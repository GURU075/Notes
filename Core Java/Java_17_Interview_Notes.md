# Java 17 — Complete Interview Revision Notes

> **Audience:** Java / Spring Boot backend developers (around 3 years' experience)  
> **Scope:** Seven high-value features available in Java 17. Not all originated in Java 17.  
> **Learning pattern:** Problem → Syntax → Production usage → Pitfalls → Interview answer.

## Contents

1. [Records](#1-records)
2. [Sealed Classes and Interfaces](#2-sealed-classes-and-interfaces)
3. [Pattern Matching for `instanceof`](#3-pattern-matching-for-instanceof)
4. [Switch Expressions](#4-switch-expressions)
5. [Text Blocks](#5-text-blocks)
6. [`Stream.toList()`](#6-streamtolist)
7. [Helpful NullPointerExceptions](#7-helpful-nullpointerexceptions)
8. [Combined Interview Questions](#8-combined-interview-questions)
9. [Final Cheat Sheet](#9-final-cheat-sheet)

## Version timeline

| Feature | Finalized / introduced | Available in Java 17? |
|---|---|---|
| Records | Java 16 (preview since 14) | Yes |
| Sealed classes | Java 17 (preview since 15) | Yes |
| Pattern matching for `instanceof` | Java 16 (preview since 14) | Yes |
| Switch expressions | Java 14 (preview since 12) | Yes |
| Text blocks | Java 15 (preview since 13) | Yes |
| `Stream.toList()` | Java 16 | Yes |
| Helpful NullPointerExceptions | Java 14 (enabled by default since 15) | Yes |

---

# 1. Records

## Definition and motivation

A **record** is a concise declaration of a class intended primarily to carry data. Java supplies the canonical constructor, component accessors, `equals`, `hashCode`, and `toString`.

**One-line interview answer:** *A record is a special data-carrier class that reduces boilerplate by automatically providing a constructor, component accessors, equality, hashing and string representation.*

### Before: regular class

```java
public final class Employee {
    private final int id;
    private final String name;

    public Employee(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public int getId() { return id; }
    public String getName() { return name; }
    // Need to implement equals, hashCode and toString as well.
}
```

### After: record

```java
public record Employee(int id, String name) {}

Employee e = new Employee(101, "Rahul");
System.out.println(e.id());   // 101
System.out.println(e.name()); // Rahul
System.out.println(e);        // Employee[id=101, name=Rahul]
```

The accessors are `id()` and `name()`, **not** `getId()` and `getName()`.

## Shallow versus deep immutability

Record component fields are private and final, but referenced objects may be mutable.

```java
record Team(String name, List<String> members) {}

List<String> source = new ArrayList<>(List.of("Amit"));
Team team = new Team("Backend", source);
team.members().add("Sneha"); // Allowed: List itself is mutable.
```

To avoid mutations through this list reference and mutations to the original input list:

```java
record Team(String name, List<String> members) {
    public Team {
        members = List.copyOf(members);
    }
}
```

`List.copyOf` produces an unmodifiable **shallow** snapshot and rejects null elements. Nested mutable elements may still change.

## Compact constructor and validation

```java
public record Employee(int id, String name, double salary) {
    public Employee {
        if (id <= 0) throw new IllegalArgumentException("Invalid ID");
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name is required");
        }
        if (salary < 0) throw new IllegalArgumentException("Negative salary");
    }
}
```

In a compact constructor, Java implicitly assigns component parameters to fields after the body completes successfully. You can validate or normalize parameters, but cannot explicitly assign the component fields in the compact constructor.

## Spring Boot request/response DTO

```java
public record OrderResponse(Long orderId, String productName, double amount) {}

@RestController
@RequestMapping("/orders")
class OrderController {
    @GetMapping("/{id}")
    OrderResponse getOrder(@PathVariable Long id) {
        return new OrderResponse(id, "Laptop", 55000);
    }
}
```

With a compatible Jackson setup, the response serializes to:

```json
{"orderId":101,"productName":"Laptop","amount":55000.0}
```

Validation annotations on record components can also be used for request DTOs with a compatible Bean Validation / Spring configuration.

## Important limitations

- Records are implicitly `final` and extend `java.lang.Record`; they **cannot extend** another class.
- Records **can implement interfaces**.
- They may declare static fields, custom methods and additional constructors, but no additional instance fields.
- No setters for record component fields; record accessors return the component value/reference.
- A regular JPA `@Entity` generally cannot be a record: standard JPA entities require a non-final entity class and an appropriate no-arg constructor. Records work well as DTOs / projections in supported frameworks.
- Records are not guaranteed to be deeply immutable.

## Five common interview Q&As

**Q1. Why were records introduced?** To remove boilerplate from simple data-carrier classes.

**Q2. Are records immutable?** Component fields are final; records are shallowly immutable, not necessarily deeply immutable.

**Q3. Can records extend classes?** No. They implicitly extend `Record`; they can implement interfaces.

**Q4. Can records have methods or constructors?** Yes, including compact constructors, methods and static members.

**Q5. Where would you use a record in Spring Boot?** Request/response DTOs, read models, and other simple data transfers; not standard JPA entity classes.

---

# 2. Sealed Classes and Interfaces

## Definition and motivation

A **sealed class/interface** restricts which classes may directly extend it or which classes/interfaces may directly implement/extend it. Java 17 finalized this feature.

**One-line interview answer:** *Sealed types provide controlled inheritance by explicitly limiting the permitted direct subclasses or implementors.*

### Without sealed

```java
abstract class Payment {
    abstract void pay(double amount);
}
// Any accessible non-final subclass may extend Payment.
```

### With sealed

```java
public abstract sealed class Payment
        permits UpiPayment, CardPayment {
    public abstract void pay(double amount);
}

final class UpiPayment extends Payment {
    @Override public void pay(double amount) {
        System.out.println("UPI: " + amount);
    }
}

final class CardPayment extends Payment {
    @Override public void pay(double amount) {
        System.out.println("Card: " + amount);
    }
}

// class CryptoPayment extends Payment {}  // Compilation error: not permitted.
```

## `sealed`, `permits`, `final`, `non-sealed`

- `sealed`: limit direct subclasses.
- `permits`: name the direct allowed subtypes.
- `final`: stop inheritance after that class.
- `non-sealed`: open inheritance again beneath a permitted subtype.

```java
sealed class Vehicle permits Car, Bike {}
final class Car extends Vehicle {}
non-sealed class Bike extends Vehicle {}
class SportsBike extends Bike {} // Valid.
```

A permitted subclass can also be `sealed` and permit another restricted set.

## Sealed class vs sealed abstract class vs sealed interface

```java
sealed class Base permits ConcreteChild {}
final class ConcreteChild extends Base {}

abstract sealed class AbstractPayment permits Upipay {}
final class Upipay extends AbstractPayment {}

sealed interface PaymentResult permits Success, Failed {}
record Success(String transactionId) implements PaymentResult {}
record Failed(String reason) implements PaymentResult {}
```

- A concrete sealed class can be instantiated unless other access restrictions apply.
- An abstract sealed class cannot be instantiated and may declare abstract methods.
- A sealed interface can permit implementing classes (including records) and extending interfaces.

## Microservices example: controlled results

```java
sealed interface PaymentResult permits Success, Failed, Pending {}
record Success(String transactionId) implements PaymentResult {}
record Failed(String message) implements PaymentResult {}
record Pending(String referenceId) implements PaymentResult {}

PaymentResult process(boolean approved) {
    return approved ? new Success("TXN-101") : new Failed("Declined");
}
```

This explicitly expresses the allowed result types. It does not replace business validation, persistence, network failure handling, or tests.

## Rules and pitfalls

- Each permitted direct subclass must be `final`, `sealed`, or `non-sealed`; records are implicitly final.
- Permitted types must directly extend or implement their sealed parent.
- Permitted types belong to the same **named module**, or, in the unnamed module, the same **package**.
- If all directly permitted types are in the same compilation unit, `permits` can sometimes be inferred.
- A sealed class may also be `abstract`; it cannot also be `final` or `non-sealed`.
- `sealed` controls the subtype graph, **not immutability**.

## Five common interview Q&As

**Q1. Why use a sealed class?** To allow a known, restricted set of subtypes instead of open inheritance.

**Q2. How is `sealed` different from `final`?** `final` bans subclassing entirely; `sealed` permits only named direct subtypes.

**Q3. Can sealed be applied to interfaces or abstract classes?** Yes, both.

**Q4. What does `non-sealed` do?** Allows unrestricted subclassing beneath that permitted class/interface (subject to normal Java access rules).

**Q5. Production example?** Model controlled outcome types such as `Success`, `Failed`, and `Pending` for a service operation.

---

# 3. Pattern Matching for `instanceof`

## Definition and motivation

Combines runtime type checking and a typed pattern-variable binding, eliminating a redundant manual cast.

**One-line interview answer:** *Pattern matching for `instanceof` checks a type and binds a typed variable on a successful match, making code shorter and safer.*

### Before

```java
Object obj = "Spring Boot";
if (obj instanceof String) {
    String value = (String) obj;
    System.out.println(value.length());
}
```

### After

```java
Object obj = "Spring Boot";
if (obj instanceof String value) {
    System.out.println(value.length());
}
```

The pattern variable exists only where the compiler can prove the match succeeded (**flow-sensitive scoping**).

## Scope with logical operators

```java
Object obj = "Java 17";
if (obj instanceof String s && s.length() > 4) {
    System.out.println(s.toUpperCase());
}
```

Right side of `&&` is evaluated only if the match succeeded.

```java
// DOES NOT COMPILE: 's' may not exist when the right side is evaluated.
// if (obj instanceof String s || s.length() > 4) { }
```

Useful early return:

```java
void printLength(Object value) {
    if (!(value instanceof String s)) {
        return;
    }
    System.out.println(s.length()); // s is definitely bound here.
}
```

A `null` reference never matches an `instanceof` type pattern.

## Backend use case

```java
void logResult(Object result) {
    if (result instanceof String message) {
        System.out.println("Message: " + message);
    } else if (result instanceof Integer code) {
        System.out.println("Code: " + code);
    }
}
```

You might similarly inspect exception subclasses in a shared handler, although separate specific Spring `@ExceptionHandler` methods are often clearer. Prefer ordinary polymorphism when subtype-dependent behavior naturally belongs inside the subtype.

## Five common interview Q&As

**Q1. What changed?** We no longer need an explicit cast after a successful `instanceof` pattern match.

**Q2. Which Java version finalized it?** Java 16; usable in Java 17.

**Q3. What is flow-sensitive scoping?** The variable is accessible only in paths where its match is guaranteed.

**Q4. What happens for `null`?** Match fails (`false`).

**Q5. Does it replace polymorphism?** No. It is useful for explicit type inspection; polymorphism often gives better extensibility for subtype behavior.

---

# 4. Switch Expressions

## Definition and motivation

Switch expressions produce values directly. Arrow case rules avoid accidental fall-through, and `yield` allows a multi-statement case to supply its result.

**One-line interview answer:** *Switch expressions return values directly, support concise arrow rules, and use `yield` to produce values from multi-statement branches.*

### Traditional switch

```java
int day = 2;
String name;
switch (day) {
    case 1:
        name = "Monday";
        break;
    case 2:
        name = "Tuesday";
        break;
    default:
        name = "Unknown";
}
```

Without a `break`, old colon-style cases may fall through.

### Switch expression

```java
int day = 2;
String name = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    default -> "Unknown";
};
```

A group of case labels can share a branch:

```java
String kind = switch (dayName) {
    case "SATURDAY", "SUNDAY" -> "Weekend";
    default -> "Weekday";
};
```

## `yield` versus `break` versus `return`

```java
int marks = 85;
String grade = switch (marks / 10) {
    case 9, 10 -> "A";
    case 8 -> {
        System.out.println("Very good");
        yield "B";
    }
    default -> "C";
};
```

- `yield`: supplies a value from a switch **expression** branch.
- `break`: exits a traditional switch **statement** (not the way to yield a result).
- `return`: exits the enclosing method.

Arrow syntax is available in **switch statements as well**, even when no value is returned.

## Exhaustiveness and enums

A switch expression must cover all possible cases.

```java
enum OrderStatus { CREATED, CONFIRMED, SHIPPED, DELIVERED }

String describe(OrderStatus status) {
    return switch (status) {
        case CREATED -> "Created";
        case CONFIRMED -> "Confirmed";
        case SHIPPED -> "On the way";
        case DELIVERED -> "Delivered";
    };
}
```

For an enum, covering every constant suffices without a `default`; adding a new constant will require reconsideration of the expression when recompiled.

For an `int` selector, usually include `default`:

```java
String label = switch (httpCode) {
    case 200 -> "OK";
    case 404 -> "Not found";
    default -> "Other";
};
```

In standard Java 17 syntax, switching on a null reference throws `NullPointerException` even with `default`. `case null` belongs to pattern-matching switch support (preview in Java 17), not the normal finalized Java 17 switch expressions taught here.

## Five common interview Q&As

**Q1. Difference between switch statement and expression?** A statement executes actions; an expression produces a value.

**Q2. Why use arrow rules?** Cleaner syntax and no fall-through between arrow cases.

**Q3. Why use `yield`?** To provide an expression result after multiple statements in one case block.

**Q4. Is `default` always required?** No; an enum switch expression can list every constant. But every switch expression must be exhaustive.

**Q5. Can switch expressions use strings/enums?** Yes. They are especially useful for status and enum mappings.

---

# 5. Text Blocks

## Definition and motivation

Text blocks use triple quotes (`"""`) to represent multiline `String` values naturally. Finalized in Java 15.

**One-line interview answer:** *Text blocks simplify multiline JSON, SQL, HTML and similar strings, avoiding noisy `\n` escapes, quote escaping, and concatenation.*

### Before

```java
String json = "{\n" +
              "  \"name\": \"Rahul\",\n" +
              "  \"city\": \"Pune\"\n" +
              "}";
```

### After

```java
String json = """
        {
          "name": "Rahul",
          "city": "Pune"
        }
        """;
```

The opening delimiter needs a line break after it; indentation is normalized to remove incidental spaces. A text block is an ordinary `String`, not a special object type.

## Spring Data JPA native query example

```java
@Query(value = """
        SELECT *
        FROM employees
        WHERE department = :department
          AND salary > :salary
        """, nativeQuery = true)
List<Employee> findEmployees(
        @Param("department") String department,
        @Param("salary") double salary);
```

Text blocks **do not** validate SQL or JSON, and do not prevent SQL injection. Continue to bind parameters. For dynamic SQL, never concatenate untrusted input directly into SQL commands.

## Formatting text blocks

```java
String template = """
        Hello %s,
        Your order number is %d.
        """;
String message = template.formatted("Rahul", 101);
```

The backslash escape `\` and newline behavior still follow Java string rules where applicable. Text blocks are not a replacement for template engines when the content is complex.

## Five common interview Q&As

**Q1. What are text blocks?** A syntax for expressing multiline `String` values using `"""`.

**Q2. When finalized?** Java 15; supported in Java 17.

**Q3. Are they a new datatype?** No, they produce normal `String` instances.

**Q4. What practical uses?** JSON samples, native SQL queries, HTML fragments, email text.

**Q5. Do they prevent SQL injection?** No; parameter binding and validation are still essential.

---

# 6. Stream.toList()

## Definition and motivation

Introduced in Java 16, `Stream.toList()` is a concise terminal operation returning an **unmodifiable list** of stream elements.

**One-line interview answer:** *`Stream.toList()` turns a Stream into an unmodifiable List without calling `collect`, unlike `Collectors.toList()` whose returned List mutability is unspecified.*

### Before (Java 8/11)

```java
List<String> names = List.of("Amit", "Rahul", "Anita");
List<String> selected = names.stream()
        .filter(name -> name.startsWith("A"))
        .collect(Collectors.toList());
```

### With Java 17

```java
List<String> selected = names.stream()
        .filter(name -> name.startsWith("A"))
        .toList();
// [Amit, Anita]
```

## Mutability differences

```java
List<String> result = Stream.of("Java", "Spring").toList();
result.add("Docker"); // UnsupportedOperationException
```

`Collectors.toList()` **does not promise** a modifiable list even though common JDK implementations return an `ArrayList`.

If a mutable list is required:

```java
List<String> mutable = Stream.of("Java", "Spring")
        .collect(Collectors.toCollection(ArrayList::new));
mutable.add("Docker"); // Works.
```

Or copy into a new mutable list:

```java
List<String> mutable = new ArrayList<>(stream.toList());
```

## Null behavior

```java
List<String> a = Stream.of("Java", null).toList();
// Permitted: contains null.

List<String> b = Stream.of("Java", null)
        .collect(Collectors.toUnmodifiableList());
// Throws NullPointerException.
```

`Collectors.toUnmodifiableList()` also returns an unmodifiable list, but rejects null values. `Collectors.toList()` makes no general mutability/type guarantee.

## Spring Boot mapping to DTOs

```java
public List<UserResponse> getActiveUsers() {
    return userRepository.findActiveUsers().stream()
            .map(user -> new UserResponse(user.getId(), user.getName()))
            .toList();
}
```

Prefer filtering in the database (`findActiveUsers()` here) rather than fetching every user and applying a stream filter for large datasets.

## Five common interview Q&As

**Q1. Which Java version introduced `Stream.toList()`?** Java 16.

**Q2. Is its result modifiable?** No; modifying it throws `UnsupportedOperationException`.

**Q3. Is `Collectors.toList()` guaranteed mutable?** No; its mutability is not specified.

**Q4. Can `Stream.toList()` contain null elements?** Yes.

**Q5. How to guarantee a mutable List?** `collect(Collectors.toCollection(ArrayList::new))`, or copy to `new ArrayList<>(...)`.

---

# 7. Helpful NullPointerExceptions

## Definition and motivation

Introduced in Java 14 and enabled by default since Java 15, this JVM improvement makes `NullPointerException` messages identify the operation and the null reference more precisely.

**One-line interview answer:** *Helpful NPEs improve debugging by identifying the specific variable, field, or method result that was null during a failed operation.*

### Example

```java
class Address { String city; }
class Employee { Address address; }

Employee emp = new Employee();
System.out.println(emp.address.city);
```

Older exception output might contain only the type and stack-trace line:

```text
java.lang.NullPointerException
    at Main.main(Main.java:10)
```

Java 17 can provide:

```text
Cannot read field "city" because "emp.address" is null
```

Another example:

```java
String name = null;
System.out.println(name.length());
// Cannot invoke "String.length()" because "name" is null
```

## Spring Boot example

```java
String city = order.getCustomer().getAddress().getCity();
```

When an intermediate method returns null, the detailed error may indicate which method result was null. Exact wording varies depending on code and debug information.

This is a **diagnostic** improvement, not automatic null prevention.

## JVM flag

Helpful messages are normally on by default in Java 17:

```bash
-XX:+ShowCodeDetailsInExceptionMessages
```

Disable them with:

```bash
-XX:-ShowCodeDetailsInExceptionMessages
```

Keep in mind some thrown NPEs or missing metadata may not have detailed messages. Do not expose raw internal error messages to external API clients; log appropriately and return safe error responses.

## Five common interview Q&As

**Q1. What changed?** Error messages can identify precisely which dereference failed.

**Q2. Which Java release introduced them?** Java 14, enabled by default since Java 15.

**Q3. Is new source syntax needed?** No, it is a JVM diagnostic feature.

**Q4. Does it prevent NPEs?** No, it only improves troubleshooting.

**Q5. Are details always available?** Not necessarily; message detail depends on how the exception arose and available JVM information.

---

# 8. Combined Interview Questions

1. **Why records over Lombok DTOs?** Records are language-supported, concise, and shallowly immutable with value-based equality; Lombok can generate flexible mutable or immutable classes and methods through annotation processing. Choose based on data and framework needs.
2. **Records versus sealed classes?** Records focus on representing data; sealed types focus on restricting inheritance. Records can implement sealed interfaces.
3. **Can a record implement a sealed interface?** Yes, if permitted. It is implicitly final.
4. **`instanceof` pattern matching versus `switch` pattern matching?** The former is finalized in Java 16. Pattern matching for `switch` was only preview functionality in Java 17, so do not present it as a standard finalized Java 17 feature.
5. **Does a switch expression need `default`?** It must be exhaustive. Exhaustively listing enum constants may eliminate the need for `default`.
6. **Does `yield` exit a Java method?** No. It provides the value of the switch expression branch.
7. **Does `Stream.toList()` return `ArrayList`?** No implementation type is guaranteed; rely only on its List interface and unmodifiable behavior.
8. **Difference between `Stream.toList()` and `List.of()`?** Both create unmodifiable lists, but `Stream.toList()` accepts null stream elements; `List.of()` rejects null arguments.
9. **Does a text block prevent injection?** No. It only improves string readability.
10. **Do helpful NPEs replace defensive coding?** No. Continue validation and correct null handling.

---

# 9. Final Cheat Sheet

| Feature | Interview keyword | Practical backend usage | Remember |
|---|---|---|---|
| Records | Data carriers, boilerplate | REST DTOs | Shallowly immutable; `name()` accessor |
| Sealed classes/interfaces | Controlled inheritance | PaymentResult variants | `sealed`, `permits`, `final`, `non-sealed` |
| Pattern `instanceof` | Type checking + binding | Handling typed variants | Flow-sensitive scope |
| Switch expressions | Return value, `yield` | Enum/status mapping | Exhaustive; arrow cases do not fall through |
| Text blocks | `"""` multiline strings | Native SQL / JSON samples | Just `String`; not SQL validation |
| `Stream.toList()` | Unmodifiable List | DTO mapping | Not the same contract as `Collectors.toList()` |
| Helpful NPEs | Precise null diagnostics | Debugging | Not NPE prevention |

## Suggested spoken answer: “What Java 17 features have you used?”

> “I have worked with features available in Java 17 such as Records for lightweight response DTOs, switch expressions for clean status mappings, pattern matching for `instanceof` to avoid manual casting, and `Stream.toList()` for concise stream processing. I also understand sealed types for controlled inheritance, text blocks for readable SQL or JSON, and helpful NullPointerExceptions for debugging. Some of these features were finalized before Java 17 but are part of the Java 17 language and JDK environment.”

**Interview practice:** For each feature, explain the old problem, show the new syntax, give one Spring Boot example, and mention one caveat.
