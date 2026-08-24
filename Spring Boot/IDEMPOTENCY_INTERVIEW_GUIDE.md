# Idempotency in Java Microservices — Interview Study Guide

> Target: Java/Spring Boot backend developers with 2–3 years of experience  
> Goal: move from a simple definition to production design and interview-ready answers  
> Scope: REST APIs, payments, databases, retries, Kafka, orchestration, concurrency, and failure handling

---

## Table of contents

1. [How to study this guide](#1-how-to-study-this-guide)
2. [Idempotency in one minute](#2-idempotency-in-one-minute)
3. [Why duplicate requests happen](#3-why-duplicate-requests-happen)
4. [HTTP methods and idempotency](#4-http-methods-and-idempotency)
5. [Who owns idempotency?](#5-who-owns-idempotency)
6. [Idempotency-key API contract](#6-idempotency-key-api-contract)
7. [Core request flow and state machine](#7-core-request-flow-and-state-machine)
8. [Database design](#8-database-design)
9. [Spring Boot implementation](#9-spring-boot-implementation)
10. [Concurrency and atomicity](#10-concurrency-and-atomicity)
11. [Failure and crash scenarios](#11-failure-and-crash-scenarios)
12. [Redis-based approaches](#12-redis-based-approaches)
13. [Idempotency in Kafka and messaging](#13-idempotency-in-kafka-and-messaging)
14. [Outbox, inbox, and saga/orchestration](#14-outbox-inbox-and-sagaorchestration)
15. [Idempotency vs related concepts](#15-idempotency-vs-related-concepts)
16. [Security, retention, and observability](#16-security-retention-and-observability)
17. [Common mistakes](#17-common-mistakes)
18. [Testing checklist](#18-testing-checklist)
19. [Scenario-based interview questions](#19-scenario-based-interview-questions)
20. [Quick quizzes](#20-quick-quizzes)
21. [Memory maps](#21-memory-maps)
22. [Final one-page revision sheet](#22-final-one-page-revision-sheet)

---

## 1. How to study this guide

### Pass 1 — concept

- Learn the definition and real-life analogy.
- Understand why retries can create duplicate side effects.
- Explain who owns idempotency in a microservice flow.

### Pass 2 — implementation

- Read the database schema and Spring Boot flow.
- Focus on the unique constraint and transaction boundary.
- Walk through two identical requests arriving simultaneously.

### Pass 3 — interview

- Answer every scenario before opening the suggested answer.
- Practice the 30-second and 2-minute answers.
- Revise the final one-page section before the interview.

### Five questions to ask for any idempotent operation

1. What is the logical operation?
2. Who generates and stores the idempotency key?
3. What prevents concurrent duplicates?
4. What result is returned for a repeated request?
5. What happens after a crash or key expiration?

---

## 2. Idempotency in one minute

### Simple definition

An operation is **idempotent** when executing the same logical operation multiple times has the same intended business effect as executing it once.

```text
Same logical request once  → one payment
Same logical request 5 times → still one payment
```

Mathematically:

```text
f(f(x)) = f(x)
```

For APIs, “same effect” is more important than “byte-for-byte identical execution.” Logging, metrics, and timestamps may change, but the protected business side effect must not be duplicated.

### Real-life analogy

Pressing an elevator button five times does not request five elevators. The system recognizes one logical request and keeps the same intended effect.

### Example: non-idempotent payment without protection

```text
Client → POST /payments → Payment Service charges ₹500
                         ← response lost
Client → retries POST /payments → charges another ₹500 ❌
```

### With an idempotency key

```http
POST /payments
Idempotency-Key: 47b2ca18-2307-4e14-bb9f-48901f4cf92a
```

```text
First request  → charge ₹500 → save result under key
Second request → find same key → return saved result
Business effect → exactly one charge
```

### 30-second interview answer

> Idempotency means repeating the same logical request does not repeat its business side effect. For a non-idempotent operation such as payment creation, the client sends a unique idempotency key. The owning service uses an atomic database uniqueness constraint to ensure only one request wins, stores the request fingerprint and outcome, and returns the stored outcome for retries. The same key with a different payload must be rejected.

### What idempotency does not mean

- It does not mean the code executes only once internally.
- It does not guarantee exactly-once network delivery.
- It does not eliminate retries.
- It does not replace database transactions.
- It does not mean every repeated response must have a new status or body.
- It does not make an operation safe if downstream side effects are not protected.

---

## 3. Why duplicate requests happen

### Common causes

- Client double-clicks a Submit/Pay button.
- Mobile app retries after a weak network connection.
- Client times out even though the server completed the operation.
- API Gateway, load balancer, SDK, or service retries automatically.
- Message broker redelivers after consumer failure.
- Consumer completes database work but crashes before acknowledging/committing offset.
- Scheduled job runs twice.
- Two workers pick the same task.
- User refreshes a page after submission.
- Orchestrator retries a participant after losing the response.

### The fundamental distributed-systems problem

```text
Caller sends request
       ↓
Server completes operation
       ↓
Response is lost
       ↓
Caller cannot know whether operation completed
```

The caller sees an **ambiguous outcome**. Retrying is necessary for reliability, but retrying can duplicate the side effect. Idempotency makes that retry safe.

### Interview insight

> Most duplicate problems are not caused by “bad users.” They arise because delivery and acknowledgements can fail independently of business execution.

---

## 4. HTTP methods and idempotency

### Method semantics

| HTTP method | Normally idempotent? | Reason |
|---|---:|---|
| `GET` | Yes | Repeating a read should not create additional business effects |
| `HEAD` | Yes | Same semantics as metadata-only read |
| `PUT` | Yes | Replacing resource state repeatedly produces the same intended state |
| `DELETE` | Yes | Deleting an already deleted resource leaves it deleted |
| `POST` | Usually no | Often creates a new resource or triggers an action each time |
| `PATCH` | Depends | Some patches set a value; others increment/append repeatedly |

### Idempotent does not mean identical response

Two `DELETE` calls may return:

```text
First DELETE  → 204 No Content
Second DELETE → 404 Not Found
```

The responses differ, but the final resource state is still “deleted.” The intended server-side effect is idempotent.

### PUT example

```http
PUT /users/101/preferences
Content-Type: application/json

{"language":"en"}
```

Sending this repeatedly keeps `language = en`.

### PATCH counterexample

```http
PATCH /wallets/101

{"operation":"increment", "amount":500}
```

Repeating this adds ₹500 each time, so it is not naturally idempotent.

### Important nuance

HTTP defines method semantics, but the application must implement them correctly. A badly designed `GET /send-email` has a side effect even though `GET` is supposed to be safe.

---

## 5. Who owns idempotency?

### Main rule

> Idempotency belongs to the service that owns the specific business operation.

```text
Booking Orchestrator
   ├── Seat Service
   ├── Payment Service
   └── Booking Service
```

Ownership:

```text
Overall checkout workflow → Booking Orchestrator
Seat reservation          → Seat Service
Payment charge            → Payment Service
Booking creation          → Booking Service
```

### Why the API Gateway should not usually own business idempotency

The gateway may:

- Validate that an `Idempotency-Key` header exists.
- Forward the key.
- Apply edge rate limits or authentication.

The gateway usually cannot decide:

- Whether two payment payloads represent the same operation.
- Whether the prior payment is complete, pending, or reversed.
- Which result is safe to replay.
- How long a business key should remain valid.

Therefore, the business service must enforce the rule.

### Orchestrator and participant idempotency are both needed

```text
Client --checkout-key--> Orchestrator
                            |
                            +--seat-operation-key--> Seat Service
                            +--payment-key---------> Payment Service
                            +--booking-key---------> Booking Service
```

- Orchestrator prevents starting the same workflow twice.
- Each participant prevents duplication of its own side effect.
- The orchestrator should derive stable participant keys, not generate a new key on every retry.

Example:

```text
checkout key: checkout-123
payment key:  checkout-123:payment
seat key:     checkout-123:seat-reservation
```

### Interview answer

> The orchestrator owns idempotency for the whole workflow, but each participating service independently owns idempotency for its business operation. This is necessary because a response can be lost between the orchestrator and any participant.

---

## 6. Idempotency-key API contract

### Request example

```http
POST /api/payments HTTP/1.1
Content-Type: application/json
Idempotency-Key: 47b2ca18-2307-4e14-bb9f-48901f4cf92a

{
  "orderId": "ORD-1001",
  "amount": 500.00,
  "currency": "INR"
}
```

### Who should generate the key?

Usually the caller that knows the logical user action:

- Frontend/mobile app for a user submission.
- Orchestrator for a participant command.
- Producer for an event/command.

Use a high-entropy value such as UUID v4, or a stable business-operation identifier when uniqueness and scope are guaranteed.

### Key scope

Do not necessarily make the raw key globally unique forever. Define scope explicitly:

```text
(tenant_id, operation_name, idempotency_key)
```

Example:

```text
(tenant-7, CREATE_PAYMENT, 47b2ca18-...)
```

Scoping prevents accidental collision across tenants and endpoints.

### Request fingerprint

Store a hash of the normalized request content:

```text
fingerprint = SHA-256(
    operation + tenant + canonicalized business payload
)
```

Why?

```text
Same key + same payload      → retry/replay allowed
Same key + different payload → reject misuse
```

Do not blindly hash the raw JSON string because whitespace and property order can differ. Canonicalize relevant fields or construct the hash from stable business values.

### Typical response policy

| Situation | Typical handling |
|---|---|
| Missing required key | Reject with a client error such as `400` |
| New key | Reserve and execute operation |
| Same key + same request + completed | Replay stored status/body or return same resource |
| Same key + different request | Reject conflict/misuse, often `409` or `422` |
| Same key currently processing | Wait briefly, return `409`, `425`, or `202`, or ask caller to retry |
| Expired/pruned key | Treat as new according to documented retention policy |

The exact status codes are an API design choice. Document them consistently.

### What result should be stored?

Depending on the API:

- Business resource ID only, then reconstruct response.
- HTTP status plus response body.
- Selected response headers.
- Operation state: `PROCESSING`, `COMPLETED`, or `FAILED`.
- Error outcome when replaying the same failure is part of the contract.

### Response replay example

First response:

```http
HTTP/1.1 201 Created
Location: /api/payments/PAY-9001

{"paymentId":"PAY-9001","status":"SUCCESS"}
```

Retry with the same key can return the stored status/body rather than creating `PAY-9002`.

---

## 7. Core request flow and state machine

### Basic flow

```text
Receive request + idempotency key
              ↓
Validate key and calculate request fingerprint
              ↓
Atomically reserve key using a unique database constraint
      ┌───────┴────────┐
      │                │
   New key         Existing key
      │                │
Execute operation      ├── fingerprint differs → reject
      │                ├── COMPLETED → replay result
Save outcome           └── PROCESSING → wait/retry response
      │
Return result
```

### State machine

```text
                reserve key
   ABSENT ─────────────────────> PROCESSING
                                    │
                       ┌────────────┴────────────┐
                       │                         │
                    success                  handled failure
                       │                         │
                       ▼                         ▼
                   COMPLETED                  FAILED
```

Possible policies:

- Roll back the reservation on failure so a retry starts fresh.
- Persist a retryable `FAILED` state.
- Persist and replay deterministic business failures.
- Move stale `PROCESSING` records to recoverable/expired state after verifying the business outcome.

The policy depends on whether execution and result storage share one atomic transaction.

### Three invariants

1. At most one winner may own a key for a given scope.
2. One key cannot represent two different request fingerprints.
3. A completed result must be recoverable for the promised retention window.

---

## 8. Database design

### Example PostgreSQL table

```sql
CREATE TABLE idempotency_request (
    id                BIGSERIAL PRIMARY KEY,
    tenant_id         VARCHAR(100) NOT NULL,
    operation_name    VARCHAR(100) NOT NULL,
    idempotency_key   VARCHAR(255) NOT NULL,
    request_hash      VARCHAR(64)  NOT NULL,
    status            VARCHAR(20)  NOT NULL,
    resource_id       VARCHAR(100),
    http_status       INTEGER,
    response_body     TEXT,
    locked_until      TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at        TIMESTAMPTZ NOT NULL,

    CONSTRAINT uk_idempotency_scope
        UNIQUE (tenant_id, operation_name, idempotency_key),

    CONSTRAINT ck_idempotency_status
        CHECK (status IN ('PROCESSING', 'COMPLETED', 'FAILED'))
);

CREATE INDEX ix_idempotency_expiry
    ON idempotency_request (expires_at);
```

### Most important line

```sql
UNIQUE (tenant_id, operation_name, idempotency_key)
```

This unique constraint is the final concurrency guard. An application-level `findByKey()` alone is not enough.

### Atomic reservation

PostgreSQL example:

```sql
INSERT INTO idempotency_request (
    tenant_id,
    operation_name,
    idempotency_key,
    request_hash,
    status,
    expires_at
)
VALUES (?, ?, ?, ?, 'PROCESSING', ?)
ON CONFLICT (tenant_id, operation_name, idempotency_key)
DO NOTHING;
```

Interpretation:

```text
rows inserted = 1 → this request owns execution
rows inserted = 0 → another request already owns/finished it
```

Use the equivalent atomic insert/upsert supported by your database. Do not implement correctness using only a JVM lock because multiple service instances do not share that lock.

### Store full response or resource reference?

| Strategy | Advantages | Risks |
|---|---|---|
| Store full response | Exact fast replay | Storage size, sensitive data, schema/version aging |
| Store resource ID | Smaller, response can be rebuilt | Rebuilt response may reflect newer state |
| Store outcome summary | Good for status APIs | May not reproduce original HTTP contract |

Choose based on API promises and data sensitivity.

### Retention

Retention must be longer than the realistic retry window:

```text
client timeout + SDK retry period + queue delay + operational replay window
```

If a key is removed too early, an old retry may execute again. If keys are stored forever, cost and privacy risks grow.

---

## 9. Spring Boot implementation

### 9.1 API models

```java
public record CreatePaymentRequest(
        String orderId,
        BigDecimal amount,
        String currency) {
}

public record PaymentResponse(
        String paymentId,
        String status) {
}

public record StoredHttpResult(
        int statusCode,
        String responseBody) {
}
```

### 9.2 Controller

```java
@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    private final IdempotentPaymentService paymentService;

    public PaymentController(IdempotentPaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @PostMapping
    public ResponseEntity<String> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestHeader("X-Tenant-Id") String tenantId,
            @Valid @RequestBody CreatePaymentRequest request) {

        StoredHttpResult result = paymentService.create(
                tenantId,
                idempotencyKey,
                request);

        return ResponseEntity
                .status(result.statusCode())
                .contentType(MediaType.APPLICATION_JSON)
                .body(result.responseBody());
    }
}
```

In production, obtain the tenant/user identity from trusted authentication context rather than trusting an arbitrary caller-provided tenant header.

### 9.3 Repository contract

```java
public interface IdempotencyStore {

    boolean tryReserve(
            String tenantId,
            String operation,
            String key,
            String requestHash,
            Instant expiresAt);

    Optional<IdempotencyRecord> find(
            String tenantId,
            String operation,
            String key);

    void complete(
            String tenantId,
            String operation,
            String key,
            String resourceId,
            int httpStatus,
            String responseBody);
}
```

### 9.4 PostgreSQL/JdbcTemplate reservation

```java
@Repository
public class JdbcIdempotencyStore implements IdempotencyStore {

    private final JdbcTemplate jdbcTemplate;

    public JdbcIdempotencyStore(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public boolean tryReserve(
            String tenantId,
            String operation,
            String key,
            String requestHash,
            Instant expiresAt) {

        int inserted = jdbcTemplate.update("""
            INSERT INTO idempotency_request (
                tenant_id, operation_name, idempotency_key,
                request_hash, status, expires_at
            )
            VALUES (?, ?, ?, ?, 'PROCESSING', ?)
            ON CONFLICT (tenant_id, operation_name, idempotency_key)
            DO NOTHING
            """,
            tenantId, operation, key, requestHash,
            Timestamp.from(expiresAt));

        return inserted == 1;
    }

    // find(...) and complete(...) omitted here for revision clarity.
}
```

### 9.5 Service flow

```java
@Service
public class IdempotentPaymentService {

    private static final String OPERATION = "CREATE_PAYMENT";

    private final IdempotencyStore idempotencyStore;
    private final PaymentRepository paymentRepository;
    private final ObjectMapper objectMapper;
    private final RequestFingerprint requestFingerprint;

    public IdempotentPaymentService(
            IdempotencyStore idempotencyStore,
            PaymentRepository paymentRepository,
            ObjectMapper objectMapper,
            RequestFingerprint requestFingerprint) {
        this.idempotencyStore = idempotencyStore;
        this.paymentRepository = paymentRepository;
        this.objectMapper = objectMapper;
        this.requestFingerprint = requestFingerprint;
    }

    @Transactional
    public StoredHttpResult create(
            String tenantId,
            String key,
            CreatePaymentRequest request) {

        validateKey(key);
        String hash = requestFingerprint.forPayment(tenantId, request);

        boolean owner = idempotencyStore.tryReserve(
                tenantId,
                OPERATION,
                key,
                hash,
                Instant.now().plus(Duration.ofHours(24)));

        if (!owner) {
            IdempotencyRecord existing = idempotencyStore
                    .find(tenantId, OPERATION, key)
                    .orElseThrow(IdempotencyConflictException::new);

            if (!MessageDigest.isEqual(
                    hash.getBytes(StandardCharsets.UTF_8),
                    existing.requestHash().getBytes(StandardCharsets.UTF_8))) {
                throw new IdempotencyKeyReuseException(
                        "Idempotency key was already used for another request");
            }

            if (existing.status() == IdempotencyStatus.COMPLETED) {
                return new StoredHttpResult(
                        existing.httpStatus(),
                        existing.responseBody());
            }

            throw new OperationInProgressException(
                    "A request with this idempotency key is still processing");
        }

        Payment payment = paymentRepository.save(
                Payment.create(
                        tenantId,
                        request.orderId(),
                        request.amount(),
                        request.currency()));

        PaymentResponse response = new PaymentResponse(
                payment.getId(),
                payment.getStatus().name());

        String responseJson = writeJson(response);

        idempotencyStore.complete(
                tenantId,
                OPERATION,
                key,
                payment.getId(),
                HttpStatus.CREATED.value(),
                responseJson);

        return new StoredHttpResult(
                HttpStatus.CREATED.value(),
                responseJson);
    }
}
```

### 9.6 Why one transaction matters

When the payment row and idempotency record are in the same database transaction:

```text
reserve key + create payment + save response
                │
                ├── commit all
                └── rollback all
```

This avoids:

```text
payment committed
process crashes
idempotency result missing
retry creates another payment
```

### Important limitation

A local database transaction cannot atomically include a remote payment provider. For external side effects, pass a stable idempotency key downstream or use a recoverable workflow. This is covered in the failure section.

### 9.7 Exception mapping

```java
@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(IdempotencyKeyReuseException.class)
    ResponseEntity<ProblemDetail> keyReuse(IdempotencyKeyReuseException error) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.UNPROCESSABLE_ENTITY,
                error.getMessage());
        return ResponseEntity
                .status(HttpStatus.UNPROCESSABLE_ENTITY)
                .body(problem);
    }

    @ExceptionHandler(OperationInProgressException.class)
    ResponseEntity<ProblemDetail> inProgress(OperationInProgressException error) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.CONFLICT,
                error.getMessage());
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .header(HttpHeaders.RETRY_AFTER, "1")
                .body(problem);
    }
}
```

### Spring transaction proxy reminder

`@Transactional` is normally applied through a Spring proxy. Calling an annotated method from another method in the same object may bypass proxy interception. Put transactional boundaries on externally invoked service methods and test rollback behavior.

---

## 10. Concurrency and atomicity

### The broken check-then-act approach

```java
if (!repository.existsByKey(key)) {
    createPayment();
    repository.saveKey(key);
}
```

Two threads can interleave:

```text
Thread A: key does not exist
Thread B: key does not exist
Thread A: create payment A
Thread B: create payment B  ❌
```

### Correct principle

Use an atomic operation backed by a uniqueness constraint:

```text
Thread A: INSERT key → success → owns work
Thread B: INSERT same key → conflict/no-op → must not execute work
```

### Why `synchronized` is insufficient

```text
Pod 1 JVM lock != Pod 2 JVM lock
```

It may protect threads in one process, but not multiple pods, restarts, batch workers, or other writers.

### Database behavior with one transaction

With `INSERT ... ON CONFLICT DO NOTHING` and a unique constraint, a concurrent insert may wait for the first transaction to commit or roll back:

```text
Request A reserves uncommitted key and executes
Request B attempts same insert and waits at database constraint

A commits   → B loses conflict and reads completed result
A rolls back → B may reserve and execute
```

Exact locking and visibility depend on the database and isolation level. Verify them with an integration test against the production database engine, not only H2.

### Optimistic vs pessimistic locking

- A unique insert is often enough to elect the first owner.
- Optimistic locking (`@Version`) helps prevent lost updates to an existing record.
- Pessimistic locking can serialize access but may increase waits and deadlocks.
- None of these automatically protects an external side effect after a process crash.

### Business uniqueness is an additional guard

For payment-per-order semantics, also consider:

```sql
UNIQUE (tenant_id, order_id, payment_purpose)
```

An idempotency key protects retries of one request. A business uniqueness constraint protects the invariant even if a buggy client generates a new key for the same prohibited duplicate operation.

---

## 11. Failure and crash scenarios

### Scenario A — failure before business work

```text
validation fails → no operation starts → key may remain unused
```

Many APIs allow the caller to correct validation and retry. Decide whether validation failures are stored.

### Scenario B — database work and key share one transaction

```text
reserve key → insert payment → update result → commit
```

- If commit succeeds, retry replays the result.
- If transaction rolls back, neither payment nor key exists; retry can execute.
- This is the cleanest case.

### Scenario C — server commits, response is lost

```text
database commit succeeds
response is lost
client retries same key
service returns stored result ✅
```

This is the main use case for idempotency.

### Scenario D — remote provider succeeds, local save fails

```text
Service → provider charges card successfully
Service crashes before saving provider result
```

A local transaction cannot roll back the provider. Safer approaches:

1. Send the same stable idempotency key to the provider.
2. On retry, query the provider using that key/reference before charging again.
3. Persist an operation record before the remote call and reconcile stale `PROCESSING` operations.
4. Model the payment as a state machine rather than assuming one synchronous transaction.

### Scenario E — record stuck in PROCESSING

Do not blindly delete it and execute again. First determine whether the side effect occurred.

Possible recovery:

```text
PROCESSING lease expires
       ↓
recovery worker claims record using version/token
       ↓
query business DB/provider by operation reference
       ├── found → mark COMPLETED
       ├── definitely absent → safely retry
       └── unknown → manual/reconciliation state
```

### Scenario F — same key, different payload

Reject it. Otherwise the server cannot know which operation the key represents.

```text
key K + amount ₹500  → accepted
key K + amount ₹900  → reject ❌
```

### Ambiguous failures deserve an UNKNOWN/PENDING state

For money movement, pretending “failed” can be dangerous when the actual outcome is unknown. A state such as `PENDING_RECONCILIATION` may be more truthful.

---

## 12. Redis-based approaches

### Basic Redis idea

```text
SET idempotency:{scope}:{key} PROCESSING NX EX 86400
```

- `NX`: create only if absent.
- `EX`: expire after a retention period.
- Winner performs work.
- Completion replaces `PROCESSING` with stored result.

### Benefits

- Very fast lookup.
- Natural TTL support.
- Useful for high-throughput short-lived deduplication.

### Risks

- Lock/key expires while the first request still runs, allowing a second owner.
- Redis data loss/failover behavior may not match the required business guarantee.
- Business database commit and Redis update are not one atomic transaction.
- Deleting a lock without checking an ownership token can delete another worker's lock.
- A cache is not automatically a durable financial ledger.

### Ownership token rule

Store a random owner token:

```text
key   = idempotency:payment:abc
value = owner-token-7f91...
```

Release or extend only if the token still matches, usually through an atomic Lua script or equivalent primitive.

### Database vs Redis

| Requirement | Prefer database | Redis may fit |
|---|---:|---:|
| Financial/business correctness | Yes | Only with carefully designed durability/recovery |
| Same transaction as domain write | Yes | No |
| Very high-throughput short TTL dedupe | Maybe | Yes |
| Exact durable response replay | Yes | Depends on persistence setup |
| Simple cache of completed keys | Possible | Yes |

### Practical approach

Use the database as the source of truth when correctness matters. Redis can be an optimization, but cache misses must not bypass the durable uniqueness rule.

---

## 13. Idempotency in Kafka and messaging

### Why consumers see duplicates

Typical at-least-once flow:

```text
Consumer reads event
Consumer updates database
Consumer crashes before committing offset
Broker redelivers event
Consumer updates database again ❌
```

### Idempotent consumer pattern — inbox/deduplication table

```sql
CREATE TABLE processed_message (
    consumer_name VARCHAR(100) NOT NULL,
    event_id      VARCHAR(100) NOT NULL,
    processed_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (consumer_name, event_id)
);
```

Consumer transaction:

```text
BEGIN
  INSERT (consumer_name, event_id)
  if duplicate → skip business effect
  else → update business tables
COMMIT
then acknowledge/commit offset according to framework strategy
```

Spring-style example:

```java
@Transactional
public void handle(PaymentCompletedEvent event) {
    boolean firstDelivery = processedMessageRepository
            .tryInsert("order-payment-consumer", event.eventId());

    if (!firstDelivery) {
        return;
    }

    orderRepository.markPaid(
            event.orderId(),
            event.paymentId());
}
```

The inbox insert and business update must be in the same database transaction.

### Event ID vs business key

- `eventId` prevents processing the same event twice.
- A business constraint protects against two different events representing the same prohibited effect.

Example:

```text
event-1: pay order-100
event-2: pay order-100
```

Event-ID deduplication does not catch this because IDs differ. A business invariant may still need `UNIQUE(order_id, payment_purpose)`.

### Kafka idempotent producer is not enough

Kafka producer idempotence prevents duplicate records caused by producer retries under its defined scope. It does **not** automatically make an external consumer database update or email/payment call idempotent.

```text
Kafka log has one event
consumer may still process it again after failure
```

### Exactly-once nuance

Kafka transactions can provide strong exactly-once processing when consuming from Kafka and producing back to Kafka with the correct transactional setup. Once the flow includes an external database, REST API, email provider, or payment gateway, that external side effect needs its own transaction/idempotency strategy.

### Interview line

> In an at-least-once consumer, I store the event ID in an inbox table with a unique key in the same transaction as the business update. Kafka producer idempotence prevents a different class of duplicates and does not automatically deduplicate external consumer side effects.

---

## 14. Outbox, inbox, and saga/orchestration

### Transactional Outbox

Problem:

```text
save order in DB
publish OrderCreated to Kafka
```

If these are independent operations:

- DB succeeds, Kafka fails → missing event.
- Kafka succeeds, DB rolls back → event describes nonexistent state.

Outbox solution:

```text
Single DB transaction:
  save order
  save outbox event with unique event ID

Publisher later reads outbox and publishes event
```

The publisher may publish twice after a crash, so consumers still need idempotency/inbox handling.

### Inbox + Outbox flow

```text
Producer Service
  domain table + outbox table (one transaction)
             ↓
          Kafka event
             ↓
Consumer Service
  inbox table + domain update (one transaction)
```

### Saga/orchestrator retries

Every command should have a stable operation ID:

```text
ReserveSeat(commandId = checkout-123:seat)
ChargePayment(commandId = checkout-123:payment)
ConfirmBooking(commandId = checkout-123:booking)
```

Compensations should also be idempotent:

```text
RefundPayment(commandId = checkout-123:refund)
ReleaseSeat(commandId = checkout-123:release-seat)
```

Why? The compensation response can also be lost and retried.

### Important rule

> In distributed workflows, both forward actions and compensating actions must tolerate duplicate delivery.

---

## 15. Idempotency vs related concepts

### Comparison table

| Concept | Purpose | Key difference |
|---|---|---|
| Idempotency | Repeated logical operation has one intended effect | Business/API semantic guarantee |
| Deduplication | Detect/remove duplicate items | Mechanism that may help implement idempotency |
| Database transaction | Atomic commit/rollback of local operations | Does not span ordinary remote calls |
| Unique constraint | Prevent duplicate stored keys/business values | Strong concurrency primitive, not the whole API contract |
| Retry | Repeat after transient failure | Creates the need for idempotency |
| Distributed lock | Temporarily elect one owner | Does not automatically store/replay outcome |
| Optimistic locking | Detect conflicting updates by version | Protects updates, not necessarily repeated commands |
| Exactly-once delivery | Message delivery claim within a defined boundary | Usually narrower than end-to-end business effect |
| Correlation ID | Trace one request/workflow through services | For observability, not duplicate prevention |
| Request ID | Identifies one network request/attempt | A retry may have a new request ID but same idempotency key |

### Correlation ID vs idempotency key

```text
Logical payment operation: idempotency key = PAYMENT-123

Attempt 1: request/correlation ID = REQ-A
Attempt 2: request/correlation ID = REQ-B
```

The idempotency key stays stable across retries. Request/trace identifiers may change per attempt.

### Idempotency vs distributed lock

```text
Lock: “Only one worker may execute now.”
Idempotency record: “This logical operation already produced this outcome.”
```

A lock disappears; an idempotency result must remain replayable for the documented window.

### Idempotency vs database uniqueness

A unique constraint is a mechanism. A full idempotency solution also defines:

- Key scope and validation.
- Same-key/different-payload behavior.
- In-progress behavior.
- Stored result/replay behavior.
- Retention and cleanup.
- Recovery from crashes.

---

## 16. Security, retention, and observability

### Security considerations

- Scope keys by authenticated tenant/user where appropriate.
- Do not let one tenant discover another tenant's prior result.
- Limit key length and accepted character set.
- Prefer high-entropy keys to prevent guessing/collision.
- Do not include secrets or card details in the key.
- Avoid storing sensitive response bodies unnecessarily.
- Encrypt sensitive stored data and apply access controls.
- Hash only after selecting stable, relevant request fields.
- Rate-limit key creation to prevent storage exhaustion.

### Cleanup job

```sql
DELETE FROM idempotency_request
WHERE expires_at < NOW()
  AND status IN ('COMPLETED', 'FAILED');
```

For large tables, delete in bounded batches or use partitioning. Never delete active/ambiguous records without a recovery policy.

### Metrics

- New keys reserved.
- Completed operations.
- Replayed responses.
- Same-key/different-payload conflicts.
- Concurrent/in-progress conflicts.
- Stale `PROCESSING` records.
- Processing latency.
- Cleanup count and table size.
- Downstream reconciliation count.
- Duplicate messages skipped per consumer.

### Logging

Useful structured fields:

```text
operation
tenantId (subject to privacy policy)
idempotencyKeyHash
requestHash
status
resourceId
replayed=true/false
correlationId
```

Avoid logging full keys or sensitive request/response bodies if that increases security risk.

---

## 17. Common mistakes

### Design mistakes

- Checking for a key and inserting later without a unique constraint.
- Generating a new key for every retry.
- Accepting the same key with a different payload.
- Storing only `PROCESSING` but not the final outcome.
- Deleting a stuck key without checking whether the side effect occurred.
- Assuming a client timeout means the server did nothing.
- Treating every `POST` as safely retryable.
- Using only an in-memory map or `synchronized` in a multi-pod service.
- Letting a Redis lock expire while work continues.
- Forgetting idempotency for compensating operations.
- Using an idempotency key as a correlation ID or vice versa.
- Keeping keys for less time than clients can retry.

### Transaction mistakes

- Saving the business result and idempotency result in separate local transactions without recovery.
- Catching an exception and accidentally committing partial work.
- Assuming `@Transactional` spans an HTTP call and remote database.
- Calling a `@Transactional` method through same-class self-invocation.
- Testing concurrency only with H2 when production uses PostgreSQL/MySQL.

### Messaging mistakes

- Assuming Kafka idempotent producer makes consumers idempotent.
- Recording `processed_message` after the business transaction commits.
- Committing the consumer offset before the business transaction safely completes.
- Deduplicating only by event ID when a business invariant also matters.
- Keeping the inbox deduplication record for less time than messages can be replayed.

### API mistakes

- No documented behavior for missing keys.
- No maximum key size.
- Replaying an old response for a changed request.
- Returning false success while an operation is actually pending/unknown.
- Storing volatile fields in the hash so harmless retries appear different.
- Exposing one user's cached response to another due to missing scope.

---

## 18. Testing checklist

### Functional tests

- New key creates exactly one resource.
- Same key and same payload returns the original resource/result.
- Same key and different payload is rejected.
- Missing/invalid key follows the API contract.
- Expired key follows the documented policy.

### Concurrency test

Send 20 simultaneous requests with the same key:

```text
expected business rows = 1
expected successful logical result = same payment/resource ID
expected duplicate side effects = 0
```

Use a real production-like database container to verify unique-index blocking and transaction visibility.

### Failure tests

- Crash/fail before reserving key.
- Fail after reservation but before business write.
- Fail after business write but before result update.
- Commit successfully but drop HTTP response.
- Remote provider succeeds but local persistence fails.
- Record remains `PROCESSING` beyond lease duration.

### Messaging tests

- Deliver the same event twice.
- Crash after database commit but before acknowledgement.
- Deliver two event IDs for the same business operation.
- Replay old messages after inbox cleanup threshold.

### Assertions that matter

Do not only assert HTTP status. Verify:

```text
business row count
external provider call count
stored key/fingerprint/status
same resource ID in replay
transaction rollback
metrics and audit trail
```

---

## 19. Scenario-based interview questions

### Scenario 1 — response lost after payment

**Question:** The payment succeeded, but Order Service timed out. What should happen on retry?

<details>
<summary>Suggested answer</summary>

Order Service must reuse the same payment idempotency key. Payment Service checks the key, verifies the request fingerprint, and returns the original payment result instead of charging again. A timeout represents an unknown outcome, not proof of failure.

</details>

### Scenario 2 — two requests arrive at the same millisecond

**Question:** Both check the database and find no key. How do you prevent duplicate creation?

<details>
<summary>Suggested answer</summary>

Do not depend on a read-before-write check. Atomically insert a scoped idempotency record protected by a database unique constraint. Only the successful insert owns execution; the loser reads or waits for the winning result.

</details>

### Scenario 3 — same key, different amount

**Question:** First request is ₹500; retry uses the same key with ₹900. What should the service do?

<details>
<summary>Suggested answer</summary>

Reject it as key misuse. Store a canonical request fingerprint with the key and compare it on reuse. Never replay the ₹500 result as if it represented the ₹900 request, and never execute a second operation under the same key.

</details>

### Scenario 4 — where should idempotency live?

**Question:** Gateway, orchestrator, or Payment Service?

<details>
<summary>Suggested answer</summary>

The gateway may require and forward the header. The orchestrator owns idempotency for the overall workflow. Payment Service independently owns payment idempotency because it controls payment state and must protect against retries from the orchestrator. Each operation owner enforces its own invariant.

</details>

### Scenario 5 — database commit and HTTP response loss

**Question:** Why store the response/result?

<details>
<summary>Suggested answer</summary>

After the commit, the caller may retry because it never received the response. Storing the resource ID or original HTTP result lets the service answer consistently without re-executing the business operation.

</details>

### Scenario 6 — Redis `SETNX` is used

**Question:** Is that enough for payment correctness?

<details>
<summary>Suggested answer</summary>

Not by itself. The key may expire during work, Redis and the payment database are not one transaction, failover may affect durability, and a lock does not preserve the completed result. For critical operations use durable business/idempotency records and downstream idempotency/reconciliation; Redis may be an optimization.

</details>

### Scenario 7 — Kafka event processed twice

**Question:** How do you make the consumer idempotent?

<details>
<summary>Suggested answer</summary>

Insert `(consumerName, eventId)` into an inbox/processed-message table with a unique constraint in the same transaction as the business update. A duplicate insert means the effect was already applied and can be skipped.

</details>

### Scenario 8 — Kafka producer idempotence enabled

**Question:** Can the consumer now safely send an email exactly once?

<details>
<summary>Suggested answer</summary>

No. Producer idempotence addresses duplicate appends caused by producer retries within Kafka's defined boundary. A consumer can still be redelivered, and email is an external side effect. Use a durable consumer/outbox workflow and provider idempotency or a deduplication strategy where available.

</details>

### Scenario 9 — stuck PROCESSING record

**Question:** Should a cleanup job delete it and retry?

<details>
<summary>Suggested answer</summary>

Not blindly. The side effect may have completed before the crash. A recovery worker should claim the stale record, query the domain database or downstream provider using a stable reference, then mark completed, retry only if definitely absent, or move it to reconciliation when the result is unknown.

</details>

### Scenario 10 — caller generates a new key after timeout

**Question:** What is wrong?

<details>
<summary>Suggested answer</summary>

The server sees a new logical operation and may execute it again. The client must keep the same key for all retries of one user action and generate a new key only for a genuinely new action.

</details>

### Scenario 11 — unique order number already exists

**Question:** Is that idempotency?

<details>
<summary>Suggested answer</summary>

It can help enforce a business invariant, but it is not a complete idempotency API. The service still needs a defined key scope, payload validation, concurrent behavior, result replay, retention, and error contract.

</details>

### Scenario 12 — retry plus idempotency

**Question:** Which one should be implemented first?

<details>
<summary>Suggested answer</summary>

They solve different problems. Retry improves availability for transient failures; idempotency makes repeated side-effecting operations safe. Before enabling automatic retries on a write, establish end-to-end idempotency and a bounded retry policy.

</details>

### Scenario 13 — compensation is called twice

**Question:** What should `refundPayment` do?

<details>
<summary>Suggested answer</summary>

It should use a stable refund operation key and return the existing refund outcome on duplicate delivery. Compensating saga steps are distributed operations and can be retried just like forward steps.

</details>

### Scenario 14 — service has five pods

**Question:** Can `ConcurrentHashMap<String, Result>` implement idempotency?

<details>
<summary>Suggested answer</summary>

No. Each pod has separate memory, entries disappear on restart, and requests can be routed to different pods. Use a shared durable store with an atomic uniqueness mechanism.

</details>

### Scenario 15 — explain your production design

<details>
<summary>Suggested answer</summary>

I define the logical operation and key scope, require a client-generated high-entropy key, calculate a canonical request fingerprint, and atomically reserve the key using a database unique constraint. The domain change and idempotency outcome commit in one transaction where possible. Retries with the same payload replay the original resource/result; different payloads are rejected. For remote side effects I propagate a stable downstream key and reconcile ambiguous `PROCESSING` states. I set retention beyond the retry window, secure tenant scoping, monitor replays/conflicts, and test concurrent and crash scenarios against the production database engine.

</details>

---

## 20. Quick quizzes

### Quiz A — true or false

1. Idempotency means the server code physically runs only once.
2. A timeout proves the server did not finish.
3. The same idempotency key should be reused across retries of one operation.
4. The same key with a different payload should execute as a new request.
5. A database unique constraint can protect concurrent key reservation.
6. `synchronized` protects all service pods.
7. Kafka idempotent producer automatically deduplicates database writes by consumers.
8. Compensating saga actions should also be idempotent.
9. A correlation ID and idempotency key always mean the same thing.
10. `DELETE` can be idempotent even when repeated responses differ.

<details>
<summary>Answers</summary>

1. False
2. False
3. True
4. False
5. True
6. False
7. False
8. True
9. False
10. True

</details>

### Quiz B — choose the owner

```text
Overall checkout flow  → ?
Seat reservation       → ?
Payment charge         → ?
Booking confirmation   → ?
```

<details>
<summary>Answer</summary>

```text
Overall checkout flow  → Checkout/Booking Orchestrator
Seat reservation       → Seat Service
Payment charge         → Payment Service
Booking confirmation   → Booking Service
```

Each owner protects its own state transition.

</details>

### Quiz C — spot the race

```java
if (idempotencyRepository.find(key).isEmpty()) {
    paymentProvider.charge(request);
    idempotencyRepository.save(key);
}
```

<details>
<summary>Answer</summary>

Two callers can both observe “empty” and both charge. The key must be atomically reserved before the effect, using a shared uniqueness constraint. The external charge also requires a stable downstream key or reconciliation because a local transaction cannot roll it back.

</details>

### Quiz D — identify the keys

```text
Attempt 1 request ID: REQ-1
Attempt 2 request ID: REQ-2
Both attempts represent one payment.
```

<details>
<summary>Answer</summary>

The request IDs may differ because they identify attempts. Both must carry the same payment idempotency key because they represent one logical operation.

</details>

### Quiz E — 60-second verbal drill

Explain these without looking back:

1. Idempotency vs retry.
2. Idempotency vs deduplication.
3. Idempotency vs distributed lock.
4. Idempotency key vs correlation ID.
5. Kafka producer idempotence vs idempotent consumer.

---

## 21. Memory maps

### 21.1 The K-H-S-R map

```text
K = KEY          → identifies one logical operation
H = HASH         → proves repeated request has same meaning
S = STATE        → PROCESSING / COMPLETED / FAILED
R = RESULT       → replay without repeating side effect
```

### 21.2 Correctness map

```text
Idempotent operation
       │
       ├── stable key across retries
       ├── scoped unique constraint
       ├── canonical request fingerprint
       ├── atomic business transaction where possible
       ├── stored/reconstructable result
       ├── downstream idempotency for remote effects
       └── retention + recovery policy
```

### 21.3 Duplicate-source map

```text
Duplicate operation
       │
       ├── HTTP retry/double click → Idempotency-Key
       ├── concurrent requests     → unique constraint
       ├── broker redelivery       → inbox/processed-message table
       ├── DB + event publishing   → transactional outbox
       └── distributed workflow    → stable step and compensation keys
```

### 21.4 Ownership map

```text
Gateway      → validate/forward key; edge policy
Orchestrator → overall workflow idempotency
Domain owner → business-operation idempotency
Consumer     → message-processing idempotency
```

### 21.5 Failure map

```text
Can local DB transaction include every effect?
       │
       ├── Yes → key + business write + result in one transaction
       │
       └── No, remote effect
              ├── propagate stable downstream key
              ├── persist operation state
              ├── query/reconcile ambiguous outcomes
              └── never assume timeout means failure
```

---

## 22. Final one-page revision sheet

> Read this section during the final 10 minutes before an interview.

### Definition

Idempotency means repeating the same logical operation produces the same intended business effect as performing it once.

```text
f(f(x)) = f(x)
```

### Why it is needed

```text
operation succeeds → response/ack is lost → caller retries
```

Without protection, payment/order/message side effects may be duplicated.

### Golden flow

```text
Receive key
  → validate and scope it
  → calculate canonical request hash
  → atomically reserve with UNIQUE constraint
      → new: execute and save result
      → existing + same hash + completed: replay result
      → existing + different hash: reject
      → existing + processing: wait/retry response
```

### Required data

```text
scope + idempotency key
request fingerprint
PROCESSING / COMPLETED / FAILED state
resource ID or stored HTTP result
created/updated/expiry timestamps
```

### Golden rules

1. The client reuses one key for all retries of one logical action.
2. A new logical action gets a new key.
3. Scope keys by tenant and operation.
4. Reject the same key with a different payload.
5. Use a database unique constraint; check-then-insert is racy.
6. Keep domain write and idempotency result in one transaction when possible.
7. A local transaction cannot roll back a remote side effect.
8. Propagate a stable key to downstream operation owners.
9. A timeout means outcome unknown, not necessarily failed.
10. Persist/reconstruct the result so retries do not re-execute.
11. Retention must exceed the complete retry/replay window.
12. Gateway forwarding does not replace service-level idempotency.
13. Orchestrator owns workflow idempotency; participants own their operations.
14. Kafka producer idempotence does not replace an idempotent consumer.
15. Inbox records and business updates belong in one transaction.
16. Forward and compensation saga steps should both be idempotent.
17. Correlation ID traces attempts; idempotency key identifies the logical operation.

### Key comparisons

| Item | Remember |
|---|---|
| Retry | Repeats an attempt; idempotency makes repetition safe |
| Unique constraint | Atomic concurrency guard, but not the full API contract |
| Distributed lock | Temporary ownership, not durable result replay |
| Transaction | Atomic within its resource boundary |
| Deduplication | A mechanism for detecting duplicate inputs |
| Correlation ID | Observability, not business duplicate prevention |
| Kafka producer idempotence | Prevents a defined class of duplicate Kafka appends |
| Inbox table | Prevents duplicate consumer business effects |

### Fast interview answer

> For a write API, the caller sends a stable idempotency key for one logical operation. The owning service scopes the key, stores a canonical request hash, and atomically reserves it using a database unique constraint. The domain change and result are committed in one transaction where possible. A retry with the same key and payload receives the original result; a different payload is rejected. For external side effects, the service propagates a stable downstream key and reconciles ambiguous outcomes. In messaging, the consumer uses an inbox unique key in the same transaction as its business update.

### Final interview self-check

Can you answer these in two minutes?

1. Why is check-then-insert unsafe?
2. Why store a request hash?
3. What happens when a request is already processing?
4. Why is a local transaction insufficient for a remote payment?
5. Who owns idempotency in an orchestrated workflow?
6. How do you make a Kafka consumer idempotent?
7. Idempotency key vs correlation ID?
8. How do you recover a stale `PROCESSING` operation?

---

## Official references

- [IETF HTTPAPI Idempotency-Key header Internet-Draft](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header)
- [Stripe API: idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- [Spring Framework: declarative transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html)
- [Spring Framework: using `@Transactional`](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Apache Kafka: message delivery semantics](https://kafka.apache.org/30/design/design/)
- [Apache Kafka: producer configuration](https://kafka.apache.org/configuration/producer-configs/)

> Note: At the time this guide was prepared, the IETF `Idempotency-Key` document was an Internet-Draft rather than a final RFC. Treat its status-code guidance as evolving and document your API contract explicitly.
