---
layout: post
title: "Understanding SOLID Principles and Clean Code"
categories: Programming
excerpt: SOLID principles and Clean Code practices help create maintainable, scalable Java software. SOLID guides class design (single responsibility, extensibility, loose coupling), while Clean Code focuses on readable names, small methods, clear errors, and consistent style, making change safer and faster.
---
# Understanding SOLID Principles and Clean Code
SOLID principles and Clean Code practices help create maintainable, scalable Java software. SOLID guides class design (single responsibility, extensibility, loose coupling), while Clean Code focuses on readable names, small methods, clear errors, and consistent style, making change safer and faster.

## 1. SOLID Principles

SOLID is about making code **easy to change** without breaking everything.

### S – Single Responsibility Principle (SRP)
**One class/module = one reason to change.**

- A class should have one clear job.
- If you change “logging rules” and that forces you to edit your business logic class, you likely broke SRP.

**Typical Java “god” class:**

```java
public class UserService {
    private final Connection connection;

    public UserService(Connection connection) {
        this.connection = connection;
    }

    public void register(User user) throws SQLException {
        // 1. validate
        if (user.getEmail() == null || !user.getEmail().contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }

        // 2. hash password
        String hash = PasswordHasher.hash(user.getPassword());
        user.setPassword(hash);

        // 3. save to DB
        PreparedStatement stmt = connection.prepareStatement(
            "INSERT INTO users (email, password) VALUES (?, ?)"
        );
        stmt.setString(1, user.getEmail());
        stmt.setString(2, user.getPassword());
        stmt.executeUpdate();

        // 4. send email
        EmailClient.sendWelcomeEmail(user.getEmail());
    }
}
```

This class handles **validation, hashing, DB, email**.

**Refactored by responsibilities:**

```java
public class UserService {
    private final UserValidator validator;
    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;
    private final MailService mailService;

    public UserService(UserValidator validator,
                       PasswordEncoder passwordEncoder,
                       UserRepository userRepository,
                       MailService mailService) {
        this.validator = validator;
        this.passwordEncoder = passwordEncoder;
        this.userRepository = userRepository;
        this.mailService = mailService;
    }

    public void register(User user) {
        validator.validate(user);
        user.setPassword(passwordEncoder.encode(user.getPassword()));
        userRepository.save(user);
        mailService.sendWelcomeEmail(user);
    }
}
```

Each class has **one reason to change**:

- `UserValidator` – validation rules changed
- `PasswordEncoder` – hashing algorithm changed
- `UserRepository` – DB schema/technology changed
- `MailService` – email provider / template changed

This is exactly how **Spring** apps are usually structured.

---

### O – Open/Closed Principle (OCP)
**Open for extension, closed for modification.**

- You should be able to add new behavior **without changing existing, tested code**.
- Use interfaces, polymorphism, configuration, etc.

Discount example in Java:

**Bad (if-else grows forever):**

```java
public class DiscountService {
    public BigDecimal applyDiscount(Customer customer, BigDecimal price) {
        if (customer.getType() == CustomerType.REGULAR) {
            return price.multiply(BigDecimal.valueOf(0.95));
        } else if (customer.getType() == CustomerType.VIP) {
            return price.multiply(BigDecimal.valueOf(0.90));
        } else if (customer.getType() == CustomerType.STAFF) {
            return price.multiply(BigDecimal.valueOf(0.85));
        }
        return price;
    }
}
```

**Better: Strategy + OCP:**

```java
public interface DiscountPolicy {
    BigDecimal apply(BigDecimal price);
}

public class RegularDiscount implements DiscountPolicy {
    @Override
    public BigDecimal apply(BigDecimal price) {
        return price.multiply(BigDecimal.valueOf(0.95));
    }
}

public class VipDiscount implements DiscountPolicy {
    @Override
    public BigDecimal apply(BigDecimal price) {
        return price.multiply(BigDecimal.valueOf(0.90));
    }
}

// etc.
```

```java
public class DiscountService {
    private final Map<CustomerType, DiscountPolicy> policies;

    public DiscountService(Map<CustomerType, DiscountPolicy> policies) {
        this.policies = policies;
    }

    public BigDecimal applyDiscount(Customer customer, BigDecimal price) {
        return policies.getOrDefault(customer.getType(), p -> p).apply(price);
    }
}
```

Adding new discount = **new class + map entry**, not changing logic inside `DiscountService`.

With Spring, you can inject this map automatically using `@Qualifier` or by collecting beans.

---

### L – Liskov Substitution Principle (LSP)
**Subtypes must be usable wherever base types are expected.**

- If `B` extends `A`, then any code that works with `A` should also work with `B` **without surprises**.

Red flags:

- Overriding methods to throw `UnsupportedOperationException`.
- Subclass changing behavior in a way that breaks client expectations.

Example case 1: if `Rectangle` has `setWidth` and `setHeight`, a `Square` subclass that changes these in unexpected ways often violates LSP. Prefer **composition** over such inheritance.

Example case 2: 
**Violation example:**

```java
public class FileStorage {
    public void save(String path, byte[] content) { ... }
}

public class ReadOnlyFileStorage extends FileStorage {
    @Override
    public void save(String path, byte[] content) {
        throw new UnsupportedOperationException("Read-only storage");
    }
}
```

Any code that expects to use `FileStorage.save()` will **break** when given `ReadOnlyFileStorage`. That’s an LSP violation.

**Better: split behavior via interfaces:**

```java
public interface FileReadable {
    byte[] read(String path);
}

public interface FileWritable {
    void save(String path, byte[] content);
}

public class FullFileStorage implements FileReadable, FileWritable { ... }

public class ReadOnlyFileStorage implements FileReadable { ... }
```

Callers that need writing depend on `FileWritable`; callers that only read depend on `FileReadable`.


---

### I – Interface Segregation Principle (ISP)
**Many small, specific interfaces > one large, general interface.**

- Clients should not be forced to depend on methods they don’t use.

In Java, this is about **small, focused interfaces**.

**Bad:**

```java
public interface UserOperations {
    void create(User user);
    void update(User user);
    void delete(User user);
    User findById(Long id);
    List<User> findAll();
    void sendWelcomeEmail(User user);
}
```

Implementors that only do DB work shouldn’t know about `sendWelcomeEmail`.

**Better:**

```java
public interface UserRepository {
    void create(User user);
    void update(User user);
    void delete(User user);
    User findById(Long id);
    List<User> findAll();
}

public interface UserNotifier {
    void sendWelcomeEmail(User user);
}
```

Classes implement only what they actually need.

---

### D – Dependency Inversion Principle (DIP)
**Depend on abstractions, not concrete implementations.**

- High-level modules (business logic) should not depend on low-level modules (DB, HTTP).
- Both should depend on interfaces/abstractions.

Typical bad example:

```java
public class ReportService {
    private final JdbcReportRepository repository = new JdbcReportRepository();

    public List<Report> getReports() {
        return repository.findAll();
    }
}
```

`ReportService` is tightly coupled to `JdbcReportRepository`.

**Better: depend on abstraction and inject implementation:**

```java
public interface ReportRepository {
    List<Report> findAll();
}

public class JdbcReportRepository implements ReportRepository {
    // ...
}

@Service
public class ReportService {
    private final ReportRepository repository;

    @Autowired
    public ReportService(ReportRepository repository) {
        this.repository = repository;
    }

    public List<Report> getReports() {
        return repository.findAll();
    }
}
```

Now unit tests can pass a fake implementation:

```java
class InMemoryReportRepository implements ReportRepository {
    // fake implementation for tests
}
```

---

## 2. Clean Code in Java

### 2.1 Method Design

Aim for **small, intention-revealing methods**.

**Bad:**

```java
public void process() {
    // 50 lines of mixed logic…
}
```

**Better:**

```java
public void process() {
    validateRequest();
    loadUser();
    calculateInvoice();
    persistInvoice();
    notifyUser();
}
```

Each private method has a clear name and small body.

---

### 2.2 Naming in Java

- Class: `UserService`, `OrderRepository`, `InvoiceCalculator`
- Interfaces: `UserRepository`, `PaymentGateway`
- Methods: `createUser`, `calculateTotal`, `findByEmail`

```java
// Bad
public void doIt(User u) { ... }

// Better
public void registerUser(User user) { ... }
```

---

### 2.3 Exceptions and Error Handling

Prefer:

- Checked exceptions when the caller **must** handle (e.g., business rule violation).
- Runtime exceptions for programming errors.

```java
public class NotEnoughBalanceException extends RuntimeException {
    public NotEnoughBalanceException(String message) {
        super(message);
    }
}

public class PaymentService {
    public void charge(Account account, BigDecimal amount) {
        if (account.getBalance().compareTo(amount) < 0) {
            throw new NotEnoughBalanceException("Insufficient funds");
        }
        // ...
    }
}
```

In Spring REST:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotEnoughBalanceException.class)
    public ResponseEntity<ErrorResponse> handleBalanceError(NotEnoughBalanceException ex) {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(ex.getMessage()));
    }
}
```

Clean separation: **domain**, **error handling**, **HTTP**.

---

## 3. Key Design Patterns in Java You Actually Use

### 3.1 Strategy in Java (Sorting / Behavior Swapping)

```java
public interface SortingStrategy {
    List<Integer> sort(List<Integer> numbers);
}

public class QuickSortStrategy implements SortingStrategy {
    @Override
    public List<Integer> sort(List<Integer> numbers) {
        // quicksort impl...
    }
}

public class MergeSortStrategy implements SortingStrategy {
    @Override
    public List<Integer> sort(List<Integer> numbers) {
        // mergesort impl...
    }
}

public class SortContext {
    private SortingStrategy strategy;

    public SortContext(SortingStrategy strategy) {
        this.strategy = strategy;
    }

    public void setStrategy(SortingStrategy strategy) {
        this.strategy = strategy;
    }

    public List<Integer> execute(List<Integer> numbers) {
        return strategy.sort(numbers);
    }
}
```

Switch strategy at runtime with `setStrategy`.

---

### 3.2 Factory in Java (with Enums)

```java
public interface Notification {
    void send(String message);
}

public class EmailNotification implements Notification { ... }
public class SmsNotification implements Notification { ... }

public enum NotificationType { EMAIL, SMS }

public class NotificationFactory {
    public static Notification create(NotificationType type) {
        return switch (type) {
            case EMAIL -> new EmailNotification();
            case SMS   -> new SmsNotification();
        };
    }
}
```

Usage:

```java
Notification n = NotificationFactory.create(NotificationType.EMAIL);
n.send("Hello");
```

In Spring, factories are often replaced by **DI with conditional beans or profiles**, but the idea is the same.

---

### 3.3 Builder Pattern (very common in Java)

Avoid telescoping constructors:

```java
// Bad
User user = new User("john", "doe", 30, "street", "city", "country");
```

Instead:

```java
public class User {
    private String firstName;
    private String lastName;
    private int age;
    private String street;
    private String city;
    private String country;

    private User() {}

    public static class Builder {
        private final User user = new User();

        public Builder firstName(String firstName) {
            user.firstName = firstName;
            return this;
        }

        public Builder lastName(String lastName) {
            user.lastName = lastName;
            return this;
        }

        public Builder age(int age) {
            user.age = age;
            return this;
        }

        public Builder address(String street, String city, String country) {
            user.street = street;
            user.city = city;
            user.country = country;
            return this;
        }

        public User build() {
            // validate here if needed
            return user;
        }
    }
}
```

Usage:

```java
User user = new User.Builder()
    .firstName("John")
    .lastName("Doe")
    .age(30)
    .address("Street", "City", "Country")
    .build();
```

---

### 3.4 Adapter Pattern (for external APIs)

```java
public interface PaymentGateway {
    void pay(BigDecimal amount);
}

// Third-party library
public class StripeClient {
    public void makeCharge(BigDecimal amount) { ... }
}

public class StripePaymentAdapter implements PaymentGateway {
    private final StripeClient stripeClient;

    public StripePaymentAdapter(StripeClient stripeClient) {
        this.stripeClient = stripeClient;
    }

    @Override
    public void pay(BigDecimal amount) {
        stripeClient.makeCharge(amount);
    }
}
```

Your code depends only on `PaymentGateway`.

---

## 4. Putting It Together in a Java/Spring Project

Imagine a **payment use case**:

- Controller (web layer)
- Service (business layer)
- Repository (data)
- Gateway/Adapter (external systems)

```java
@RestController
@RequestMapping("/payments")
public class PaymentController {

    private final PaymentService service;

    public PaymentController(PaymentService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<Void> pay(@RequestBody PaymentRequest request) {
        service.pay(request);
        return ResponseEntity.ok().build();
    }
}
```

```java
@Service
public class PaymentService {

    private final PaymentRepository repository;
    private final PaymentGateway gateway;
    private final PaymentValidator validator;

    public PaymentService(PaymentRepository repository,
                          PaymentGateway gateway,
                          PaymentValidator validator) {
        this.repository = repository;
        this.gateway = gateway;
        this.validator = validator;
    }

    public void pay(PaymentRequest request) {
        validator.validate(request);                  // SRP (validation separated)
        Payment payment = Payment.from(request);      // Factory/B. method
        gateway.pay(payment.getAmount());            // DIP (interface)
        repository.save(payment);                    // DIP (interface)
    }
}
```

Patterns and principles visible:

- **SRP**: validator, gateway, repository, service each have single focus.
- **DIP**: service depends on `PaymentRepository` & `PaymentGateway` interfaces.
- **Adapter**: concrete gateway classes adapt external APIs.
- **Clean Code**: names are clear, methods small and focused.
- **OCP**: new payment gateway → new class implementing `PaymentGateway`, no change in `PaymentService`.

---

## 5. How You Can Practice (Java-Focused)

1. **Pick one service class you wrote** (in Spring or plain Java):
   - List what responsibilities it has.
   - Extract validator / repository / external client into separate classes.
2. **Introduce interfaces**:
   - For any class that does I/O (`*Repository`, `*Client`), define an interface.
   - Inject the interface into your services.
3. **Replace big `if/else` on type**:
   - With a `Map<Enum, Strategy>` or `Map<String, Strategy>`.
4. **Add tests**:
   - Write unit tests for services using **fake implementations** of the interfaces.
   - This will show you the power of DIP.

---
