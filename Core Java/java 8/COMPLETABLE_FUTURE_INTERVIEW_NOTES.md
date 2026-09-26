# Java 8 CompletableFuture — Practical and Interview Notes

> Target: Java developers with 3–4 years of experience  
> Goal: understand asynchronous execution, chaining, combining, error handling, executors, and common interview questions

---

## Table of contents

1. [What is CompletableFuture?](#1-what-is-completablefuture)
2. [Why do we need it?](#2-why-do-we-need-it)
3. [Complete runnable example](#3-complete-runnable-example)
4. [`supplyAsync()` and `runAsync()`](#4-supplyasync-and-runasync)
5. [`thenApply()`](#5-thenapply)
6. [`thenCompose()`](#6-thencompose)
7. [`thenCombine()`](#7-thencombine)
8. [`allOf()`](#8-allof)
9. [Error handling](#9-error-handling)
10. [`join()` vs `get()`](#10-join-vs-get)
11. [Synchronous vs Async continuation methods](#11-synchronous-vs-async-continuation-methods)
12. [Executors and production usage](#12-executors-and-production-usage)
13. [Common mistakes](#13-common-mistakes)
14. [Interview questions](#14-interview-questions)
15. [Quick revision sheet](#15-quick-revision-sheet)

---

## 1. What is CompletableFuture?

`CompletableFuture<T>` represents a computation that may produce a value of type `T` in the future.

```java
CompletableFuture<Customer>
```

Think of it as a box that will contain a `Customer` later:

```text
Now:    CompletableFuture<Customer> → not completed
Later:  CompletableFuture<Customer> → Customer object available
```

It was introduced in Java 8 and supports:

- Starting tasks asynchronously.
- Chaining dependent operations.
- Running independent operations concurrently.
- Combining multiple results.
- Handling success and failure.
- Building a pipeline without blocking after every step.

### Interview definition

> `CompletableFuture` is a Java 8 API for composing asynchronous and non-blocking-style computation stages. It represents a result that may become available later and provides methods for transforming, chaining, combining, and recovering that result.

---

## 2. Why do we need it?

Imagine an Order API needs:

- Customer details from Customer Service.
- Delivery details from Delivery Service.

### Sequential execution

```java
Customer customer = customerService.getCustomer(101); // 500 ms
Delivery delivery = deliveryService.getDelivery(101); // 500 ms
```

Approximate total time:

```text
500 ms + 500 ms = 1000 ms
```

### Concurrent execution

```java
CompletableFuture<Customer> customerFuture = getCustomerAsync(101);
CompletableFuture<Delivery> deliveryFuture = getDeliveryAsync(101);
```

Both operations can run at the same time:

```text
               ┌── Customer call: 500 ms ──┐
Start together ─┤                           ├── Combine
               └── Delivery call: 500 ms ──┘
```

Approximate total time can be close to 500 ms, assuming:

- The calls are independent.
- Enough threads/connections are available.
- The downstream systems can handle the concurrency.

`CompletableFuture` does not make a single slow operation faster. It helps when useful work can overlap.

---

## 3. Complete runnable example

Save this as `CompletableFutureDemo.java`.

```java
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CompletableFutureDemo {

    private static final ExecutorService EXECUTOR =
            Executors.newFixedThreadPool(4);

    static class Customer {
        private final int id;
        private final String name;

        Customer(int id, String name) {
            this.id = id;
            this.name = name;
        }

        int getId() {
            return id;
        }

        String getName() {
            return name;
        }
    }

    static class Delivery {
        private final String estimate;

        Delivery(String estimate) {
            this.estimate = estimate;
        }

        String getEstimate() {
            return estimate;
        }
    }

    static CompletableFuture<Customer> getCustomerAsync(int id) {
        return CompletableFuture.supplyAsync(() -> {
            pause(500);
            System.out.println("Fetched customer on "
                    + Thread.currentThread().getName());
            return new Customer(id, "Anita");
        }, EXECUTOR);
    }

    static CompletableFuture<List<String>> getOrdersAsync(int customerId) {
        return CompletableFuture.supplyAsync(() -> {
            pause(500);
            System.out.println("Fetched orders for customer " + customerId);
            return Arrays.asList("Order-101", "Order-102");
        }, EXECUTOR);
    }

    static CompletableFuture<Delivery> getDeliveryAsync(int customerId) {
        return CompletableFuture.supplyAsync(() -> {
            pause(500);
            System.out.println("Fetched delivery for customer " + customerId);
            return new Delivery("Tomorrow");
        }, EXECUTOR);
    }

    static CompletableFuture<Customer> getFailingCustomerAsync() {
        return CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Customer service unavailable");
        }, EXECUTOR);
    }

    static void pause(long milliseconds) {
        try {
            Thread.sleep(milliseconds);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }

    public static void main(String[] args) {
        try {
            // 1. Start an asynchronous task that produces a value.
            CompletableFuture<Customer> customerFuture =
                    getCustomerAsync(101);

            System.out.println("Main thread can do other work");

            Customer customer = customerFuture.join();
            System.out.println("Customer: " + customer.getName());

            // 2. Transform a result.
            CompletableFuture<String> greetingFuture =
                    getCustomerAsync(101)
                            .thenApply(c -> "Hello, " + c.getName());

            System.out.println(greetingFuture.join());

            // 3. Start a dependent asynchronous operation.
            CompletableFuture<List<String>> ordersFuture =
                    getCustomerAsync(101)
                            .thenCompose(c -> getOrdersAsync(c.getId()));

            System.out.println("Orders: " + ordersFuture.join());

            // 4. Combine two independent asynchronous operations.
            CompletableFuture<Customer> customerForPage =
                    getCustomerAsync(101);

            CompletableFuture<Delivery> deliveryForPage =
                    getDeliveryAsync(101);

            CompletableFuture<String> pageFuture =
                    customerForPage.thenCombine(
                            deliveryForPage,
                            (c, d) -> c.getName()
                                    + " - delivery: "
                                    + d.getEstimate()
                    );

            System.out.println("Page: " + pageFuture.join());

            // 5. Wait for multiple independent operations.
            CompletableFuture<Customer> customerForAll =
                    getCustomerAsync(101);

            CompletableFuture<List<String>> ordersForAll =
                    getOrdersAsync(101);

            CompletableFuture<Delivery> deliveryForAll =
                    getDeliveryAsync(101);

            CompletableFuture<Void> all =
                    CompletableFuture.allOf(
                            customerForAll,
                            ordersForAll,
                            deliveryForAll
                    );

            all.join();

            System.out.println("All finished:");
            System.out.println(customerForAll.join().getName());
            System.out.println(ordersForAll.join());
            System.out.println(deliveryForAll.join().getEstimate());

            // 6. Recover from failure.
            CompletableFuture<String> recovered =
                    getFailingCustomerAsync()
                            .thenApply(Customer::getName)
                            .exceptionally(error -> "Guest");

            System.out.println("Recovered name: " + recovered.join());

            // 7. Inspect success or failure.
            CompletableFuture<String> handled =
                    getFailingCustomerAsync()
                            .handle((c, error) -> {
                                if (error != null) {
                                    return "Customer unavailable";
                                }
                                return c.getName();
                            });

            System.out.println("Handled result: " + handled.join());
        } finally {
            EXECUTOR.shutdown();
        }
    }
}
```

Compile and run:

```bash
javac CompletableFutureDemo.java
java CompletableFutureDemo
```

The output order can change because asynchronous operations can finish in different orders.

---

## 4. `supplyAsync()` and `runAsync()`

### `supplyAsync()` — task returns a value

```java
CompletableFuture<Customer> future =
        CompletableFuture.supplyAsync(
                () -> customerService.fetch(101),
                executor
        );
```

The lambda is a `Supplier<Customer>`:

```text
() → Customer
```

The result type is:

```text
CompletableFuture<Customer>
```

### `runAsync()` — task returns no value

```java
CompletableFuture<Void> future =
        CompletableFuture.runAsync(
                () -> auditService.writeLog("Order created"),
                executor
        );
```

The lambda is a `Runnable`:

```text
() → no result
```

Therefore, the future type is `CompletableFuture<Void>`.

### Comparison

| Method | Lambda returns | Future type |
|---|---|---|
| `supplyAsync()` | A value | `CompletableFuture<T>` |
| `runAsync()` | Nothing | `CompletableFuture<Void>` |

### Thread flow

```text
Main thread:   submit task ── continue other work ── join if result is needed
Worker thread:             └── execute operation ── complete future
```

### Immediate `join()` problem

```java
Customer customer = CompletableFuture
        .supplyAsync(() -> customerService.fetch(101), executor)
        .join();
```

This starts asynchronous work and immediately blocks the caller. There is no overlap with useful caller work.

It is usually better to build the pipeline first and wait only at the system boundary if waiting is necessary.

---

## 5. `thenApply()`

Use `thenApply()` to transform a completed value into another normal value.

```java
CompletableFuture<String> nameFuture =
        getCustomerAsync(101)
                .thenApply(customer -> customer.getName());
```

Transformation:

```text
Customer → String
```

Future transformation:

```text
CompletableFuture<Customer>
             ↓ thenApply
CompletableFuture<String>
```

### Multiple transformations

```java
CompletableFuture<String> result =
        getCustomerAsync(101)
                .thenApply(Customer::getName)
                .thenApply(String::toUpperCase)
                .thenApply(name -> "Hello, " + name);
```

Flow:

```text
Customer → name → uppercase name → greeting
```

### When to use it

- Extract a field from an object.
- Convert an object into a DTO.
- Format the completed result.
- Perform a synchronous calculation using the result.

### Do not use it for a function returning another future

This creates a nested future:

```java
CompletableFuture<CompletableFuture<List<String>>> nested =
        getCustomerAsync(101)
                .thenApply(customer ->
                        getOrdersAsync(customer.getId()));
```

Use `thenCompose()` for that situation.

---

## 6. `thenCompose()`

Use `thenCompose()` when the next asynchronous operation depends on the previous result.

Requirement:

```text
Fetch customer first
       ↓
Use customer ID to fetch orders
```

Code:

```java
CompletableFuture<List<String>> ordersFuture =
        getCustomerAsync(101)
                .thenCompose(customer ->
                        getOrdersAsync(customer.getId()));
```

`getOrdersAsync()` already returns:

```text
CompletableFuture<List<String>>
```

`thenCompose()` flattens the result:

```text
Without thenCompose:
CompletableFuture<CompletableFuture<List<String>>>

With thenCompose:
CompletableFuture<List<String>>
```

### Relation to `map` and `flatMap`

```text
Optional.map       ≈ CompletableFuture.thenApply
Optional.flatMap   ≈ CompletableFuture.thenCompose
Stream.map         ≈ CompletableFuture.thenApply
Stream.flatMap     ≈ CompletableFuture.thenCompose
```

Conceptually:

```text
thenApply:
T → R

thenCompose:
T → CompletableFuture<R>
```

### Practical example: user followed by orders

```java
CompletableFuture<List<String>> result =
        findUserByEmailAsync("anita@example.com")
                .thenCompose(user ->
                        findOrdersByUserIdAsync(user.getId()));
```

The order call cannot start until the user call provides the ID. Therefore these operations are dependent and are not started together.

### Interview rule

> If the lambda returns a normal value, use `thenApply`. If the lambda returns a `CompletableFuture`, use `thenCompose` to avoid a nested future.

---

## 7. `thenCombine()`

Use `thenCombine()` when two asynchronous operations are independent but you need both results.

```java
CompletableFuture<Customer> customerFuture =
        getCustomerAsync(101);

CompletableFuture<Delivery> deliveryFuture =
        getDeliveryAsync(101);

CompletableFuture<String> pageFuture =
        customerFuture.thenCombine(
                deliveryFuture,
                (customer, delivery) ->
                        customer.getName()
                                + " - "
                                + delivery.getEstimate()
        );
```

Flow:

```text
               ┌── Fetch customer ──┐
Start together ─┤                    ├── Combine into page result
               └── Fetch delivery ──┘
```

### Dependent vs independent operations

```text
Customer ID needed to fetch orders
→ dependent
→ thenCompose

Customer and delivery can be fetched separately
→ independent
→ thenCombine
```

### Important mistake

This is sequential even though both methods are asynchronous:

```java
CompletableFuture<Result> result =
        getCustomerAsync(101)
                .thenCompose(customer ->
                        getDeliveryAsync(101)
                                .thenApply(delivery ->
                                        new Result(customer, delivery)));
```

Delivery is started only after the customer completes. If the calls are independent, create both futures first and use `thenCombine()`.

---

## 8. `allOf()`

Use `allOf()` to wait for several futures.

```java
CompletableFuture<Customer> customerFuture =
        getCustomerAsync(101);

CompletableFuture<List<String>> ordersFuture =
        getOrdersAsync(101);

CompletableFuture<Delivery> deliveryFuture =
        getDeliveryAsync(101);

CompletableFuture<Void> all =
        CompletableFuture.allOf(
                customerFuture,
                ordersFuture,
                deliveryFuture
        );

all.join();
```

### Important point

`allOf()` returns:

```text
CompletableFuture<Void>
```

It does not automatically return a list containing all results.

After it completes, read the original futures:

```java
Customer customer = customerFuture.join();
List<String> orders = ordersFuture.join();
Delivery delivery = deliveryFuture.join();
```

These individual `join()` calls should return immediately after `all.join()` completes successfully.

### Converting a list of futures into a future of list

```java
static <T> CompletableFuture<List<T>> sequence(
        List<CompletableFuture<T>> futures) {

    CompletableFuture<Void> all = CompletableFuture.allOf(
            futures.toArray(new CompletableFuture[0]));

    return all.thenApply(ignored ->
            futures.stream()
                    .map(CompletableFuture::join)
                    .collect(java.util.stream.Collectors.toList()));
}
```

Usage:

```java
List<CompletableFuture<String>> futures = Arrays.asList(
        CompletableFuture.supplyAsync(() -> "A", EXECUTOR),
        CompletableFuture.supplyAsync(() -> "B", EXECUTOR),
        CompletableFuture.supplyAsync(() -> "C", EXECUTOR)
);

CompletableFuture<List<String>> results = sequence(futures);
System.out.println(results.join()); // [A, B, C]
```

### Failure behavior

If any input future completes exceptionally, the `allOf()` future also completes exceptionally. `allOf()` does not automatically recover failures or cancel the other tasks.

---

## 9. Error handling

### `exceptionally()` — recover only from failure

```java
CompletableFuture<String> result =
        getFailingCustomerAsync()
                .thenApply(Customer::getName)
                .exceptionally(error -> "Guest");
```

Flow:

```text
Success → use customer name
Failure → return "Guest"
```

`exceptionally()` receives the error and must return a fallback value of the future's result type.

### `handle()` — process success or failure

```java
CompletableFuture<String> result =
        getCustomerAsync(101)
                .handle((customer, error) -> {
                    if (error != null) {
                        return "Customer unavailable";
                    }
                    return customer.getName();
                });
```

| Outcome | `customer` | `error` |
|---|---|---|
| Success | Customer value | `null` |
| Failure | Usually `null` | Exception/wrapper |

### `whenComplete()` — observe without changing the result

```java
CompletableFuture<Customer> future =
        getCustomerAsync(101)
                .whenComplete((customer, error) -> {
                    if (error != null) {
                        System.out.println("Call failed: " + error.getMessage());
                    } else {
                        System.out.println("Call succeeded: " + customer.getName());
                    }
                });
```

`whenComplete()` is typically used for logging, metrics, or cleanup. It normally preserves the original result or failure rather than converting it into a fallback.

### Comparison

| Method | Runs on success? | Runs on failure? | Can transform/recover? |
|---|---:|---:|---:|
| `exceptionally` | No | Yes | Yes, returns fallback |
| `handle` | Yes | Yes | Yes |
| `whenComplete` | Yes | Yes | Mainly observes |

### Business fallback warning

This may be acceptable for recommendations:

```java
.exceptionally(error -> Collections.emptyList())
```

This is dangerous for payment:

```java
.exceptionally(error -> new PaymentResponse("SUCCESS")) // Wrong
```

A fallback must preserve correct business meaning. A payment timeout may mean the outcome is unknown, not successful or failed.

### Exceptions in `join()`

An asynchronous failure is usually wrapped in `CompletionException`:

```java
try {
    future.join();
} catch (java.util.concurrent.CompletionException e) {
    Throwable originalCause = e.getCause();
    System.out.println(originalCause.getMessage());
}
```

---

## 10. `join()` vs `get()`

Both methods return the value and wait if the future is incomplete.

### `join()`

```java
Customer customer = future.join();
```

- Throws unchecked `CompletionException` on failure.
- Does not require checked-exception handling.

### `get()`

```java
try {
    Customer customer = future.get();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} catch (java.util.concurrent.ExecutionException e) {
    Throwable cause = e.getCause();
}
```

- Throws checked `InterruptedException`.
- Throws checked `ExecutionException` for task failure.
- Also has a timeout overload:

```java
Customer customer = future.get(2, java.util.concurrent.TimeUnit.SECONDS);
```

### Comparison

| Method | Waits? | Failure type | Checked exception handling? |
|---|---:|---|---:|
| `join()` | Yes | `CompletionException` | No |
| `get()` | Yes | `ExecutionException` | Yes |
| `get(timeout, unit)` | Yes, bounded | `ExecutionException`/`TimeoutException` | Yes |

Neither method is non-blocking. Both are waiting points.

---

## 11. Synchronous vs Async continuation methods

Many continuation methods have an `Async` version:

```java
thenApply(...)
thenApplyAsync(...)

thenCompose(...)
thenComposeAsync(...)

thenAccept(...)
thenAcceptAsync(...)
```

### Without `Async`

```java
future.thenApply(value -> transform(value));
```

The continuation may run on the thread that completes the previous stage, or immediately on the caller thread if the stage is already complete.

It does not automatically create a new thread.

### With `Async`

```java
future.thenApplyAsync(
        value -> transform(value),
        executor
);
```

The continuation is submitted to an executor.

### Do not use `Async` everywhere automatically

Each asynchronous boundary adds:

- Executor scheduling.
- Queueing.
- Context switching.
- Operational complexity.

Small transformations such as extracting a field usually do not need another executor submission.

Use an async variant when moving expensive work to a suitable executor is intentional.

---

## 12. Executors and production usage

### Default executor

Without an explicit executor:

```java
CompletableFuture.supplyAsync(() -> service.fetch());
```

the task normally uses `ForkJoinPool.commonPool()`.

The common pool is shared by unrelated application work. Blocking HTTP, database, or file calls can occupy its worker threads and affect other tasks.

### Dedicated executor

```java
ExecutorService customerExecutor =
        Executors.newFixedThreadPool(10);

CompletableFuture<Customer> customer =
        CompletableFuture.supplyAsync(
                () -> customerService.fetch(101),
                customerExecutor
        );
```

In a real Spring Boot application, define a managed executor bean rather than repeatedly creating executors per request.

```java
@Configuration
public class AsyncConfiguration {

    @Bean(name = "customerExecutor", destroyMethod = "shutdown")
    public ExecutorService customerExecutor() {
        return Executors.newFixedThreadPool(10);
    }
}
```

Inject it:

```java
@Service
public class CustomerFacade {

    private final Executor customerExecutor;
    private final CustomerService customerService;

    public CustomerFacade(
            @Qualifier("customerExecutor") Executor customerExecutor,
            CustomerService customerService) {
        this.customerExecutor = customerExecutor;
        this.customerService = customerService;
    }

    public CompletableFuture<Customer> getCustomer(int id) {
        return CompletableFuture.supplyAsync(
                () -> customerService.fetch(id),
                customerExecutor
        );
    }
}
```

### Executor sizing considerations

Do not choose pool size randomly. Consider:

- Whether work is CPU-bound or blocking I/O.
- Downstream connection-pool capacity.
- Downstream concurrency limits.
- Expected request rate and latency.
- Queue capacity.
- Rejection policy.
- Memory and timeout budget.

Creating 100 threads does not help if the HTTP connection pool has only 10 connections.

### Timeouts and cancellation

Java 8 does not provide the later convenience methods `orTimeout()` and `completeOnTimeout()`. A Java 8 application commonly combines `CompletableFuture` with:

- HTTP client connection/read timeouts.
- `get(timeout, unit)` at a blocking boundary.
- A scheduled timeout completion strategy.
- Resilience libraries such as a time limiter.

Important:

> A caller timing out does not guarantee that the underlying remote call or side effect stopped.

For payments and writes, use idempotency because the operation might complete after the caller stops waiting.

### Thread-local context

Thread-local values do not automatically move to worker threads:

- Logging MDC/correlation ID.
- Security context.
- Request context.
- Transaction context.

Do not assume a Spring `@Transactional` transaction automatically continues inside a `supplyAsync()` task running on another thread.

Propagate the required context intentionally, or redesign the transaction boundary.

---

## 13. Common mistakes

### 1. Calling `join()` immediately

```java
CompletableFuture.supplyAsync(task, executor).join();
```

This often removes the concurrency benefit.

### 2. Using `thenApply()` for an asynchronous function

```java
future.thenApply(value -> callAsync(value));
```

This creates a nested future. Use `thenCompose()`.

### 3. Using `thenCompose()` for independent calls

This starts the second operation after the first. If calls are independent, start both first and use `thenCombine()` or `allOf()`.

### 4. Blocking the common ForkJoinPool

Blocking I/O can occupy common-pool threads. Use a suitable bounded executor.

### 5. Creating a new executor for every HTTP request

This leaks resources and creates too many threads. Use application-managed shared executors.

### 6. No timeout

A slow downstream call can keep a task and its resources occupied indefinitely.

### 7. Assuming cancellation stops remote work

Cancellation is not guaranteed to stop an underlying network operation or business side effect.

### 8. Returning a misleading fallback

Do not convert every failure into a fake successful response.

### 9. Ignoring exception wrappers

Failures can appear inside `CompletionException` or `ExecutionException`. Inspect the cause.

### 10. Forgetting context propagation

MDC, security, request context, and thread-bound transactions do not automatically propagate to another thread.

### 11. Excessive async boundaries

Using every `*Async` method can add queueing and context-switch overhead without benefit.

### 12. Unbounded fan-out

Starting thousands of calls simultaneously can overload your executor and downstream service. Bound concurrency.

---

## 14. Interview questions

### 1. What is `CompletableFuture`?

It represents a result that may become available later and provides a fluent API for asynchronous execution, transformation, composition, combination, and error handling.

### 2. What is the difference between `supplyAsync()` and `runAsync()`?

`supplyAsync()` produces a value and returns `CompletableFuture<T>`. `runAsync()` produces no value and returns `CompletableFuture<Void>`.

### 3. What is the difference between `thenApply()` and `thenCompose()`?

```text
thenApply:   T → R
thenCompose: T → CompletableFuture<R>
```

`thenCompose()` flattens the nested future and is used for dependent asynchronous calls.

### 4. `thenCompose()` vs `thenCombine()`?

- `thenCompose()`: the next call depends on the first result.
- `thenCombine()`: two independent calls can start concurrently and their results are combined.

### 5. What does `allOf()` return?

It returns `CompletableFuture<Void>`. Results must normally be read from the original futures after completion.

### 6. `exceptionally()` vs `handle()`?

- `exceptionally()` handles only failure and returns a fallback.
- `handle()` runs for both success and failure and can transform either outcome.

### 7. What is `whenComplete()` used for?

It observes success or failure, commonly for logging or metrics, while normally preserving the original outcome.

### 8. `join()` vs `get()`?

- Both wait for completion.
- `join()` throws unchecked `CompletionException`.
- `get()` throws checked `InterruptedException` and `ExecutionException`.
- `get()` also has a timeout overload.

### 9. Which thread runs `thenApply()`?

It may run on the thread completing the previous stage, or immediately on the caller if the previous stage is already complete. `thenApplyAsync()` submits it to an executor.

### 10. Which executor does `supplyAsync()` use by default?

Normally `ForkJoinPool.commonPool()` when no executor is supplied.

### 11. Why can the common pool be risky for blocking operations?

It is shared. Blocking its workers can reduce capacity for unrelated tasks and cause starvation or high latency.

### 12. Does an HTTP timeout stop a `CompletableFuture` task?

Not necessarily. The caller may stop waiting while the task or remote operation continues.

### 13. Does `@Transactional` propagate into `supplyAsync()`?

Not automatically. Traditional Spring transactions are generally thread-bound, and `supplyAsync()` executes on another thread.

### 14. How do you execute two independent service calls efficiently?

Start both futures before waiting, then combine them with `thenCombine()` or coordinate several with `allOf()`.

### 15. What happens if one future supplied to `allOf()` fails?

The combined future completes exceptionally. Other tasks are not automatically cancelled.

### 16. Can `CompletableFuture` guarantee non-blocking I/O?

No. It can move a blocking operation to another thread, but the operation still blocks that worker. True non-blocking I/O depends on the underlying client and programming model.

### Scenario question

You must fetch a user, use the user ID to fetch orders, and independently fetch current promotions.

One possible structure:

```java
CompletableFuture<List<Order>> ordersFuture =
        getUserAsync(email)
                .thenCompose(user -> getOrdersAsync(user.getId()));

CompletableFuture<List<Promotion>> promotionsFuture =
        getPromotionsAsync();

CompletableFuture<HomePage> pageFuture =
        ordersFuture.thenCombine(
                promotionsFuture,
                HomePage::new
        );
```

Explanation:

- User and orders are dependent, so use `thenCompose()`.
- Promotions are independent, so start them separately.
- Combine orders and promotions using `thenCombine()`.

---

## 15. Quick revision sheet

### Method map

```text
supplyAsync     → start async task that returns a value
runAsync        → start async task with no result

thenApply       → T → R
thenCompose     → T → CompletableFuture<R>
thenCombine     → combine two independent future results
allOf           → wait for many futures; returns Future<Void>

exceptionally   → recover from failure
handle          → transform success or failure
whenComplete    → observe success or failure

join            → wait; unchecked CompletionException
get             → wait; checked exceptions
```

### Main comparison

| Requirement | Use |
|---|---|
| Start a task returning a value | `supplyAsync()` |
| Start a task returning nothing | `runAsync()` |
| Transform completed value | `thenApply()` |
| Start dependent asynchronous call | `thenCompose()` |
| Combine two independent calls | `thenCombine()` |
| Wait for many calls | `allOf()` |
| Failure-only fallback | `exceptionally()` |
| Handle success or failure | `handle()` |
| Log success or failure | `whenComplete()` |

### Memory rule

```text
thenApply   = map
thenCompose = flatMap
thenCombine = independent futures meet
```

### Production checklist

- Are the operations actually independent?
- Is the underlying work blocking or non-blocking?
- Am I using the correct bounded executor?
- Does executor size match downstream connection capacity?
- Is there a timeout/deadline?
- Is the fallback valid for the business?
- Can a timed-out side effect still complete?
- Is idempotency required?
- Do MDC, security, or transaction contexts need propagation?
- Am I waiting only at the required boundary?

### Interview-ready answer

> `CompletableFuture` represents a result that becomes available later. I use `supplyAsync` to start value-producing work, `thenApply` to transform a completed value, `thenCompose` for dependent asynchronous calls, and `thenCombine` for independent calls that can run concurrently. I use `allOf` to coordinate several tasks and `exceptionally`, `handle`, or `whenComplete` for failure handling and observation. For production blocking I/O, I use a bounded dedicated executor, configure downstream timeouts, avoid immediate blocking, and account for context propagation and idempotent side effects.

### Self-test

1. Why does `thenApply()` sometimes create a nested future?
2. When should you choose `thenCompose()`?
3. How is `thenCombine()` different from `thenCompose()`?
4. Why does `allOf()` return `Void`?
5. What is the difference between `exceptionally()` and `handle()`?
6. Why can the common pool be risky for database and HTTP calls?
7. Does `join()` make code non-blocking?
8. Does a caller timeout guarantee that asynchronous work stopped?
9. Does a Spring transaction automatically cross an async thread boundary?
10. Why should executors be bounded and managed by the application?

