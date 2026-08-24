# Spring Transactions — Interview Revision Guide

For Java/Spring Boot backend interviews (2–3 years experience).

## 1. What Is a Transaction?

A transaction groups database operations into one logical unit of work.

```text
BEGIN
  create order
  reduce stock
  save payment
COMMIT                 // all succeed

ROLLBACK               // any required operation fails
```

Transactions follow **ACID**:

- **Atomicity:** all operations succeed or all are undone.
- **Consistency:** data moves from one valid state to another.
- **Isolation:** concurrent transactions do not interfere beyond the configured level.
- **Durability:** committed data survives failures.

In Spring, place the transaction around a complete business use case, normally in the service layer.

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        inventoryRepository.reduceStock(order.getProductId());
    }
}
```

If `reduceStock()` fails with an exception that triggers rollback, the order insert is also rolled back.

## 2. How `@Transactional` Works Internally

Spring normally implements declarative transactions using **AOP proxies**.

```text
Controller -> Spring proxy -> TransactionManager -> target service method
                     |                              |
                     +-- begin                      |
                     +-- commit on success <--------+
                     +-- rollback on failure <------+
```

Conceptually, the proxy performs:

```java
TransactionStatus tx = transactionManager.getTransaction(...);
try {
    Object result = targetMethod.invoke(...);
    transactionManager.commit(tx);
    return result;
} catch (Throwable ex) {
    transactionManager.rollback(tx); // according to rollback rules
    throw ex;
}
```

Important components:

- `@Transactional` supplies transaction metadata.
- A Spring AOP proxy intercepts an eligible method call.
- `PlatformTransactionManager` coordinates begin, commit and rollback.
- For JPA, Spring commonly uses `JpaTransactionManager`.
- For JDBC, Spring commonly uses `DataSourceTransactionManager`.
- The transaction's resources are associated with the current thread.

### Proxy types

- **JDK dynamic proxy:** proxies interfaces.
- **CGLIB/class-based proxy:** creates a subclass of the target class.

Practical interview rule: call a transactional method through the Spring-managed bean/proxy. Private methods cannot be advised by normal proxy-based AOP, and final methods cannot be overridden by a class proxy.

## 3. Rollback Rules

By default, Spring rolls back for:

- `RuntimeException` and its subclasses
- `Error`

By default, it does **not** roll back for checked exceptions.

```java
@Transactional(rollbackFor = IOException.class)
public void importFile() throws IOException {
    // rolls back if IOException escapes the method
}
```

```java
@Transactional(noRollbackFor = BusinessWarningException.class)
public void process() {
    // commits even if this runtime exception escapes
}
```

### Checked vs unchecked exceptions

| Type | Examples | Compiler requires handling? | Default Spring rollback? |
|---|---|---:|---:|
| Checked | `IOException`, custom `Exception` | Yes | No |
| Unchecked | `NullPointerException`, custom `RuntimeException` | No | Yes |
| Error | `OutOfMemoryError` | No | Yes |

Common trap: if the method catches and suppresses an exception, the proxy sees normal completion and may commit.

```java
@Transactional
public void placeOrder() {
    try {
        inventoryRepository.reduceStock();
    } catch (RuntimeException ex) {
        // Suppressed: transaction may commit.
        // Prefer rethrowing, or explicitly mark rollback-only when appropriate.
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

If an inner `REQUIRED` method marks a shared transaction rollback-only and the outer method tries to commit, Spring can throw `UnexpectedRollbackException`.

## 4. Transaction Propagation

Propagation answers: **What should this method do when a transaction already exists?**

| Propagation | Existing transaction | No existing transaction | Typical use |
|---|---|---|---|
| `REQUIRED` | Join it | Create one | Default business operations |
| `REQUIRES_NEW` | Suspend it; create new | Create one | Independent audit operation |
| `SUPPORTS` | Join it | Run without one | Optional transactional read |
| `MANDATORY` | Join it | Throw exception | Must be called inside a transaction |
| `NOT_SUPPORTED` | Suspend it | Run without one | Explicit non-transactional work |
| `NEVER` | Throw exception | Run without one | Must not execute in a transaction |
| `NESTED` | Create savepoint | Create transaction | Partial rollback with savepoints |

### `REQUIRED` — default

```java
@Transactional // Propagation.REQUIRED
public void createOrder() {
    paymentService.pay(); // joins the same transaction if also REQUIRED
}
```

Both methods share one physical transaction. A rollback normally affects all their database work.

### `REQUIRES_NEW`

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAuditLog(AuditLog log) {
    auditRepository.save(log);
}
```

The outer transaction is suspended. The new transaction commits or rolls back independently. It needs another database connection while the outer one is suspended, so connection-pool sizing matters.

### `NESTED`

Uses a savepoint inside the same physical transaction. The nested part can roll back to its savepoint, but an outer rollback still removes everything. Support depends on the transaction manager and resource; it is commonly associated with JDBC savepoints and should not be assumed for every JPA setup.

### Propagation call example

```java
@Transactional
public void placeOrder() {             // T1
    orderRepository.save(...);
    auditService.record(...);          // T2 only if call crosses proxy
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void record(...) { ... }
```

Annotation settings on an internal self-call are not newly applied because the call does not pass through the proxy.

## 5. Isolation Levels

Isolation controls what one transaction may observe from concurrent transactions.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public Order loadOrder(Long id) { ... }
```

| Isolation | Dirty read | Non-repeatable read | Phantom read | Concurrency |
|---|---:|---:|---:|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible | Highest |
| `READ_COMMITTED` | Prevented | Possible | Possible | High |
| `REPEATABLE_READ` | Prevented | Prevented* | Possible* | Medium |
| `SERIALIZABLE` | Prevented | Prevented | Prevented | Lowest |

`*` Exact behavior is database-specific. MVCC databases and vendors can provide stronger behavior than the SQL-level summary suggests.

- `DEFAULT` means use the database's default isolation level.
- Higher isolation usually increases blocking, retries or serialization failures.
- Choose isolation using business invariants and actual database behavior, not only the textbook table.

## 6. Concurrency Problems

### Dirty read

T2 reads data changed by T1 before T1 commits. If T1 rolls back, T2 used data that never became permanent.

### Non-repeatable read

T1 reads the same row twice; T2 updates and commits it between those reads, so T1 gets different values.

### Phantom read

T1 repeats a range query; T2 inserts or deletes matching rows, so the result set changes.

### Lost update

Two transactions read the same value and both update it. The later write overwrites the earlier write.

```text
stock = 10
T1 reads 10        T2 reads 10
T1 writes 9        T2 writes 9
Expected 8, actual 9 -> one update was lost
```

Lost updates are commonly handled using atomic SQL, optimistic locking, or pessimistic locking. Do not assume an isolation level alone solves every lost-update pattern on every database.

```sql
UPDATE product SET stock = stock - 1
WHERE id = ? AND stock > 0;
```

## 7. Optimistic Locking and `@Version`

Optimistic locking assumes conflicts are uncommon. It does not hold a database lock for the whole user workflow; instead, it detects whether the row changed before update.

```java
@Entity
public class Product {
    @Id
    private Long id;

    private int stock;

    @Version
    private Long version;
}
```

Conceptual SQL:

```sql
UPDATE product
SET stock = ?, version = version + 1
WHERE id = ? AND version = ?;
```

If zero rows are updated, another transaction changed the row. JPA reports an optimistic-lock failure (commonly translated by Spring to `ObjectOptimisticLockingFailureException`). The application normally retries the complete operation or reports a conflict.

Use optimistic locking when:

- reads greatly outnumber conflicting writes;
- short retries are acceptable;
- you want good concurrency without long-held locks.

Do not manually change the `@Version` field. JPA manages it.

## 8. Pessimistic Locking

Pessimistic locking locks data while a transaction is using it, expecting that conflicts are likely.

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select p from Product p where p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}
```

Typically this results in SQL similar to `SELECT ... FOR UPDATE`. Keep the surrounding transaction short.

Use it when:

- contention is high;
- overselling or double-processing must be prevented immediately;
- retrying conflicts would be expensive.

Trade-offs: blocking, reduced throughput, lock timeouts and deadlocks. Always access shared resources in a consistent order when possible.

### Optimistic vs pessimistic

| Optimistic | Pessimistic |
|---|---|
| Detects conflict at write/flush | Prevents or blocks conflicting access |
| Uses version comparison | Uses database locks |
| Better for low contention | Useful for high contention |
| May require retry | May wait, time out or deadlock |

## 9. Timeout

```java
@Transactional(timeout = 5) // seconds
public void generateReport() { ... }
```

A timeout limits how long the transaction may run. When enforced and exceeded, the transaction is rolled back. Exact enforcement depends on the transaction manager, database driver and executed operations; it is not a guaranteed cancellation of arbitrary Java computation.

Use timeouts to prevent transactions and locks from being held indefinitely. Configure realistic values—too short causes avoidable failures.

## 10. `readOnly`

```java
@Transactional(readOnly = true)
public OrderView getOrder(Long id) { ... }
```

`readOnly = true` expresses intent and may enable ORM/driver/database optimizations, such as reduced dirty checking in some setups. It is generally a **hint**, not a universal security rule that guarantees writes will fail.

Use it for query-only service methods. Do not depend on it as the sole protection against accidental writes.

## 11. Self-Invocation Problem

```java
@Service
public class ReportService {

    public void generate() {
        saveAudit(); // direct call on this; proxy is bypassed
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAudit() { ... }
}
```

`generate()` calls `saveAudit()` on the same object, so the call does not go through Spring's proxy. The `REQUIRES_NEW` advice is therefore not applied.

Preferred fix: move the transactional operation to another Spring service and inject it.

```java
@Service
class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAudit() { ... }
}
```

Other options exist (self-proxy injection, `AopContext`, AspectJ weaving, `TransactionTemplate`), but separating responsibilities is usually clearer and easier to test.

## 12. Transaction Boundaries

A transaction boundary defines where a transaction starts and ends.

```text
HTTP request
  Controller
    Service.placeOrder()  <-- BEGIN
      save order
      update stock
      create outbox row
    return                <-- COMMIT
```

Good boundaries:

- surround one complete business operation;
- usually start at a public service method;
- are short and contain related database work;
- avoid waiting for user input or slow remote calls.

Why avoid network calls inside long database transactions? The transaction holds connections and possibly locks while waiting, increasing contention and failure complexity. For reliable event publication, consider the transactional outbox pattern.

Also remember:

- Returning normally usually causes commit.
- An eligible exception must escape the proxy boundary to trigger default rollback.
- A transaction is generally thread-bound; it does not automatically follow work submitted to another thread or an `@Async` method.
- Class-level `@Transactional` supplies defaults; a method-level annotation can override them.

## 13. Multiple Databases / Transaction Managers

If an application has multiple data sources, define and select the correct transaction manager.

```java
@Transactional(transactionManager = "ordersTransactionManager")
public void saveOrder() { ... }
```

```java
@Transactional(transactionManager = "billingTransactionManager")
public void saveInvoice() { ... }
```

Two independent local transaction managers do **not** make one atomic transaction across both databases. One commit can succeed while the other fails.

Options when cross-resource consistency is required:

- XA/JTA distributed transactions (strong atomicity, greater operational complexity);
- redesign around one owner/database;
- outbox, messaging and Saga patterns for eventual consistency.

## 14. Transactions in Microservices

`@Transactional` normally covers resources managed inside one service/process and one configured transaction manager. It cannot roll back a database commit already made by another microservice.

```text
Order Service DB -- local transaction
       |
       +-- message/API --> Payment Service DB -- separate local transaction
```

Common distributed consistency techniques:

- local transaction per service;
- transactional outbox to atomically save business data and an event;
- idempotent consumers so duplicate messages are safe;
- retries with backoff and dead-letter handling;
- Saga compensation for multi-step business workflows.

Avoid treating a chain of synchronous HTTP calls as one database transaction.

## 15. Saga vs `@Transactional`

| `@Transactional` | Saga |
|---|---|
| Usually one service/local resource boundary | Multiple services and local transactions |
| ACID commit/rollback | Eventual consistency |
| Technical rollback | Business compensation |
| Simple for local operations | Handles long-running distributed workflows |

Example Saga:

```text
1. Order Service: create PENDING order
2. Payment Service: charge payment
3. Inventory Service: reserve stock
4. Order Service: confirm order

If stock reservation fails:
5. Payment Service: refund payment       // compensation
6. Order Service: cancel order           // compensation
```

Compensation is a new business action, not a database rollback. It can also fail, so steps must be observable, retryable and idempotent.

Saga styles:

- **Choreography:** services react to events; less central control, but flow can become difficult to follow.
- **Orchestration:** a coordinator commands each step; clearer workflow, but the orchestrator becomes important infrastructure.

Use local `@Transactional` inside each Saga step. Saga and `@Transactional` complement each other rather than compete.

## 16. Common Interview Questions and Short Answers

### 1. What does `@Transactional` do?

It tells Spring to execute an eligible method within a transaction using a transaction manager: begin before invocation, then commit on success or roll back according to rollback rules.

### 2. Why is `@Transactional` usually placed on the service layer?

A service method represents a complete business use case and may coordinate several repository operations that must succeed or fail together.

### 3. Does Spring roll back for every exception?

No. By default it rolls back for `RuntimeException` and `Error`, but not checked exceptions. Use `rollbackFor` when required.

### 4. What happens if I catch the exception?

If it is suppressed and the method returns normally, the proxy normally attempts to commit unless the transaction was already marked rollback-only.

### 5. What is the default propagation?

`REQUIRED`: join an existing transaction or create a new one.

### 6. `REQUIRED` vs `REQUIRES_NEW`?

`REQUIRED` shares the caller's transaction. `REQUIRES_NEW` suspends it and uses an independent transaction.

### 7. `REQUIRES_NEW` vs `NESTED`?

`REQUIRES_NEW` uses a separate physical transaction. `NESTED` normally uses a savepoint within the existing physical transaction, subject to resource support.

### 8. Why does `@Transactional` not work on a self-invoked method?

The direct `this.method()` call bypasses the Spring proxy, so the transactional interceptor does not see it.

### 9. What is an isolation level?

It controls visibility and interference between concurrent transactions, balancing correctness against concurrency.

### 10. How do you prevent lost updates?

Use an atomic SQL update, optimistic locking with `@Version`, or a suitable pessimistic lock, depending on contention and business needs.

### 11. What does `@Version` do?

JPA includes the version in updates and increments it. If the expected version no longer matches, it detects a concurrent modification.

### 12. Is `readOnly = true` a guarantee that no write occurs?

No. It is primarily a transaction/optimization hint; exact behavior depends on the stack.

### 13. Can `@Transactional` cover two microservices?

No, not as a normal local Spring transaction. Each service owns its local transaction; distributed workflows use techniques such as outbox and Saga.

### 14. Can a transaction cross an `@Async` call?

Normally no. Spring transaction context is thread-bound, and the asynchronous method runs on another thread.

### 15. What is `UnexpectedRollbackException`?

It commonly occurs when an inner participant marks a shared transaction rollback-only, but outer code catches the original failure and then attempts to commit.

### 16. Optimistic vs pessimistic locking—which should you choose?

Use optimistic locking for low conflict and high concurrency. Use pessimistic locking when conflicts are frequent or immediate exclusive access is necessary.

### 17. Is `@Transactional` useful in a Saga?

Yes. Each service uses it for its own local atomic step, while the Saga coordinates consistency across services.

## 17. Rapid Revision Checklist

- `@Transactional` is normally proxy-based.
- External calls through a Spring bean are intercepted; self-invocation is not.
- Default propagation: `REQUIRED`.
- Default rollback: unchecked exceptions and `Error`.
- Checked exception: add `rollbackFor` if rollback is desired.
- Caught/suppressed exception may allow commit.
- `REQUIRES_NEW`: independent transaction; outer transaction is suspended.
- `NESTED`: savepoint semantics, only when supported.
- Isolation controls concurrent visibility; database behavior matters.
- `@Version`: optimistic conflict detection.
- Pessimistic lock: blocking and deadlock risk; keep transaction short.
- `timeout` and `readOnly` behavior depends partly on the transaction stack.
- Transactions are generally thread-bound.
- Multiple local transaction managers do not provide automatic atomicity.
- `@Transactional` is local; Saga coordinates distributed business consistency.

