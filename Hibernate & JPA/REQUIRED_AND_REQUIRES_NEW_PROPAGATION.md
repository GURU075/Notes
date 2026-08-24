# Spring Propagation: `REQUIRED` and `REQUIRES_NEW`

This guide explains the two most important Spring transaction propagation types using a `placeOrder()` business flow.

## 1. `REQUIRED`

### Meaning

`REQUIRED` means:

> Join the existing transaction. If no transaction exists, create a new one.

It is Spring's default propagation type:

```java
@Transactional
```

is the same as:

```java
@Transactional(propagation = Propagation.REQUIRED)
```

### Business example

Placing an order requires two operations:

1. Save the order.
2. Save the payment.

Both operations belong to the same business flow, so both should succeed or fail together.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    @Transactional // REQUIRED: creates T1
    public void placeOrder(Order order) {
        orderRepository.save(order);       // uses T1
        paymentService.savePayment(order); // uses the same T1
    }
}
```

```java
@Service
public class PaymentService {

    private final PaymentRepository paymentRepository;

    @Transactional(propagation = Propagation.REQUIRED)
    public void savePayment(Order order) {
        paymentRepository.save(...); // joins T1
    }
}
```

### Successful flow

```text
placeOrder()
   |
   +-- no existing transaction
   +-- create T1
   |
   +-- save order using T1
   |
   +-- call savePayment()
   +-- savePayment() finds existing T1
   +-- join T1
   +-- save payment using T1
   |
   +-- everything succeeds
   +-- commit T1
```

Final result:

```text
Order   -> saved
Payment -> saved
```

### Failure flow

Suppose saving the payment throws a `RuntimeException`:

```java
@Transactional(propagation = Propagation.REQUIRED)
public void savePayment(Order order) {
    paymentRepository.save(...);
    throw new RuntimeException("Payment failed");
}
```

Transaction flow:

```text
placeOrder()
   |
   +-- create T1
   +-- save order using T1
   |
   +-- call savePayment()
   +-- join the same T1
   +-- save payment using T1
   |
   +-- payment exception occurs
   +-- T1 is marked for rollback
   +-- rollback T1
```

Final result:

```text
Order   -> rolled back
Payment -> rolled back
```

### Why use `REQUIRED`?

Use `REQUIRED` when all operations are part of one business unit and must succeed or fail together.

Examples:

- save order and reduce stock;
- create employee and assign department;
- transfer money by debiting one account and crediting another;
- save invoice and invoice items.

### Interview answer

> `REQUIRED` uses one shared transaction for the complete business flow. It joins an existing transaction or creates one if none exists. If a participating operation fails and causes rollback, all changes in that transaction are rolled back.

---

## 2. `REQUIRES_NEW`

### Meaning

`REQUIRES_NEW` means:

> Always create a new, independent transaction.

If transaction `T1` already exists, Spring:

1. Pauses `T1`.
2. Creates a new transaction `T2`.
3. Completes `T2` independently.
4. Resumes `T1`.

### Business example: independent audit log

Suppose an audit record must remain saved even when placing the order fails.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final AuditService auditService;
    private final PaymentService paymentService;

    @Transactional // REQUIRED: creates T1
    public void placeOrder(Order order) {
        orderRepository.save(order);       // T1
        auditService.saveAuditLog(order);  // independent T2
        paymentService.makePayment(order); // T1
    }
}
```

```java
@Service
public class AuditService {

    private final AuditRepository auditRepository;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAuditLog(Order order) {
        auditRepository.save(...); // uses T2
    }
}
```

### Outer transaction fails after `T2` commits

```text
placeOrder()
   |
   +-- create T1
   +-- save order using T1
   |
   +-- call saveAuditLog()
   +-- pause T1
   +-- create T2
   +-- save audit log using T2
   +-- commit T2
   +-- resume T1
   |
   +-- make payment using T1
   +-- payment fails in T1
   +-- rollback T1
```

Final result:

```text
Order     -> rolled back because it belongs to T1
Payment   -> rolled back because it belongs to T1
Audit log -> remains saved because T2 already committed
```

An outer rollback cannot undo a successfully committed independent `T2`.

### Inner `T2` fails, but `T1` continues

The outer method must handle the exception if the business decision is to continue:

```java
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order); // T1

    try {
        auditService.saveAuditLog(order); // T2
    } catch (RuntimeException ex) {
        // Log the problem and allow T1 to continue.
    }

    paymentService.makePayment(order); // T1
}
```

Transaction flow:

```text
placeOrder()
   |
   +-- create T1
   +-- save order using T1
   |
   +-- call saveAuditLog()
   +-- pause T1
   +-- create T2
   +-- audit exception occurs
   +-- rollback T2 only
   +-- exception is handled by placeOrder()
   +-- resume T1
   |
   +-- save payment using T1
   +-- commit T1
```

Final result:

```text
Order     -> saved through T1
Payment   -> saved through T1
Audit log -> rolled back through T2
```

If the `T2` exception is **not caught**, it escapes from `placeOrder()`. A runtime exception will then normally cause `T1` to roll back as well.

### Why use `REQUIRES_NEW`?

Use it when an operation needs an independent commit or rollback.

Common business cases:

- save audit logs independently;
- record failure details even when the main operation rolls back;
- store an independent operation history;
- process each batch item in its own transaction.

Do not use it for operations that must remain atomic with the main business flow.

### Interview answer

> `REQUIRES_NEW` suspends the current transaction and creates an independent transaction. The inner transaction commits or rolls back separately, so an outer rollback cannot undo an inner transaction that has already committed.

---

## 3. `REQUIRED` vs `REQUIRES_NEW`

| Point | `REQUIRED` | `REQUIRES_NEW` |
|---|---|---|
| Existing transaction | Joins it | Suspends it |
| Transaction used by inner method | Same `T1` | New `T2` |
| Inner and outer work independent? | No | Yes |
| Outer transaction fails | All `T1` work rolls back | Committed `T2` remains |
| Inner transaction fails | Shared `T1` is normally rolled back | Only `T2` rolls back if exception is handled |
| Common use | Main business flow | Audit/failure/history record |

### Easy memory rule

```text
REQUIRED
   -> One business flow
   -> One shared transaction
   -> All succeed or all roll back

REQUIRES_NEW
   -> Independent business operation
   -> Separate transaction
   -> Separate commit or rollback
```

## 4. Important Proxy Rule

For `REQUIRES_NEW` to create `T2`, the call should normally go through another Spring-managed bean.

This does not work as expected with normal proxy-based transactions:

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder() {
        saveAuditLog(); // self-invocation: proxy is bypassed
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAuditLog() { ... }
}
```

Preferred structure:

```text
OrderService.placeOrder()
   |
   +-- call AuditService proxy
          |
          +-- apply REQUIRES_NEW
          +-- create T2
```

Move `saveAuditLog()` into a separate `AuditService` Spring bean so the call passes through Spring's transaction proxy.

## 5. Final Revision

```text
REQUIRED:
T1 -> save order -> save payment -> failure -> rollback everything in T1

REQUIRES_NEW:
T1 -> save order
   -> pause T1
   -> T2 saves and commits audit
   -> resume T1
   -> payment failure
   -> rollback T1, but T2 remains committed
```
