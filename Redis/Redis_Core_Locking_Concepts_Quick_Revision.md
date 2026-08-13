# Redis Concurrency Concepts — Quick Revision Notes

## 1. Redis TTL

**TTL = Time To Live**

TTL allows a Redis key to automatically expire after a specific amount of time.

Example:

```redis
SET seat-lock:A1 LOCK-123 PX 300000
```

Meaning:

```text
Key   = seat-lock:A1
Value = LOCK-123
TTL   = 300000 milliseconds = 5 minutes
```

After 5 minutes, the key becomes expired.

### Why TTL is useful

TTL is useful for temporary data such as:

- seat locks
- OTPs
- login sessions
- cache entries
- rate-limit counters
- temporary tokens

### Interview answer

> Redis TTL allows a key to automatically expire after a configured duration. It is useful for temporary data because the application does not need to manually delete every expired key.

---

# 2. Redis Expiration

Redis automatically handles expired keys.

There are two important mechanisms:

## Lazy expiration

When an expired key is accessed, Redis checks its expiry time.

```text
GET key
   ↓
Has TTL expired?
   ↓
Yes
   ↓
Remove/treat key as expired
   ↓
Return nil
```

## Active expiration

Redis also periodically checks keys with expiration times and removes expired keys.

Therefore, normally we do not need an application scheduler just to remove TTL-based temporary keys.

---

# 3. Atomic Operation

**Atomic means an operation happens completely or not at all, without another command interfering in the middle.**

Example of a non-atomic flow:

```text
Check A1
Check A2
Create A1
Create A2
```

Another request could execute between those commands.

That creates a race condition.

Atomic version:

```text
Check A1 and A2
If both are free:
    create both
Otherwise:
    create none
```

This behaves as one logical operation.

### Interview answer

> An atomic operation is an indivisible operation. Other Redis commands cannot interfere in the middle of the decision.

---

# 4. Race Condition

A race condition happens when multiple requests access or modify the same data at nearly the same time and the final result depends on execution timing.

Example:

```text
User A checks A1 -> free
User B checks A1 -> free

User A creates lock
User B also tries to create lock
```

Both users initially saw the same seat as free.

This is a concurrency problem.

We solve such problems using atomic Redis operations, Lua scripts, or other locking techniques.

---

# 5. Redis Lua Scripting

Redis supports Lua scripts.

A Lua script allows us to send multiple Redis operations together and execute them atomically.

Example idea:

```lua
if redis.call('EXISTS', KEYS[1]) == 0 then
    redis.call('SET', KEYS[1], ARGV[1], 'PX', ARGV[2])
    return 1
end

return 0
```

Conceptually:

```text
Check key
    +
Create key
```

happen atomically.

### Why Lua is useful

Without Lua:

```text
GET
↓
time gap
↓
SET / DELETE
```

Another request can run during the gap.

With Lua:

```text
Check + Modify
```

happen in one Redis-side operation.

### Interview answer

> Redis Lua scripting is useful when multiple Redis commands need to execute atomically. It prevents race-condition windows between separate commands.

---

# 6. KEYS and ARGV in Lua

Redis Lua scripts usually receive two types of inputs.

## KEYS

`KEYS` contains Redis key names.

Example:

```text
KEYS[1] = seat-lock:{show:101}:seat:A1
KEYS[2] = seat-lock:{show:101}:seat:A2
```

## ARGV

`ARGV` contains values and normal arguments.

Example:

```text
ARGV[1] = LOCK-123|501
ARGV[2] = 300000
```

Important:

```text
KEYS -> Redis keys
ARGV -> values/configuration
```

Lua arrays start from index `1`.

---

# 7. Distributed Locking

A distributed lock is a lock shared across multiple application instances.

Imagine your Spring Boot application is running on three servers:

```text
Instance 1
Instance 2
Instance 3
```

All three can receive requests.

A Java `synchronized` block only protects threads inside one JVM.

It does not automatically coordinate all three application instances.

Redis can act as a shared coordination system:

```text
Instance 1 ─┐
Instance 2 ─┼──> Redis
Instance 3 ─┘
```

All instances check the same Redis lock.

### Interview answer

> A distributed lock coordinates access to a shared resource across multiple application instances or processes.

---

# 8. Why `synchronized` Is Not Enough

Example:

```java
synchronized void lockSeat() {
}
```

This can protect threads inside:

```text
Application Instance 1
```

But if another request reaches:

```text
Application Instance 2
```

it has a different JVM and a different monitor lock.

Therefore:

```text
synchronized = JVM-level coordination
Redis lock   = distributed coordination
```

---

# 9. Compare-and-Delete

Compare-and-delete means:

> Delete a lock only if its current value still matches the expected value.

Example:

Expected:

```text
LOCK-A|501
```

Current Redis value:

```text
LOCK-B|502
```

They are different.

Therefore:

```text
DO NOT DELETE
```

Lua concept:

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end

return 0
```

This protects a newer owner's lock from an older delayed request.

---

# 10. Why GET + DELETE Is Unsafe

This looks safe:

```java
String current = redis.get(key);

if (expected.equals(current)) {
    redis.delete(key);
}
```

But there is a race window.

Example:

```text
1. GET returns LOCK-A
2. LOCK-A expires
3. Another user creates LOCK-B
4. DELETE executes
5. LOCK-B is accidentally deleted
```

The problem is the time gap between:

```text
GET
and
DELETE
```

Lua solves this by doing:

```text
Compare + Delete
```

atomically.

---

# 11. All-or-Nothing Multi-Key Operation

Suppose a user wants:

```text
A1
A2
A3
```

Bad flow:

```text
Lock A1 -> success
Lock A2 -> success
Lock A3 -> failed
```

Now the request partially succeeded.

This is undesirable.

Correct behavior:

```text
If A1, A2 and A3 are all free:
    lock all

Otherwise:
    lock none
```

This is an all-or-nothing operation.

Lua is useful for implementing this atomically.

---

# 12. Redis `SET NX`

Redis supports:

```redis
SET key value NX PX 300000
```

Meaning:

```text
NX -> set only if the key does not already exist
PX -> add TTL in milliseconds
```

For a single resource, this can be very useful.

Example:

```redis
SET seat:A1 LOCK-123 NX PX 300000
```

If `A1` does not exist:

```text
lock created
```

If it already exists:

```text
operation fails
```

However, for multiple related keys, separate `SET NX` commands can cause partial success.

That is one reason Lua is useful for multi-key locking.

---

# 13. Redis Cluster Hash Tags

Example keys:

```text
seat-lock:{show:101}:seat:A1
seat-lock:{show:101}:seat:A2
seat-lock:{show:101}:group:LOCK-123
```

The part:

```text
{show:101}
```

is called a **Redis Cluster hash tag**.

Redis Cluster uses the content inside `{}` to determine the hash slot.

Therefore these keys can be colocated in the same slot:

```text
{show:101}
{show:101}
{show:101}
```

This is useful when a multi-key Lua script needs to operate on related keys together.

### Interview answer

> Redis Cluster hash tags ensure related keys use the same hash slot, which is important for multi-key operations in a clustered Redis setup.

---

# 14. Temporary Lock vs Permanent State

Redis locks are generally temporary.

Example:

```text
Redis:
A1 -> temporary lock
```

Permanent business data should normally remain in the durable database.

Example:

```text
PostgreSQL:
A1 -> BOOKED
```

Therefore:

```text
Redis      = temporary coordination
Database   = durable business state
```

TTL expiration of a Redis lock does not automatically change permanent database records.

---

# 15. Fail-Safe Lock Ownership

A lock value should contain some ownership identity.

Example:

```text
LOCK-123|501
```

Where:

```text
LOCK-123 = lock ID
501      = user ID
```

Why?

Because before extending or deleting the lock, the application can verify:

```text
Does this lock still belong to this request/user?
```

This prevents one user from accidentally deleting or modifying another user's lock.

---

# 16. Lock Extension

Sometimes a temporary lock is close to expiration while an important operation is starting.

Example:

```text
Current TTL = 2 seconds
```

The application may first:

```text
verify ownership
+
extend TTL
```

Using:

```redis
PEXPIRE key 60000
```

This gives another:

```text
60 seconds
```

The extension should be performed safely with ownership verification.

Lua can make:

```text
Verify + Extend
```

atomic.

---

# 17. Important Redis Commands

| Command | Purpose |
|---|---|
| `SET` | Store a value |
| `GET` | Read a value |
| `EXISTS` | Check whether a key exists |
| `DEL` | Delete a key |
| `TTL` | Get remaining TTL in seconds |
| `PTTL` | Get remaining TTL in milliseconds |
| `EXPIRE` | Set TTL in seconds |
| `PEXPIRE` | Set TTL in milliseconds |
| `SET ... NX` | Create only if key does not exist |
| `SET ... PX` | Set expiration in milliseconds |

Example:

```redis
SET my-key value PX 300000
```

```redis
TTL my-key
```

```redis
PTTL my-key
```

---

# 18. Main Concepts to Remember

```text
Redis TTL
        ↓
Automatic temporary expiry

Atomic Operation
        ↓
No interference in the middle

Race Condition
        ↓
Concurrent requests producing incorrect results

Lua Script
        ↓
Execute multiple Redis operations atomically

Distributed Lock
        ↓
Coordinate multiple application instances

Compare-and-Delete
        ↓
Delete only if ownership still matches

SET NX
        ↓
Create only if key does not already exist

Cluster Hash Tag
        ↓
Keep related keys in the same Redis Cluster slot
```

---

# 19. Common Interview Questions

## What is TTL in Redis?

> TTL means Time To Live. It defines how long a Redis key should exist before it automatically expires.

## Why use Redis for temporary locks?

> Redis is fast, supports atomic operations, and provides native TTL, making it useful for short-lived distributed locks.

## What is a race condition?

> A race condition occurs when concurrent requests access the same resource and the result depends on the timing of their operations.

## Why use Lua scripts in Redis?

> Lua scripts allow multiple Redis commands to execute atomically, preventing another command from interfering in the middle.

## What is a distributed lock?

> A distributed lock coordinates access to a shared resource across multiple application instances or processes.

## Why is GET followed by DELETE unsafe?

> Another request can modify the key between GET and DELETE. Compare-and-delete should therefore be atomic.

## What does `SET NX` mean?

> `NX` means Redis should set the key only if the key does not already exist.

## What does `PX` mean?

> `PX` specifies the key expiration time in milliseconds.

## What are Redis Cluster hash tags?

> The value inside `{}` in a Redis key is used to force related keys into the same Redis Cluster hash slot.

---

# 20. Strong Interview Answer

> Redis provides several useful concurrency concepts for temporary locking. TTL automatically expires abandoned locks, while `SET NX` can create a key only if it does not already exist. For more complex multi-key operations, Redis Lua scripts allow checks and updates to execute atomically, which prevents race conditions. Lock ownership should be stored in the value so operations such as extension and deletion can use compare-and-delete semantics. In a distributed application, Redis can coordinate multiple service instances, and Redis Cluster hash tags can colocate related keys for multi-key operations.

---

# 21. Quick Revision

```text
TTL               = automatic expiration
Atomicity         = operation cannot be interrupted midway
Race condition    = timing-related concurrency bug
Lua               = atomic multi-command Redis logic
Distributed lock  = shared lock across application instances
Compare-and-delete= delete only when value still matches
SET NX            = set only if absent
PX                = TTL in milliseconds
Hash tag {...}    = same Redis Cluster slot
```

## One-Line Memory Rule

> Redis locking is mainly about temporary ownership + TTL + atomic operations + race-condition prevention.
