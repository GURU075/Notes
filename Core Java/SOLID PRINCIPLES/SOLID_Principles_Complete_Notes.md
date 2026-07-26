# SOLID Principles in Java and Spring Boot

## 1. Overview

**SOLID** is a group of five object-oriented design principles that help developers write code that is:

- Easy to understand
- Easy to test
- Easy to maintain
- Flexible when requirements change
- Less tightly coupled
- Easier to extend without breaking existing functionality

SOLID stands for:

| Letter | Principle |
|---|---|
| S | Single Responsibility Principle |
| O | Open/Closed Principle |
| L | Liskov Substitution Principle |
| I | Interface Segregation Principle |
| D | Dependency Inversion Principle |

These principles are especially important in Java backend development because enterprise applications usually contain many services, repositories, controllers, integrations, and business rules.

---

## 2. Why SOLID Matters

Without SOLID principles, code often becomes:

- Large and difficult to understand
- Tightly coupled
- Hard to test
- Risky to modify
- Full of repeated conditions
- Dependent on concrete implementations
- Difficult to reuse

Example of poor design:

```java
public class OrderService {

    public void placeOrder(Order order) {
        validateOrder(order);
        saveOrder(order);
        sendEmail(order);
        generateInvoice(order);
        processPayment(order);
    }
}
```

This class performs many unrelated responsibilities:

- Validation
- Database operations
- Email communication
- Invoice generation
- Payment processing

A change in any one responsibility can force changes in the same class.

SOLID helps separate these responsibilities into focused components.

---

# 3. Single Responsibility Principle

## 3.1 Definition

> A class should have only one reason to change.

This means a class should be responsible for one specific part of the application.

It does **not** mean that a class should contain only one method.

It means all methods in the class should support one focused responsibility.

---

## 3.2 Bad Example

```java
public class EmployeeService {

    public void saveEmployee(Employee employee) {
        System.out.println("Saving employee");
    }

    public void calculateSalary(Employee employee) {
        System.out.println("Calculating salary");
    }

    public void sendEmail(Employee employee) {
        System.out.println("Sending email");
    }

    public void generateReport(Employee employee) {
        System.out.println("Generating report");
    }
}
```

### Problems

The class has multiple responsibilities:

- Saving employee data
- Calculating salary
- Sending emails
- Generating reports

Possible reasons to modify this class:

- Database changes
- Salary rule changes
- Email provider changes
- Report format changes

This violates SRP.

---

## 3.3 Improved Design

```java
public class EmployeeRepository {

    public void save(Employee employee) {
        System.out.println("Saving employee");
    }
}
```

```java
public class SalaryCalculator {

    public BigDecimal calculate(Employee employee) {
        return employee.getBasicSalary()
                .add(employee.getAllowance());
    }
}
```

```java
public class EmailService {

    public void sendWelcomeEmail(Employee employee) {
        System.out.println("Sending welcome email");
    }
}
```

```java
public class EmployeeReportGenerator {

    public void generate(Employee employee) {
        System.out.println("Generating employee report");
    }
}
```

Now every class has one focused responsibility.

---

## 3.4 Spring Boot Example

```java
@RestController
@RequestMapping("/employees")
@RequiredArgsConstructor
public class EmployeeController {

    private final EmployeeService employeeService;

    @PostMapping
    public ResponseEntity<EmployeeResponse> create(
            @RequestBody EmployeeRequest request) {

        return ResponseEntity.ok(
                employeeService.createEmployee(request)
        );
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class EmployeeService {

    private final EmployeeRepository employeeRepository;
    private final EmployeeMapper employeeMapper;

    public EmployeeResponse createEmployee(
            EmployeeRequest request) {

        Employee employee =
                employeeMapper.toEntity(request);

        Employee saved =
                employeeRepository.save(employee);

        return employeeMapper.toResponse(saved);
    }
}
```

```java
@Component
public class EmployeeMapper {

    public Employee toEntity(EmployeeRequest request) {
        return new Employee(
                request.name(),
                request.department()
        );
    }

    public EmployeeResponse toResponse(Employee employee) {
        return new EmployeeResponse(
                employee.getId(),
                employee.getName(),
                employee.getDepartment()
        );
    }
}
```

Responsibilities are separated:

- Controller handles HTTP requests
- Service handles business logic
- Repository handles persistence
- Mapper handles object conversion

---

## 3.5 How to Identify SRP Violations

Look for classes with names such as:

```text
Manager
Helper
Utility
Processor
CommonService
EverythingService
```

These names may indicate that too many responsibilities have been placed in one class.

Other warning signs:

- Very large classes
- Many unrelated imports
- Many injected dependencies
- Frequent changes for unrelated requirements
- Methods working on unrelated domains
- Difficult unit testing

---

## 3.6 Interview Answer

> The Single Responsibility Principle says that a class should have one reason to change. In a Spring Boot application, I separate HTTP handling into controllers, business logic into services, persistence into repositories, and mapping into mapper classes. This keeps each component focused and makes testing and maintenance easier.

---

# 4. Open/Closed Principle

## 4.1 Definition

> Software entities should be open for extension but closed for modification.

This means:

- New behavior should be added through extension
- Existing stable code should not require repeated modification

The goal is not to never modify code. The goal is to avoid changing tested code every time a new variation is added.

---

## 4.2 Bad Example

```java
public class PaymentService {

    public void pay(String paymentType, BigDecimal amount) {

        if ("CREDIT_CARD".equals(paymentType)) {
            System.out.println("Credit card payment");

        } else if ("UPI".equals(paymentType)) {
            System.out.println("UPI payment");

        } else if ("NET_BANKING".equals(paymentType)) {
            System.out.println("Net banking payment");
        }
    }
}
```

Every new payment type requires modifying this method.

Example:

```text
Wallet payment
Cash on delivery
PayPal
Bank transfer
```

The class is not closed for modification.

---

## 4.3 Improved Design Using Strategy Pattern

```java
public interface PaymentProcessor {

    void pay(BigDecimal amount);
}
```

```java
@Component("CREDIT_CARD")
public class CreditCardPaymentProcessor
        implements PaymentProcessor {

    @Override
    public void pay(BigDecimal amount) {
        System.out.println(
                "Paid using credit card: " + amount
        );
    }
}
```

```java
@Component("UPI")
public class UpiPaymentProcessor
        implements PaymentProcessor {

    @Override
    public void pay(BigDecimal amount) {
        System.out.println(
                "Paid using UPI: " + amount
        );
    }
}
```

```java
@Component("NET_BANKING")
public class NetBankingPaymentProcessor
        implements PaymentProcessor {

    @Override
    public void pay(BigDecimal amount) {
        System.out.println(
                "Paid using net banking: " + amount
        );
    }
}
```

```java
@Service
public class PaymentService {

    private final Map<String, PaymentProcessor> processors;

    public PaymentService(
            Map<String, PaymentProcessor> processors) {
        this.processors = processors;
    }

    public void pay(
            String paymentType,
            BigDecimal amount) {

        PaymentProcessor processor =
                processors.get(paymentType);

        if (processor == null) {
            throw new IllegalArgumentException(
                    "Unsupported payment type: "
                            + paymentType
            );
        }

        processor.pay(amount);
    }
}
```

To add a new payment method, create a new implementation:

```java
@Component("WALLET")
public class WalletPaymentProcessor
        implements PaymentProcessor {

    @Override
    public void pay(BigDecimal amount) {
        System.out.println(
                "Paid using wallet: " + amount
        );
    }
}
```

No modification is required in `PaymentService`.

---

## 4.4 Another Example: Notification Service

Bad design:

```java
public void send(String type, String message) {

    if ("EMAIL".equals(type)) {
        // email logic
    } else if ("SMS".equals(type)) {
        // SMS logic
    } else if ("PUSH".equals(type)) {
        // push notification logic
    }
}
```

Better design:

```java
public interface NotificationSender {

    void send(String message);
}
```

```java
@Component("EMAIL")
public class EmailNotificationSender
        implements NotificationSender {

    @Override
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}
```

```java
@Component("SMS")
public class SmsNotificationSender
        implements NotificationSender {

    @Override
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}
```

---

## 4.5 Techniques Used to Follow OCP

Common techniques:

- Interfaces
- Abstract classes
- Polymorphism
- Strategy pattern
- Factory pattern
- Template Method pattern
- Dependency injection
- Configuration-driven behavior
- Plugin architecture

---

## 4.6 Interview Answer

> The Open/Closed Principle states that software components should be open for extension but closed for modification. For example, instead of adding multiple `if-else` conditions for payment types, I define a `PaymentProcessor` interface and create separate implementations. A new payment type is added by creating a new implementation rather than changing existing tested logic.

---

# 5. Liskov Substitution Principle

## 5.1 Definition

> Objects of a parent type should be replaceable with objects of a child type without breaking the expected behavior.

In simple terms:

If class `B` extends class `A`, then wherever `A` is expected, `B` should work correctly without surprising the caller.

---

## 5.2 Bad Example

```java
public class Bird {

    public void fly() {
        System.out.println("Bird is flying");
    }
}
```

```java
public class Sparrow extends Bird {
}
```

```java
public class Ostrich extends Bird {

    @Override
    public void fly() {
        throw new UnsupportedOperationException(
                "Ostrich cannot fly"
        );
    }
}
```

Usage:

```java
public void makeBirdFly(Bird bird) {
    bird.fly();
}
```

This works for `Sparrow`, but fails for `Ostrich`.

`Ostrich` cannot safely substitute `Bird` because the parent class incorrectly assumes that every bird can fly.

---

## 5.3 Improved Design

```java
public abstract class Bird {

    public abstract void eat();
}
```

```java
public interface FlyingBird {

    void fly();
}
```

```java
public class Sparrow
        extends Bird
        implements FlyingBird {

    @Override
    public void eat() {
        System.out.println("Sparrow is eating");
    }

    @Override
    public void fly() {
        System.out.println("Sparrow is flying");
    }
}
```

```java
public class Ostrich extends Bird {

    @Override
    public void eat() {
        System.out.println("Ostrich is eating");
    }
}
```

Now only birds capable of flying implement `FlyingBird`.

---

## 5.4 Banking Example

Bad design:

```java
public class Account {

    public void withdraw(BigDecimal amount) {
        System.out.println("Amount withdrawn");
    }
}
```

```java
public class FixedDepositAccount extends Account {

    @Override
    public void withdraw(BigDecimal amount) {
        throw new UnsupportedOperationException(
                "Withdrawal is not allowed"
        );
    }
}
```

If the system expects every `Account` to support withdrawal, the fixed deposit account breaks that expectation.

Better design:

```java
public interface Account {

    BigDecimal getBalance();
}
```

```java
public interface WithdrawableAccount
        extends Account {

    void withdraw(BigDecimal amount);
}
```

```java
public class SavingsAccount
        implements WithdrawableAccount {

    @Override
    public BigDecimal getBalance() {
        return BigDecimal.ZERO;
    }

    @Override
    public void withdraw(BigDecimal amount) {
        System.out.println("Amount withdrawn");
    }
}
```

```java
public class FixedDepositAccount
        implements Account {

    @Override
    public BigDecimal getBalance() {
        return BigDecimal.ZERO;
    }
}
```

---

## 5.5 Important Rules for LSP

A child class should not:

- Throw unexpected exceptions
- Remove behavior promised by the parent
- Strengthen method preconditions
- Weaken method postconditions
- Return invalid or surprising results
- Change expected side effects
- Break invariants of the parent class

---

## 5.6 Method Contract Example

Parent contract:

```java
public class DiscountCalculator {

    public double calculate(double amount) {
        return amount * 0.10;
    }
}
```

A subclass should not unexpectedly return a negative discount or throw an exception for valid positive input.

The caller relies on the contract of the parent type.

---

## 5.7 LSP and Composition

Sometimes inheritance is the wrong design.

Instead of:

```java
public class Ostrich extends FlyingBird {
}
```

prefer composition:

```java
public class Bird {

    private final MovementBehavior movementBehavior;

    public Bird(MovementBehavior movementBehavior) {
        this.movementBehavior = movementBehavior;
    }

    public void move() {
        movementBehavior.move();
    }
}
```

Different behaviors:

```java
public class FlyingMovement
        implements MovementBehavior {

    @Override
    public void move() {
        System.out.println("Flying");
    }
}
```

```java
public class RunningMovement
        implements MovementBehavior {

    @Override
    public void move() {
        System.out.println("Running");
    }
}
```

---

## 5.8 Interview Answer

> The Liskov Substitution Principle says that a subclass should be usable wherever its parent type is expected without breaking behavior. If a child class throws unsupported-operation exceptions for methods promised by the parent, the hierarchy is probably incorrect. I solve this by creating smaller capabilities through interfaces or by preferring composition over inheritance.

---

# 6. Interface Segregation Principle

## 6.1 Definition

> Clients should not be forced to depend on methods they do not use.

Instead of creating one large interface, create smaller and focused interfaces.

---

## 6.2 Bad Example

```java
public interface Worker {

    void work();

    void eat();

    void sleep();
}
```

```java
public class HumanWorker implements Worker {

    @Override
    public void work() {
        System.out.println("Human is working");
    }

    @Override
    public void eat() {
        System.out.println("Human is eating");
    }

    @Override
    public void sleep() {
        System.out.println("Human is sleeping");
    }
}
```

```java
public class RobotWorker implements Worker {

    @Override
    public void work() {
        System.out.println("Robot is working");
    }

    @Override
    public void eat() {
        throw new UnsupportedOperationException();
    }

    @Override
    public void sleep() {
        throw new UnsupportedOperationException();
    }
}
```

The robot is forced to implement methods it does not need.

---

## 6.3 Improved Design

```java
public interface Workable {

    void work();
}
```

```java
public interface Eatable {

    void eat();
}
```

```java
public interface Sleepable {

    void sleep();
}
```

```java
public class HumanWorker
        implements Workable, Eatable, Sleepable {

    @Override
    public void work() {
        System.out.println("Human is working");
    }

    @Override
    public void eat() {
        System.out.println("Human is eating");
    }

    @Override
    public void sleep() {
        System.out.println("Human is sleeping");
    }
}
```

```java
public class RobotWorker implements Workable {

    @Override
    public void work() {
        System.out.println("Robot is working");
    }
}
```

---

## 6.4 Repository Example

Bad interface:

```java
public interface UserRepository {

    User save(User user);

    User update(User user);

    void delete(Long id);

    User findById(Long id);

    List<User> findAll();

    void exportToCsv();

    void sendReport();
}
```

This mixes:

- Persistence
- Export
- Reporting

Better:

```java
public interface UserRepository {

    User save(User user);

    Optional<User> findById(Long id);

    void deleteById(Long id);
}
```

```java
public interface UserExporter {

    byte[] exportToCsv(List<User> users);
}
```

```java
public interface UserReportSender {

    void sendReport(List<User> users);
}
```

---

## 6.5 Spring Service Interface Example

Bad:

```java
public interface EmployeeOperations {

    Employee create(Employee employee);

    void processPayroll();

    byte[] generateReport();

    void sendEmail();

    void uploadDocument();
}
```

Better:

```java
public interface EmployeeManagementService {

    Employee create(Employee employee);
}
```

```java
public interface PayrollService {

    void processPayroll();
}
```

```java
public interface ReportService {

    byte[] generateReport();
}
```

```java
public interface DocumentStorageService {

    void uploadDocument();
}
```

---

## 6.6 Interface Segregation Does Not Mean Tiny Interfaces Everywhere

Do not create one interface per method without a valid design reason.

Bad overengineering:

```java
public interface UserSaver {
    User save(User user);
}

public interface UserFinder {
    User find(Long id);
}

public interface UserDeleter {
    void delete(Long id);
}
```

This may be unnecessary if all methods represent one cohesive repository responsibility.

The goal is **cohesion**, not maximum fragmentation.

---

## 6.7 Interview Answer

> The Interface Segregation Principle says that clients should not depend on methods they do not use. I avoid large interfaces that combine unrelated capabilities. For example, instead of forcing a robot to implement `eat()` and `sleep()`, I create focused interfaces such as `Workable`, `Eatable`, and `Sleepable`. In Spring applications, I keep service contracts focused on a cohesive business capability.

---

# 7. Dependency Inversion Principle

## 7.1 Definition

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

Also:

> Abstractions should not depend on details. Details should depend on abstractions.

This is one of the most important principles in Spring because dependency injection is built around it.

---

## 7.2 Bad Example

```java
public class OrderService {

    private final EmailNotificationService
            notificationService =
            new EmailNotificationService();

    public void placeOrder(Order order) {
        System.out.println("Saving order");
        notificationService.send(order);
    }
}
```

Problems:

- `OrderService` directly creates a concrete implementation
- Difficult to replace email with SMS
- Difficult to mock in unit tests
- Strong coupling
- Violates Dependency Inversion

---

## 7.3 Improved Design

Create an abstraction:

```java
public interface NotificationService {

    void send(Order order);
}
```

Implementation:

```java
@Service
public class EmailNotificationService
        implements NotificationService {

    @Override
    public void send(Order order) {
        System.out.println(
                "Sending order email"
        );
    }
}
```

High-level service:

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final NotificationService
            notificationService;

    public void placeOrder(Order order) {
        System.out.println("Saving order");
        notificationService.send(order);
    }
}
```

Now `OrderService` depends on the interface, not the concrete email class.

---

## 7.4 Replacing the Implementation

```java
@Service
@Primary
public class SmsNotificationService
        implements NotificationService {

    @Override
    public void send(Order order) {
        System.out.println(
                "Sending order SMS"
        );
    }
}
```

The high-level `OrderService` does not require modification.

---

## 7.5 Constructor Injection

Recommended:

```java
@Service
public class OrderService {

    private final NotificationService
            notificationService;

    public OrderService(
            NotificationService notificationService) {
        this.notificationService =
                notificationService;
    }
}
```

With Lombok:

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final NotificationService
            notificationService;
}
```

Advantages:

- Dependencies are explicit
- Easier unit testing
- Supports immutability
- Prevents partially initialized objects
- Avoids hidden dependencies
- Works well with DIP

---

## 7.6 Field Injection Should Be Avoided

```java
@Service
public class OrderService {

    @Autowired
    private NotificationService notificationService;
}
```

Problems:

- Difficult to instantiate in unit tests
- Dependency is hidden
- Field cannot easily be final
- Encourages classes with too many dependencies

Constructor injection is generally better.

---

## 7.7 Unit Testing Benefit

```java
class OrderServiceTest {

    @Test
    void shouldSendNotificationAfterOrder() {

        NotificationService notificationService =
                mock(NotificationService.class);

        OrderService orderService =
                new OrderService(notificationService);

        Order order = new Order();

        orderService.placeOrder(order);

        verify(notificationService).send(order);
    }
}
```

The test does not need a real email provider.

---

## 7.8 DIP vs Dependency Injection

These are related but not identical.

### Dependency Inversion Principle

A design principle:

```text
Depend on abstractions, not concrete classes.
```

### Dependency Injection

A technique:

```text
Dependencies are supplied from outside the class.
```

Spring's IoC container performs dependency injection and helps implement DIP.

Example:

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final NotificationService notificationService;
}
```

Spring provides the implementation at runtime.

---

## 7.9 Interview Answer

> The Dependency Inversion Principle says that high-level business modules should depend on abstractions rather than concrete implementations. In Spring Boot, I define an interface such as `NotificationService`, create implementations such as email or SMS, and inject the abstraction through the constructor. This reduces coupling and makes the code easier to test and extend. Dependency injection is the mechanism Spring uses to help implement this principle.

---

# 8. SOLID in a Complete Spring Boot Example

## 8.1 Requirement

Create an order and:

- Validate it
- Save it
- Process payment
- Send notification

---

## 8.2 Domain Classes

```java
public record OrderRequest(
        Long customerId,
        BigDecimal amount,
        String paymentType
) {
}
```

```java
public record OrderResponse(
        Long id,
        String status
) {
}
```

---

## 8.3 Validation Responsibility

```java
@Component
public class OrderValidator {

    public void validate(OrderRequest request) {

        if (request.customerId() == null) {
            throw new IllegalArgumentException(
                    "Customer ID is required"
            );
        }

        if (request.amount() == null ||
                request.amount()
                        .compareTo(BigDecimal.ZERO) <= 0) {

            throw new IllegalArgumentException(
                    "Amount must be positive"
            );
        }
    }
}
```

This follows SRP.

---

## 8.4 Payment Abstraction

```java
public interface PaymentProcessor {

    void process(BigDecimal amount);
}
```

```java
@Component("UPI")
public class UpiPaymentProcessor
        implements PaymentProcessor {

    @Override
    public void process(BigDecimal amount) {
        System.out.println(
                "Processing UPI payment"
        );
    }
}
```

```java
@Component("CARD")
public class CardPaymentProcessor
        implements PaymentProcessor {

    @Override
    public void process(BigDecimal amount) {
        System.out.println(
                "Processing card payment"
        );
    }
}
```

This supports OCP and DIP.

---

## 8.5 Notification Abstraction

```java
public interface OrderNotificationService {

    void sendOrderConfirmation(Long orderId);
}
```

```java
@Service
public class EmailOrderNotificationService
        implements OrderNotificationService {

    @Override
    public void sendOrderConfirmation(
            Long orderId) {

        System.out.println(
                "Order confirmation sent: "
                        + orderId
        );
    }
}
```

This supports DIP.

---

## 8.6 Order Service

```java
@Service
public class OrderService {

    private final OrderValidator orderValidator;
    private final OrderRepository orderRepository;
    private final Map<String, PaymentProcessor>
            paymentProcessors;
    private final OrderNotificationService
            notificationService;

    public OrderService(
            OrderValidator orderValidator,
            OrderRepository orderRepository,
            Map<String, PaymentProcessor>
                    paymentProcessors,
            OrderNotificationService
                    notificationService) {

        this.orderValidator = orderValidator;
        this.orderRepository = orderRepository;
        this.paymentProcessors = paymentProcessors;
        this.notificationService =
                notificationService;
    }

    public OrderResponse createOrder(
            OrderRequest request) {

        orderValidator.validate(request);

        PaymentProcessor processor =
                paymentProcessors.get(
                        request.paymentType()
                );

        if (processor == null) {
            throw new IllegalArgumentException(
                    "Unsupported payment type"
            );
        }

        processor.process(request.amount());

        Order order = new Order();
        order.setCustomerId(
                request.customerId()
        );
        order.setAmount(request.amount());
        order.setStatus("CREATED");

        Order saved =
                orderRepository.save(order);

        notificationService
                .sendOrderConfirmation(
                        saved.getId()
                );

        return new OrderResponse(
                saved.getId(),
                saved.getStatus()
        );
    }
}
```

---

## 8.7 Which SOLID Principles Are Used?

### Single Responsibility Principle

- `OrderValidator` validates
- `OrderRepository` persists
- `PaymentProcessor` handles payment
- `OrderNotificationService` sends notifications

### Open/Closed Principle

New payment types are added using new implementations.

### Liskov Substitution Principle

Every `PaymentProcessor` implementation must correctly support `process()`.

### Interface Segregation Principle

Payment and notification capabilities use focused interfaces.

### Dependency Inversion Principle

`OrderService` depends on abstractions instead of concrete classes.

---

# 9. SOLID and Common Design Patterns

SOLID principles are not design patterns, but they support good use of design patterns.

| Principle | Common Related Patterns |
|---|---|
| SRP | Facade, Service Layer, Repository |
| OCP | Strategy, Factory, Template Method |
| LSP | Strategy, Composition |
| ISP | Adapter, Role Interfaces |
| DIP | Dependency Injection, Factory, Bridge |

Example:

```text
OCP + Strategy Pattern
```

You create new strategy implementations without modifying existing service logic.

---

# 10. Common Misunderstandings

## 10.1 SRP Does Not Mean One Method Per Class

A class may have many methods if they all support one responsibility.

Example:

```java
public class EmployeeRepository {

    Employee save(Employee employee);

    Optional<Employee> findById(Long id);

    void deleteById(Long id);
}
```

All methods relate to employee persistence.

---

## 10.2 OCP Does Not Mean Code Is Never Modified

Bug fixes and requirement changes may require modification.

OCP means stable business flows should not require repeated modification for every new variation.

---

## 10.3 LSP Is Not Only About Inheritance Syntax

Code may compile correctly but still violate LSP behaviorally.

Example:

```java
@Override
public void withdraw(BigDecimal amount) {
    throw new UnsupportedOperationException();
}
```

The compiler accepts it, but the design may violate the parent contract.

---

## 10.4 ISP Does Not Mean Hundreds of Tiny Interfaces

Split interfaces when clients are forced to depend on unrelated methods.

Do not fragment cohesive interfaces unnecessarily.

---

## 10.5 DIP Does Not Mean Every Class Needs an Interface

Do not create meaningless interfaces for every simple class.

Example:

```java
EmployeeMapper
EmployeeMapperImpl
```

An interface may not be necessary if:

- There is only one stable implementation
- No alternate behavior is expected
- Testing does not require substitution
- The abstraction does not represent a meaningful business contract

Use abstractions where they reduce coupling or represent genuine variation.

---

# 11. Best Practices

## 11.1 Prefer Composition Over Inheritance

Instead of forcing unrelated subclasses into one hierarchy, inject behavior.

```java
public class ReportService {

    private final ReportFormatter formatter;

    public ReportService(
            ReportFormatter formatter) {
        this.formatter = formatter;
    }
}
```

---

## 11.2 Keep Controllers Thin

Controller should:

- Accept request
- Validate input
- Call service
- Return response

Avoid putting business logic in controllers.

---

## 11.3 Keep Services Focused

Avoid services with dozens of unrelated operations.

Bad:

```text
CommonService
UtilityService
ApplicationManager
```

Prefer domain-focused names:

```text
OrderService
PaymentService
InventoryService
```

---

## 11.4 Depend on Business Abstractions

Good:

```java
private final PaymentProcessor paymentProcessor;
```

Less flexible:

```java
private final StripePaymentProcessor
        stripePaymentProcessor;
```

---

## 11.5 Avoid Excessive `if-else`

Repeated type-based `if-else` chains often indicate a missing Strategy pattern or polymorphism.

---

## 11.6 Write Contract-Based Tests

For multiple implementations, test that each implementation respects the same contract.

Example:

```text
All PaymentProcessor implementations:
- Reject negative values
- Process valid amounts
- Return consistent status
```

This supports LSP.

---

## 11.7 Do Not Overengineer

SOLID should simplify change, not make simple code unnecessarily complex.

For a small, stable requirement, a direct implementation may be better than creating many layers and interfaces.

Apply SOLID based on:

- Expected change
- Business complexity
- Number of variations
- Testing needs
- Team conventions

---

# 12. Interview Questions and Answers

## 12.1 What is SOLID?

> SOLID is a set of five object-oriented design principles: Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion. They help create maintainable, testable, extensible, and loosely coupled software.

---

## 12.2 Explain SRP with a Spring Boot example.

> SRP says a class should have one reason to change. In Spring Boot, a controller handles HTTP communication, a service handles business logic, a repository handles persistence, and a mapper converts DTOs. This separation keeps each class focused.

---

## 12.3 Explain OCP with a payment example.

> Instead of modifying a payment service every time a new payment type is introduced, I define a `PaymentProcessor` interface and create implementations such as UPI, card, and wallet. The service works with the interface, so new payment types are added without modifying existing logic.

---

## 12.4 Explain LSP with an example.

> LSP means a child object should replace its parent without breaking expected behavior. If `FixedDepositAccount` extends `Account` but throws an unsupported exception from `withdraw()`, it violates LSP because callers expect every Account to support withdrawal. I would split the abstraction into `Account` and `WithdrawableAccount`.

---

## 12.5 Explain ISP with an example.

> ISP says clients should not depend on methods they do not need. Instead of one large `Worker` interface containing `work()`, `eat()`, and `sleep()`, I create focused interfaces. A robot implements only `Workable`, while a human can implement all three.

---

## 12.6 Explain DIP with Spring.

> DIP says high-level modules should depend on abstractions, not concrete classes. In Spring, I inject an interface such as `NotificationService` through the constructor. Spring provides an implementation such as `EmailNotificationService`. This reduces coupling and improves testability.

---

## 12.7 Difference Between DIP and Dependency Injection

> Dependency Inversion is a design principle, while dependency injection is a technique used to supply dependencies from outside a class. Spring's IoC container uses dependency injection to help us implement DIP.

---

## 12.8 Which SOLID principle is violated by a large interface?

> Interface Segregation Principle, because implementations may be forced to depend on methods they do not need. It may also indicate an SRP violation if the interface combines unrelated responsibilities.

---

## 12.9 Which SOLID principle is violated by many `if-else` blocks for types?

> Usually the Open/Closed Principle. If each new type requires modifying the same method, polymorphism or the Strategy pattern may be more appropriate.

---

## 12.10 Can one design violate multiple SOLID principles?

> Yes. A large service that creates concrete dependencies, contains many unrelated methods, and uses repeated type conditions can violate SRP, OCP, and DIP at the same time.

---

## 12.11 Do you always create interfaces for services?

> No. I create interfaces when they represent a meaningful contract, multiple implementations are expected, external boundaries need abstraction, or substitution improves testing. Creating interfaces for every class without a reason adds unnecessary complexity.

---

## 12.12 How does SOLID improve testing?

> Focused classes are easier to test, abstractions can be mocked, behavior can be substituted, and changes are isolated. For example, an OrderService depending on a PaymentProcessor interface can be tested using a mock payment processor without calling a real payment gateway.

---

# 13. Scenario-Based Interview Questions

## 13.1 A service has 15 injected dependencies. What does it indicate?

Possible answer:

> It may indicate that the service has too many responsibilities and violates SRP. I would examine whether responsibilities such as validation, mapping, notification, persistence, and external integration can be moved into focused collaborators. However, dependency count alone is only a warning sign, not automatic proof.

---

## 13.2 Every new document type requires adding another `if-else`. What would you do?

> I would define a `DocumentProcessor` interface and create separate implementations for PDF, CSV, XML, and other formats. A registry or factory can select the implementation. This follows OCP and makes new types easier to add.

---

## 13.3 A subclass throws `UnsupportedOperationException`. Is that always an LSP violation?

> Not always, but it is a strong warning sign. If the parent contract promises the operation for all subtypes, then it is an LSP violation. The hierarchy should be redesigned using smaller interfaces or composition.

---

## 13.4 A class directly creates an external API client. What problem does it cause?

> It creates tight coupling and makes unit testing difficult. I would inject an abstraction such as `PaymentGateway` or `NotificationClient` so the business service does not depend directly on the concrete client.

---

# 14. Quick Revision Notes

## 14.1 SOLID in One Line

```text
S -> One responsibility
O -> Extend without modifying stable code
L -> Child must safely replace parent
I -> Small and focused interfaces
D -> Depend on abstractions
```

---

## 14.2 Fast Examples

```text
SRP:
Controller, Service, Repository separated

OCP:
PaymentProcessor implementations

LSP:
Ostrich should not inherit fly()

ISP:
Robot should not implement eat()

DIP:
OrderService depends on NotificationService
```

---

## 14.3 Common Warning Signs

```text
SRP violation:
Large class with unrelated methods

OCP violation:
Repeated if-else for every new type

LSP violation:
Subclass throws unsupported operation

ISP violation:
Class implements unused methods

DIP violation:
High-level class creates concrete dependency
```

---

## 14.4 Interview Memory Trick

```text
S -> Separate responsibilities
O -> Open to extend
L -> Legal substitution
I -> Interfaces should be focused
D -> Depend on abstraction
```

---

# 15. Final Interview Answer

> SOLID is a set of five object-oriented design principles used to create maintainable and extensible systems. SRP keeps each class focused on one responsibility. OCP allows new behavior through extension instead of repeatedly modifying stable code. LSP ensures subtypes preserve the behavior promised by their parent type. ISP avoids forcing clients to depend on unnecessary methods. DIP makes high-level business logic depend on abstractions rather than concrete implementations. In Spring Boot, these principles are commonly implemented through layered architecture, interfaces, constructor injection, dependency injection, strategy implementations, and focused services.

---

# 16. Revision Checklist

Before an interview, make sure you can:

- Define all five SOLID principles
- Give one simple example for each
- Explain SRP using Controller-Service-Repository
- Explain OCP using payment strategies
- Explain LSP using Bird or Account examples
- Explain ISP using Worker interfaces
- Explain DIP using Spring constructor injection
- Differentiate DIP and dependency injection
- Explain why not every class needs an interface
- Identify SOLID violations in existing code
- Explain how SOLID improves testing
- Discuss overengineering and practical balance

---

# 17. Folder Recommendation

Store this file under:

```text
Java Backend Interview Notes/
└── 01 Core Java/
    └── SOLID Principles/
        └── SOLID_Principles_Complete_Notes.md
```

You can replace the five separate SOLID files with this single consolidated document.
