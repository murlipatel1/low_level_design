# Day 3 - SOLID: SRP and OCP

**Theme:** One reason to change per unit, and extension without modifying stable core logic.

---

## Learning goals

By the end of this document, you should be able to:

- Explain SRP and OCP in practical LLD language.
- Refactor a god class into focused collaborators.
- Design extension points that absorb new variants without rewriting core flows.

---

## 1) Single Responsibility Principle (SRP)

SRP means a module should have one primary axis of change.  
This is not about counting methods; it is about who forces edits when requirements evolve.

### Typical SRP violation signals

- A class mixes authentication, persistence, and notifications.
- Unrelated changes trigger retesting of the same file.
- Multiple teams repeatedly conflict on one class.

### Practical refactor direction

- Split by change drivers (security, storage, messaging).
- Keep orchestration in a thin coordinator class.
- Push details into collaborators with narrow contracts.

---

## 2) Open/Closed Principle (OCP)

OCP means you add behavior by extension (new classes/strategies), not by editing stable core code.

### Common OCP mechanisms

- Interface polymorphism (`Shape`, `PaymentMethod`, `PipelineStep`)
- Strategy objects
- Startup-time registration

Balanced approach: do not over-engineer every possible extension point. Add OCP boundaries where variation is likely and frequent.

---

## 3) SRP case study: Refactor `UserManager`

If one class handles login, DB writes, and email alerts, it has multiple reasons to change.

### Responsibility split

| Responsibility | Suggested type | Typical change driver |
|----------------|----------------|-----------------------|
| Authentication | `Authenticator` | Auth/security policy |
| Persistence | `UserRepository` | Schema/storage changes |
| Notifications | `UserNotificationService` | Channel/template changes |
| Flow coordination | `UserService` | Use-case orchestration |

### Example orchestration

```java
public class UserService {
    private final Authenticator authenticator;
    private final UserRepository repository;
    private final UserNotificationService notifications;

    public Session login(LoginRequest request) {
        return authenticator.authenticate(request);
    }

    public User register(RegisterRequest request) {
        User user = User.create(request.email(), request.password());
        repository.save(user);
        notifications.sendWelcome(user);
        return user;
    }
}
```

Result: each unit changes independently, and tests become more focused.

---

## 4) OCP case study: Shape area calculator

Bad design uses `switch(shapeType)` in calculator logic.  
Every new shape edits existing logic, which breaks OCP.

### OCP-compliant model

```java
public interface Shape {
    double area();
}
```

```java
public final class AreaCalculator {
    public double totalArea(List<Shape> shapes) {
        return shapes.stream().mapToDouble(Shape::area).sum();
    }
}
```

Adding `RegularHexagon` or `RegularPentagon` requires only new `Shape` implementations.  
`AreaCalculator` remains untouched.

---

## 5) SRP + OCP case study: CI/CD pipeline

A `BuildPipeline` doing checkout, compile, test, upload, and Slack notification in one class violates SRP.

Split each stage into a narrow step:

| Stage | Type |
|-------|------|
| Checkout | `SourceCheckoutStep` |
| Compile | `CompileStep` |
| Test | `TestStep` |
| Upload | `ArtifactUploadStep` |
| Notify | `SlackNotificationStep` |

Shared contract:

```java
public interface PipelineStep {
    StepResult execute(PipelineContext context);
}
```

Thin orchestrator:

```java
public class BuildPipeline {
    private final List<PipelineStep> steps;

    public PipelineResult run(PipelineContext context) {
        for (PipelineStep step : steps) {
            StepResult result = step.execute(context);
            if (result.failed()) return PipelineResult.failed(step.name(), result.message());
        }
        return PipelineResult.success();
    }
}
```

OCP benefit: adding a new step (for example `DockerPublishStep`) is additive, not invasive.

---

## 6) Stable core vs extension point: Report generator

Scenario: today PDF output, next quarter HTML.

Stable core:

- Data collection and report model construction
- Generation orchestration flow

Extension point:

```java
public interface ReportRenderer {
    byte[] render(ReportModel model);
}
```

Implementations:

- `PdfReportRenderer`
- `HtmlReportRenderer`

Keep `ReportGenerator` dependent on `ReportRenderer`; avoid `if (format == ...)` in core logic.

---

## 7) SRP and OCP together

- SRP produces small cohesive parts.
- OCP lets those parts grow through new implementations.
- Combined effect: stable orchestrators and scalable change management.

---

## 8) When SRP over-splitting hurts

Avoid splitting tiny, cohesive code into many micro-classes before real variation appears.  
If rules always change together, excessive fragmentation adds complexity without reducing change risk.

Use SRP where you observe independent stakeholders, independent release cycles, or repeated conflict/blast radius.

---

## 9) Self-check with answers

1. **Two reasons one class might change?**  
   Example: auth policy updates and email template changes. If both hit one class, SRP is stressed.

2. **Why does naive `switch(shapeType)` violate OCP?**  
   Every new shape modifies stable existing logic instead of extending via new types.

3. **One case where early SRP split is harmful?**  
   Breaking a small validator into many one-method classes when all rules evolve together.

---

## 10) Interview quick lines

- SRP is one reason to change, not one method per class.
- OCP means extension by new implementations, not edits to tested core.
- Orchestrators should coordinate, not implement every concern directly.
- Prefer adding new classes over extending large conditional logic blocks.

---

## 11) Bridge to next days

- **Day 4 (LSP):** OCP depends on substitutable implementations.
- **Day 5 (DIP):** SRP/OCP designs become stronger when orchestrators depend on abstractions.

---

## Day 3 checkpoint

- [x] I can split mixed-responsibility classes by change driver.
- [x] I can design extension points for likely variants.
- [x] I can keep core orchestrators stable while adding new behavior.
