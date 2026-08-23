# Resilience4j Study Guide: Beginner to Interview-Ready

> Target: Java/Spring Boot backend developers with 2–3 years of experience  
> Scope: Circuit Breaker, Retry, Rate Limiter, Bulkhead, and Time Limiter  
> Goal: understand **why**, **when**, **how**, and **what can go wrong**

---

## Table of contents

1. [How to use this guide](#1-how-to-use-this-guide)
2. [The big picture](#2-the-big-picture)
3. [Spring Boot setup](#3-spring-boot-setup)
4. [Circuit Breaker](#4-circuit-breaker)
5. [Retry](#5-retry)
6. [Rate Limiter](#6-rate-limiter)
7. [Bulkhead](#7-bulkhead)
8. [Time Limiter](#8-time-limiter)
9. [Pattern comparison](#9-pattern-comparison)
10. [Combining patterns safely](#10-combining-patterns-safely)
11. [Fallbacks](#11-fallbacks)
12. [Monitoring and testing](#12-monitoring-and-testing)
13. [Common mistakes](#13-common-mistakes)
14. [Scenario-based interview questions](#14-scenario-based-interview-questions)
15. [Quick quizzes](#15-quick-quizzes)
16. [Memory maps](#16-memory-maps)
17. [Final one-page revision sheet](#17-final-one-page-revision-sheet)

---

## 1. How to use this guide

### First pass — understand

- Read the analogy and request flow for each pattern.
- Explain the pattern aloud in your own words.
- Focus on the problem it solves, not the annotation name.

### Second pass — implement

- Type the Spring Boot examples instead of copying them.
- Change one configuration value and predict the behavior.
- Simulate failures, delays, and traffic bursts.

### Third pass — interview practice

- Answer each scenario before expanding the suggested answer.
- Complete the quizzes without looking back.
- Revise only the final one-page sheet before an interview.

### Active-recall card used in every chapter

For each pattern, answer:

1. What problem does it solve?
2. What does it measure or limit?
3. What happens when its rule is violated?
4. Which configuration values matter most?
5. What is its most common misuse?

---

## 2. The big picture

### What is resilience?

Resilience is the ability of a service to continue behaving predictably when a dependency is slow, unavailable, overloaded, or returning errors.

```text
Client → Order Service → Payment Service → Bank API
```

If the Bank API becomes slow, the problem can spread:

```text
Slow Bank API
  → Payment threads wait
  → Payment connection pool fills
  → Order requests wait
  → Order Service becomes unhealthy
  → Entire user flow fails
```

Resilience4j supplies small, composable fault-tolerance patterns. It does **not** repair a dependency. It protects your service and controls how it reacts.

### Five patterns in one view

| Pattern | Main question | Protects against | Typical rejection/failure |
|---|---|---|---|
| Circuit Breaker | Is this dependency failing too often? | Repeated calls to an unhealthy dependency | `CallNotPermittedException` |
| Retry | Was this failure temporary? | Transient failures | Original exception after attempts finish |
| Rate Limiter | Are too many calls arriving over time? | Excess request rate | `RequestNotPermitted` |
| Bulkhead | Are too many calls executing at once? | Resource/thread exhaustion | `BulkheadFullException` |
| Time Limiter | Is this operation taking too long? | Unbounded waiting | `TimeoutException` |

### Interview-ready summary

> Circuit Breaker stops calls when failure rate is high. Retry repeats safe operations after transient failures. Rate Limiter controls calls per time period. Bulkhead limits concurrent work to isolate resources. Time Limiter bounds how long an asynchronous call may take.

### Important design rule

Apply resilience at the **outbound dependency boundary**:

```text
Controller → Business Service → protected client call → Remote Service
                                ^
                         Resilience4j here
```

Avoid wrapping an entire business flow when only one remote call is unreliable. Each dependency may need different limits.

---

## 3. Spring Boot setup

### Maven dependencies — Spring Boot 3

Use a Resilience4j version compatible with your Java and Spring Boot versions. The example below uses the current 2.x line for Java 17/Spring Boot 3; verify your organization's supported version before upgrading. Keep it in dependency management or a Maven property rather than scattering the version.

```xml
<properties>
    <java.version>17</java.version>
    <resilience4j.version>2.4.0</resilience4j.version>
</properties>

<dependencies>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
        <version>${resilience4j.version}</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

For WebFlux/Reactor types, also add `resilience4j-reactor`.

### A simple client used in examples

```java
@Service
public class PaymentClient {

    private final RestClient restClient;

    public PaymentClient(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("http://payment-service").build();
    }

    public PaymentResponse charge(PaymentRequest request) {
        return restClient.post()
                .uri("/payments")
                .body(request)
                .retrieve()
                .body(PaymentResponse.class);
    }
}
```

> `RestClient` is only an example HTTP client. The resilience concepts also apply to `WebClient`, OpenFeign, database calls, message brokers, and SDK calls.

### Configuration structure

```yaml
resilience4j:
  circuitbreaker:
    configs:       # reusable named/default configurations
      default: { }
    instances:     # actual instances referenced by annotations
      paymentService: { }
```

The annotation name and configuration instance name must match:

```java
@CircuitBreaker(name = "paymentService")
```

---

## 4. Circuit Breaker

### 4.1 What problem does it solve?

When a dependency is repeatedly failing, continuing to call it wastes threads, connections, CPU, and user time. A Circuit Breaker temporarily blocks calls so the dependency and caller can recover.

### 4.2 Real-life analogy

An electrical circuit breaker cuts power when it detects a dangerous condition. It later allows a small test to check whether normal operation is safe.

### 4.3 States

```text
                 failure/slow-call rate reaches threshold
       ┌──────────────────────────────────────────────┐
       │                                              ▼
   CLOSED ─────────────────────────────────────────> OPEN
     ▲                                                  │
     │                                                  │ wait duration passes
     │ enough trial calls succeed                       ▼
     └──────────────────────────────────────────── HALF_OPEN
                                                        │
                                                        │ trial calls fail
                                                        └──────> OPEN
```

- **CLOSED**: calls are allowed and outcomes are recorded.
- **OPEN**: calls are rejected immediately; the remote service is not called.
- **HALF_OPEN**: a limited number of trial calls are allowed.
- Special administrative states also exist, but CLOSED/OPEN/HALF_OPEN are the main interview focus.

### 4.4 Request flow

```text
Request
  → Circuit CLOSED?
      → Yes: call Payment Service
          → record success/failure/slow call
          → recalculate rates when enough calls exist
      → No, OPEN: reject immediately or use fallback
      → HALF_OPEN: allow only configured trial calls
```

### 4.5 Spring Boot annotation example

```java
@Service
public class PaymentService {

    private final PaymentClient paymentClient;

    public PaymentService(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse pay(PaymentRequest request) {
        return paymentClient.charge(request);
    }

    private PaymentResponse paymentFallback(
            PaymentRequest request,
            Throwable error) {
        return PaymentResponse.pending("Payment temporarily unavailable");
    }
}
```

### 4.6 Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        slowCallRateThreshold: 50
        slowCallDurationThreshold: 2s
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.example.payment.InvalidPaymentRequestException
```

### 4.7 Key properties

| Property | Meaning | Example interpretation |
|---|---|---|
| `slidingWindowType` | `COUNT_BASED` or `TIME_BASED` measurement window | Last 10 calls vs calls in last 10 seconds |
| `slidingWindowSize` | Size of measurement window | Store/aggregate 10 calls or 10 seconds |
| `minimumNumberOfCalls` | Required sample size before rate is evaluated | Do not open based on only 1–2 calls |
| `failureRateThreshold` | Failure percentage that opens circuit | Open at 50% or more failures |
| `slowCallDurationThreshold` | Duration after which a call counts as slow | A call taking at least 2 seconds is slow |
| `slowCallRateThreshold` | Slow-call percentage that opens circuit | Open if at least 50% are slow |
| `waitDurationInOpenState` | How long OPEN lasts before trials | Wait 10 seconds |
| `permittedNumberOfCallsInHalfOpenState` | Number of recovery trial calls | Test with 3 calls |
| `recordExceptions` | Exceptions counted as failures | Count I/O and timeout problems |
| `ignoreExceptions` | Exceptions not counted as failures | Do not blame dependency for bad input |

### 4.8 Worked example

Configuration:

```text
slidingWindowSize = 10
minimumNumberOfCalls = 5
failureRateThreshold = 50%
```

Outcomes after five calls:

```text
Success, Failure, Failure, Success, Failure
Failures = 3/5 = 60%
```

The failure rate exceeds 50%, so the circuit moves from CLOSED to OPEN.

### 4.9 What Circuit Breaker does not do

- It does not limit traffic while CLOSED.
- It does not retry a failed call.
- It does not automatically make a synchronous call time out.
- It does not share state across service instances by default; each application instance has its own circuit state.
- It does not replace correct HTTP/client connection and read timeouts.

### 4.10 Common mistakes

- Setting `minimumNumberOfCalls` too low and opening on a tiny sample.
- Counting business validation exceptions as dependency failures.
- Using one breaker for several unrelated dependencies.
- Assuming OPEN means the dependency is checked continuously; calls are normally blocked until a transition permits trials.
- Returning a fake “success” from fallback and hiding a real failure.
- Confusing the sliding window with permitted traffic. It measures calls; it does not cap them.

### 4.11 Check yourself

**Question:** With a window of 10, minimum calls of 5, and threshold of 50%, can the circuit open after the first 2 calls both fail?

<details>
<summary>Answer</summary>

No. The configured minimum of 5 calls has not been reached.

</details>

**Interview line:**

> A Circuit Breaker observes recent outcomes. When the configured failure or slow-call rate is reached after a minimum sample, it opens and fails fast. After a wait period, it permits limited half-open trial calls to decide whether to close or reopen.

---

## 5. Retry

### 5.1 What problem does it solve?

Some failures are temporary: a brief network interruption, short-lived `503`, or optimistic-lock conflict. Retry gives the operation another chance.

### 5.2 Real-life analogy

You call someone, the signal drops, and you call again after waiting. You do not call 100 times immediately, and you do not retry if they clearly say the number is invalid.

### 5.3 Request flow

```text
Attempt 1
  → success: return result
  → retryable failure: wait
       → Attempt 2
           → success: return result
           → retryable failure: wait
                → Attempt 3
                    → success or final failure/fallback
  → non-retryable failure: fail immediately
```

### 5.4 The idempotency rule

Retry can repeat side effects. Before retrying a write operation, ensure it is idempotent.

```text
Order Service → Payment Service charges ₹500 → response is lost
Order Service retries → possible second charge
```

Safer design:

```http
POST /payments
Idempotency-Key: payment-order-123
```

The Payment Service—the owner of the payment operation—stores or recognizes the key and returns the prior result instead of charging twice.

### 5.5 Spring Boot example

```java
@Service
public class InventoryService {

    private final InventoryClient inventoryClient;

    public InventoryService(InventoryClient inventoryClient) {
        this.inventoryClient = inventoryClient;
    }

    @Retry(name = "inventoryService", fallbackMethod = "stockFallback")
    public StockResponse getStock(String productId) {
        return inventoryClient.fetchStock(productId);
    }

    private StockResponse stockFallback(String productId, Throwable error) {
        return StockResponse.unknown(productId);
    }
}
```

### 5.6 Configuration

```yaml
resilience4j:
  retry:
    instances:
      inventoryService:
        maxAttempts: 3
        waitDuration: 200ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        exponentialMaxWaitDuration: 2s
        retryExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.example.inventory.ProductNotFoundException
```

### 5.7 Key properties

| Property | Meaning |
|---|---|
| `maxAttempts` | Total attempts, including the first call |
| `waitDuration` | Delay between attempts |
| `enableExponentialBackoff` | Increase delay after successive failures |
| `exponentialBackoffMultiplier` | Delay growth factor |
| `exponentialMaxWaitDuration` | Upper bound for exponential delay |
| `enableRandomizedWait` | Add randomness to reduce synchronized retries |
| `retryExceptions` | Exceptions that should trigger another attempt |
| `ignoreExceptions` | Exceptions that should fail without retry |

> `maxAttempts: 3` means one original attempt plus at most two additional attempts—not three retries after the original.

### 5.8 Backoff and jitter

Without backoff:

```text
0ms → attempt → 100ms → attempt → 100ms → attempt
```

With exponential backoff:

```text
0ms → attempt → 200ms → attempt → 400ms → attempt
```

With jitter/randomness, different service instances do not all retry at exactly the same moment. This reduces a **retry storm** or **thundering herd**.

### 5.9 When to retry

Good candidates:

- Connection reset or temporary I/O failure.
- `503 Service Unavailable` when the dependency signals a transient problem.
- `429 Too Many Requests` only when respecting the server's retry guidance and backing off.
- Optimistic-lock conflict when repeating the business operation is safe.

Usually do not retry:

- Validation errors and most other `4xx` responses.
- Authentication/authorization failure without refreshing credentials.
- Deterministic business rejection such as insufficient balance.
- Non-idempotent write with no deduplication/idempotency protection.
- Calls already consuming most of the request's latency budget.

### 5.10 Common mistakes

- Retrying every exception, including permanent failures.
- Using many immediate retries and multiplying dependency load.
- Forgetting that retries increase latency.
- Retrying at multiple layers: HTTP client, service, gateway, and job runner.
- Retrying a payment without an idempotency key.
- Assuming three attempts means three retries.

### 5.11 Quick calculation

Suppose each call can take 1 second and configuration allows 3 total attempts with 200 ms then 400 ms backoff:

```text
Worst approximate time = 1s + 0.2s + 1s + 0.4s + 1s = 3.6s
```

This must fit inside the end-to-end latency budget.

### 5.12 Check yourself

**Question:** Should an invalid card number be retried?

<details>
<summary>Answer</summary>

No. It is a deterministic client/business error. Another identical attempt will not fix it.

</details>

**Interview line:**

> Retry is for transient failures and must be bounded. I retry only selected exceptions, use backoff and jitter, respect the end-to-end time budget, and require idempotency for operations with side effects.

---

## 6. Rate Limiter

### 6.1 What problem does it solve?

A Rate Limiter controls how many calls may pass during a time period. It protects capacity, enforces quotas, and prevents one caller from overwhelming a service or dependency.

### 6.2 Real-life analogy

A venue admits only 100 people every 10 minutes. Even if the entrance is empty at this instant, admissions are controlled by the time-based quota.

### 6.3 Request flow

```text
Request
  → permission available in current refresh period?
      → yes: consume permission and call dependency
      → no: wait up to timeoutDuration
          → permission arrives: call dependency
          → still unavailable: reject
```

### 6.4 Spring Boot example

```java
@Service
public class SmsService {

    private final SmsClient smsClient;

    public SmsService(SmsClient smsClient) {
        this.smsClient = smsClient;
    }

    @RateLimiter(name = "smsProvider", fallbackMethod = "rateLimitFallback")
    public SmsResponse send(SmsRequest request) {
        return smsClient.send(request);
    }

    private SmsResponse rateLimitFallback(
            SmsRequest request,
            RequestNotPermitted error) {
        return SmsResponse.queued("SMS will be sent later");
    }
}
```

### 6.5 Configuration

```yaml
resilience4j:
  ratelimiter:
    instances:
      smsProvider:
        limitForPeriod: 20
        limitRefreshPeriod: 1s
        timeoutDuration: 0
```

### 6.6 Key properties

| Property | Meaning | Example |
|---|---|---|
| `limitForPeriod` | Permissions created for each refresh cycle | 20 calls |
| `limitRefreshPeriod` | How often permissions refresh | Every 1 second |
| `timeoutDuration` | Maximum time a caller waits for permission | `0` means fail fast |

### 6.7 Important behavior

- `20 per second` is a time-based allowance, not a concurrency limit.
- Twenty quick calls may be admitted together; use Bulkhead to cap simultaneous execution.
- A local Rate Limiter applies independently inside each application instance.

```text
3 pods × 20 calls/second each = up to roughly 60 calls/second overall
```

For a strict cluster-wide/business quota, use a distributed limiter or enforce the quota at a shared gateway/provider.

### 6.8 Where to use it

- Outbound third-party API quotas.
- Per-user or per-plan request quotas.
- Expensive endpoints.
- Smoothing load toward a limited dependency.

### 6.9 Common mistakes

- Confusing rate with concurrency.
- Forgetting that each pod has its own local limiter.
- Setting a long `timeoutDuration` and making request threads wait.
- Applying one global limit when different customers require different quotas.
- Retrying rejected calls immediately, defeating the rate limit.

### 6.10 Check yourself

**Question:** Can a limiter configured for 100 calls per second still allow 100 concurrent calls?

<details>
<summary>Answer</summary>

Yes. Rate Limiter controls admissions over time, not how many admitted calls are executing simultaneously. Pair it with Bulkhead when concurrency also matters.

</details>

**Interview line:**

> Rate Limiter controls throughput over a time window. It grants permissions based on a refresh period and either waits briefly or rejects when none are available. A local limiter is per service instance, not automatically distributed.

---

## 7. Bulkhead

### 7.1 What problem does it solve?

A Bulkhead limits concurrent executions so one slow dependency cannot consume every thread or connection and bring down unrelated operations.

### 7.2 Real-life analogy

A ship is divided into watertight compartments. Damage in one compartment does not flood the entire ship.

### 7.3 Semaphore vs thread-pool Bulkhead

| Type | How it works | Best fit | Important trade-off |
|---|---|---|---|
| Semaphore Bulkhead | Allows only N concurrent callers; caller runs on its current thread | Simple synchronous calls | Does not create thread isolation |
| Thread-pool Bulkhead | Executes work in a separate bounded executor and queue | Asynchronous/thread-isolated work | Context propagation and queue sizing need care |

### 7.4 Semaphore request flow

```text
Request
  → concurrency permit available?
      → yes: acquire permit → execute → release permit
      → no: wait up to maxWaitDuration
          → permit arrives: execute
          → otherwise: reject
```

### 7.5 Semaphore Bulkhead example

```java
@Service
public class ReportService {

    private final ReportClient reportClient;

    public ReportService(ReportClient reportClient) {
        this.reportClient = reportClient;
    }

    @Bulkhead(
        name = "reportService",
        type = Bulkhead.Type.SEMAPHORE,
        fallbackMethod = "reportFallback"
    )
    public Report generate(String customerId) {
        return reportClient.generate(customerId);
    }

    private Report reportFallback(String customerId, Throwable error) {
        return Report.pending(customerId);
    }
}
```

```yaml
resilience4j:
  bulkhead:
    instances:
      reportService:
        maxConcurrentCalls: 10
        maxWaitDuration: 0
```

### 7.6 Thread-pool Bulkhead example

```java
@Bulkhead(
    name = "invoiceService",
    type = Bulkhead.Type.THREADPOOL,
    fallbackMethod = "invoiceFallback"
)
public CompletableFuture<Invoice> generateInvoice(String orderId) {
    // The annotation's thread-pool Bulkhead runs this method on its bounded executor.
    return CompletableFuture.completedFuture(invoiceClient.generate(orderId));
}

private CompletableFuture<Invoice> invoiceFallback(
        String orderId,
        Throwable error) {
    return CompletableFuture.completedFuture(Invoice.pending(orderId));
}
```

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      invoiceService:
        coreThreadPoolSize: 4
        maxThreadPoolSize: 8
        queueCapacity: 20
        keepAliveDuration: 20ms
```

> With annotation-based async code, verify which executor actually runs the work. Avoid accidentally adding an unnecessary second executor through `supplyAsync` defaults; choose and test the execution model deliberately.

### 7.7 Key properties

Semaphore:

| Property | Meaning |
|---|---|
| `maxConcurrentCalls` | Maximum simultaneous protected executions |
| `maxWaitDuration` | How long to wait for a concurrency permit |

Thread pool:

| Property | Meaning |
|---|---|
| `coreThreadPoolSize` | Normal worker count |
| `maxThreadPoolSize` | Maximum worker count |
| `queueCapacity` | Number of waiting tasks |
| `keepAliveDuration` | Idle time before extra threads are removed |

### 7.8 Sizing intuition

Little's Law gives a rough starting point:

```text
concurrency ≈ throughput × average latency
```

If a dependency receives 20 calls/second and averages 0.2 seconds:

```text
expected concurrency ≈ 20 × 0.2 = 4
```

Do not turn this rough estimate directly into production configuration. Also consider latency spikes, connection-pool capacity, downstream limits, memory, and load-test results.

### 7.9 Why a large queue is dangerous

```text
Large queue → requests wait invisibly → latency grows → stale work executes later
```

A bounded small queue or fail-fast rejection often gives more predictable behavior.

### 7.10 Common mistakes

- Confusing concurrency with calls per second.
- Setting `maxConcurrentCalls` higher than the HTTP/database connection pool.
- Creating a huge queue that hides overload and increases latency.
- Forgetting thread-local/security/logging context propagation with a thread-pool Bulkhead.
- Using a thread pool automatically for every call, adding context switches without need.
- Providing no handling for `BulkheadFullException`.

### 7.11 Check yourself

**Question:** A dependency permits 50 calls/second but only 5 simultaneous connections. Which patterns might you combine?

<details>
<summary>Answer</summary>

Rate Limiter for at most 50 admissions per second and Bulkhead for at most 5 concurrent calls.

</details>

**Interview line:**

> Bulkhead isolates resources by bounding concurrency. A semaphore Bulkhead limits callers on their current threads, while a thread-pool Bulkhead adds a dedicated bounded executor and queue. It prevents one slow dependency from exhausting the whole service.

---

## 8. Time Limiter

### 8.1 What problem does it solve?

A Time Limiter stops waiting for an asynchronous operation after a configured duration. It protects the caller's latency budget.

### 8.2 Real-life analogy

You wait five minutes for a taxi. After that, you cancel and use another option instead of waiting indefinitely.

### 8.3 Request flow

```text
Start async operation and timer
  → operation completes before timeout: return result
  → timeout expires first: complete exceptionally with TimeoutException
      → optionally request cancellation of running Future
      → execute fallback if configured
```

### 8.4 Spring Boot example

```java
@Service
public class RecommendationService {

    private final RecommendationClient client;
    private final Executor recommendationExecutor;

    public RecommendationService(
            RecommendationClient client,
            @Qualifier("recommendationExecutor") Executor recommendationExecutor) {
        this.client = client;
        this.recommendationExecutor = recommendationExecutor;
    }

    @TimeLimiter(name = "recommendationService", fallbackMethod = "defaultRecommendations")
    public CompletableFuture<List<String>> recommendations(String userId) {
        return CompletableFuture.supplyAsync(
                () -> client.fetch(userId),
                recommendationExecutor);
    }

    private CompletableFuture<List<String>> defaultRecommendations(
            String userId,
            Throwable error) {
        return CompletableFuture.completedFuture(List.of("popular-item"));
    }
}
```

```yaml
resilience4j:
  timelimiter:
    instances:
      recommendationService:
        timeoutDuration: 800ms
        cancelRunningFuture: true
```

### 8.5 Key properties

| Property | Meaning |
|---|---|
| `timeoutDuration` | Maximum allowed duration for the asynchronous result |
| `cancelRunningFuture` | Whether to request cancellation when time expires |

### 8.6 Critical distinction: timeout vs cancellation

A caller receiving a timeout does not guarantee that the underlying remote request or side effect stopped.

```text
Caller times out at 800ms
Remote payment may still complete at 1200ms
```

Therefore:

- Configure connection/read/response timeouts in the HTTP client too.
- Treat cancellation as cooperative; the underlying task may ignore interruption.
- Use idempotency for side effects even when a Time Limiter exists.
- Verify behavior for Reactor/Netty or other non-blocking clients separately.

### 8.7 Choosing a timeout

Start from the end-to-end budget and work backward:

```text
API budget                         2,000ms
- gateway/network allowance         200ms
- application work                  300ms
- fallback/response allowance       200ms
------------------------------------------------
maximum dependency budget         1,300ms
```

The caller's timeout should also align with downstream timeouts. A downstream timeout longer than the entire request budget is usually wasteful.

### 8.8 Common mistakes

- Believing timeout definitely cancels the remote operation.
- Applying `@TimeLimiter` to an ordinary synchronous return type and expecting it to become asynchronous.
- Using only Time Limiter while leaving HTTP connection/read timeouts unbounded.
- Setting a timeout below normal latency and causing false failures.
- Making retry attempts whose combined duration exceeds the parent request deadline.

### 8.9 Check yourself

**Question:** If the caller timed out, is it safe to assume a payment did not happen?

<details>
<summary>Answer</summary>

No. The caller stopped waiting, but the underlying operation may have completed. Query by an idempotency/payment key before starting another charge.

</details>

**Interview line:**

> Time Limiter bounds how long an asynchronous result may take. A timeout limits caller waiting, but it does not guarantee that the underlying remote side effect stopped, so client timeouts and idempotency are still necessary.

---

## 9. Pattern comparison

### 9.1 Core differences

| Question | Circuit Breaker | Retry | Rate Limiter | Bulkhead | Time Limiter |
|---|---|---|---|---|---|
| Main dimension | Failure/slow-call percentage | Attempt count | Calls per period | Concurrent executions | Duration |
| Primary goal | Fail fast when dependency is unhealthy | Recover from transient failure | Control throughput/quota | Isolate bounded resources | Bound waiting time |
| Calls dependency? | Not while OPEN | Yes, multiple times | Only with permission | Only with capacity | Starts it, then stops waiting |
| Uses history? | Yes, sliding window | Current execution attempts | Current refresh-cycle permissions | Current active/queued work | Current execution timer |
| Typical fallback trigger | Circuit open or recorded call failure | Attempts exhausted | No permission | No capacity | Timeout |
| Biggest risk | Bad thresholds hide/overreact to failure | Load amplification/duplicate side effects | Per-instance limit mistaken for global | Bad pool/queue sizing | Work continues after caller timeout |

### 9.2 Commonly confused pairs

#### Circuit Breaker vs Retry

```text
Retry: “Try again because the failure may be temporary.”
Circuit Breaker: “Stop trying because recent evidence says the dependency is unhealthy.”
```

#### Rate Limiter vs Bulkhead

```text
Rate Limiter: 100 calls per second.
Bulkhead: only 10 calls executing at the same time.
```

Example:

- 100 calls can finish in 100 ms: low concurrency, high rate.
- 10 calls can each run for 30 seconds: low rate, high sustained concurrency.

#### Time Limiter vs Circuit Breaker slow-call threshold

```text
Time Limiter: fails this individual operation when its deadline passes.
Circuit Breaker slow-call rule: observes many slow outcomes and may block future calls.
```

#### Client timeout vs Time Limiter

- HTTP client timeout controls network/client behavior.
- Time Limiter controls the decorated asynchronous result.
- Use aligned layers; do not assume one perfectly cancels the other.

---

## 10. Combining patterns safely

### 10.1 Example business scenario

An Order Service calls a Payment Service that:

- Occasionally returns transient `503` responses.
- Must answer within 1.5 seconds.
- Supports only 20 concurrent requests.
- Allows 100 requests per second.
- Can become unhealthy for several minutes.

Possible protection:

```text
Order request
  → Rate Limiter: permission within provider quota?
  → Bulkhead: concurrency capacity available?
  → Time Limiter/client timeout: enough time remaining?
  → Retry: repeat only selected transient failure, with idempotency key
  → Circuit Breaker: stop calls when recent health is poor
  → Payment Service
```

The exact order is a design decision because it changes metrics and behavior.

### 10.2 Annotation aspect order

With Spring annotations, multiple resilience aspects are nested. The documented default is conceptually:

```text
Retry(
  CircuitBreaker(
    RateLimiter(
      TimeLimiter(
        Bulkhead(
          businessFunction
        )
      )
    )
  )
)
```

Do not memorize this without understanding consequences:

- An outer Retry can invoke the inner chain multiple times.
- Depending on placement, each retry attempt may be recorded separately by the Circuit Breaker.
- A Rate Limiter inside Retry may require permission for every attempt; outside it may permit the logical request once.
- A Time Limiter around each attempt differs from one deadline around all attempts.

When order must be unambiguous, prefer functional decoration or explicitly configure/test aspect order.

### 10.3 Functional composition example

```java
Supplier<PaymentResponse> remoteCall =
        () -> paymentClient.charge(request);

Supplier<PaymentResponse> protectedCall =
        Decorators.ofSupplier(remoteCall)
                .withBulkhead(bulkhead)
                .withRateLimiter(rateLimiter)
                .withCircuitBreaker(circuitBreaker)
                .withRetry(retry)
                .decorate();

return protectedCall.get();
```

Read and test the actual nesting used by your version/API. The wrapper order determines which component observes rejections, retries, and failures.

### 10.4 A practical design checklist

Before combining patterns, decide:

1. Is the operation idempotent?
2. Which failures are transient?
3. What is the total latency/deadline budget?
4. Is the quota per instance, user, tenant, or entire cluster?
5. What is the dependency's safe concurrency?
6. Should rejected/timeout outcomes count as Circuit Breaker failures?
7. Is fallback semantically honest?
8. Which layer owns each retry and timeout?
9. What metrics and alerts prove the behavior in production?

### 10.5 Suggested starting policy for a read-only dependency

This is an example, not a universal production default:

```text
- Strict HTTP connect/response timeout
- At most 2–3 total attempts
- Backoff plus jitter
- Circuit Breaker with a meaningful minimum sample
- Small dependency-specific Bulkhead
- Rate Limiter only when a quota/capacity policy exists
- Cached/default fallback only if the business accepts stale/partial data
```

---

## 11. Fallbacks

### 11.1 What is a fallback?

A fallback is an alternate response or action when the protected call cannot produce its normal result.

Good fallbacks:

- Return cached product recommendations with a “stale” marker.
- Queue a notification for later delivery.
- Return partial, clearly labeled data.
- Return a domain-specific “pending” state.
- Fail with a controlled error and correct HTTP status.

Dangerous fallbacks:

- Return an empty list that looks like a real “no data” result.
- Claim a payment succeeded when its status is unknown.
- Catch every exception and hide programming bugs.
- Call another equally unhealthy service and create a failure chain.

### 11.2 Fallback signature rule

The fallback normally needs:

- The same input parameters as the protected method.
- One additional exception parameter at the end.
- A compatible return type.

```java
public PaymentResponse pay(PaymentRequest request) { ... }

private PaymentResponse paymentFallback(
        PaymentRequest request,
        Throwable error) { ... }
```

For `CompletableFuture<T>`, fallback should return `CompletableFuture<T>`.

### 11.3 Better than “always return success”

```java
private PaymentResponse paymentFallback(
        PaymentRequest request,
        CallNotPermittedException error) {
    return PaymentResponse.pending("Dependency circuit is open");
}

private PaymentResponse paymentFallback(
        PaymentRequest request,
        Throwable error) {
    throw new PaymentUnavailableException("Payment status unavailable", error);
}
```

Specific fallbacks can preserve meaning for known failure types. Keep error handling observable through logs and metrics.

---

## 12. Monitoring and testing

### 12.1 Actuator configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  endpoint:
    health:
      show-details: always

resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        registerHealthIndicator: true
        eventConsumerBufferSize: 20
```

Secure operational endpoints appropriately in real environments; do not expose everything publicly.

### 12.2 Metrics worth watching

- Successful, failed, slow, not-permitted, and ignored calls.
- Circuit Breaker state transitions.
- Retry attempt/success/failure counts.
- Rate Limiter available permissions and rejected calls.
- Bulkhead available concurrent calls, queue depth, and rejections.
- Time Limiter successes, failures, and timeouts.
- Dependency latency percentiles and HTTP status distribution.
- Fallback usage rate.

### 12.3 Testing strategy

#### Circuit Breaker

- Stub dependency failures.
- Make at least `minimumNumberOfCalls` calls.
- Assert state becomes OPEN.
- Assert later calls fail fast and do not hit the stub.
- Advance time/wait and verify HALF_OPEN recovery behavior.

#### Retry

- Fail twice and succeed on the third call.
- Verify exactly three total dependency calls.
- Verify a non-retryable exception produces one call.
- Verify a retried write uses the same idempotency key.

#### Rate Limiter

- Exhaust permissions in a refresh period.
- Verify excess calls reject or wait as configured.
- Verify permissions refresh.

#### Bulkhead

- Block N concurrent calls.
- Send one more call.
- Verify rejection/wait behavior and that other endpoints remain responsive.

#### Time Limiter

- Make the async dependency exceed `timeoutDuration`.
- Verify `TimeoutException` or fallback.
- Verify whether underlying work actually stops.

### 12.4 Test behavior, not only annotations

An annotation being present proves nothing. Integration tests should verify:

```text
number of remote calls
state transitions
elapsed time
exception/fallback result
idempotency key reuse
metrics/events
```

---

## 13. Common mistakes

### 13.1 Spring AOP mistakes

#### Self-invocation

```java
public void checkout() {
    pay(); // same-object call may bypass the Spring proxy
}

@Retry(name = "paymentService")
public void pay() { ... }
```

Spring proxy-based annotations are normally applied when another bean calls through the proxy. Move the protected method to a separate Spring bean or use programmatic decoration.

#### Private annotated method

Do not expect proxy-based interception on a private method. Put the annotation on an externally invoked Spring bean method.

#### Wrong fallback signature

If parameter or return types do not match, fallback resolution fails at runtime.

#### Name mismatch

```java
@Retry(name = "inventory")
```

will not use an instance configured only as `inventoryService`.

### 13.2 Design mistakes

- **Retry storm:** many callers retry an overloaded dependency together.
- **Retry multiplication:** three layers each retry three times, potentially producing 27 calls.
- **Timeout mismatch:** outer request expires while inner work and retries continue.
- **False success:** fallback hides an unknown or failed business result.
- **Global breaker misuse:** unrelated tenants/dependencies affect one shared circuit.
- **Local quota assumption:** 10 requests/second on each of 20 pods is not 10 globally.
- **Oversized Bulkhead:** limit exceeds downstream capacity.
- **No observability:** fallback quietly becomes the normal path.
- **Wrong exception classification:** validation errors open a dependency circuit.
- **No idempotency:** retry duplicates a charge/order/message.

### 13.3 Configuration mistakes

- Durations with missing or misunderstood units.
- Tiny Circuit Breaker sample sizes.
- Very long waits for Rate Limiter/Bulkhead permissions.
- Huge thread-pool queues.
- Changing several policies together without load/failure testing.
- Copying production thresholds from another service with different traffic and latency.

---

## 14. Scenario-based interview questions

### Scenario 1 — Payment response lost

**Question:** Payment Service charged the card, but the response was lost. Should Order Service retry?

<details>
<summary>Suggested answer</summary>

Only with payment-level idempotency. Order Service should reuse the same idempotency/payment key. Payment Service must return the stored result for that key instead of charging again. A timeout cannot prove that the charge failed.

</details>

### Scenario 2 — Dependency fails 60% of the time

**Question:** Which pattern prevents continuously wasting resources?

<details>
<summary>Suggested answer</summary>

Circuit Breaker. Configure a meaningful minimum sample and failure-rate threshold. When it opens, calls fail fast; after a wait, half-open trial calls test recovery. Retry alone may increase load.

</details>

### Scenario 3 — Third-party quota

**Question:** A provider allows 1,000 calls/minute across the whole company, but your service has five pods. Is a local Resilience4j Rate Limiter sufficient?

<details>
<summary>Suggested answer</summary>

Not for a strict global quota. Each pod maintains local state, so independent 1,000/minute limits could permit roughly 5,000/minute. Divide capacity conservatively or use a shared/distributed/gateway limiter, with headroom and provider response handling.

</details>

### Scenario 4 — Slow report service consumes all threads

**Question:** Circuit Breaker threshold has not yet been reached, but unrelated APIs are becoming slow. What is missing?

<details>
<summary>Suggested answer</summary>

A dependency-specific Bulkhead and strict timeouts. The Bulkhead caps concurrent report calls so they cannot consume all caller resources. Timeouts bound how long those resources remain occupied.

</details>

### Scenario 5 — 100 requests/second vs 10 concurrent requests

**Question:** Are these equivalent limits?

<details>
<summary>Suggested answer</summary>

No. Calls per second is a Rate Limiter concern; active simultaneous calls is a Bulkhead concern. The relationship depends on call latency.

</details>

### Scenario 6 — Circuit opens on bad requests

**Question:** Product lookup receives many invalid product IDs and its circuit opens. What is wrong?

<details>
<summary>Suggested answer</summary>

The exception classification is wrong. Expected client/business errors such as invalid IDs or `404` may need to be ignored by the Circuit Breaker because they do not prove the dependency is unhealthy.

</details>

### Scenario 7 — Three retries at gateway and service

**Question:** Why is this risky?

<details>
<summary>Suggested answer</summary>

Retries multiply across layers, amplify load, increase latency, and can duplicate side effects. Assign retry ownership to one appropriate layer and keep a shared deadline/attempt budget.

</details>

### Scenario 8 — Fallback returns empty list

**Question:** Is an empty recommendation list a good fallback?

<details>
<summary>Suggested answer</summary>

Only if consumers can distinguish “no recommendations exist” from “recommendation service unavailable.” Otherwise it silently changes business meaning. A stale marker, partial-status field, or controlled error is safer.

</details>

### Scenario 9 — Time Limiter fired, task still running

**Question:** Is that a Resilience4j bug?

<details>
<summary>Suggested answer</summary>

Not necessarily. The caller's future can time out while the underlying operation ignores interruption or cannot be cancelled. Configure the HTTP client itself, use cooperative cancellation where supported, and keep side effects idempotent.

</details>

### Scenario 10 — How would you choose thresholds?

<details>
<summary>Suggested answer</summary>

Start from the dependency SLO, normal/error latency distributions, traffic volume, downstream capacity, and the caller's end-to-end budget. Choose a statistically meaningful Circuit Breaker sample, safe concurrency, bounded retries, and timeouts above normal tail latency but below the overall deadline. Validate with load and fault-injection tests, then tune using production metrics.

</details>

### Scenario 11 — Where should policies live?

**Question:** API Gateway or business service?

<details>
<summary>Suggested answer</summary>

Both may have different responsibilities. A gateway can enforce edge traffic quotas and gateway timeouts. The calling business service should protect each outbound dependency because it knows operation semantics, idempotency, acceptable fallback, and downstream capacity. The service owning a side effect must enforce idempotency for that operation.

</details>

### Scenario 12 — Retry plus Circuit Breaker ordering

**Question:** Why does order matter?

<details>
<summary>Suggested answer</summary>

If the Circuit Breaker is inside Retry, it can observe each attempt. If it is outside, it may observe one final logical outcome after retries. This changes call counts, failure rates, and when the circuit opens. The desired semantics must be explicit and verified with tests/metrics.

</details>

---

## 15. Quick quizzes

### Quiz A — choose the pattern

1. Stop calling a dependency that has recently failed 70% of calls.
2. Allow only 200 calls per minute.
3. Permit only 8 simultaneous report generations.
4. Give a temporary network error two more chances.
5. Stop waiting after 900 ms.

<details>
<summary>Answers</summary>

1. Circuit Breaker
2. Rate Limiter
3. Bulkhead
4. Retry
5. Time Limiter

</details>

### Quiz B — true or false

1. An OPEN Circuit Breaker still calls the remote service to check every request.
2. `maxAttempts: 3` means at most three total calls.
3. Rate Limiter and Bulkhead solve the same problem.
4. A timeout guarantees a remote payment was cancelled.
5. A local Rate Limiter is automatically shared across all pods.
6. Retry is safe for every `POST` request.
7. A slow-call rate can open a Circuit Breaker even when calls eventually succeed.
8. A fallback must preserve business meaning.

<details>
<summary>Answers</summary>

1. False
2. True
3. False
4. False
5. False
6. False
7. True
8. True

</details>

### Quiz C — configuration reading

```yaml
resilience4j:
  circuitbreaker:
    instances:
      catalog:
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 15s
        permittedNumberOfCallsInHalfOpenState: 4
```

Answer without looking back:

1. Can it open after four failed calls?
2. At ten calls, how many failures meet a 50% threshold?
3. How long does it remain OPEN before recovery trials are allowed?
4. How many trial calls are permitted in HALF_OPEN?

<details>
<summary>Answers</summary>

1. No; the minimum sample is 10 calls.
2. Five failures out of ten meet 50%.
3. 15 seconds.
4. Four.

</details>

### Quiz D — spot the risks

```yaml
resilience4j:
  retry:
    instances:
      payment:
        maxAttempts: 10
        waitDuration: 0
        retryExceptions:
          - java.lang.Exception
```

<details>
<summary>Answer</summary>

- Ten immediate attempts can overload the dependency.
- Retrying every exception includes permanent/business/programming failures.
- Latency and load multiply.
- Payment may be duplicated without idempotency.
- There is no backoff or jitter.
- The total deadline is not considered.

</details>

### Quiz E — 60-second verbal drill

Explain all five without using their names first:

```text
1. Stops future calls based on recent unhealthy outcomes.
2. Repeats selected transient failures.
3. Controls admissions per period.
4. Caps simultaneous work.
5. Bounds waiting duration.
```

Now name each pattern and give one production example.

---

## 16. Memory maps

### 16.1 The five-word map

```text
Circuit Breaker → HEALTH
Retry          → ATTEMPTS
Rate Limiter   → RATE
Bulkhead       → CONCURRENCY
Time Limiter   → TIME
```

### 16.2 The “H-A-R-C-T” recall sequence

```text
H = Health history       → Circuit Breaker
A = Attempts             → Retry
R = Rate over time       → Rate Limiter
C = Concurrent capacity  → Bulkhead
T = Time budget          → Time Limiter
```

Say: **“Health, Attempts, Rate, Concurrency, Time.”**

### 16.3 Failure-control map

```text
                       Remote call problem
                              │
              ┌───────────────┼───────────────┐
              │               │               │
           Failing          Too busy        Too slow
              │               │               │
      ┌───────┴───────┐   ┌───┴────┐          │
      │               │   │        │          │
 temporary       repeatedly rate  concurrent  deadline
      │               │   │        │          │
    Retry         Circuit  Rate   Bulkhead    Time
                  Breaker  Limiter             Limiter
```

### 16.4 Configuration memory map

```text
Circuit Breaker
  → window + minimum calls + failure/slow threshold
  → open wait + half-open trials

Retry
  → attempts + wait/backoff/jitter
  → retry vs ignore exceptions

Rate Limiter
  → permissions + refresh period + permission wait

Bulkhead
  → max concurrency + wait
  OR pool size + queue

Time Limiter
  → timeout + cancellation request
```

### 16.5 Safe-operation memory map

```text
Before retrying a write:
  WHO owns the side effect?
  WHAT is the idempotency key?
  WHERE is the previous result stored?
  HOW long is the total deadline?
```

---

## 17. Final one-page revision sheet

> Read this section in the final 10 minutes before an interview.

### Resilience4j in one sentence

Resilience4j is a lightweight Java fault-tolerance library that applies composable policies around calls so a service reacts predictably to dependency failure, slowness, and overload.

### The five patterns

| Pattern | Remember | Main settings | One warning |
|---|---|---|---|
| Circuit Breaker | Stop calls based on recent health | window, minimum calls, failure/slow rate, open wait, half-open calls | It does not limit traffic while CLOSED |
| Retry | Repeat transient failure | attempts, wait, backoff/jitter, exception list | Requires bounded load and idempotency |
| Rate Limiter | Calls per time period | limit, refresh period, permission timeout | Local instance is not a cluster-wide quota |
| Bulkhead | Concurrent capacity | concurrent calls/wait or pool/queue | Large queues create hidden latency |
| Time Limiter | Maximum waiting time | timeout, cancel future | Timeout does not guarantee remote cancellation |

### Circuit Breaker states

```text
CLOSED --high failure/slow rate--> OPEN
OPEN --wait duration--> HALF_OPEN
HALF_OPEN --trial success--> CLOSED
HALF_OPEN --trial failure--> OPEN
```

### Golden differences

```text
Circuit Breaker = recent health
Retry           = number of attempts
Rate Limiter    = throughput over time
Bulkhead        = simultaneous work
Time Limiter    = duration of one async operation
```

### Golden rules

1. Protect each outbound dependency separately.
2. Retry only transient, selected failures.
3. Make side-effecting retries idempotent at the service that owns the operation.
4. Use backoff and jitter; avoid retry storms.
5. Keep total attempts inside the end-to-end deadline.
6. Rate is not concurrency: use Rate Limiter and Bulkhead for different constraints.
7. Local policy state is normally per application instance.
8. A timeout stops waiting; it may not stop remote work.
9. Classify business/client errors separately from dependency failures.
10. Make fallbacks honest, observable, and domain-appropriate.
11. Spring proxy annotations can be bypassed by self-invocation.
12. Combining pattern order changes behavior—test the actual call count and metrics.

### Fast interview answer

> I apply resilience around outbound dependency calls. I use a Circuit Breaker to fail fast when recent failure or slow-call rates are high, Retry with backoff only for safe transient failures, Rate Limiter for time-based quotas, Bulkhead for bounded concurrency and resource isolation, and Time Limiter for an asynchronous deadline. I align these with client timeouts, ensure idempotency for side effects, use meaningful fallbacks, and verify behavior using Actuator metrics and failure tests.

### Final self-check

Can you answer all six in under two minutes?

1. Circuit Breaker vs Retry?
2. Rate Limiter vs Bulkhead?
3. Time Limiter vs HTTP client timeout?
4. Why does retry require idempotency?
5. Why does annotation order matter?
6. Why is a local Rate Limiter not a global quota?

If yes, you are ready for the typical 2–3 year Java backend Resilience4j interview discussion.

---

## Official references

- [Resilience4j Spring Boot getting started](https://resilience4j.readme.io/docs/getting-started-3)
- [Resilience4j project and modules](https://github.com/resilience4j/resilience4j)
- [Official Spring Boot 3 demo configuration](https://github.com/resilience4j/resilience4j-spring-boot3-demo/blob/master/src/main/resources/application.yml)
