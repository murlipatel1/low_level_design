# Day 6 - Behavioral Patterns Part 2 and Week 2 Capstone

**Theme:** Complete the behavioral set and apply patterns in realistic mini-architectures.

---

## Learning goals

By the end of this document, you should be able to:

- Build request pipelines with Chain of Responsibility.
- Reuse fixed workflows with Template Method.
- Coordinate peer interactions through Mediator.
- Support undo/rollback through Memento.
- Add operations over stable hierarchies with Visitor.
- Evaluate small DSLs using Interpreter.

---

## 1) Chain of Responsibility

### Intent

Pass requests through handlers that may process, forward, or short-circuit.

### Typical use

- HTTP middleware chain
- Approval tiers
- Validation pipelines

### HTTP chain sketch

```java
public abstract class HttpHandler {
    private HttpHandler next;

    public HttpHandler setNext(HttpHandler next) {
        this.next = next;
        return next;
    }

    public final HttpResponse handle(HttpRequest request) {
        HttpResponse response = tryHandle(request);
        if (response != null) return response;
        if (next != null) return next.handle(request);
        return HttpResponse.notFound();
    }

    protected abstract HttpResponse tryHandle(HttpRequest request);
}
```

CoR vs Decorator: CoR can stop the chain early; decorator usually wraps and always delegates.

---

## 2) Template Method

### Intent

Define a stable algorithm skeleton in base class, let subclasses implement variable steps.

### Typical use

- ETL pipelines
- Test lifecycle hooks
- Parse workflows

### ETL template sketch

```java
public abstract class DataPipeline {
    public final PipelineResult run(PipelineContext ctx) {
        open(ctx);
        try {
            List<RawRecord> raw = extract(ctx);
            List<Record> transformed = transform(raw, ctx);
            load(transformed, ctx);
            return PipelineResult.success(transformed.size());
        } finally {
            close(ctx);
        }
    }

    protected void open(PipelineContext ctx) {}
    protected abstract List<RawRecord> extract(PipelineContext ctx);
    protected abstract List<Record> transform(List<RawRecord> raw, PipelineContext ctx);
    protected abstract void load(List<Record> data, PipelineContext ctx);
    protected void close(PipelineContext ctx) {}
}
```

Use Strategy instead when most workflow steps vary independently.

---

## 3) Mediator

### Intent

Centralize object-to-object interaction logic to reduce many-to-many coupling.

### Typical use

- Event bus
- Chat room coordination
- Complex UI dialog coordination

### Event bus sketch

```java
public interface EventHandler<T extends DomainEvent> {
    void handle(T event);
}

public final class EventBus {
    private final Map<Class<?>, List<EventHandler<?>>> handlers = new HashMap<>();

    public <T extends DomainEvent> void subscribe(Class<T> type, EventHandler<T> handler) {
        handlers.computeIfAbsent(type, k -> new ArrayList<>()).add(handler);
    }

    @SuppressWarnings("unchecked")
    public <T extends DomainEvent> void publish(T event) {
        List<EventHandler<?>> list = handlers.getOrDefault(event.getClass(), List.of());
        for (EventHandler<?> h : list) ((EventHandler<T>) h).handle(event);
    }
}
```

Mediator vs Observer: mediator routes and coordinates peers centrally; observer is often direct subject-listener notification.

---

## 4) Memento

### Intent

Capture internal state snapshots and restore later without exposing internals.

### Typical use

- Editor undo checkpoints
- Transaction rollback points
- Config rollback before risky operations

### Editor memento sketch

```java
public final class EditorMemento {
    private final String text;
    private final int cursor;

    EditorMemento(String text, int cursor) {
        this.text = text;
        this.cursor = cursor;
    }

    String text() { return text; }
    int cursor() { return cursor; }
}

public final class CodeEditor {
    private StringBuilder buffer = new StringBuilder();
    private int cursor;

    public EditorMemento save() {
        return new EditorMemento(buffer.toString(), cursor);
    }

    public void restore(EditorMemento m) {
        buffer = new StringBuilder(m.text());
        cursor = m.cursor();
    }
}
```

Command + Memento is a common undo pairing.

---

## 5) Visitor

### Intent

Add new operations over a stable object hierarchy without changing element classes.

### Typical use

- AST evaluation/printing/type-checking
- Pricing/tax/shipping passes over item hierarchies

### AST visitor sketch

```java
public interface Expr {
    <R> R accept(ExprVisitor<R> visitor);
}

public interface ExprVisitor<R> {
    R visitNumber(NumberNode n);
    R visitAdd(AddNode n);
}
```

Trade-off:

- Easy to add new visitors (operations).
- Costly to add new element types (must update all visitors).

---

## 6) Interpreter

### Intent

Represent grammar constructs as expressions and interpret/evaluate them.

### Typical use

- Small boolean/search/filter DSLs
- Lightweight rules engines

### Boolean expression sketch

```java
public interface BoolExpr {
    boolean interpret(Map<String, Boolean> env);
}

public final class Constant implements BoolExpr {
    private final boolean value;
    public Constant(boolean value) { this.value = value; }
    @Override public boolean interpret(Map<String, Boolean> env) { return value; }
}

public final class AndExpr implements BoolExpr {
    private final BoolExpr left, right;
    public AndExpr(BoolExpr left, BoolExpr right) { this.left = left; this.right = right; }
    @Override public boolean interpret(Map<String, Boolean> env) {
        return left.interpret(env) && right.interpret(env);
    }
}
```

Interpreter fits grammar evaluation; CoR fits staged validation workflows.

---

## 7) Week 2 capstones

Complete at least two with:

- UML-style sketch
- Pattern-to-class mapping sentence
- Trade-off paragraph
- 3 initial tests

### Capstone A: Logging framework (deep direction)

Pattern usage:

- Singleton/DI singleton for logger factory/registry
- CoR for level filtering to appender handling
- Strategy for formatter choice
- Decorator for async/buffered appenders
- Observer hooks for metrics/error events
- Template method in appender pipeline (`format -> write`)

### Capstone B: Rate limiter (deep direction)

Pattern usage:

- Strategy for token bucket vs sliding window
- Factory method for algorithm selection by config
- Singleton/registry for per-tenant limiter instances (per process)
- Decorator for logging/metrics around limiter

---

## 8) Full-week review answers

1. **Two patterns that wrap object and difference?**  
   Adapter changes interface; Decorator preserves interface and adds behavior; Proxy preserves interface and controls access.

2. **Why Visitor shifts OCP pressure to operations?**  
   New operation means new visitor (easy), but new element type requires touching every visitor (harder).

3. **Interpreter vs CoR for validation?**  
   Interpreter for rule language parsing/evaluation; CoR for sequential processing checks.

---

## 9) Week 2 to Week 3 bridge

Week 3 applies these patterns in complete systems (logging, rate limiting, cache, plugin architecture).  
Use this document’s capstone sketches as implementation blueprints.

---

## Day 6 checkpoint

- [x] I can model request pipelines with CoR.
- [x] I can identify when Template Method vs Strategy is appropriate.
- [x] I can use Mediator for peer decoupling.
- [x] I can pair Command and Memento for undo/rollback.
- [x] I can explain Visitor and Interpreter trade-offs clearly.
