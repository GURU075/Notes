# Java Generics

## 1. Overview

Java Generics allow classes, interfaces, and methods to work with
different data types while maintaining compile-time type safety. Instead
of writing separate code for every data type, you write reusable code
that works for many types. Generics remove unnecessary casting and
prevent many runtime `ClassCastException`s.

------------------------------------------------------------------------

## 2. Why It Matters

### Where it is used

-   Java Collections (`List<T>`, `Map<K,V>`, `Set<E>`)
-   Spring Boot (`JpaRepository<User, Long>`, `ResponseEntity<T>`,
    `Optional<T>`)
-   Stream API (`Function<T,R>`, `Predicate<T>`, `Consumer<T>`)
-   Libraries and Frameworks

### Why you should learn it

-   Frequently asked in 2--5 years Java interviews.
-   Makes APIs reusable and type-safe.
-   Required to understand Spring, Streams, Collections, and modern
    Java.

### Problems it solves

-   Eliminates explicit casting.
-   Prevents invalid object insertion.
-   Detects type errors at compile time.

------------------------------------------------------------------------

## 3. Key Concepts

### Generic Type Parameter (`T`)

A placeholder representing a data type decided by the caller.

``` java
class Box<T> {
    private T value;
}
```

### Common Naming Conventions

  Symbol   Meaning
  -------- -------------
  T        Type
  E        Element
  K        Key
  V        Value
  R        Return Type
  N        Number

### Generic Class

``` java
class Box<T>{
    private T value;

    public void set(T value){
        this.value = value;
    }

    public T get(){
        return value;
    }
}
```

### Generic Method

``` java
public static <T> void print(T value){
    System.out.println(value);
}
```

### Multiple Type Parameters

``` java
class Pair<K,V>{
    K key;
    V value;
}
```

### Bounded Generics

Restrict acceptable types.

``` java
class Calculator<T extends Number>{ }
```

### Wildcards

#### Unbounded

``` java
List<?>
```

Accepts any generic list.

#### Upper Bound

``` java
List<? extends Number>
```

Read-only producer of Number types.

#### Lower Bound

``` java
List<? super Integer>
```

Useful for adding Integers.

### PECS Rule

**Producer Extends Consumer Super**

-   Read → `extends`
-   Write → `super`

### Type Erasure

Generics exist only during compilation.

``` java
List<String>
```

becomes

``` java
List
```

at runtime.

------------------------------------------------------------------------

## 4. Syntax or Formula

``` java
List<String> names = new ArrayList<>();

class Box<T>{}

class Pair<K,V>{}

public static <T> T identity(T value){
    return value;
}

class Calculator<T extends Number>{}

List<?>
List<? extends Number>
List<? super Integer>
```

------------------------------------------------------------------------

## 5. Real-world Spring Examples

``` java
JpaRepository<User, Long>

ResponseEntity<User>

Optional<User>

List<Employee>

Map<String, Object>

Function<Employee,String>

Predicate<Employee>
```

------------------------------------------------------------------------

## 6. Interview Questions

### Q1. What are Generics?

Generics allow classes, interfaces, and methods to work with multiple
data types while providing compile-time type safety and eliminating
explicit casting.

### Q2. Why were Generics introduced?

To remove runtime ClassCastException caused by Object collections and
eliminate manual casting.

### Q3. Difference between List`<Object>`{=html} and List\<?\>?

-   `List<Object>` stores only Object references.
-   `List<?>` can reference a list of any type.

### Q4. Difference between `<T>` and `<?>`?

-   `<T>` declares a type parameter.
-   `<?>` represents an unknown existing type.

### Q5. What is Type Erasure?

Generic type information is removed by the compiler for backward
compatibility.

### Q6. Explain PECS.

Producer → `extends`

Consumer → `super`

------------------------------------------------------------------------

## 7. Common Mistakes

❌ Using raw types

``` java
List list = new ArrayList();
```

✅

``` java
List<String> list = new ArrayList<>();
```

❌

``` java
List<Object> list = new ArrayList<Integer>();
```

✅

``` java
List<?> list = new ArrayList<Integer>();
```

------------------------------------------------------------------------

## 8. Best Practices

-   Always use Generics instead of raw types.
-   Prefer diamond operator (`<>`).
-   Use `extends` for reading.
-   Use `super` for writing.
-   Use meaningful type parameter names.
-   Avoid unnecessary wildcards.

------------------------------------------------------------------------

## 9. Quick Revision

### Remember

-   Generics = Compile-time Type Safety
-   No Casting
-   Reusable Code
-   `T` = Type Variable
-   `extends` = Upper Bound
-   `super` = Lower Bound
-   `?` = Unknown Type
-   PECS = Producer Extends Consumer Super
-   Type Erasure = Runtime removes generic type

### Interview One-liners

-   Generics prevent ClassCastException.
-   They improve readability and reusability.
-   `List<Integer>` is **not** a subtype of `List<Object>`.
-   Generics work only at compile time.
-   JVM does not know `List<String>` vs `List<Integer>` after
    compilation.

------------------------------------------------------------------------

## 10. Revision Checklist

-   [ ] Why Generics?
-   [ ] Generic Class
-   [ ] Generic Method
-   [ ] Multiple Type Parameters
-   [ ] Bounded Types
-   [ ] Wildcards
-   [ ] PECS
-   [ ] Type Erasure
-   [ ] Spring Examples
-   [ ] Interview Questions
