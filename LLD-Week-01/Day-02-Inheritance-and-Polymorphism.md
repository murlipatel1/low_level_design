# Day 2 - Inheritance and Polymorphism

**Theme:** Model true `is-a` relationships and runtime-substitutable behavior.  
This day builds the foundation for OCP (Day 3), LSP (Day 4), and composition-first thinking (Day 5).

---

## Learning goals

By the end of this document, you should be able to:

- Use inheritance only for real specialization, not just code reuse.
- Use polymorphism to remove repeated `if/else` dispatch logic.
- Keep interfaces honest so each implementation can satisfy the same contract.
- Spot fragile base classes and decide when composition is better.

---

## 1) Inheritance in LLD

Inheritance fits when:

- Subtype is truly a specialization of the base concept.
- Base contract and shared behavior are stable.
- Hierarchy stays shallow and understandable.

Inheritance hurts when:

- Subclasses disable, no-op, or throw for inherited methods.
- It is used only to reuse utility code.
- Tree depth makes behavior lookup hard during maintenance.

Interview-friendly line: start with a small interface or shallow abstract base, then grow only when domain semantics clearly support it.

---

## 2) Polymorphism in practice

Polymorphism lets callers depend on one contract while runtime implementation provides specific behavior.

Examples:

- `PaymentMethod.pay(invoice)` with card, wallet, and UPI implementations
- `NotificationChannel.send(message)` with email, SMS, and push implementations

Design goal: replace condition chains with dispatch through a contract.  
Overloading is not the same tool: it is compile-time selection, not runtime extensibility.

---

## 3) Interface vs abstract class

| Tool | Best use |
|------|----------|
| Interface | Capability contract, multiple implementations |
| Abstract class | Shared defaults/state for closely related types |

Rule of thumb: program to interfaces and keep inheritance minimal.

---

## 4) Case study: Document hierarchy

A clean 2-level hierarchy:

```text
        Document (abstract)
        /              \
PdfDocument      MarkdownDocument
```

Keep common metadata and universally required behavior on base type.  
Keep format-specific behaviors on concrete types.

```java
public abstract class Document {
    private final DocumentId id;
    private String title;
    private final Instant createdAt;

    public DocumentId getId() { return id; }
    public String getTitle() { return title; }
    public Instant getCreatedAt() { return createdAt; }

    public abstract String renderPlainText();
    public abstract int wordCount();
}
```

`PdfDocument` may expose page-related APIs; `MarkdownDocument` may expose markdown/HTML conversion.  
Callers typed as `Document` should not need PDF-specific details.

---

## 5) Case study: Replace string dispatch with polymorphism

Problem pattern repeated in systems:

```java
if (type == "EMAIL") ...
else if (type == "SMS") ...
```

Better design:

```java
public interface NotificationChannel {
    ChannelType type();
    void send(NotificationMessage message);
}
```

Use a registry/dispatcher to map `ChannelType` to implementation once, then dispatch through `channel.send(message)`.  
Adding new channels becomes additive: new class + registration, with no edits in existing channel logic.

---

## 6) Composition over wrong inheritance: Stack example

Bad pattern: `Stack extends ArrayList`

Why it is wrong:

- Stack is not semantically an array list.
- Inherited random-index operations break LIFO constraints.
- Contract mismatch leads to LSP issues.

Composition-based fix:

```java
public class Stack<E> {
    private final Deque<E> items = new ArrayDeque<>();

    public void push(E e) { items.push(e); }
    public E pop() { return items.pop(); }
    public int size() { return items.size(); }
}
```

This exposes only stack-safe operations and keeps invariants protected.

---

## 7) Design practice: PaymentMethod contract

A minimal, honest interface:

```java
public interface PaymentMethod {
    PaymentResult pay(Invoice invoice);
    PaymentMethodType type();
}
```

Implementations:

- `CreditCardPayment`
- `WalletPayment`
- `UpiPayment`

Checkout code stays polymorphic:

```java
public class CheckoutService {
    public Receipt checkout(Invoice invoice, PaymentMethod method) {
        PaymentResult result = method.pay(invoice);
        if (!result.isSuccess()) throw new PaymentFailedException(result.reason());
        return Receipt.from(invoice, result);
    }
}
```

Avoid putting subtype-specific methods (for example `saveCard()`) on `PaymentMethod`; separate those into narrower interfaces or services.

---

## 8) Design practice: Extensible Shape model

Use polymorphism to avoid central type switching:

```java
public interface Shape {
    double area();
}
```

Calculator:

```java
public final class AreaCalculator {
    public double totalArea(List<Shape> shapes) {
        return shapes.stream().mapToDouble(Shape::area).sum();
    }
}
```

Adding `Triangle` requires only a new implementation, not calculator changes.

If only some shapes support perimeter, prefer a separate capability interface:

```java
public interface HasPerimeter {
    double perimeter();
}
```

Do not use default methods that throw unsupported exceptions for non-applicable types.

---

## 9) Fragile base class risk

Default behavior in base classes can silently break subclasses when:

- Base implementation changes assumptions.
- New default methods conflict with subtype semantics.
- Subclasses inherit behavior they never intended.

Use small contracts and composition to reduce these ripple effects.

---

## 10) Self-check with answers

1. **Signal to replace inheritance with composition?**  
   A subclass must throw or no-op inherited methods, meaning contract mismatch.

2. **Polymorphism without saying parent/child?**  
   Clients call a contract; runtime picks concrete implementation behavior without type-branching in callers.

3. **Why risky to add defaults on a base type?**  
   Subclasses inherit changed behavior automatically, causing subtle regressions and contract drift.

---

## 11) Interview quick lines

- Inheritance is for specialization; composition is for assembling behavior.
- Polymorphism removes repeated branching and supports extension.
- Honest interfaces prevent LSP violations.
- If a subtype cannot support a base method naturally, the hierarchy is likely wrong.

---

## 12) Bridge to upcoming days

- **Day 3 (OCP):** polymorphic extension points reduce edits to existing logic.
- **Day 4 (LSP):** interface design must preserve substitutability.
- **Day 5:** composition helps when variation dimensions multiply.

---

## Day 2 checkpoint

- [x] I can identify when inheritance is semantically valid.
- [x] I can refactor type-condition chains into polymorphism.
- [x] I can keep contracts minimal and meaningful for all implementations.
- [x] I can explain why composition often outperforms deep inheritance.
