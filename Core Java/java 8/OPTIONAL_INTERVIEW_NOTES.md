# Java 8 Optional — Practical and Interview Notes

> Target: Java developers with 3–4 years of experience  
> Goal: understand how to represent missing values safely and use `Optional` correctly in production code

---

## Table of contents

1. [What is Optional?](#1-what-is-optional)
2. [The problem before Optional](#2-the-problem-before-optional)
3. [Creating an Optional](#3-creating-an-optional)
4. [Reading and consuming values](#4-reading-and-consuming-values)
5. [`orElse()` vs `orElseGet()`](#5-orelse-vs-orelseget)
6. [`orElseThrow()`](#6-orelsethrow)
7. [`map()`](#7-map)
8. [`flatMap()`](#8-flatmap)
9. [`filter()`](#9-filter)
10. [Complete runnable example](#10-complete-runnable-example)
11. [Where to use Optional](#11-where-to-use-optional)
12. [Where to avoid Optional](#12-where-to-avoid-optional)
13. [Primitive Optional types](#13-primitive-optional-types)
14. [Java 8 limitations](#14-java-8-limitations)
15. [Common mistakes](#15-common-mistakes)
16. [Interview questions](#16-interview-questions)
17. [Quick revision sheet](#17-quick-revision-sheet)

---

## 1. What is Optional?

`Optional<T>` is a container that either:

- Contains one non-null value of type `T`, or
- Contains no value

```text
Optional<User>
    ├── contains one User
    └── empty
```

It was introduced in Java 8 to represent the possible absence of a return value explicitly.

```java
Optional<User> findUserById(int id)
```

The method signature tells the caller:

> A user may or may not be found, so absence must be handled.

### Interview definition

> `Optional` is a Java 8 container that contains either one non-null value or no value. Its main use is as a method return type that explicitly communicates that a result may be absent.

### Important clarification

`Optional` does not guarantee that a `NullPointerException` can never occur. It only gives you a better API for modeling an absent result. Incorrect code can still:

- Assign `null` to an `Optional` variable.
- Call `get()` on an empty `Optional`.
- Dereference null inside a mapping function.
- Pass null into `Optional.of()`.

---

## 2. The problem before Optional

### Traditional null-returning method

```java
public User findUserById(int id) {
    return null;
}
```

The method signature does not show that absence is possible:

```java
User findUserById(int id)
```

The caller may forget to check:

```java
User user = findUserById(101);
System.out.println(user.getName());
```

If the user is not found:

```text
NullPointerException
```

### Traditional defensive code

```java
User user = findUserById(101);

if (user != null) {
    System.out.println(user.getName());
} else {
    System.out.println("User not found");
}
```

This works, but every caller must remember the convention.

### With Optional

```java
public Optional<User> findUserById(int id) {
    User user = searchDatabase(id);
    return Optional.ofNullable(user);
}
```

Caller:

```java
String name = findUserById(101)
        .map(User::getName)
        .orElse("User not found");
```

The possibility of absence is part of the return type.

---

## 3. Creating an Optional

### 3.1 `Optional.of()`

Use `of()` only when the value is guaranteed to be non-null:

```java
String name = "Anita";

Optional<String> optionalName = Optional.of(name);
```

Result:

```text
Optional[Anita]
```

Passing null throws `NullPointerException` immediately:

```java
String name = null;

Optional<String> optionalName = Optional.of(name); // Exception
```

Use `of()` when null would indicate a programming error and you want fail-fast behavior.

### 3.2 `Optional.ofNullable()`

Use it when the input may be null:

```java
String name = null;

Optional<String> optionalName = Optional.ofNullable(name);
```

Result:

```text
Optional.empty
```

With a non-null value:

```java
Optional<String> optionalName =
        Optional.ofNullable("Anita");
```

Result:

```text
Optional[Anita]
```

### 3.3 `Optional.empty()`

Use it to explicitly represent no result:

```java
Optional<User> user = Optional.empty();
```

Example method:

```java
public Optional<User> findUserById(int id) {
    if (id == 101) {
        return Optional.of(new User(101, "Anita"));
    }

    return Optional.empty();
}
```

### Comparison

| Method | Accepts null? | Result for null |
|---|---:|---|
| `Optional.of(value)` | No | Throws `NullPointerException` |
| `Optional.ofNullable(value)` | Yes | `Optional.empty()` |
| `Optional.empty()` | No value supplied | Empty Optional |

### Memory rule

```text
Guaranteed value → of()
Possibly null    → ofNullable()
No value         → empty()
```

---

## 4. Reading and consuming values

### 4.1 `isPresent()`

```java
Optional<String> name = Optional.of("Anita");

if (name.isPresent()) {
    System.out.println(name.get());
}
```

`isPresent()` returns `true` when a value exists.

This works, but it can recreate traditional null-check-style programming:

```java
if (value != null) {
    // use value
}
```

Prefer `ifPresent()`, `map()`, `filter()`, or `orElseThrow()` when they express the requirement more clearly.

### 4.2 `ifPresent()`

It executes a `Consumer` only when the value exists:

```java
Optional<String> name = Optional.of("Anita");

name.ifPresent(value ->
        System.out.println("Name: " + value));
```

Method-reference form:

```java
name.ifPresent(System.out::println);
```

If the Optional is empty, the Consumer does not run.

### 4.3 `get()`

```java
Optional<String> name = Optional.of("Anita");

String result = name.get();
```

This returns `"Anita"`.

But this is unsafe:

```java
Optional<String> name = Optional.empty();

String result = name.get();
```

It throws:

```text
NoSuchElementException
```

Prefer:

```java
String result = name.orElse("Unknown");
```

or:

```java
String result = name.orElseThrow(() ->
        new IllegalStateException("Name is missing"));
```

### Interview point

> Calling `get()` without checking defeats the purpose of Optional. It replaces a possible `NullPointerException` with a possible `NoSuchElementException`.

---

## 5. `orElse()` vs `orElseGet()`

This is one of the most frequently asked Optional interview questions.

### 5.1 `orElse()`

Returns the contained value, or the supplied default value when empty:

```java
Optional<String> name = Optional.empty();

String result = name.orElse("Guest");

System.out.println(result); // Guest
```

When a value exists:

```java
Optional<String> name = Optional.of("Anita");

String result = name.orElse("Guest");

System.out.println(result); // Anita
```

### Eager evaluation

The expression passed to `orElse()` is evaluated even when the Optional contains a value.

```java
static String createDefaultName() {
    System.out.println("Creating default name");
    return "Guest";
}

Optional<String> name = Optional.of("Anita");

String result = name.orElse(createDefaultName());
```

Output includes:

```text
Creating default name
```

The default result is discarded, but the method still executes because Java evaluates method arguments before calling `orElse()`.

### 5.2 `orElseGet()`

It accepts a `Supplier` that runs only if the Optional is empty:

```java
Optional<String> name = Optional.of("Anita");

String result = name.orElseGet(() -> createDefaultName());
```

`createDefaultName()` does not execute because the value exists.

When empty:

```java
Optional<String> name = Optional.empty();

String result = name.orElseGet(() -> createDefaultName());
```

Now the supplier executes and returns `"Guest"`.

### Comparison

| Method | Parameter | Fallback evaluation |
|---|---|---|
| `orElse(value)` | Direct value | Eager: expression always evaluated |
| `orElseGet(supplier)` | `Supplier<T>` | Lazy: supplier runs only when empty |

### Which should you use?

For a simple constant:

```java
String name = optionalName.orElse("Guest");
```

For expensive work or a method call:

```java
User user = optionalUser.orElseGet(() ->
        userService.loadDefaultUser());
```

Use `orElseGet()` when the fallback involves:

- A database query.
- A network call.
- Object construction.
- Expensive computation.
- Side effects.

---

## 6. `orElseThrow()`

Use it when absence should be treated as an error:

```java
User user = findUserById(101)
        .orElseThrow(() ->
                new UserNotFoundException("User 101 not found"));
```

Repository/service example:

```java
public User getRequiredUser(int id) {
    return userRepository.findById(id)
            .orElseThrow(() ->
                    new UserNotFoundException(
                            "User not found: " + id));
}
```

### Java 8 syntax

Java 8 requires an exception supplier:

```java
optional.orElseThrow(() ->
        new IllegalStateException("Value missing"));
```

The no-argument form below was added after Java 8:

```java
optional.orElseThrow(); // Not Java 8
```

---

## 7. `map()`

Use `map()` when:

- A value may exist.
- You want to transform it.
- Your transformation returns a normal value.

### Basic example

```java
Optional<User> optionalUser =
        Optional.of(new User(101, "Anita"));

Optional<String> optionalName =
        optionalUser.map(User::getName);
```

Transformation:

```text
Optional<User>
      ↓ map(User::getName)
Optional<String>
```

If the user exists, `getName()` runs.

If it is empty, the mapping function does not run:

```java
Optional<User> optionalUser = Optional.empty();

Optional<String> optionalName =
        optionalUser.map(User::getName);

System.out.println(optionalName); // Optional.empty
```

### Chained transformations

```java
String name = findUserById(101)
        .map(User::getName)
        .map(String::toUpperCase)
        .orElse("UNKNOWN");
```

Flow:

```text
Optional<User>
    → Optional<String>
    → Optional<uppercase String>
    → final String
```

### `map()` and null transformation results

If the mapping function returns null, `Optional.map()` produces `Optional.empty()`:

```java
Optional<String> optionalName =
        optionalUser.map(User::getName);
```

Conceptually, the mapping result is treated like `Optional.ofNullable(result)`.

However, do not intentionally use null-heavy object models simply because `map()` can absorb a null result.

### Function signature

Conceptually:

```text
T → R
```

Example:

```text
User → String
```

---

## 8. `flatMap()`

Use `flatMap()` when the transformation already returns an `Optional`.

### Domain model

```java
class User {
    private Address address;

    public Optional<Address> getAddress() {
        return Optional.ofNullable(address);
    }
}
```

### Problem with `map()`

```java
Optional<Optional<Address>> address =
        optionalUser.map(User::getAddress);
```

`getAddress()` already returns `Optional<Address>`, so `map()` wraps that Optional again:

```text
Optional<Optional<Address>>
```

### Solution with `flatMap()`

```java
Optional<Address> address =
        optionalUser.flatMap(User::getAddress);
```

`flatMap()` removes the unnecessary additional wrapper.

### Nested navigation example

```java
Optional<String> city =
        findUserById(101)
                .flatMap(User::getAddress)
                .flatMap(Address::getCity)
                .map(String::toUpperCase);
```

Flow:

```text
Optional<User>
    ↓ flatMap
Optional<Address>
    ↓ flatMap
Optional<String>
    ↓ map
Optional<uppercase String>
```

If the user, address, or city is absent, the remaining functions are skipped and the result stays empty.

### `map()` vs `flatMap()`

```text
Function returns normal value → map()
Function returns Optional     → flatMap()
```

Conceptually:

```text
map:
T → R

flatMap:
T → Optional<R>
```

### Connection with CompletableFuture

```text
Optional.map()      ≈ CompletableFuture.thenApply()
Optional.flatMap()  ≈ CompletableFuture.thenCompose()
```

Both `flatMap()` and `thenCompose()` prevent nested containers.

---

## 9. `filter()`

Use `filter()` to keep a value only when it satisfies a condition.

```java
Optional<User> adultUser =
        optionalUser.filter(user -> user.getAge() >= 18);
```

Behavior:

```text
User exists and age >= 18 → Optional<User>
User exists but age < 18  → Optional.empty
User does not exist       → Optional.empty
```

### Practical example

```java
String email = findUserById(101)
        .filter(User::isActive)
        .map(User::getEmail)
        .orElse("No active user email");
```

Only an active user's email is returned.

### Multiple filters

```java
String result = findUserById(101)
        .filter(User::isActive)
        .filter(user -> user.getAge() >= 18)
        .map(User::getName)
        .orElse("No active adult user");
```

### String validation example

```java
Optional<String> validName =
        Optional.of("Anita")
                .filter(name -> !name.trim().isEmpty());
```

---

## 10. Complete runnable example

Save as `OptionalDemo.java`.

```java
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

public class OptionalDemo {

    static class Address {
        private final String city;

        Address(String city) {
            this.city = city;
        }

        Optional<String> getCity() {
            return Optional.ofNullable(city);
        }
    }

    static class User {
        private final int id;
        private final String name;
        private final int age;
        private final boolean active;
        private final Address address;

        User(
                int id,
                String name,
                int age,
                boolean active,
                Address address) {
            this.id = id;
            this.name = name;
            this.age = age;
            this.active = active;
            this.address = address;
        }

        int getId() {
            return id;
        }

        String getName() {
            return name;
        }

        int getAge() {
            return age;
        }

        boolean isActive() {
            return active;
        }

        Optional<Address> getAddress() {
            return Optional.ofNullable(address);
        }
    }

    private static final List<User> USERS = Arrays.asList(
            new User(
                    101,
                    "Anita",
                    28,
                    true,
                    new Address("Bengaluru")),
            new User(
                    102,
                    "Rahul",
                    16,
                    false,
                    null)
    );

    static Optional<User> findUserById(int id) {
        return USERS.stream()
                .filter(user -> user.getId() == id)
                .findFirst();
    }

    public static void main(String[] args) {

        // 1. Get the value or throw an exception.
        User user = findUserById(101)
                .orElseThrow(() ->
                        new IllegalArgumentException("User not found"));

        System.out.println(user.getName());

        // 2. Transform User into String.
        String name = findUserById(101)
                .map(User::getName)
                .orElse("Unknown");

        System.out.println(name);

        // 3. Keep only an active adult user.
        String activeAdult = findUserById(101)
                .filter(User::isActive)
                .filter(foundUser -> foundUser.getAge() >= 18)
                .map(User::getName)
                .orElse("No active adult user");

        System.out.println(activeAdult);

        // 4. Navigate nested Optional values.
        String city = findUserById(101)
                .flatMap(User::getAddress)
                .flatMap(Address::getCity)
                .orElse("City unavailable");

        System.out.println(city);

        // 5. Missing user produces a fallback.
        String missingUser = findUserById(999)
                .map(User::getName)
                .orElse("User not found");

        System.out.println(missingUser);

        // 6. Execute action only when a value exists.
        findUserById(101)
                .ifPresent(foundUser ->
                        System.out.println(
                                "Found: " + foundUser.getName()));
    }
}
```

Compile and run:

```bash
javac OptionalDemo.java
java OptionalDemo
```

Expected output:

```text
Anita
Anita
Anita
Bengaluru
User not found
Found: Anita
```

---

## 11. Where to use Optional

The primary use is as a return type when a method may legitimately produce no result:

```java
Optional<User> findById(int id);

Optional<Order> findLatestOrder(int customerId);

Optional<String> findMiddleName(int userId);
```

### Repository example

```java
public interface UserRepository {
    Optional<User> findByEmail(String email);
}
```

### Service example

```java
public User getRequiredUser(String email) {
    return userRepository.findByEmail(email)
            .orElseThrow(() ->
                    new UserNotFoundException(email));
}
```

### When empty is normal

```java
public Optional<Discount> findApplicableDiscount(Order order) {
    // An order may legitimately have no discount.
}
```

Returning Optional is useful because absence is an expected result, not necessarily an exceptional condition.

---

## 12. Where to avoid Optional

### 12.1 Usually avoid it as a method parameter

Avoid:

```java
public void createUser(
        String name,
        Optional<String> middleName) {
}
```

It creates multiple representations of absence:

```java
createUser("Anita", Optional.empty());
createUser("Anita", null); // Still technically possible
```

Prefer a request object, overload, or a clearly validated nullable field depending on the API design.

### 12.2 Usually avoid it as a JPA entity field

Avoid:

```java
@Entity
class User {
    private Optional<String> middleName;
}
```

Map the actual database value:

```java
@Entity
class User {
    private String middleName;

    public Optional<String> getMiddleName() {
        return Optional.ofNullable(middleName);
    }
}
```

### 12.3 Be careful with DTO fields

```java
class UserResponse {
    private Optional<String> middleName;
}
```

Serialization frameworks may require additional support and the resulting JSON contract may be unclear. A nullable field or explicit presence model may be better.

### 12.4 Do not return null from an Optional-returning method

Wrong:

```java
public Optional<User> findUser(int id) {
    return null;
}
```

Correct:

```java
public Optional<User> findUser(int id) {
    return Optional.empty();
}
```

The Optional reference itself should not be null.

### 12.5 Usually avoid `Optional<List<T>>`

Prefer:

```java
List<Order> findOrders();
```

Return an empty list when no orders exist:

```java
return Collections.emptyList();
```

Usually avoid:

```java
Optional<List<Order>> findOrders();
```

An empty collection already represents “no elements.” Use `Optional<List<T>>` only if “not calculated/not available” and “calculated but empty” have genuinely different business meanings.

### 12.6 Do not use Optional everywhere

Optional is not a replacement for:

- Input validation.
- Required constructor fields.
- Empty collections.
- Error handling.
- Domain-specific result types.

---

## 13. Primitive Optional types

Java 8 provides specialized Optional classes:

```java
OptionalInt
OptionalLong
OptionalDouble
```

Example:

```java
OptionalInt maximum =
        java.util.stream.IntStream
                .of(10, 20, 30)
                .max();

int result = maximum.orElse(0);
```

These avoid boxing primitive values into wrapper objects such as `Integer`, `Long`, or `Double`.

```text
Optional<Integer> → boxed integer
OptionalInt       → primitive-specialized container
```

---

## 14. Java 8 limitations

These methods are not available in Java 8:

```java
optional.isEmpty();          // Added in Java 11
optional.ifPresentOrElse();  // Added in Java 9
optional.or(...);            // Added in Java 9
optional.stream();           // Added in Java 9
optional.orElseThrow();      // No-argument version added after Java 8
```

### Java 8 alternatives

Instead of `isEmpty()`:

```java
if (!optional.isPresent()) {
    // Empty
}
```

Instead of `ifPresentOrElse()`:

```java
if (optional.isPresent()) {
    System.out.println(optional.get());
} else {
    System.out.println("Value missing");
}
```

Instead of no-argument `orElseThrow()`:

```java
optional.orElseThrow(() ->
        new IllegalStateException("Value missing"));
```

---

## 15. Common mistakes

### 1. Calling `get()` without checking

```java
User user = optionalUser.get();
```

Prefer:

```java
User user = optionalUser.orElseThrow(() ->
        new UserNotFoundException());
```

### 2. Using `orElse()` for expensive work

```java
User user = optionalUser.orElse(
        loadDefaultUserFromDatabase());
```

The database method executes even when the user exists.

Prefer:

```java
User user = optionalUser.orElseGet(
        () -> loadDefaultUserFromDatabase());
```

### 3. Creating nested Optional values

```java
Optional<Optional<Address>> address =
        optionalUser.map(User::getAddress);
```

Prefer:

```java
Optional<Address> address =
        optionalUser.flatMap(User::getAddress);
```

### 4. Using only `isPresent()` and `get()`

Verbose style:

```java
if (optionalUser.isPresent()) {
    return optionalUser.get().getName();
}
return "Unknown";
```

Prefer:

```java
return optionalUser
        .map(User::getName)
        .orElse("Unknown");
```

### 5. Returning null instead of `Optional.empty()`

```java
return null; // Wrong for Optional-returning method
```

Use:

```java
return Optional.empty();
```

### 6. Using Optional as every field or parameter

Optional is mainly intended to communicate optional method results. It is not automatically the best type for every field, parameter, DTO, or entity property.

### 7. Wrapping an Optional again

Wrong:

```java
Optional<Optional<User>> result =
        Optional.of(findUserById(101));
```

The method already returns `Optional<User>`, so use it directly.

### 8. Using Optional instead of validation

This is still invalid input:

```java
Optional.ofNullable(request.getEmail())
        .orElse("unknown@example.com");
```

If email is required, validate and reject the request instead of silently creating a fake value.

### 9. Treating Optional as an exception replacement

Use Optional when absence is a meaningful possibility. Use an exception when an operation failed unexpectedly or a required invariant was violated.

### 10. Assigning null to Optional

Wrong:

```java
Optional<User> user = null;
```

Correct:

```java
Optional<User> user = Optional.empty();
```

---

## 16. Interview questions

### 1. What is Optional?

`Optional` is a Java 8 container that either contains one non-null value or is empty. It is mainly used as a return type to represent an absent result explicitly.

### 2. Why was Optional introduced?

It makes the possibility of no result visible in the method signature and provides operations for transforming, filtering, consuming, defaulting, or rejecting an absent value.

### 3. `of()` vs `ofNullable()`?

```text
of(value)         → value must not be null
ofNullable(value) → null becomes Optional.empty()
```

### 4. `orElse()` vs `orElseGet()`?

```text
orElse()    → fallback expression is evaluated eagerly
orElseGet() → fallback supplier executes only when empty
```

### 5. `map()` vs `flatMap()`?

```text
map()     → mapping function returns a normal value
flatMap() → mapping function returns Optional
```

`flatMap()` avoids `Optional<Optional<T>>`.

### 6. Why should `get()` generally be avoided?

It throws `NoSuchElementException` when empty and does not express how absence should be handled.

### 7. Can Optional contain null?

No. It contains a non-null value or is empty.

### 8. Can an Optional variable itself be null?

Technically any reference can be null, but an Optional reference should never be null. Use `Optional.empty()`.

### 9. Should Optional be used as a method parameter?

Usually no. It creates additional calling complexity and still permits passing a null Optional reference. It is mainly designed for return types.

### 10. Should a method return `Optional<List<T>>`?

Usually no. Return an empty list to mean no elements. Use Optional only if absence and an existing-but-empty collection have distinct business meanings.

### 11. Is Optional serializable?

`java.util.Optional` does not implement `Serializable`. Do not treat it as a general-purpose serializable entity or DTO field.

### 12. How does `map()` handle a null mapping result?

It produces `Optional.empty()`.

### 13. What does `filter()` do?

It keeps the contained value when the predicate is true; otherwise, it returns an empty Optional.

### 14. What happens if the mapping function is not called because Optional is empty?

The result remains empty and subsequent transformations are skipped until a terminal/default operation handles absence.

### 15. Optional vs exception?

- Optional: no value is an expected possible result.
- Exception: an operation failed or a required condition was violated.

### 16. Why use `OptionalInt`?

It represents an optional primitive integer without boxing it into `Integer`.

### Coding question: safely obtain a city

```java
String city = findUserById(id)
        .flatMap(User::getAddress)
        .flatMap(Address::getCity)
        .orElse("City unavailable");
```

### Coding question: return active adult user's name

```java
String name = findUserById(id)
        .filter(User::isActive)
        .filter(user -> user.getAge() >= 18)
        .map(User::getName)
        .orElseThrow(() ->
                new IllegalStateException(
                        "Active adult user not found"));
```

---

## 17. Quick revision sheet

### Method map

```text
Optional.of(value)
→ value must be non-null

Optional.ofNullable(value)
→ null becomes empty

Optional.empty()
→ no value

isPresent()
→ check whether value exists

ifPresent(consumer)
→ execute action when present

map(function)
→ transform T into R

flatMap(function)
→ transform T into Optional<R>

filter(predicate)
→ keep value only if condition passes

orElse(value)
→ eager fallback

orElseGet(supplier)
→ lazy fallback

orElseThrow(exceptionSupplier)
→ throw when empty
```

### Decision map

```text
Need to transform a present value?
    → map()

Transformation already returns Optional?
    → flatMap()

Need to validate/keep value conditionally?
    → filter()

Need a simple constant default?
    → orElse()

Need an expensive or lazy default?
    → orElseGet()

Absence should be an error?
    → orElseThrow()

Need to perform an action only when present?
    → ifPresent()
```

### Most important comparison

| Requirement | Method |
|---|---|
| Create with guaranteed non-null value | `of()` |
| Create from possibly null value | `ofNullable()` |
| Transform contained value | `map()` |
| Chain method returning Optional | `flatMap()` |
| Keep value based on condition | `filter()` |
| Simple eager fallback | `orElse()` |
| Lazy fallback | `orElseGet()` |
| Throw when absent | `orElseThrow()` |
| Execute Consumer when present | `ifPresent()` |

### Golden rules

1. Use Optional mainly as a method return type.
2. Never return null from an Optional-returning method.
3. Avoid calling `get()` without a presence guarantee.
4. Use `of()` only for guaranteed non-null values.
5. Use `ofNullable()` when converting a possibly null value.
6. Use `map()` for normal return values.
7. Use `flatMap()` for functions already returning Optional.
8. Use `orElseGet()` for expensive or side-effecting fallbacks.
9. Usually return empty collections instead of `Optional<List<T>>`.
10. Avoid Optional fields in JPA entities and use caution in DTOs.

### Interview-ready answer

> `Optional` represents a value that may be absent. I mainly use it as a return type. I use `map` when transforming the contained value, `flatMap` when the transformation already returns an Optional, `filter` for conditional presence, `orElseGet` for lazy fallbacks, and `orElseThrow` when absence is exceptional. I avoid unsafe `get()`, returning null from Optional methods, and generally avoid using Optional for entity fields, method parameters, and collections.

### Self-test

1. Why does `Optional.of(null)` fail?
2. When should you use `ofNullable()`?
3. Why can `orElse()` perform unnecessary work?
4. When should you use `orElseGet()`?
5. What is the difference between `map()` and `flatMap()`?
6. Why is `Optional<Optional<T>>` usually a design warning?
7. What does `filter()` return when its condition is false?
8. Why should an Optional-returning method never return null?
9. Why is an empty list usually better than `Optional<List<T>>`?
10. Which useful Optional methods are unavailable in Java 8?

