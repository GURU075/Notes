# Redis — Simple Notes and Interview Questions (Java / Spring Boot)

## 1. What is Redis?

**Redis** (REmote DIctionary Server) is a fast, in-memory data store. It stores data mainly in **RAM** and supports multiple data structures such as strings, hashes, lists, sets and sorted sets.

Redis is commonly used for **caching, sessions, OTPs, rate limiting, counters, distributed locks and messaging**.

> **Remember:** Redis is not just a cache. It is a data store with optional persistence, replication and more.

## 2. Why do we use Redis?

Suppose a Spring Boot application gets a product from MySQL. If thousands of users repeatedly request the same product, repeatedly querying MySQL adds load and latency.

**Without Redis**

```text
Client → Spring Boot API → MySQL → Response
Client → Spring Boot API → MySQL → Response (again)
```

**With Redis (cache-aside pattern)**

```text
                     ┌── Cache HIT ──→ Return cached product
Client → Spring API ─┤
                     └── Cache MISS → MySQL → Store in Redis → Return product
```

**Benefit:** Fewer database queries, less database load, and often faster API responses.

### Typical flow

1. An API receives `GET /products/101`.
2. The application checks Redis for `product:101`.
3. If found (**cache hit**), return it immediately.
4. If absent (**cache miss**), load it from MySQL.
5. Cache the result in Redis, usually with a **TTL** (expiration).
6. Return the response.

## 3. Where does Redis run in microservices?

Redis normally runs as a **separate service/server**, not inside the Spring Boot application. It may run in Docker, Kubernetes, a VM or a managed cloud service.

```text
Frontend
   |
API Gateway
   |
Spring Boot Product Service ──→ Redis Server (cache)
   |
   └───────────────────────────→ MySQL (source of truth)
```

Multiple Spring Boot instances can share the same Redis service. Protect it with network restrictions and authentication; never expose it publicly without appropriate controls.

## 4. Common real-world use cases

| Use case | Example | Important detail |
|---|---|---|
| Caching | Product details, reference data | Set a TTL and invalidate on updates |
| Session management | Store session state shared across API instances | Spring Session can use Redis |
| OTP storage | Store verification code for 5 minutes | TTL; secure verification; rate limiting |
| Rate limiting | 100 requests per user per minute | Atomic counters or Lua scripts |
| Distributed locking | Prevent simultaneous conflicting operations | Correctness needs special care |
| Temporary reservations | Hold a movie seat for 5 minutes | Redis TTL alone is not the final booking guarantee |
| Counters | Page visits, attempts | `INCR` is atomic |
| Pub/Sub | Broadcast transient messages | Subscribers can miss messages when disconnected |

### Ticket booking example

In a movie ticket booking app, two users may attempt seat **A1** at once. The application can use Redis to hold a *temporary reservation*:

```text
Key:   seat-lock:show-42:A1
Value: reservation-unique-token
TTL:   300 seconds
```

Example command for a temporary lock:

```redis
SET seat-lock:show-42:A1 reservation-uuid NX EX 300
```

- `NX` means create the key **only if it does not already exist**.
- `EX 300` means expire after **300 seconds**.
- The booking must also be protected at the **database level**, e.g. unique constraints or transactions, because locks may expire, processes can crash and network failures happen.
- Unlock only if the stored token still belongs to the caller (use an atomic compare-and-delete script); do **not** blindly `DEL` a lock that may now belong to someone else.

## 5. Redis data types — easy examples

### String — simple value, cache, counter

```redis
SET user:1:name "Guru"
GET user:1:name
SET otp:user:1 "123456" EX 300
INCR api:requests:count
```

### Hash — related fields of an object

```redis
HSET user:1 name "Guru" city "Pune"
HGET user:1 name
HGETALL user:1
```

### List — ordered entries

```redis
LPUSH tasks "task-1"
LPUSH tasks "task-2"
LRANGE tasks 0 -1
```

### Set — unique, unordered members

```redis
SADD active:users 1 2 3 2
SMEMBERS active:users
```

### Sorted Set — ordered by score

```redis
ZADD leaderboard 100 player1 200 player2
ZREVRANGE leaderboard 0 -1 WITHSCORES
```

**Also know:** Redis Streams provide persistent-log-style messaging with consumer groups; they differ from basic Pub/Sub.

## 6. Important commands

| Command | Purpose |
|---|---|
| `SET key value` | Store a value |
| `GET key` | Read a value |
| `DEL key` | Delete a key |
| `EXPIRE key 60` | Set TTL to 60 seconds |
| `TTL key` | See remaining expiration time |
| `SET key val NX EX 60` | Conditionally create expiring key |
| `INCR key` | Atomically increment counter |
| `HSET`, `HGET` | Write/read hash fields |
| `SADD`, `SMEMBERS` | Add/read set members |
| `ZADD`, `ZRANGE` | Use sorted sets |
| `SCAN` | Iterate keys incrementally |

**Production tip:** Avoid `KEYS *` on large production databases; prefer `SCAN` for incremental iteration.

## 7. Redis vs MySQL

| Redis | MySQL |
|---|---|
| Primarily in-memory | Primarily disk-backed relational database |
| Key-based access and specialized structures | SQL, relations, joins and transactions |
| Excellent for frequently accessed or temporary data | Commonly the primary source of business records |
| Optional persistence (RDB/AOF) | Durable transactional storage is a primary design goal |
| Supports TTL per key | No directly equivalent automatic TTL for each row |

**Practical design:** MySQL/PostgreSQL stores confirmed orders; Redis speeds up repeated reads and manages short-lived data.

## 8. Spring Boot integration (cache)

### Maven dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

### Configuration (Spring Boot 3.x example)

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

In production, configure credentials and TLS as appropriate and source secrets from secure configuration, not hardcoded values.

### Enable caching

```java
@SpringBootApplication
@EnableCaching
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### Read from cache

```java
@Service
public class ProductService {
    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return repository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found"));
    }
}
```

- First call (cache miss): database query runs.
- Later call (cache hit): method body is bypassed and the cached value is returned.
- This requires Redis to be configured as the **Spring Cache provider**, not merely a Redis dependency.
- Choose a serializer suitable for your model; don't assume arbitrary JPA entities serialize safely.
- Spring's proxy-based caching may be bypassed by **self-invocation** (calling a cached method from another method on the same instance).

### Updating/deleting cached data

```java
@CacheEvict(value = "products", key = "#id")
public void deleteProduct(Long id) {
    repository.deleteById(id);
}
```

`@CachePut` can update a cache entry with a method's returned value. Think about transaction boundaries and cache consistency, especially when database writes fail or roll back.

### TTL configuration example

```java
@Bean
RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10));

    return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
}
```

Imports for this snippet include `Duration`, `RedisCacheConfiguration`, `RedisCacheManager`, and `RedisConnectionFactory`.

## 9. Key caching concepts

**Cache hit:** Requested data exists in Redis.

**Cache miss:** Data is absent; load it from the source.

**TTL (Time To Live):** How long a key lives before expiry. Example: `EX 300` = 5 minutes.

**Cache invalidation:** Removing/updating cached data when the real data changes. Stale data is a common caching bug.

**Cache-aside:** Application reads cache first and fills it from DB on misses.

**Write-through:** Writes update both cache and persistent store through a coordinated path.

**Cache penetration:** Requests for nonexistent keys repeatedly hit the DB; consider negative caching and validation.

**Cache breakdown / stampede:** A popular key expires and many requests hit the DB simultaneously; solutions include request coalescing, TTL jitter and careful refresh strategies.

**Cache avalanche:** Many keys expire together, overwhelming the DB; jitter TTLs and stagger refreshes.

## 10. Redis reliability and limitations

- **Persistence:** RDB snapshots and AOF logging can recover data, with trade-offs in speed, storage and possible data loss.
- **Replication:** Redis can replicate data to replicas for availability and read scaling; replication is normally asynchronous.
- **High availability:** Redis Sentinel or managed services can automate monitoring/failover.
- **Cluster:** Redis Cluster distributes keys across nodes for horizontal scale.
- **Memory is limited:** Configure capacity and an appropriate `maxmemory-policy`. Depending on policy, Redis may evict keys or reject writes when full.
- **Security:** Authentication, TLS, private networking, least privilege; avoid storing sensitive plaintext when possible.
- **Distributed locks:** Timeouts, crashes and failovers can compromise assumptions. For critical operations, consider fencing tokens and database-level safeguards.

---

# Redis Interview Questions with Answers

## Beginner — must prepare

### Q1. What is Redis?
**Answer:** Redis is a fast, in-memory data store supporting key-value access and multiple data structures. We use it for caching, sessions, counters, rate limiting, temporary data and messaging.

### Q2. Why is Redis faster than querying MySQL?
**Answer:** Redis primarily serves data from memory and typically uses simple key lookups. MySQL often performs SQL processing and accesses relational storage. Actual latency depends on networking, workload and query design.

### Q3. Is Redis a database or only a cache?
**Answer:** It is a data store that can be used as a database, cache, message broker and more. It also supports optional disk persistence.

### Q4. What is a cache hit and cache miss?
**Answer:** A hit means data is found in Redis. A miss means it isn't found and the app normally fetches the data from the original source.

### Q5. What is TTL? Why do we use it?
**Answer:** Time To Live is a key's expiration period. TTL helps remove temporary or outdated entries, such as a five-minute OTP or cached product data.

### Q6. What is the difference between Redis and MySQL?
**Answer:** Redis is in-memory and excels at fast key-based operations and temporary data; MySQL is a relational SQL database typically used for durable business records and complex querying.

### Q7. What is Redis caching in Spring Boot?
**Answer:** Spring's cache abstraction can use Redis as the backing cache. Methods annotated with `@Cacheable` can return cached data rather than execute database queries on every call.

### Q8. What is `@Cacheable` vs `@CachePut` vs `@CacheEvict`?
**Answer:** `@Cacheable` returns cached results or executes and stores on a miss. `@CachePut` executes the method and stores the returned result. `@CacheEvict` removes selected entries.

### Q9. What happens after an entry expires?
**Answer:** Redis treats the entry as absent; the next read gets a miss. In a cache-aside design, the application reloads it from the database and caches a fresh value.

### Q10. Where is Redis deployed in microservices?
**Answer:** Usually separately as a shared infrastructure service, e.g. Docker, Kubernetes, a server or managed cloud Redis. Services connect to it over the network.

## Intermediate — common for 3-year Java developers

### Q11. Explain the cache-aside pattern.
**Answer:** The application checks Redis first. On a miss, it reads from the database, puts the result in Redis with TTL and returns the result. Updates must invalidate or refresh the cache.

### Q12. What is the N+1 query problem? Does Redis solve it?
**Answer:** N+1 means one query loads parent records and N additional queries load related records. Fix it using fetch joins, projections, entity graphs, batching or query design. Redis may reduce repeated reads but does **not** fix the underlying N+1 query pattern.

### Q13. How do you ensure cached data stays consistent after a DB update?
**Answer:** Common strategies include evicting/refreshing the relevant key after a successful DB commit, using TTL, and event-driven invalidation. Strict consistency may require using the database as the authoritative source rather than trusting cache entries.

### Q14. What is cache stampede?
**Answer:** Many requests simultaneously miss one hot key, frequently right after its expiry, and overload the database. Mitigate with synchronized loading/request coalescing, TTL jitter, stale-while-revalidate designs, or controlled locking.

### Q15. How would you implement a basic rate limiter with Redis?
**Answer:** Use atomic counters with expiration, or a Lua script for atomic increment-plus-expiry behavior. Advanced designs use sliding windows or token buckets. Be careful that setting a TTL separately from `INCR` isn't atomic unless coordinated.

### Q16. Can Redis store Java objects?
**Answer:** Yes. Spring Data Redis can serialize objects, commonly as JSON or another configured format. Pick stable serialization and avoid unintentionally caching large, cyclic JPA entity graphs.

### Q17. What are Redis data types?
**Answer:** Strings, hashes, lists, sets, sorted sets, streams and others. Use strings for values/counters, hashes for object fields, sets for unique membership, sorted sets for ranking and streams for event logs.

### Q18. Redis Pub/Sub vs Kafka — are they the same?
**Answer:** No. Basic Redis Pub/Sub is transient and doesn't replay missed messages; Kafka persists records and supports replay and consumer-group processing. Redis Streams offer a different, more durable messaging option than Redis Pub/Sub.

### Q19. What happens if Redis goes down?
**Answer:** A caching application should ideally fall back to its database or degrade gracefully, subject to system design. If Redis is used for sessions, locks or coordination, outages can disrupt functionality. Use timeouts, monitoring, appropriate failover and deliberate fallback behavior.

### Q20. What is Redis eviction?
**Answer:** When a configured memory limit is reached, Redis may evict entries according to the selected memory policy. TTL expiration and memory-pressure eviction are different mechanisms.

## Advanced / scenario-based

### Q21. Two customers try to book the same seat. How can Redis help?
**Answer:** Use `SET lock-key unique-token NX EX seconds` for a short reservation lock, then verify and commit the booking in a database transaction with uniqueness constraints. Release the lock only when the unique token matches. Redis alone must not be the final guarantee of no double booking.

### Q22. Why not use `SETNX` followed by a separate `EXPIRE` for a lock?
**Answer:** The process could crash between commands, leaving a lock without TTL. A single `SET key value NX EX seconds` is atomic for creation plus expiry.

### Q23. What is the difference between RDB and AOF persistence?
**Answer:** RDB creates point-in-time snapshots. AOF logs write operations for replay (with configurable fsync/durability trade-offs). Organizations choose based on recovery requirements and overhead; some use both.

### Q24. Is Redis single-threaded?
**Answer:** Command execution is largely organized around a single-threaded event loop in typical Redis deployments, but modern Redis also uses background and optional I/O threads. Saying “Redis has only one thread” is inaccurate.

### Q25. What is Redis Cluster vs replication?
**Answer:** Replication copies data between primary and replica nodes. Redis Cluster also partitions keys across multiple primary nodes and can use replicas for availability.

### Q26. How do you troubleshoot slow Redis-backed APIs?
**Answer:** Measure cache hit rate, key size, serialization overhead, network latency, Redis CPU/memory, slow commands, eviction rates and DB fallback spikes. Verify timeouts, connection pools and the effect of key expiry.

### Q27. Why shouldn't Redis be the only source of truth for confirmed orders?
**Answer:** Confirmed business transactions generally need strong consistency and durable invariants. Cache expiry, failover, eviction and asynchronous replication complicate guarantees; a transactional persistent database is commonly safer.

### Q28. How can a Redis lock fail even if `NX EX` is used?
**Answer:** A holder can exceed the TTL, lose connectivity or encounter failover; a different holder may then acquire the lock while the old holder is still running. Critical systems need stronger protections, such as fencing tokens and database constraints.

---

## One-minute interview explanation

> “In our Spring Boot architecture, Redis can be used as a shared in-memory cache. For frequently accessed product data, the service checks Redis first, and on a cache miss it fetches from the database and stores the response with a TTL. This reduces database load and usually improves response time. Redis also supports counters, session storage, rate limiting and short-lived reservation locks. For confirmed transactions, the database remains the source of truth, while cache invalidation and failover handling are important design considerations.”

## Quick revision checklist

- [ ] Explain why Redis is used.
- [ ] Draw Spring Boot → Redis → database flow.
- [ ] Understand cache hit, miss, TTL and invalidation.
- [ ] Know `@Cacheable`, `@CachePut`, `@CacheEvict`.
- [ ] Know strings, hashes, sets, sorted sets and streams.
- [ ] Explain rate limiting and temporary seat locking.
- [ ] Differentiate Redis Pub/Sub vs Kafka.
- [ ] Explain cache stampede and stale data.
- [ ] Explain RDB vs AOF, replication and Redis Cluster.
- [ ] State why DB constraints are still needed for bookings.
