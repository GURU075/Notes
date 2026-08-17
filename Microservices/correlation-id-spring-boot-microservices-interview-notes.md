# Correlation ID in Spring Boot Microservices — Simple Interview Notes

## 1. What is a Correlation ID?

A **Correlation ID** is a unique ID attached to one request flow.

Example:

```text
Client -> API Gateway -> Show Service -> Venue Service
```

All logs for that request can use the same value:

```text
X-Correlation-ID: checkout-request-123
```

Example logs:

```text
[correlationId=checkout-request-123] Gateway received request
[correlationId=checkout-request-123] Show Service loaded show
[correlationId=checkout-request-123] Venue Service loaded venue
```

### Interview line

> A correlation ID is a unique identifier propagated across services for one request flow. It helps us connect logs from different microservices and debug a request end to end.

---

## 2. What problem does it solve?

Without a correlation ID, logs from multiple services are difficult to connect.

```text
Client -> Gateway -> Booking Service -> Payment Service
```

With one shared ID:

```text
Gateway         -> request-789
Booking Service -> request-789
Payment Service -> request-789
```

we can search the same value everywhere.

### Interview line

> In microservices, one request can travel through multiple applications. Correlation ID gives all those logs a common searchable value, which makes troubleshooting much easier.

---

## 3. Where is the Correlation ID stored?

In this project, it is sent in the HTTP header:

```http
X-Correlation-ID: request-123
```

### Interview line

> We usually propagate the correlation ID using an HTTP header. In our project, the header is `X-Correlation-ID`.

---

## 4. Why generate it at the API Gateway?

The API Gateway is the common entry point.

```text
Client
   |
   v
API Gateway
   |
   +--> Service A
   +--> Service B
```

The Gateway can guarantee that each request entering internal services has a correlation ID.

### Interview line

> We generate the correlation ID at the API Gateway because it is the common entry point and can ensure every downstream request has one consistent identifier.

---

## 5. What if the client already sends an ID?

If it is valid and safe, preserve it.

```http
X-Correlation-ID: mobile-request-456
```

This helps connect client-side logs with backend logs.

### Interview line

> If the client sends a safe and valid correlation ID, we preserve it so client logs and backend logs can refer to the same request.

---

## 6. What if the ID is missing?

The Gateway generates a UUID.

Example:

```text
550e8400-e29b-41d4-a716-446655440000
```

### Interview line

> If the correlation ID is missing, the Gateway generates a UUID and forwards that same value through the request flow.

---

## 7. What if the incoming ID is invalid?

This project accepts values that:

```text
Length: 1 to 128 characters
First character: letter or digit
Remaining characters: letters, digits, ., _, -
```

Examples:

```text
Valid:
checkout-123
mobile.app_request_456
550e8400-e29b-41d4-a716-446655440000

Invalid:
request with spaces
value containing newline
value longer than 128 characters
```

Unsafe values are replaced with a UUID.

### Interview line

> We validate client-provided correlation IDs because headers are untrusted input. If the value is missing, empty, too long, or contains unsafe characters, we replace it with a generated UUID.

---

## 8. Complete request flow

```text
Client Request
      |
      v
CorrelationIdFilter
      |
      |-- valid incoming ID -> preserve
      |
      |-- missing/unsafe ID -> generate UUID
      |
      v
Put ID into MDC
      |
      v
Ensure one X-Correlation-ID header
      |
      v
Spring Cloud Gateway route
      |
      v
Downstream Service
      |
      v
Return same X-Correlation-ID to client
```

### Interview line

> The Gateway filter validates or generates the ID, stores it in MDC for logging, forwards it to downstream services, and also returns it in the response.

---

## 9. Why use a filter?

Correlation-ID handling should happen before routing.

Example:

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {
}
```

### Interview line

> We implement correlation-ID handling as a global filter because it should execute early for every incoming request before Gateway routing happens.

---

## 10. Why `OncePerRequestFilter`?

This project uses Spring Cloud Gateway Server Web MVC on the Servlet stack.

It is useful because it can:

```text
run before Gateway route handling
wrap HttpServletRequest
set response headers
work for routed and local endpoints
clean MDC in finally
```

### Interview line

> Since our Gateway uses the Servlet stack, we use `OncePerRequestFilter` so the correlation logic runs at the request boundary and we can reliably clean MDC in a `finally` block.

---

## 11. Important constants

```java
public static final String HEADER_NAME = "X-Correlation-ID";
public static final String MDC_KEY = "correlationId";
```

### Interview line

> We keep the HTTP header name and MDC key as constants so the same names are used consistently across filters, logs, and tests.

---

## 12. Why use `HttpServletRequestWrapper`?

Servlet request headers are effectively immutable.

This does not exist:

```java
request.setHeader(...);
```

So the filter wraps the request and overrides:

```java
getHeader()
getHeaders()
getHeaderNames()
```

### Interview line

> Servlet request headers cannot be directly changed, so we wrap the request using `HttpServletRequestWrapper` and expose the final correlation ID through the header methods.

---

## 13. Why override all three header methods?

Framework code may read headers in different ways:

```java
getHeader()
getHeaders()
getHeaderNames()
```

### Interview line

> We override all relevant header methods because frameworks may access headers differently, and this keeps the canonical correlation ID visible consistently.

---

## 14. What is MDC?

MDC means:

```text
Mapped Diagnostic Context
```

Example:

```java
MDC.put("correlationId", correlationId);
```

Then logs can automatically include:

```text
[correlationId=request-123]
```

### Interview line

> MDC is a logging context that lets us attach request-scoped values like correlation ID to log statements automatically without passing the ID through every method.

---

## 15. Example logging pattern

```yaml
logging:
  pattern:
    correlation: "[correlationId=%X{correlationId:-none}] "
```

`%X{correlationId}` reads the MDC value.

### Interview line

> Once the ID is placed in MDC, our logging pattern reads it automatically and prints it with each request log.

---

## 16. Why is MDC cleanup important?

Servlet server threads are reused.

```text
Thread-7 -> Request A
Thread-7 -> later Request B
```

If Request A's ID remains in MDC, Request B may log the wrong ID.

Use:

```java
try {
    filterChain.doFilter(request, response);
}
finally {
    restoreMdc(previousValue);
}
```

### Interview line

> MDC cleanup is mandatory because application-server threads are reused. If we do not clean it, a later request handled by the same thread could get the previous request's correlation ID.

---

## 17. Why clean MDC in `finally`?

Because `finally` runs for both success and exception paths.

### Interview line

> We clean MDC inside `finally` so cleanup happens for both successful requests and exceptional requests.

---

## 18. How is the ID propagated?

The same ID should travel across services:

```text
Client
   |
   | request-123
   v
Gateway
   |
   | request-123
   v
Show Service
   |
   | request-123
   v
Venue Service
```

Do not generate a new ID in every service.

### Interview line

> Each downstream service should preserve and forward the same correlation ID. Generating a new ID in every service would break end-to-end log correlation.

---

## 19. Does Gateway propagation automatically populate downstream MDC?

No.

Each downstream service should:

```text
1. Read X-Correlation-ID
2. Put it into its own MDC
3. Include it in logs
4. Forward it when calling another service
```

### Interview line

> Forwarding the header through the Gateway does not automatically populate MDC in other services. Each service must extract the header into its own logging context or use a tracing framework.

---

## 20. Why return the ID to the client?

The Gateway can set:

```java
response.setHeader("X-Correlation-ID", correlationId);
```

Then the client can report:

```text
My request failed.
Correlation ID: checkout-request-123
```

### Interview line

> We return the correlation ID in the response so the client or support team can provide that exact ID when reporting a failed request.

---

## 21. Correlation ID vs Trace ID

```text
Correlation ID
= Connect related logs

Trace ID
= Distributed tracing with spans, timing and dependencies
```

### Interview line

> Correlation ID is mainly used to connect logs for one request, while a trace ID belongs to distributed tracing and connects multiple spans with timing and dependency information.

---

## 22. Correlation ID vs Idempotency Key

```text
Correlation ID
= Observability and debugging

Idempotency Key
= Prevent duplicate business operations
```

Never use correlation ID to decide whether a payment or booking was already processed.

### Interview line

> Correlation ID is for observability, whereas an idempotency key protects against duplicate business operations. They solve completely different problems.

---

## 23. Correlation ID vs User ID

```text
Correlation ID -> request flow
User ID        -> user
```

Do not use correlation ID for authentication or authorization.

### Interview line

> A correlation ID identifies a request, not a user, so it must never be used as an authentication or authorization mechanism.

---

## 24. Should sensitive information be stored in a Correlation ID?

No.

Do not encode:

```text
username
email
access token
booking details
sensitive information
```

Use an opaque value such as a UUID.

### Interview line

> Correlation IDs should be opaque values. We should not encode usernames, emails, tokens, or other sensitive business information inside them.

---

## 25. What about asynchronous processing?

Traditional MDC is thread-local.

If work moves from:

```text
Thread 1 -> Thread 5
```

the MDC value may not automatically move with it.

### Interview line

> Traditional MDC is thread-local, so when execution moves to another thread, we may need explicit context propagation or a tracing framework.

---

## 26. Common mistakes

### Mistake 1: Generate a new ID in every service

Wrong:

```text
Gateway         -> A123
Booking Service -> B456
Payment Service -> C789
```

Correct:

```text
Gateway         -> A123
Booking Service -> A123
Payment Service -> A123
```

### Interview line

> The same request should keep the same correlation ID across services; otherwise end-to-end correlation is lost.

### Mistake 2: Trust any client value

### Interview line

> Correlation-ID headers are untrusted input, so we validate their format and size before storing them in logs.

### Mistake 3: Forget the response header

### Interview line

> Returning the correlation ID in the response makes production support easier because users can report the exact request identifier.

### Mistake 4: Forget MDC cleanup

### Interview line

> MDC should always be cleaned after request processing because server threads are reused.

---

## 27. ShowSeat example

```http
POST /api/bookings
X-Correlation-ID: booking-request-123
```

Flow:

```text
Client
   |
   v
API Gateway
   |
   v
Booking Service
   |
   v
Seat Service
   |
   v
Payment Service
```

All of them use:

```text
booking-request-123
```

### Interview line

> In a booking application, the same correlation ID can travel from Gateway to Booking Service, Seat Service, and Payment Service, allowing us to search the complete request flow from one ID.

---

## 28. What should we test?

```text
1. Valid client ID is preserved
2. Missing ID generates UUID
3. Unsafe ID is replaced
4. ID reaches downstream service
5. Same ID is returned to client
```

### Interview line

> I would test both correlation-ID creation and propagation: preserve valid values, generate missing values, replace unsafe values, verify downstream forwarding, and verify the response header.

---

## 29. Manual testing

With custom ID:

```powershell
$response = Invoke-WebRequest `
  -Uri "http://localhost:8088/api/event/getAll" `
  -Headers @{ "X-Correlation-ID" = "manual-test-123" }

$response.Headers["X-Correlation-ID"]
```

Expected:

```text
manual-test-123
```

Without ID:

```powershell
$response = Invoke-WebRequest `
  -Uri "http://localhost:8088/api/event/getAll"

$response.Headers["X-Correlation-ID"]
```

Expected:

```text
Generated UUID
```

---

# 30. Full Interview Answer

> A correlation ID is a unique identifier used to track one request across multiple microservices. We normally propagate it through an HTTP header such as `X-Correlation-ID`. At the API Gateway, I use a global filter that checks whether a valid ID is already present. If it is present, I preserve it; otherwise I generate a UUID. I put that value into MDC so it automatically appears in logs, forward the same header to downstream services, and return it in the response. Each downstream service should also read the header into its own logging context and propagate it when calling another service. MDC must be cleaned after request processing because server threads are reused.

---

# 31. 30-Second Interview Answer

> Correlation ID helps track one request across multiple microservices. We usually generate or validate it at the API Gateway, pass the same ID using an HTTP header, store it in MDC for logging, and propagate it to downstream services. This makes distributed debugging much easier.

---

# 32. One-Minute Revision

```text
Correlation ID
= Unique ID for one request flow

Purpose
= Logging, debugging, observability

Header
= X-Correlation-ID

Gateway
= Preserve valid ID or generate UUID

MDC
= Put ID automatically into logs

Propagation
= Same ID across downstream services

Response
= Return same ID to client

MDC cleanup
= Mandatory because threads are reused

Correlation ID != User ID
Correlation ID != Idempotency Key

Correlation ID
= Connect logs

Trace ID
= Distributed trace with spans and timing
```

---

# 33. Quick Interview Q&A

### Q1. What is a correlation ID?

> It is an identifier propagated across services involved in one request so their logs can be connected.

### Q2. Why generate it at API Gateway?

> The Gateway is the common entry point, so it can guarantee that every downstream request has a consistent identifier.

### Q3. Why preserve an existing ID?

> It allows the client and backend systems to refer to the same request, provided the incoming value is valid and safe.

### Q4. Why use UUID?

> UUID can be generated locally without a database call and has a very low probability of collision.

### Q5. Why use MDC?

> MDC automatically adds request-related context such as correlation ID to logs without passing it through every method.

### Q6. Why clean MDC?

> Application-server threads are reused, so an old ID could otherwise appear in another request's logs.

### Q7. Does Gateway propagation automatically populate downstream MDC?

> No. Each downstream service needs to extract the header into its own MDC or use tracing/context-propagation infrastructure.

### Q8. Correlation ID vs Trace ID?

> Correlation ID mainly connects logs, while a trace ID is part of distributed tracing and links spans, dependencies, and timings.

### Q9. Correlation ID vs Idempotency Key?

> Correlation ID is for observability; idempotency key prevents duplicate business operations.

### Q10. Is correlation ID used for security?

> No. It is not authentication, authorization, user identity, or replay protection.
