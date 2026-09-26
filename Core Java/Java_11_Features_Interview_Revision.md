# Java 11 Features — Interview Revision Notes

## 1. Overview

Java 11 introduced several improvements that made common Java tasks simpler, cleaner, and more suitable for modern backend applications.

Important Java 11 features for interviews:

1. New `String` methods
2. `Optional.isEmpty()`
3. `Files.readString()` and `Files.writeString()`
4. `Predicate.not()`
5. `var` in lambda parameters
6. Standard `HttpClient`
7. Direct execution of Java source files
8. Removal of some Java EE modules
9. JVM and GC improvements
10. Java Flight Recorder

---

# 2. New String Methods

Java 11 introduced several useful methods:

```java
isBlank()
strip()
stripLeading()
stripTrailing()
lines()
repeat()
```

---

## 2.1 `isBlank()`

### Problem before Java 11

To check whether a String was empty or contained only spaces:

```java
String name = "   ";

if (name.trim().isEmpty()) {
    System.out.println("Blank");
}
```

We had to perform:

```text
trim()
   ↓
isEmpty()
```

### Java 11

```java
if (name.isBlank()) {
    System.out.println("Blank");
}
```

### `isEmpty()` vs `isBlank()`

```java
"".isEmpty();      // true
"   ".isEmpty();   // false

"".isBlank();      // true
"   ".isBlank();   // true
```

### Practical usage

Useful for request validation:

```java
@PostMapping("/users")
public void createUser(@RequestParam String username) {

    if (username == null || username.isBlank()) {
        throw new IllegalArgumentException("Username is required");
    }
}
```

Common use cases:

- Request validation
- Form validation
- Configuration validation
- Input checking

### Interview answer

> `isBlank()` was introduced to provide a direct and readable way to check whether a String is empty or contains only whitespace.

---

# 3. `strip()`

## Problem before Java 11

Java developers commonly used:

```java
String name = "   Gururaj   ";

String result = name.trim();
```

For normal spaces, it works correctly.

However, `trim()` is an older method and does not recognize all Unicode whitespace characters.

## Java 11

```java
String result = name.strip();
```

`strip()` is Unicode-aware.

### Example

```java
String s = "\u2003Hello Java\u2003";

System.out.println("[" + s.trim() + "]");
System.out.println("[" + s.strip() + "]");
```

Conceptually:

```text
trim()
→ may leave Unicode whitespace

strip()
→ removes Unicode-aware whitespace
```

### Additional methods

```java
stripLeading();
stripTrailing();
```

Example:

```java
String s = "   Java   ";

System.out.println("[" + s.stripLeading() + "]");
System.out.println("[" + s.stripTrailing() + "]");
```

### Easy difference

```text
trim()  → older, basic whitespace handling
strip() → Java 11, Unicode-aware whitespace handling
```

### Practical usage

Useful when handling:

- User input
- International text
- API request values
- External data
- Text normalization

### Interview answer

> `trim()` removes basic whitespace characters, while `strip()` uses Unicode-aware whitespace detection and is more suitable for modern international text.

---

# 4. `lines()`

Suppose we have:

```java
String data =
        "Java\nSpring Boot\nKafka\nRedis";
```

## Before Java 11

```java
String[] lines = data.split("\n");

for (String line : lines) {
    System.out.println(line);
}
```

## Java 11

```java
data.lines()
    .forEach(System.out::println);
```

`lines()` returns:

```java
Stream<String>
```

Because it returns a Stream, we can perform Stream operations.

Example:

```java
long count =
        data.lines()
            .filter(line -> !line.isBlank())
            .count();
```

### Practical backend example

Suppose we have application logs:

```java
String logs = Files.readString(Path.of("app.log"));

logs.lines()
    .filter(line -> line.contains("ERROR"))
    .forEach(System.out::println);
```

### Common use cases

- Log processing
- Multiline API responses
- Text file processing
- Configuration files
- Parsing text data

### Interview answer

> `lines()` converts a multiline String into a `Stream<String>`, making it easier to process each line using the Stream API.

---

# 5. `repeat()`

## Before Java 11

To repeat a String:

```java
String result = "";

for (int i = 0; i < 20; i++) {
    result += "-";
}
```

## Java 11

```java
String result = "-".repeat(20);
```

Output:

```text
--------------------
```

### Practical usage

```java
System.out.println("=".repeat(50));
System.out.println("APPLICATION STARTED");
System.out.println("=".repeat(50));
```

Commonly used for:

- Formatting
- CLI applications
- Logs
- Test data
- Generated text

### Interview answer

> `repeat()` provides a simple built-in way to repeat a String multiple times without writing loops.

---

# 6. `Optional.isEmpty()`

Suppose:

```java
Optional<User> user =
        userRepository.findById(10L);
```

## Before Java 11

```java
if (!user.isPresent()) {
    throw new RuntimeException("User not found");
}
```

The expression:

```java
!user.isPresent()
```

is slightly harder to read.

## Java 11

```java
if (user.isEmpty()) {
    throw new RuntimeException("User not found");
}
```

### Practical backend example

```java
Optional<Order> order =
        orderRepository.findById(orderId);

if (order.isEmpty()) {
    throw new OrderNotFoundException();
}
```

### Before vs Now

```text
Before:
!optional.isPresent()

Java 11:
optional.isEmpty()
```

### Why introduced?

Mainly to improve readability and make the API more expressive.

### Interview answer

> `Optional.isEmpty()` was introduced as the opposite of `isPresent()` so empty-value checks become more readable.

---

# 7. `Files.readString()`

## Problem before Java 11

Reading a text file often required multiple steps:

```java
byte[] bytes =
        Files.readAllBytes(Paths.get("data.txt"));

String content =
        new String(bytes, StandardCharsets.UTF_8);
```

Flow:

```text
File
 ↓
byte[]
 ↓
String
```

## Java 11

```java
String content =
        Files.readString(Path.of("data.txt"));
```

### Practical example

Suppose `config.json` contains:

```json
{
  "application": "ShowSeat"
}
```

Read it:

```java
String json =
        Files.readString(Path.of("config.json"));

System.out.println(json);
```

### Practical uses

- Small JSON files
- Configuration files
- Templates
- Test data
- Text files

### Important point

For very large files, avoid loading the whole file into memory using `readString()`.

Use streaming APIs instead.

### Interview answer

> `Files.readString()` was introduced to reduce boilerplate when reading small text files into a String.

---

# 8. `Files.writeString()`

## Before Java 11

```java
String data = "Hello Java";

Files.write(
        Paths.get("data.txt"),
        data.getBytes(StandardCharsets.UTF_8)
);
```

Flow:

```text
String
 ↓
byte[]
 ↓
File
```

## Java 11

```java
Files.writeString(
        Path.of("data.txt"),
        "Hello Java"
);
```

### Practical example

```java
String report =
        "Total orders: 100\n" +
        "Success: 95\n" +
        "Failed: 5";

Files.writeString(
        Path.of("report.txt"),
        report
);
```

### Interview answer

> `Files.writeString()` simplifies writing String content directly to a file without manually converting it into bytes.

---

# 9. `Predicate.not()`

Suppose:

```java
List<String> names =
        List.of(
            "Java",
            "",
            "Spring",
            " ",
            "Kafka"
        );
```

We want only nonblank Strings.

## Before Java 11

```java
names.stream()
     .filter(name -> !name.isBlank())
     .forEach(System.out::println);
```

This works.

But suppose we want to use a method reference:

```java
.filter(String::isBlank)
```

This keeps blank Strings.

We need the opposite.

## Java 11

```java
names.stream()
     .filter(Predicate.not(String::isBlank))
     .forEach(System.out::println);
```

Output:

```text
Java
Spring
Kafka
```

### Practical example

```java
List<String> tags =
        List.of(
            "java",
            "",
            "spring",
            " ",
            "microservices"
        );

List<String> cleanTags =
        tags.stream()
            .filter(Predicate.not(String::isBlank))
            .toList();
```

### Concept

```text
Predicate:
String::isBlank

Negated Predicate:
Predicate.not(String::isBlank)
```

### Why introduced?

To improve readability when negating method references.

### Interview answer

> `Predicate.not()` makes it easier to negate an existing Predicate or method reference, especially inside Stream operations.

---

# 10. `var` in Lambda Parameters

`var` itself was introduced in Java 10 for local variables.

Example:

```java
var name = "Gururaj";
var count = 10;
```

Compiler understands the types automatically.

Java 11 extended `var` to lambda parameters.

## Before Java 11

```java
(a, b) -> a + b
```

Or:

```java
(Integer a, Integer b) -> a + b
```

## Java 11

```java
(var a, var b) -> a + b
```

Example:

```java
BiFunction<Integer, Integer, Integer> sum =
        (var a, var b) -> a + b;

System.out.println(sum.apply(10, 20));
```

Output:

```text
30
```

## Why was this introduced?

One important reason is annotation support.

Example:

```java
(@NotNull var name) -> name.toUpperCase()
```

### Important rule

Valid:

```java
(var a, var b) -> a + b
```

Invalid:

```java
(var a, b) -> a + b
```

If one lambda parameter uses `var`, all lambda parameters must use `var`.

### Practical usage

This is less common in normal business logic.

Main uses:

- Annotated lambda parameters
- More consistent lambda syntax
- Type inference with metadata

### Interview answer

> Java 11 added support for `var` in lambda parameters, mainly to allow annotations and modifiers while still using type inference.

---

# 11. Standard `HttpClient`

This is one of the most important Java 11 features.

Suppose:

```text
Order Service
      ↓
Payment Service
```

The Order Service needs to make an HTTP request to Payment Service.

---

## Before Java 11

Java had older APIs such as:

```java
HttpURLConnection
```

Example:

```java
URL url =
        new URL("https://api.example.com/users");

HttpURLConnection connection =
        (HttpURLConnection) url.openConnection();

connection.setRequestMethod("GET");

InputStream input =
        connection.getInputStream();
```

This API was relatively verbose.

Because of this, developers commonly used:

```text
Apache HttpClient
OkHttp
```

Spring applications also use:

```text
RestTemplate
WebClient
RestClient
```

---

## Java 11

Java introduced a modern standard HTTP client.

Main classes:

```text
HttpClient
HttpRequest
HttpResponse
```

Example:

```java
HttpClient client =
        HttpClient.newHttpClient();

HttpRequest request =
        HttpRequest.newBuilder()
                .uri(
                    URI.create(
                        "https://api.example.com/users"
                    )
                )
                .GET()
                .build();

HttpResponse<String> response =
        client.send(
            request,
            HttpResponse.BodyHandlers.ofString()
        );

System.out.println(response.statusCode());
System.out.println(response.body());
```

### Flow

```text
Create HttpClient
       ↓
Create HttpRequest
       ↓
client.send()
       ↓
HttpResponse
```

### Why introduced?

To provide Java developers with:

- A modern HTTP API
- Cleaner syntax
- HTTP/2 support
- Asynchronous communication
- Standard JDK support without requiring external libraries

---

# 12. Asynchronous HTTP Requests

Java 11 `HttpClient` supports:

```java
sendAsync()
```

Example:

```java
CompletableFuture<HttpResponse<String>> future =
        client.sendAsync(
                request,
                HttpResponse.BodyHandlers.ofString()
        );
```

Then:

```java
future
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println);
```

### Synchronous flow

```text
Thread
  ↓
Call external API
  ↓
WAIT
  ↓
Response
  ↓
Continue
```

### Asynchronous flow

```text
Send request
     ↓
Do not block waiting
     ↓
CompletableFuture returned
     ↓
Response arrives
     ↓
thenApply()
     ↓
thenAccept()
```

### Practical backend usage

Useful when:

- Calling external APIs
- Multiple independent API calls
- Async integrations
- Non-blocking workflows
- Improving responsiveness

### Interview answer

> Java 11 introduced a standard `HttpClient` with support for HTTP/1.1, HTTP/2, synchronous calls using `send()`, and asynchronous calls using `sendAsync()`, which returns a `CompletableFuture`.

---

# 13. Direct Execution of Java Source Files

## Before Java 11

Normally:

```bash
javac Hello.java
java Hello
```

First compile, then run.

## Java 11

You can directly run:

```bash
java Hello.java
```

Example:

```java
public class Hello {

    public static void main(String[] args) {
        System.out.println("Hello Java 11");
    }
}
```

Run:

```bash
java Hello.java
```

### Practical usage

Useful for:

- Small scripts
- Quick experiments
- Learning
- Utility programs
- Simple command-line tools

### Interview answer

> Java 11 supports single-file source-code execution, allowing a Java source file to be run directly using `java FileName.java`.

---

# 14. Java EE Modules Removed

Java 8 included some Java EE-related modules inside the JDK.

Examples:

```text
JAXB
JAX-WS
CORBA
Java Activation Framework
```

In Java 11, these modules were removed from the JDK.

Because of this, an application that works on Java 8 may fail after migration to Java 11.

Example:

```java
import javax.xml.bind.*;
```

This may require an explicit Maven or Gradle dependency when using Java 11.

### Practical migration issue

```text
Java 8
Application works
     ↓
Upgrade to Java 11
     ↓
JAXB classes missing
     ↓
Add JAXB dependency explicitly
```

### Interview answer

> Some Java EE modules such as JAXB and JAX-WS were removed from the JDK in Java 11, so applications using them need to add those libraries explicitly as dependencies.

---

# 15. Garbage Collection Improvements

For Java 11 interviews, mainly understand these:

```text
G1 GC
ZGC
Epsilon GC
```

## G1 GC

G1 means:

```text
Garbage First
```

It divides the heap into regions and prioritizes regions containing more garbage.

Conceptually:

```text
Heap
 ↓
Multiple regions
 ↓
Find regions containing more garbage
 ↓
Clean those regions first
```

G1 is the default garbage collector in Java 11.

---

## ZGC

ZGC was introduced experimentally in Java 11.

Purpose:

```text
Very low pause times
Large heaps
```

Basic awareness is enough for most backend interviews.

---

## Epsilon GC

Epsilon is a no-op garbage collector.

It allocates memory but does not reclaim it.

Mostly useful for:

- Testing
- Performance experiments
- Short-lived programs

---

# 16. Java Flight Recorder

Java Flight Recorder, or JFR, is a JVM diagnostic and monitoring tool.

It can collect information about:

```text
CPU usage
Memory allocation
Garbage collection
Threads
Locks
Exceptions
I/O activity
```

### Practical usage

Suppose a production application becomes slow.

JFR can help investigate:

```text
High CPU?
Too much GC?
Thread contention?
Memory allocation?
Blocked threads?
```

### Interview answer

> Java Flight Recorder is a low-overhead JVM monitoring and profiling tool used to collect runtime information for diagnosing production performance problems.

---

# 17. Before Java 11 vs Java 11 Summary

```text
Before Java 11                       Java 11
--------------------------------------------------------------

str.trim().isEmpty()                 str.isBlank()

str.trim()                           str.strip()

text.split("\n")                     text.lines()

loop to repeat String                str.repeat(n)

!optional.isPresent()                optional.isEmpty()

readAllBytes + new String            Files.readString()

String → byte[] → Files.write        Files.writeString()

x -> !x.isBlank()                    Predicate.not(String::isBlank)

Explicit/implicit lambda params      var lambda parameters

HttpURLConnection / libraries        HttpClient
```

---

# 18. Why Were These Features Introduced?

Most Java 11 features do not mean Java was previously unable to perform these operations.

The same tasks were often possible before.

The improvements were mainly introduced to make Java code:

```text
Shorter
Cleaner
More readable
Less error-prone
More expressive
Better suited for modern applications
```

For example:

```text
Before:
!optional.isPresent()

Now:
optional.isEmpty()
```

Both work.

But the Java 11 version communicates the intention more clearly.

---

# 19. Most Important Features for Backend Interviews

Priority 1:

```text
String methods
Optional.isEmpty()
Files.readString()/writeString()
Predicate.not()
var in lambda
HttpClient
```

Priority 2:

```text
Direct Java file execution
Java EE module removal
G1 GC
ZGC awareness
Java Flight Recorder
```

---

# 20. Interview Question: Tell Me About Java 11 Features

A good answer:

> Java 11 introduced several useful improvements. It added new String methods such as `isBlank()`, `strip()`, `lines()`, and `repeat()`. It added `Optional.isEmpty()`, `Files.readString()` and `Files.writeString()`, `Predicate.not()`, and support for `var` in lambda parameters. One of the major additions was the standard `HttpClient`, which supports HTTP/1.1, HTTP/2, and asynchronous requests using `CompletableFuture`. Java 11 also supports running single Java source files directly. From a platform perspective, some Java EE modules such as JAXB and JAX-WS were removed, and Java 11 included JVM improvements such as G1 enhancements, ZGC, and Java Flight Recorder.

---

# 21. Short 30-Second Interview Answer

> Java 11 added useful APIs such as new String methods, `Optional.isEmpty()`, `Files.readString()` and `writeString()`, `Predicate.not()`, and `var` in lambda parameters. One major feature is the standard `HttpClient` API with HTTP/2 and asynchronous support using `CompletableFuture`. It also supports running Java source files directly and removed some Java EE modules such as JAXB and JAX-WS from the JDK.

---

# 22. Quick Revision

```text
Java 11
│
├── String
│   ├── isBlank()
│   ├── strip()
│   ├── stripLeading()
│   ├── stripTrailing()
│   ├── lines()
│   └── repeat()
│
├── Optional
│   └── isEmpty()
│
├── Files
│   ├── readString()
│   └── writeString()
│
├── Streams
│   └── Predicate.not()
│
├── Lambda
│   └── var parameters
│
├── Networking
│   └── HttpClient
│       ├── send()
│       ├── sendAsync()
│       ├── HTTP/1.1
│       └── HTTP/2
│
├── Execution
│   └── java File.java
│
├── Migration
│   └── Java EE modules removed
│
└── JVM
    ├── G1 GC
    ├── ZGC
    ├── Epsilon GC
    └── Java Flight Recorder
```

---

# Final Memory Trick

Remember:

```text
S O F P V H

S → String improvements
O → Optional.isEmpty()
F → Files read/write String
P → Predicate.not()
V → var in lambda
H → HttpClient
```

These six are the main Java 11 interview features you should remember first.
