# Java Streams, Collectors, `forEach`, and Optional — Quick Revision Notes

## 1. Quick Method Selection Table

| Method | Used On | Main Purpose | Returns a Value? | Use When |
|---|---|---|---|---|
| `map()` | Stream / Optional | Transform one value into another | Yes | You want to change each element |
| `filter()` | Stream | Keep only matching elements | Yes | You want to remove unwanted elements |
| `collect()` | Stream | Combine stream elements into a collection or result | Yes | You want a `List`, `Set`, `Map`, grouped data, etc. |
| `toList()` | Stream | Collect elements into a list | Yes | Final output should be a `List` |
| `groupingBy()` | Collector | Group elements using a key | Yes | You want department-wise, category-wise, or value-wise grouping |
| `counting()` | Collector | Count elements in each group | Yes, as `Long` | You want frequency/count of grouped elements |
| `forEach()` | Stream / Map / Collection | Perform an action for every element | No | You want to print, save, log, or execute logic |
| `ifPresent()` | Optional | Perform an action if a value exists | No | Do something only when value is present |
| `ifPresentOrElse()` | Optional | Perform one action if present and another if empty | No | You need both present and empty handling |
| `orElse()` | Optional | Return value or default value | Yes | You need a fallback value |
| `orElseGet()` | Optional | Return value or lazily create default value | Yes | Default calculation is expensive |
| `orElseThrow()` | Optional | Return value or throw exception | Yes | Missing value is an error |
| `Optional.map()` | Optional | Transform the value inside Optional | Yes, as `Optional` | You want one property from an optional object |

---

# 2. `map()` vs `collect()`

## `map()`

`map()` transforms each individual element.

```java
List<Integer> doubledNumbers = numbers.stream()
        .map(num -> num * 2)
        .toList();
```

Input:

```java
[10, 20, 30]
```

Output:

```java
[20, 40, 60]
```

### Meaning

```java
.map(num -> num * 2)
```

Read it as:

> Take every number and convert it into number multiplied by two.

---

## `collect()`

`collect()` combines all stream elements into one final result.

Examples of final results:

- `List`
- `Set`
- `Map`
- grouped data
- frequency map
- joined string

```java
List<Integer> result = numbers.stream()
        .collect(Collectors.toList());
```

### Easy rule

```text
map()     -> transforms each element
collect() -> combines all elements into one result
```

You cannot normally replace `collect()` with `map()` when the required result is a `Map`.

---

# 3. Frequency Map Using `groupingBy()` and `counting()`

## Example

```java
List<Integer> numbers = List.of(10, 20, 10, 30, 20, 10);

Map<Integer, Long> frequencyMap = numbers.stream()
        .collect(Collectors.groupingBy(
                num -> num,
                Collectors.counting()
        ));

System.out.println(frequencyMap);
```

Output:

```text
{10=3, 20=2, 30=1}
```

Meaning:

- `10` appears 3 times
- `20` appears 2 times
- `30` appears 1 time

---

## Understanding the Syntax

```java
Collectors.groupingBy(
        num -> num,
        Collectors.counting()
)
```

`groupingBy()` accepts two important things:

```text
groupingBy(grouping key, operation on each group)
```

In this example:

```java
num -> num
```

means:

> Use the number itself as the map key.

And:

```java
Collectors.counting()
```

means:

> Count how many elements are present in every group.

Internally, grouping is conceptually like:

```text
10 -> [10, 10, 10]
20 -> [20, 20]
30 -> [30]
```

After `counting()`:

```text
10 -> 3
20 -> 2
30 -> 1
```

---

## Why the Count Type Is `Long`

`Collectors.counting()` returns `Long`.

Therefore:

```java
Map<Integer, Long>
```

is correct.

This is not correct:

```java
Map<Integer, Integer>
```

when using `Collectors.counting()` directly.

---

## `num -> num` Explained

A lambda has this structure:

```text
input -> output
```

Example:

```java
num -> num
```

Input is `num`.

Output is the same `num`.

It is called an identity operation because it returns the same value.

The same logic can be written as:

```java
Function.identity()
```

Complete example:

```java
Map<Integer, Long> frequencyMap = numbers.stream()
        .collect(Collectors.groupingBy(
                Function.identity(),
                Collectors.counting()
        ));
```

Required import:

```java
import java.util.function.Function;
```

---

# 4. Frequency Map Without Streams

Use this version first if the Stream syntax feels difficult.

```java
Map<Integer, Integer> frequencyMap = new HashMap<>();

for (Integer num : numbers) {
    frequencyMap.put(
            num,
            frequencyMap.getOrDefault(num, 0) + 1
    );
}
```

### Step-by-step

For the first `10`:

```java
frequencyMap.getOrDefault(10, 0)
```

returns:

```text
0
```

Then:

```text
0 + 1 = 1
```

Map becomes:

```text
{10=1}
```

For the next `10`:

```java
frequencyMap.getOrDefault(10, 0)
```

returns:

```text
1
```

Then:

```text
1 + 1 = 2
```

Map becomes:

```text
{10=2}
```

---

# 5. Grouping Employees by Department

Suppose:

```java
class Employee {
    private String name;
    private String department;
    private double salary;

    // getters and setters
}
```

Group employees by department:

```java
Map<String, List<Employee>> employeesByDepartment =
        employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment
                ));
```

Result conceptually:

```text
IT -> [employee1, employee2]
HR -> [employee3]
Sales -> [employee4, employee5]
```

Here:

```java
Employee::getDepartment
```

is the grouping key.

It is equivalent to:

```java
employee -> employee.getDepartment()
```

---

# 6. Highest-Paid Employee in Each Department

```java
Map<String, Optional<Employee>> highPayEmpInEachDepartment =
        employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.maxBy(
                                Comparator.comparingDouble(Employee::getSalary)
                        )
                ));
```

## Why the Value Is `Optional<Employee>`

The result type is:

```java
Map<String, Optional<Employee>>
```

because `maxBy()` returns an `Optional`.

It returns `Optional` because a group could theoretically be empty.

---

## Understanding the Collector

```java
Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.maxBy(
                Comparator.comparingDouble(Employee::getSalary)
        )
)
```

Read it as:

> Group employees by department, then find the employee with the highest salary in every department.

---

# 7. Map `forEach()` Syntax

For a map:

```java
map.forEach((key, value) -> {
    // logic
});
```

Example:

```java
frequencyMap.forEach((number, count) -> {
    System.out.println(number + " = " + count);
});
```

Output:

```text
10 = 3
20 = 2
30 = 1
```

For the employee map:

```java
highPayEmpInEachDepartment.forEach((department, employeeOptional) -> {
    // department is the key
    // employeeOptional is the value
});
```

---

# 8. Why This Code Is Incorrect

Incorrect code:

```java
highPayEmpInEachDepartment.forEach((dep, emp) -> {
    System.out.print(dep + " = ");
    System.out.println(
            emp.ifPresentOrElse(
                    value -> value.getSalary(),
                    " "
            )
    );
});
```

There are two problems.

## Problem 1: `ifPresentOrElse()` Returns `void`

You cannot write:

```java
System.out.println(emp.ifPresentOrElse(...));
```

because `println()` expects a value, but `ifPresentOrElse()` returns nothing.

Its conceptual return type is:

```java
void ifPresentOrElse(...)
```

---

## Problem 2: The Second Argument Must Be a `Runnable`

Incorrect:

```java
" "
```

Correct:

```java
() -> System.out.println("No Employee")
```

The empty case needs an action, not a String.

---

# 9. Correct Use of `ifPresentOrElse()`

```java
highPayEmpInEachDepartment.forEach((dep, emp) -> {
    System.out.print(dep + " = ");

    emp.ifPresentOrElse(
            value -> System.out.println(value.getSalary()),
            () -> System.out.println("No Employee")
    );
});
```

Read it as:

> If employee is present, print salary. Otherwise, print "No Employee".

---

## Expanded Form

```java
highPayEmpInEachDepartment.forEach((dep, emp) -> {

    System.out.print(dep + " = ");

    if (emp.isPresent()) {
        Employee value = emp.get();
        System.out.println(value.getSalary());
    } else {
        System.out.println("No Employee");
    }
});
```

The Stream/Optional version is shorter, but both represent the same idea.

---

# 10. `ifPresent()`

Use `ifPresent()` when you only care about the present case.

```java
employeeOptional.ifPresent(
        employee -> System.out.println(employee.getSalary())
);
```

Equivalent traditional code:

```java
if (employeeOptional.isPresent()) {
    Employee employee = employeeOptional.get();
    System.out.println(employee.getSalary());
}
```

Remember:

```text
ifPresent() performs an action.
It does not return a value.
```

---

# 11. `ifPresentOrElse()`

Use it when you need both cases.

```java
employeeOptional.ifPresentOrElse(
        employee -> System.out.println(employee.getSalary()),
        () -> System.out.println("Employee not found")
);
```

First argument:

```java
employee -> System.out.println(employee.getSalary())
```

This is a `Consumer`.

It accepts one value and performs an action.

Second argument:

```java
() -> System.out.println("Employee not found")
```

This is a `Runnable`.

It takes no input and performs an action.

---

# 12. `orElse()`

Use `orElse()` when you need a returned value.

```java
Employee employee = employeeOptional.orElse(defaultEmployee);
```

For salary:

```java
double salary = employeeOptional
        .map(Employee::getSalary)
        .orElse(0.0);
```

Complete printing example:

```java
highPayEmpInEachDepartment.forEach((dep, emp) -> {
    double salary = emp
            .map(Employee::getSalary)
            .orElse(0.0);

    System.out.println(dep + " = " + salary);
});
```

### Important

```java
map(Employee::getSalary)
```

converts:

```text
Optional<Employee>
```

into:

```text
Optional<Double>
```

Then:

```java
orElse(0.0)
```

returns salary if present, otherwise `0.0`.

---

# 13. `orElseGet()`

`orElseGet()` accepts a supplier.

```java
Employee employee = employeeOptional.orElseGet(
        () -> createDefaultEmployee()
);
```

Difference:

```java
orElse(createDefaultEmployee())
```

may call `createDefaultEmployee()` even when the Optional contains a value.

```java
orElseGet(() -> createDefaultEmployee())
```

calls it only when the Optional is empty.

Use `orElseGet()` when creating the default value is expensive.

---

# 14. `orElseThrow()`

Use when an absent value should cause an error.

```java
Employee employee = employeeOptional.orElseThrow(
        () -> new RuntimeException("Employee not found")
);
```

Common Spring example:

```java
Employee employee = employeeRepository.findById(id)
        .orElseThrow(
                () -> new RuntimeException(
                        "Employee not found with id: " + id
                )
        );
```

---

# 15. `Optional.map()`

`Optional.map()` transforms the value inside an Optional.

```java
Optional<Employee> employeeOptional = ...;

Optional<Double> salaryOptional =
        employeeOptional.map(Employee::getSalary);
```

If employee exists:

```text
Optional<Employee>
        ↓ map
Optional<Double>
```

If employee does not exist:

```text
Optional.empty()
        ↓ map
Optional.empty()
```

It does not throw an error simply because the Optional is empty.

---

# 16. Which Optional Method Should You Use?

## Perform an Action

Use:

```java
ifPresent()
```

or:

```java
ifPresentOrElse()
```

Examples:

- print a value
- send a notification
- call a method
- log information

---

## Return a Value

Use:

```java
orElse()
```

```java
orElseGet()
```

```java
orElseThrow()
```

Examples:

- assign employee to a variable
- get salary or default salary
- get entity or throw exception

---

## Transform a Value

Use:

```java
map()
```

Example:

```java
Optional<Double> salary =
        employeeOptional.map(Employee::getSalary);
```

---

# 17. Common Mistakes

## Mistake 1: Printing `ifPresentOrElse()`

Incorrect:

```java
System.out.println(
        optional.ifPresentOrElse(...)
);
```

Reason:

```text
ifPresentOrElse() returns void.
```

Correct:

```java
optional.ifPresentOrElse(
        value -> System.out.println(value),
        () -> System.out.println("Empty")
);
```

---

## Mistake 2: Returning a Value Inside `ifPresent()`

Incorrect:

```java
optional.ifPresent(value -> value.getSalary());
```

This calls `getSalary()`, but the result is not stored or printed.

Correct:

```java
optional.ifPresent(
        value -> System.out.println(value.getSalary())
);
```

Or, to obtain the salary:

```java
double salary = optional
        .map(Employee::getSalary)
        .orElse(0.0);
```

---

## Mistake 3: Using `map()` Instead of `collect()`

Incorrect expectation:

```java
numbers.stream()
        .map(...)
```

`map()` alone cannot create a frequency map.

Correct:

```java
numbers.stream()
        .collect(Collectors.groupingBy(
                num -> num,
                Collectors.counting()
        ));
```

---

## Mistake 4: Calling `get()` Without Checking

Risky:

```java
Employee employee = employeeOptional.get();
```

This throws `NoSuchElementException` if empty.

Better:

```java
Employee employee = employeeOptional.orElseThrow(
        () -> new RuntimeException("Employee not found")
);
```

---

# 18. Syntax Memory Templates

## Frequency Map

```java
Map<Integer, Long> frequencyMap = numbers.stream()
        .collect(Collectors.groupingBy(
                num -> num,
                Collectors.counting()
        ));
```

Read:

> Group by number and count every group.

---

## Group by Department

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment
        ));
```

Read:

> Group employees by department.

---

## Highest Salary in Each Department

```java
Map<String, Optional<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(
                        Comparator.comparingDouble(Employee::getSalary)
                )
        ));
```

Read:

> Group by department and select the maximum salary employee.

---

## Print Optional Value or Fallback Message

```java
optional.ifPresentOrElse(
        value -> System.out.println(value),
        () -> System.out.println("Value not found")
);
```

---

## Get Optional Property or Default

```java
double salary = employeeOptional
        .map(Employee::getSalary)
        .orElse(0.0);
```

---

# 19. Interview Answers

## What is the difference between `map()` and `collect()`?

`map()` is an intermediate operation used to transform every stream element.  
`collect()` is a terminal operation used to combine stream elements into a final result such as a `List`, `Set`, or `Map`.

---

## Why does `Collectors.counting()` return `Long`?

It returns `Long` because a stream can theoretically contain a very large number of elements, more than the maximum value supported by `Integer`.

---

## What does `groupingBy()` do?

`groupingBy()` groups stream elements based on a classifier function and returns a `Map`.

Example:

```java
Collectors.groupingBy(Employee::getDepartment)
```

groups employees using department as the map key.

---

## Why does `maxBy()` return `Optional`?

`maxBy()` returns `Optional` because the stream or group may be empty, and there may be no maximum element.

---

## Difference between `ifPresentOrElse()` and `orElse()`?

`ifPresentOrElse()` performs actions and returns nothing.

`orElse()` returns the contained value or a default value.

---

# 20. Practice Questions

## Question 1

Count frequency of words.

Input:

```java
List<String> words =
        List.of("java", "spring", "java", "sql", "spring", "java");
```

Expected result:

```text
{java=3, spring=2, sql=1}
```

Solution:

```java
Map<String, Long> result = words.stream()
        .collect(Collectors.groupingBy(
                word -> word,
                Collectors.counting()
        ));
```

---

## Question 2

Group employees by department.

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment
        ));
```

---

## Question 3

Print an employee's name if present, otherwise print `"Not Found"`.

```java
employeeOptional.ifPresentOrElse(
        employee -> System.out.println(employee.getName()),
        () -> System.out.println("Not Found")
);
```

---

## Question 4

Get employee salary, or `0.0` when employee is absent.

```java
double salary = employeeOptional
        .map(Employee::getSalary)
        .orElse(0.0);
```

---

# 21. Final Quick Revision

```text
map()
Transforms every element.

collect()
Combines all elements into one result.

groupingBy()
Creates groups and returns a Map.

counting()
Counts elements in every group and returns Long.

forEach()
Performs an action for every element and returns void.

ifPresent()
Runs an action only when Optional has a value.

ifPresentOrElse()
Runs one action when present and another when empty.

orElse()
Returns value or default value.

orElseGet()
Lazily creates a default value.

orElseThrow()
Returns value or throws exception.

Optional.map()
Transforms the value inside Optional.
```

## One-Line Memory Rule

```text
Need to transform?  -> map()
Need to combine?    -> collect()
Need to group?      -> groupingBy()
Need to count?      -> counting()
Need to perform?    -> forEach() / ifPresent()
Need a value?       -> orElse() / orElseGet() / orElseThrow()
```
