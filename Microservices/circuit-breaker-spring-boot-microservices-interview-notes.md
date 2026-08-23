# Circuit Breaker in Spring Boot Microservices — Simple Point-Wise Notes

## 1. What is a Circuit Breaker?

A **Circuit Breaker** is a resiliency pattern used in microservices.

Its purpose is:

```text
Stop repeatedly calling a failing downstream service.
```

Example:

```text
Booking Service
      |
      v
Payment Service
```

If Payment Service is down, Booking Service should not keep calling it again and again.

### Interview line

> Circuit Breaker is a resiliency pattern that stops repeated calls to a failing downstream service and helps prevent cascading failures.

---

## 2. Why do we need it?

Suppose:

```text
Booking Service -> Payment Service
```

Payment Service becomes slow or unavailable.

Without Circuit Breaker:

```text
Request 1 -> Payment Service -> waits
Request 2 -> Payment Service -> waits
Request 3 -> Payment Service -> waits
Request 4 -> Payment Service -> waits
```

This can cause:

- More threads waiting
- More network calls
- Higher latency
- Resource exhaustion
- Cascading failure

### Interview line

> We use Circuit Breaker because continuously calling an unhealthy service wastes resources and can cause failures to spread to other services.

---

## 3. Simple Real-Life Example

Think about an electrical circuit breaker.

```text
Too much current
      |
      v
Circuit Breaker trips
      |
      v
Current flow stops
```

In microservices:

```text
Too many failures
      |
      v
Circuit Breaker opens
      |
      v
Calls to failing service stop temporarily
```

### Interview line

> It works like an electrical circuit breaker: when failures cross a threshold, it opens the circuit and temporarily stops calls to the unhealthy service.

---

## 4. Main States

A Circuit Breaker has 3 main states:

```text
1. CLOSED
2. OPEN
3. HALF_OPEN
```

### CLOSED

```text
Everything is working normally.
Calls are allowed.
Failures are monitored.
```

### Interview line

> In CLOSED state, requests are allowed to reach the downstream service and the Circuit Breaker keeps monitoring success and failure rates.

### OPEN

```text
Failure threshold crossed.
Calls are blocked temporarily.
```

### Interview line

> In OPEN state, the Circuit Breaker blocks calls to the failing service for a configured period so the service gets time to recover.

### HALF_OPEN

```text
After waiting, allow a few test calls.
```

If test calls succeed:

```text
HALF_OPEN -> CLOSED
```

If they fail:

```text
HALF_OPEN -> OPEN
```

### Interview line

> In HALF_OPEN state, a limited number of requests are allowed to test whether the downstream service has recovered.

---

## 5. State Flow

```text
CLOSED
  |
  | failure rate crosses threshold
  v
OPEN
  |
  | wait duration finishes
  v
HALF_OPEN
  |      |
  |      |
success  failure
  |      |
  v      v
CLOSED  OPEN
```

### Interview line

> The normal state flow is CLOSED to OPEN when failures cross the threshold, then OPEN to HALF_OPEN after a wait period, and finally back to CLOSED if test calls succeed.

---

## 6. ShowSeat Example

```text
Booking Service -> Payment Service
```

Payment Service healthy:

```text
Booking Service
      |
      v
Circuit Breaker CLOSED
      |
      v
Payment Service
      |
      v
Payment Success
```

Payment Service failing:

```text
Booking Service
      |
      v
Circuit Breaker OPEN
      |
      X
Payment Service
```

Instead, return fallback or a temporary-unavailable response.

### Interview line

> For example, if Payment Service becomes unavailable, Booking Service can open the Circuit Breaker and stop making unnecessary calls until Payment Service recovers.

---

## 7. Failure Rate Threshold

This decides:

```text
When should the circuit open?
```

Example:

```text
failureRateThreshold = 50%
```

If 10 recent calls contain 5 failures, failure rate is 50%.

### Interview line

> Failure rate threshold defines how many failed calls are acceptable before the Circuit Breaker moves to OPEN state.

---

## 8. Sliding Window

Circuit Breaker checks recent calls, not all calls since startup.

Example:

```text
slidingWindowSize = 10
```

Meaning:

```text
Look at the recent 10 calls.
```

### Interview line

> Sliding window defines the recent set of calls used to calculate the failure or slow-call rate.

---

## 9. Count-Based Sliding Window

Example:

```text
slidingWindowType = COUNT_BASED
slidingWindowSize = 10
```

Meaning:

```text
Look at the last 10 calls.
```

### Interview line

> In count-based sliding window, Circuit Breaker calculates failure rate using the most recent configured number of calls.

---

## 10. Time-Based Sliding Window

Example:

```text
slidingWindowType = TIME_BASED
slidingWindowSize = 10
```

Meaning:

```text
Look at calls from the recent configured time period.
```

### Interview line

> In time-based sliding window, failure statistics are calculated using calls that occurred during a configured recent time period.

---

## 11. Minimum Number of Calls

Suppose the first request fails.

```text
1 request
1 failure
= 100% failure
```

We usually do not want the circuit to open based on one request.

Example:

```text
minimumNumberOfCalls = 10
```

### Interview line

> Minimum number of calls prevents the Circuit Breaker from making a decision based on too little data.

---

## 12. Wait Duration in OPEN State

Example:

```text
waitDurationInOpenState = 10 seconds
```

Flow:

```text
OPEN
 |
 | wait 10 seconds
 v
HALF_OPEN
```

### Interview line

> Wait duration in OPEN state defines how long calls are blocked before the Circuit Breaker starts testing the service again.

---

## 13. Permitted Calls in HALF_OPEN

Example:

```text
permittedNumberOfCallsInHalfOpenState = 3
```

Meaning:

```text
Allow only 3 test requests.
```

### Interview line

> Permitted calls in HALF_OPEN state controls how many test requests are allowed before deciding whether the service has recovered.

---

## 14. Slow Calls

A service can be unhealthy even if it does not throw an exception.

Example:

```text
Payment Service response time = 20 seconds
```

Circuit Breaker can also monitor slow-call behavior.

### Interview line

> Circuit Breaker can also treat very slow responses as unhealthy, not only exceptions.

---

## 15. Circuit Breaker vs Timeout

### Timeout

```text
How long should I wait for one request?
```

### Circuit Breaker

```text
Should I continue calling this service at all?
```

Easy difference:

```text
Timeout
= Protect one request.

Circuit Breaker
= Protect repeated requests and the system.
```

### Interview line

> Timeout limits how long one call can wait, while Circuit Breaker stops repeated calls after the downstream service shows a failure pattern.

---

## 16. Circuit Breaker vs Retry

### Retry

```text
Try the failed request again.
```

### Circuit Breaker

```text
Stop calling when the service is repeatedly unhealthy.
```

Too many retries can make an outage worse.

### Interview line

> Retry handles temporary failures by trying again, while Circuit Breaker stops calls when repeated failures indicate the downstream service is unhealthy.

---

## 17. Circuit Breaker vs Bulkhead

### Circuit Breaker

```text
Stops calls to unhealthy dependency.
```

### Bulkhead

```text
Limits how many resources one dependency can consume.
```

### Interview line

> Circuit Breaker prevents repeated calls to a failing dependency, while Bulkhead isolates resources so one failing dependency cannot consume all application capacity.

---

## 18. Circuit Breaker vs Rate Limiter

### Circuit Breaker

```text
Protects from failing downstream dependency.
```

### Rate Limiter

```text
Protects from too many incoming requests.
```

### Interview line

> Circuit Breaker reacts to dependency health, whereas Rate Limiter controls how many requests are allowed in a given period.

---

## 19. Fallback

A fallback is an alternate behavior when the protected call cannot complete.

Example:

```text
Payment Service unavailable
        |
        v
Fallback
        |
        v
"Payment service temporarily unavailable"
```

### Interview line

> A fallback provides an alternative response or behavior when the downstream call cannot be completed.

---

## 20. Never Return Fake Success

Bad:

```text
Payment failed
      |
      v
Fallback says "Payment successful"
```

Better:

```text
Payment is temporarily unavailable.
```

### Interview line

> Fallback should not hide critical business failures. For important operations like payments, I prefer a clear failure or pending response rather than fake success.

---

## 21. Common Spring Boot Library

A common library is:

```text
Resilience4j
```

It supports:

- Circuit Breaker
- Retry
- Rate Limiter
- Bulkhead
- Time Limiter

### Interview line

> In Spring Boot, Resilience4j is commonly used to implement Circuit Breaker and other resiliency patterns.

---

## 22. Basic Resilience4j Example

```java
@CircuitBreaker(
        name = "paymentService",
        fallbackMethod = "paymentFallback"
)
public PaymentResponse makePayment(PaymentRequest request) {
    return paymentClient.makePayment(request);
}
```

### Interview line

> With Resilience4j, I can use `@CircuitBreaker` around a downstream call and configure a fallback method for failure handling.

---

## 23. What does `name` mean?

```java
@CircuitBreaker(name = "paymentService")
```

This connects the annotation with its configuration.

Example:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
```

### Interview line

> The Circuit Breaker name links the annotated method to a specific Resilience4j configuration instance.

---

## 24. Fallback Method Example

```java
public PaymentResponse paymentFallback(
        PaymentRequest request,
        Throwable throwable) {

    return new PaymentResponse(
            "Payment service temporarily unavailable"
    );
}
```

### Interview line

> The fallback method should have a compatible return type and parameters so Resilience4j can call it when the protected operation fails.

---

## 25. Example Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
```

Meaning:

```text
Observe last 10 calls
Start evaluating after 5 calls
Open when failure rate reaches threshold
Stay OPEN for 10 seconds
Then allow 3 test calls
```

### Interview line

> I configure Circuit Breaker behavior using properties such as sliding window size, minimum calls, failure threshold, open-state duration, and permitted half-open calls.

---

## 26. Complete Request Flow

```text
Client
   |
   v
Booking Service
   |
   v
Circuit Breaker
   |
   | CLOSED?
   |
   +---- YES ----> Call Payment Service
   |
   +---- OPEN ----> Do not call Payment Service
                         |
                         v
                     Fallback
```

---

## 27. Failure Example

Suppose:

```text
failureRateThreshold = 50%
minimumNumberOfCalls = 5
```

Recent calls:

```text
Failure
Success
Failure
Failure
Success
```

Failures:

```text
3 / 5 = 60%
```

So:

```text
CLOSED -> OPEN
```

---

## 28. Cascading Failure

Without Circuit Breaker:

```text
Payment Service DOWN
       |
       v
Booking Service waits
       |
       v
More requests wait
       |
       v
Booking Service becomes slow
       |
       v
Other parts of system suffer
```

### Interview line

> Circuit Breaker helps prevent cascading failure by failing fast instead of allowing an unhealthy dependency to consume more resources.

---

## 29. Fail Fast

When the circuit is OPEN:

```text
Do not call unhealthy service.
Do not wait for timeout.
Return immediately.
```

### Interview line

> When the circuit is OPEN, requests fail fast instead of waiting for the downstream service to time out.

---

## 30. Where Should It Be Applied?

Usually around:

- Remote service calls
- External APIs
- Third-party services
- Other unreliable network dependencies

Not around every normal local Java method.

### Interview line

> Circuit Breaker is mainly useful around remote or unreliable dependencies where network or service failures can occur.

---

## 31. Caller-Side Pattern

Suppose:

```text
Booking Service -> Payment Service
```

Usually Booking Service owns the Circuit Breaker because it is the caller.

### Interview line

> Circuit Breaker is typically implemented on the caller side because the caller must protect itself from a failing downstream dependency.

---

## 32. With Feign / RestClient / WebClient

The concept stays the same:

```text
Service
   |
   v
Circuit Breaker
   |
   v
HTTP Client
   |
   +--> Feign
   +--> RestClient
   +--> WebClient
```

### Interview line

> Circuit Breaker wraps the remote service call, so the same pattern can be used around Feign, RestClient, or WebClient calls.

---

## 33. Common Mistakes

### Mistake 1: No Timeout

Circuit Breaker should usually be combined with sensible timeouts.

### Interview line

> Circuit Breaker should normally be combined with proper timeouts because we still need to limit how long an individual call can wait.

### Mistake 2: Too Many Retries

Too many retries can overload an already failing service.

### Interview line

> Retry settings must be conservative because aggressive retries can increase load during an outage.

### Mistake 3: Fake-Success Fallback

Never hide a real business failure.

### Interview line

> Fallback must preserve business correctness and should never convert a critical failure into fake success.

### Mistake 4: Circuit Opens Too Early

Use `minimumNumberOfCalls`.

### Interview line

> I configure a minimum number of calls so the Circuit Breaker does not react to one isolated failure.

### Mistake 5: Circuit Opens Too Late

Thresholds should be tuned properly.

### Interview line

> Circuit Breaker thresholds should be tuned using realistic traffic and failure behavior rather than arbitrary values.

---

## 34. Circuit Breaker Does Not Fix the Service

```text
Circuit Breaker != Repair mechanism
```

It only protects the caller.

### Interview line

> Circuit Breaker does not fix the failing service; it protects the caller and gives the downstream service time to recover.

---

## 35. Monitoring

Important things to monitor:

- Circuit state
- Failure rate
- Slow-call rate
- Number of blocked calls
- State transitions

### Interview line

> Circuit Breaker metrics and state transitions should be monitored so we can detect unhealthy dependencies and understand production failures.

---

## 36. Timeout + Retry + Circuit Breaker

Remember:

```text
Timeout
= How long do I wait?

Retry
= Should I try again?

Circuit Breaker
= Should I call this service at all?
```

### Interview line

> Timeout, Retry, and Circuit Breaker solve different failure problems and are often combined carefully as part of a resiliency strategy.

---

# 37. Full Interview Answer

> Circuit Breaker is a resiliency design pattern used to protect a service from repeatedly calling an unhealthy downstream dependency. Normally the circuit stays CLOSED and requests are allowed. If the failure rate crosses a configured threshold, it moves to OPEN state and fails requests quickly without calling the downstream service. After a configured wait duration, it moves to HALF_OPEN and allows a few test requests. If those succeed, it closes again; otherwise it opens again. In Spring Boot, Resilience4j is commonly used to implement this pattern.

---

# 38. 30-Second Interview Answer

> Circuit Breaker prevents repeated calls to a failing downstream service. It has three main states: CLOSED, OPEN, and HALF_OPEN. In CLOSED state calls are allowed, in OPEN state calls are blocked, and in HALF_OPEN state a few test requests are allowed to check recovery. It helps us fail fast and prevent cascading failures.

---

# 39. Quick Interview Q&A

### Q1. What problem does Circuit Breaker solve?

> It prevents repeated calls to an unhealthy downstream service and helps avoid cascading failures.

### Q2. What are the main states?

> CLOSED, OPEN, and HALF_OPEN.

### Q3. What happens in CLOSED state?

> Calls are allowed and failures are monitored.

### Q4. What happens in OPEN state?

> Calls to the protected downstream service are blocked temporarily.

### Q5. What happens in HALF_OPEN state?

> A limited number of calls are allowed to test whether the service has recovered.

### Q6. Circuit Breaker vs Retry?

> Retry attempts the failed operation again, while Circuit Breaker stops calls after repeated failures.

### Q7. Circuit Breaker vs Timeout?

> Timeout limits one request's waiting time, while Circuit Breaker reacts to a pattern of repeated failures or slow calls.

### Q8. What is fallback?

> Fallback is an alternative behavior or response when the protected call cannot be completed.

### Q9. Why is Circuit Breaker useful in microservices?

> Because network calls and downstream services can fail independently, and Circuit Breaker prevents those failures from spreading.

### Q10. Which library is commonly used in Spring Boot?

> Resilience4j is commonly used.

---

# 40. One-Minute Revision

```text
Circuit Breaker
= Stop repeatedly calling a failing service

Purpose
= Resilience + prevent cascading failure

CLOSED
= Calls allowed

OPEN
= Calls blocked

HALF_OPEN
= Few test calls allowed

Failure threshold
= When to open

Sliding window
= Which recent calls to measure

Minimum calls
= Enough data before decision

Wait duration
= How long circuit stays OPEN

Fallback
= Alternative behavior

Timeout
= How long to wait for one call

Retry
= Try failed call again

Circuit Breaker
= Should we call the service at all?

Common Spring library
= Resilience4j
```

# Final Memory Line

> **Circuit Breaker checks the health pattern of a downstream service. If failures become too high, it stops calls temporarily, tests recovery later, and protects the rest of the system from cascading failure.**
