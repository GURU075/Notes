# Spring Boot JPA/Hibernate — `save()`, `flush()`, and `commit()`

## The core idea

- **`save()`** → Hibernate manages the entity in the **Persistence Context**.
- **`flush()`** → Hibernate executes pending `INSERT` / `UPDATE` / `DELETE` SQL against the database **inside the current transaction**.
- **`commit()`** → The database makes the transaction **permanent**.
- **`rollback()`** → The database undoes the changes made by that transaction.

## Graphical explanation

https://github.com/GURU075/Notes/blob/main/Hibernate%20%26%20JPA/save_flush_commit.png

## 1. `save()`

```java
repository.save(entity);
```

`save()` tells Spring Data JPA/Hibernate to persist the entity.

The entity becomes managed in Hibernate's **Persistence Context**.

Conceptually:

```text
Java Application
      |
      | save(entity)
      v
Hibernate Persistence Context
      |
      | Entity is managed
      |
      v
Database
      |
      | SQL may not have executed yet
```

Do not think of `save()` as meaning:

> "The row is definitely already inserted into the database."

Hibernate can defer the SQL until a flush is required.

---

## 2. What is the Persistence Context?

The Persistence Context is Hibernate's in-memory working area for entities participating in the current persistence operation/transaction.

For example:

```text
Persistence Context

IdempotencyKey
-------------------------
userId = 10
itemId = 50
status = PROCESSING
```

Hibernate tracks the entity and knows that it needs to eventually synchronize this state with the database.

---

## 3. `flush()`

```java
repository.flush();
```

`flush()` tells Hibernate:

> "Synchronize the changes currently tracked in the Persistence Context with the database."

Hibernate checks for pending changes:

```text
New entity       -> INSERT
Changed entity   -> UPDATE
Deleted entity   -> DELETE
```

For example:

```sql
INSERT INTO idempotency_key
    (user_id, item_id, status)
VALUES
    (10, 50, 'PROCESSING');
```

### Important

The `INSERT` is actually executed by the database when the flush happens.

It is **not** stored in some special "flush area" waiting for commit.

The database has now processed the change as part of the **current database transaction**.

However, the transaction has not necessarily been committed yet.

---

## 4. `commit()`

After the SQL has been executed, the transaction can be committed.

```text
flush()
   |
   | INSERT executed
   v
Database transaction
   |
   | COMMIT
   v
Changes become permanent
```

Therefore:

> **Flush executes the SQL. Commit finalizes the transaction.**

Commit normally does **not** execute the `INSERT` again.

---

## 5. `saveAndFlush()`

```java
repository.saveAndFlush(entity);
```

Conceptually:

```java
repository.save(entity);
repository.flush();
```

So:

```text
saveAndFlush()
      |
      +--> save()
      |      |
      |      +--> Entity managed by Hibernate
      |
      +--> flush()
             |
             +--> Pending SQL executed in DB transaction
```

But:

> **`saveAndFlush()` still does NOT commit the transaction.**

---

## 6. What happens if an exception occurs after `flush()`?

Consider:

```java
@Transactional
public void test() {

    repository.save(entity);

    repository.flush();

    throw new RuntimeException();
}
```

Flow:

```text
save()
  |
  v
Persistence Context
  |
  | flush()
  v
INSERT executed in DB
  |
  | exception
  v
ROLLBACK
  |
  v
INSERT is undone
```

So even though the `INSERT` was executed during `flush()`, the transaction can still be rolled back.

That is why:

```text
flush != commit
```

---

## 7. Where does an `exists` check happen?

Suppose you call:

```java
repository.existsByUserIdAndItemId(userId, itemId);
```

Hibernate may execute a query against the database, conceptually:

```sql
SELECT ...
FROM idempotency_key
WHERE user_id = ?
  AND item_id = ?;
```

So the existence check is ultimately against the **database**, not simply against some separate "flush storage."

There is one important Hibernate detail:

With the normal `FlushMode.AUTO`, Hibernate may automatically flush pending changes before certain queries when required to keep query results consistent.

Therefore, do not use this oversimplified model:

```text
save() -> memory only
commit() -> INSERT
```

The better model is:

```text
save()
   |
   v
Persistence Context
   |
   | explicit flush()
   | OR automatic flush when needed
   v
Actual SQL executes in DB transaction
   |
   v
commit()
   |
   v
Transaction becomes permanent
```

---

## 8. Idempotency-key example

Suppose your database has a uniqueness constraint such as:

```text
(user_id, item_id)
```

and your application does:

```java
@Transactional
public void process(Long userId, Long itemId) {

    IdempotencyKey key = new IdempotencyKey();
    key.setUserId(userId);
    key.setItemId(itemId);
    key.setStatus("PROCESSING");

    repository.saveAndFlush(key);

    // Continue processing...
}
```

The flow is:

```text
saveAndFlush()
      |
      +--> save()
      |      |
      |      v
      |   Persistence Context
      |
      +--> flush()
             |
             v
      INSERT executed
             |
             v
      Current DB transaction
      (not committed yet)
             |
             v
      Business logic
             |
        success?
        /     \
      YES     NO
       |       |
       v       v
    COMMIT   ROLLBACK
       |       |
       v       v
   Permanent  INSERT undone
```

The database uniqueness constraint is what provides the actual duplicate protection.

`flush()` is about **when Hibernate synchronizes SQL with the database**.

---

## 9. The simplest mental model

Remember these three lines:

```text
save()
  = "Hibernate, track this entity."

flush()
  = "Hibernate, execute the pending SQL in the current DB transaction."

commit()
  = "Database, make this transaction final."
```

### One final distinction

```text
save()
  -> Persistence Context

flush()
  -> Database transaction

commit()
  -> Permanent database state
```

And:

```text
flush() + exception + rollback()
  -> Database changes are undone
```

---

## Summary table

| Operation | Main responsibility | SQL executed? | Transaction committed? |
|---|---|---:|---:|
| `save()` | Put/manage entity in Persistence Context | Not necessarily immediately | No |
| `flush()` | Synchronize Persistence Context with DB | Yes | No |
| `saveAndFlush()` | `save()` + `flush()` | Yes | No |
| `commit()` | Finalize transaction | SQL was already flushed or may be flushed as part of commit | Yes |
| `rollback()` | Undo transaction changes | N/A | No |

## The sentence to remember

> **`save()` changes Hibernate's managed state; `flush()` executes the pending SQL in the current transaction; `commit()` makes that transaction permanent.**
