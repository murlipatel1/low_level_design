# Day 4 - SOLID: LSP and ISP

**Theme:** Honest subtype contracts and right-sized interfaces.

---

## Learning goals

By the end of this document, you should be able to:

- Explain LSP as substitutability that preserves caller expectations.
- Identify contract breaks such as stronger preconditions or unsupported methods.
- Apply ISP to split fat interfaces into role-based contracts.

---

## 1) Liskov Substitution Principle (LSP)

LSP means: if code works with a type `T`, it should also work with any subtype `S` of `T` without surprises.

Caller expectations usually include:

- Accepted inputs
- Failure behavior
- Side effects
- Postconditions/invariants

Subtypes must preserve that behavior story.

### Common LSP failures

- `Square extends Rectangle` where width and height are expected to vary independently.
- `ReadOnlyFile extends File` but `write()` throws.
- `Penguin extends Bird` when callers assume all birds can fly.

When capabilities differ, split interfaces or model separate hierarchies.

---

## 2) Interface Segregation Principle (ISP)

ISP means clients should not depend on methods they do not use.

Prefer small role-based interfaces over one large “kitchen sink” interface.

Why it matters:

- Prevents `UnsupportedOperationException` stubs.
- Reduces mocking/testing friction.
- Keeps dependencies explicit and focused.

---

## 3) LSP case study: Payment contract design

Goal: `CreditCard`, `Wallet`, and `UPI` should all be usable through one `PaymentMethod` abstraction.

Recommended core contract:

```java
public interface PaymentMethod {
    PaymentResult pay(PaymentRequest request);
    PaymentMethodType type();
}
```

### Plain-English contract for `pay`

Preconditions:

- Request is valid and non-null.
- Amount is positive and currency supported.
- Required channel context is available.

Postconditions:

- On success: returns success with transaction id and charged amount.
- On expected failure: returns structured failure code (not random exception style differences by subtype).
- Idempotency is respected for repeated requests.

### Handling method-specific input (for example UPI VPA)

Use a shared `PaymentRequest` with `PaymentContext` instead of subtype-only methods or caller-side casts.

This keeps one stable `pay` signature and maintains substitutability.

---

## 4) ISP case study: Logging interfaces

Fat logger anti-pattern:

```java
interface ILogger {
    void info(String msg);
    void audit(AuditEvent e);
    Span trace(String span);
}
```

Better split:

```java
public interface Loggable {
    void log(Level level, String message, Map<String, Object> fields);
}

public interface Auditable {
    void audit(AuditEvent event);
}

public interface Traceable {
    Span startSpan(String name);
}
```

Now each service depends only on what it actually needs.

Example:

- `HealthCheckService` depends only on `Loggable`.
- `CheckoutService` may depend on both `Loggable` and `Auditable`.

One concrete class can still implement multiple interfaces where appropriate.

---

## 5) Combined ISP drill: Refactor fat `IWorker`

Bad shape:

```text
interface IWorker {
  void writeCode();
  void attendMeeting();
  void reviewPR();
  void manageTeam();
}
```

Role-based split:

- `CodeAuthor`
- `MeetingParticipant`
- `CodeReviewer`
- `TeamManager`

This prevents role classes from implementing “not my job” methods and keeps capabilities explicit.

---

## 6) Defensive copying, encapsulation, and LSP

If a contract implies snapshot/read-only behavior, returning internal mutable collections breaks both:

- **Encapsulation:** callers can mutate internals directly.
- **LSP:** subtype may violate expectations by returning a live mutable structure.

Use defensive copies or immutable views to preserve contract behavior.

---

## 7) Implementation inheritance vs interface inheritance

Inheritance from concrete implementation can pull inherited state/default behavior that subtypes never intended, creating fragile base class risks.

Interface-based modeling is often safer for LSP because it defines explicit behavior contracts without forcing inherited internal mechanics.

---

## 8) Self-check with answers

1. **When does `UnsupportedOperationException` signal an ISP issue?**  
   When classes are forced to implement methods they do not need because the interface is too broad.

2. **How does defensive copying support LSP?**  
   It keeps returned data behavior consistent with contract expectations across all implementations.

3. **Why is implementation inheritance often riskier for LSP?**  
   Base-class changes can silently alter subtype behavior and break substitutability.

---

## 9) Interview quick lines

- LSP: subtypes must honor caller expectations, not just method names.
- ISP: no client should depend on methods it does not use.
- `UnsupportedOperationException` is often a design smell, not a solution.
- Capability interfaces (`Refundable`, `Traceable`, `CodeReviewer`) keep contracts honest.

---

## 10) Bridge to Day 5

Narrow honest interfaces make DIP easier: high-level modules can depend on small abstractions instead of concrete heavyweight classes.

---

## Day 4 checkpoint

- [x] I can define LSP using behavior expectations.
- [x] I can design `PaymentMethod` to keep all implementations substitutable.
- [x] I can split fat interfaces into role-focused contracts.
- [x] I can explain why unsupported methods usually indicate ISP/LSP issues.
