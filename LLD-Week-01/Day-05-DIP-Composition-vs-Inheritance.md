# Day 5 - SOLID: DIP and Composition vs Inheritance

**Theme:** High-level policy depends on abstractions, and behavior composition is preferred when variation grows.

---

## Learning goals

By the end of this document, you should be able to:

- Explain Dependency Inversion Principle (DIP) in practical design terms.
- Refactor services so infrastructure details stay behind interfaces.
- Choose composition or inheritance using a clear decision rule.

---

## 1) Dependency Inversion Principle (DIP)

DIP has two parts:

1. High-level modules should not depend on low-level modules; both depend on abstractions.
2. Abstractions should not depend on details; details should depend on abstractions.

High-level modules are use cases and business policy (`UserService`, `PlaceOrderService`).  
Low-level details are DB drivers, HTTP clients, framework integrations, and storage-specific code.

### Practical wiring rule

- Domain/application code receives abstractions by constructor injection.
- Concrete selection happens at the composition root (`main`, bootstrap, DI config, tests).

---

## 2) Repository pattern with DIP

Target design:

```text
UserService -> UserRepository <- MySqlUserRepository
                             <- MongoUserRepository
                             <- InMemoryUserRepository (tests)
```

### Abstraction

```java
public interface UserRepository {
    Optional<User> findByEmail(Email email);
    User save(User user);
    void delete(UserId id);
}
```

### High-level service

```java
public class UserService {
    private final UserRepository users;

    public UserService(UserRepository users) {
        this.users = users;
    }

    public User register(RegisterRequest request) {
        if (users.findByEmail(request.email()).isPresent()) {
            throw new DuplicateEmailException(request.email());
        }
        User user = User.create(request.email(), request.password());
        return users.save(user);
    }
}
```

`UserService` contains no MySQL/Mongo imports and no storage-specific logic.

### Testability benefit

Inject `InMemoryUserRepository` in tests for fast and deterministic unit tests without database setup.

---

## 3) Composition root and transaction boundary

Concrete repository choice belongs only in composition root code:

```java
UserRepository repo = config.useMongo()
    ? new MongoUserRepository(mongoClient)
    : new MySqlUserRepository(dataSource);
UserService userService = new UserService(repo);
```

For multi-repository atomic workflows, keep transaction control behind an abstraction (`TransactionRunner` or `UnitOfWork`) so policy code remains infrastructure-agnostic.

---

## 4) Composition vs inheritance

### Composition (has-a / uses-a)

Behavior is assembled from collaborators.

Pros:

- Flexible feature combinations
- Better isolation in tests
- Easier extension via wrappers/strategies

### Inheritance (is-a)

Behavior is reused through subtype hierarchy.

Pros:

- Concise when domain hierarchy is truly stable and semantic

### Decision rule

- If behaviors vary independently or combine in many ways, prefer composition.
- If type is a true specialization with a stable shared contract, inheritance can be valid.

---

## 5) Plugin architecture preview (DIP + OCP + composition)

Design a core analyzer depending only on `AnalyzerPlugin` abstraction:

```java
public interface AnalyzerPlugin {
    String id();
    List<Finding> analyze(SourceFile file);
}
```

```java
public final class CodeAnalyzerEngine {
    private final List<AnalyzerPlugin> plugins;

    public AnalysisReport run(List<SourceFile> files) {
        List<Finding> all = new ArrayList<>();
        for (SourceFile file : files) {
            for (AnalyzerPlugin plugin : plugins) {
                all.addAll(plugin.analyze(file));
            }
        }
        return AnalysisReport.of(all);
    }
}
```

Adding a new plugin should not require engine changes.

---

## 6) Refactor drill: DIP violation to clean design

Violation:

```text
class UserService {
  void register(User u) {
    new MySqlConnection(...).insert(u);
  }
}
```

Refactor:

- Introduce `UserRepository` interface.
- Inject `UserRepository` into `UserService`.
- Move SQL/JDBC logic into `MySqlUserRepository`.
- Instantiate concrete implementation only in bootstrap/wiring.

---

## 7) Refactor drill: HTTP client feature growth

Problem: optional logging + retries + auth headers.

Inheritance approach causes class explosion:

- `LoggingHttpClient`
- `RetryHttpClient`
- `AuthHttpClient`
- `LoggingRetryAuthHttpClient`
- plus many other combinations

Composition/decorator approach:

- `HttpClient` interface with `execute(request)`
- `AuthHeaderClient`, `RetryClient`, `LoggingClient` wrappers
- Chain wrappers at composition root

This supports additive growth with minimal churn and better testability.

---

## 8) Honest downside of composition

Composition can introduce:

- More classes and wiring boilerplate
- Deeper call chains that require good tracing/logging

It trades simplicity in class count for flexibility in behavior evolution.

---

## 9) SOLID recap

| Letter | One-liner |
|--------|-----------|
| S | One reason to change per module |
| O | Extend behavior without editing stable cores |
| L | Subtypes preserve caller expectations |
| I | Clients depend on small focused interfaces |
| D | Policies depend on abstractions, details implement them |

---

## 10) Self-check with answers

1. **Where should `new MongoUserRepository()` appear?**  
   Only in composition root/bootstrap or test setup, never inside policy/domain services.

2. **One disadvantage of composition vs inheritance?**  
   More boilerplate and wiring complexity.

3. **How does DIP improve unit tests?**  
   Services accept interfaces, so tests inject fakes/stubs and avoid real infrastructure dependencies.

---

## 11) Interview quick lines

- DIP is about dependency direction, not any specific framework.
- DI is a technique; DIP is the design principle.
- `new Concrete()` belongs at wiring boundaries, not domain policy code.
- Composition is usually better when feature combinations grow.

---

## Day 5 checkpoint

- [x] I can design `UserService` against `UserRepository` abstraction.
- [x] I can place concrete infrastructure creation at composition root.
- [x] I can explain plugin-style extension through interfaces.
- [x] I can compare composition and inheritance with trade-offs.
