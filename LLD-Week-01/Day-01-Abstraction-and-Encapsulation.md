# Day 1 - Abstraction and Encapsulation

**Theme:** Hide complexity and protect invariants.  
These two principles make systems maintainable before inheritance and design patterns enter the picture.

---

## Learning goals

By the end of this document, you should be able to:

- Explain abstraction as what an object exposes vs how it works internally.
- Explain encapsulation as state + behavior kept behind a controlled boundary.
- Design small APIs that do not leak internal representation.
- Identify leaky abstractions and anemic domain models in LLD discussions.

---

## 1) Abstraction

Abstraction is the simplified view offered to callers: meaningful operations and data they need, without exposing implementation details.

- **Data abstraction:** expose behavior like `account.withdraw(amount)` instead of allowing direct field mutation.
- **Procedural abstraction:** use intent-focused operations like `calculateShipping()` instead of exposing step-by-step internals.

### LLD perspective

In low-level design interviews, abstraction appears in:

- Interface design
- Public method choices
- Domain-oriented naming

Good abstraction matches how the system talks about the problem, not how code is stored internally.

### Common abstraction mistakes

- A class exposing too many unrelated methods.
- Returning internal mutable references.
- Naming by framework details instead of domain concepts.

### Quick quality checklist

- Public API reads like valid business actions.
- Illegal states cannot be produced through normal usage.
- Internals can change without breaking callers.

---

## 2) Encapsulation

Encapsulation means packaging state and behavior together and restricting direct state mutation so rules stay centralized.

Language features differ (`private`, module boundaries, properties), but the design goal is always the same: **one authority per invariant**.

### Typical invariants

- Balance cannot be negative after a successful debit.
- Email is normalized before persistence.
- Order total always matches line-item calculations.

Encapsulation ensures these rules are enforced through methods, not scattered across controllers or services.

### Abstraction vs encapsulation

| Concept | Core question |
|--------|----------------|
| Abstraction | What do I show the world? |
| Encapsulation | How do I protect and evolve internals safely? |

They reinforce each other: clean abstraction usually requires strong encapsulation.

---

## 3) Case study: BankAccount invariants and API design

A strong `BankAccount` model should enforce at least these invariants:

| Invariant | Why it matters | Method-level enforcement |
|-----------|----------------|--------------------------|
| Balance never goes negative on successful debit | Prevents overdrafts and inconsistent ledgers | `withdraw(amount)` validates amount and available balance |
| Monetary values use one safe representation | Avoids floating-point rounding bugs | Accept `Money`/validated decimal types, avoid raw `double` fields |
| State transitions are legal | Protects lifecycle and compliance behavior | `deposit()`/`withdraw()` check account state before mutation |

Behavior-rich API shape:

- `deposit(Money amount)`
- `withdraw(Money amount)`
- `getBalance()` returning immutable value/snapshot
- Optional `transferTo(BankAccount other, Money amount)` for rule centralization

Design principle: callers should never perform direct balance arithmetic.

---

## 4) Case study: From anemic User model to encapsulated model

When `email`, `passwordHash`, and `role` are public:

- Security checks are bypassed.
- Data normalization becomes inconsistent.
- Business rules spread across layers.
- Refactors become expensive due to widespread field coupling.

An encapsulated model exposes intent-driven behavior:

```java
public final class User {
    private final UserId id;
    private Email email;
    private PasswordHash passwordHash;
    private Role role;

    public User(UserId id, Email email, PasswordHash passwordHash, Role role) { ... }

    public Email getEmail() { return email; }

    public boolean authenticate(String plainPassword) {
        return passwordHash.matches(plainPassword);
    }

    public void changeEmail(Email newEmail) {
        this.email = newEmail;
    }

    public void promoteTo(Role newRole, AuthorizationContext ctx) {
        if (!ctx.canAssignRole(newRole)) throw new ForbiddenException();
        this.role = newRole;
    }
}
```

This keeps validation and authorization inside domain behavior rather than in scattered callers.

---

## 5) Getters/setters vs behavior-rich methods

| Style | Best use case | Main trade-off |
|-------|---------------|----------------|
| Getters/setters for most fields | DTOs, boundary payloads, framework mapping objects | Fast to write, but can create anemic domain logic |
| Behavior-rich methods | Core domain objects with invariants | Higher upfront design effort, much stronger consistency |

Rule of thumb: if a change has a rule ("only if", "must equal", "cannot when"), use a method, not a raw setter.

---

## 6) Design pattern practice: AppConfig abstraction

Create a stable configuration interface so callers request typed values without knowing source details.

```java
public interface AppConfig {
    String getString(String key);
    int getInt(String key);
    Duration getDuration(String key);
    boolean getBoolean(String key, boolean defaultValue);
    Optional<String> getOptional(String key);
}
```

Keep these details internal:

- `.env`/YAML/cloud parameter source adapters
- Precedence and merge rules
- Parsing, caching, reload logic
- Raw source maps and client handles

This enables swapping configuration backends without changing call sites.

---

## 7) Design pattern practice: RateLimiter facade

Expose the minimum contract services actually need:

```java
public interface RateLimiter {
    boolean tryAcquire(String key, int cost);
    boolean wouldAllow(String key, int cost);
}
```

Do not expose:

- Bucket/window internals
- Threading/clock infrastructure
- Redis keys/sharding details
- Locking or scheduler choices

This preserves freedom to replace algorithms or infrastructure with no API churn.

---

## 8) Leaky abstraction example and refactor

### Leaky design

```java
public class Order {
    public List<LineItem> items = new ArrayList<>();
    public double discountPercent;
    public String status;
}
```

Problems:

- External mutation of critical state
- Invalid discounts possible
- Lifecycle rules unenforced

### Encapsulated design

```java
public class Order {
    private final List<LineItem> items = new ArrayList<>();
    private BigDecimal discountPercent = BigDecimal.ZERO;
    private OrderStatus status = OrderStatus.DRAFT;

    public void addLineItem(LineItem item) { ... }
    public void applyDiscount(BigDecimal percent) { ... }
    public void confirm() { ... }
    public Money total() { ... }
    public List<LineItem> lineItemsSnapshot() { return List.copyOf(items); }
}
```

Result: the public API represents business actions while internals remain protected and replaceable.

---

## 9) Self-check questions with answers

1. **Where does strong encapsulation help refactoring most?**  
   When internal data structure changes (for example `ArrayList` to another storage model) but callers rely only on behavior methods.

2. **Why is returning a private list reference unsafe?**  
   Callers can mutate object internals directly, bypassing validation and violating invariants.

3. **How does abstraction improve testability?**  
   Small interfaces let tests inject fakes/mocks and verify behavior via contract, not internal implementation details.

---

## 10) Interview quick-recall lines

- Abstraction is what the system can do; encapsulation is how state stays safe while doing it.
- Public mutable fields are usually a signal that invariants are not protected.
- DTOs can be data-centric; domain entities should usually be behavior-centric.
- Returning mutable internals is a classic encapsulation leak.

---

## 11) Bridge to Week 1

- **SRP (Day 3):** weak encapsulation usually leads to classes with mixed responsibilities.
- **LSP (Day 4):** honest abstractions reduce subtype surprises.
- **DIP (Day 5):** stable interfaces (`AppConfig`, `RateLimiter`) make internals swappable.

---

## Day 1 checkpoint

- [x] I can explain abstraction vs encapsulation clearly.
- [x] I can identify and fix leaky object APIs.
- [x] I can design interfaces that hide implementation details.
