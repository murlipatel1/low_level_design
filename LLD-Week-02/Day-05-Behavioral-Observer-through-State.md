# Day 5 - Behavioral Patterns: Observer through State

**Theme:** Model collaboration, runtime behavior changes, and workflow control between objects.

---

## Learning goals

By the end of this document, you should be able to:

- Use Observer for one-to-many event notifications.
- Use Strategy for swappable algorithms.
- Use Command for executable requests with queue/undo support.
- Use Iterator to hide traversal/pagination internals.
- Use State for lifecycle-driven behavior transitions.

---

## 1) Observer

### Intent

Subject notifies multiple observers when state changes.

### Typical use

- Build status fanout to Slack/email/PR checks
- Metrics threshold alerts
- Stock/ticker subscriptions

### Core sketch

```java
public interface BuildListener {
    void onStatusChanged(BuildEvent event);
}

public final class BuildSystem {
    private final List<BuildListener> listeners = new ArrayList<>();
    private BuildStatus status = BuildStatus.RUNNING;

    public void addListener(BuildListener listener) { listeners.add(listener); }

    public void complete(boolean success) {
        status = success ? BuildStatus.SUCCESS : BuildStatus.FAILED;
        BuildEvent event = new BuildEvent(status, "build-42");
        listeners.forEach(l -> l.onStatusChanged(event));
    }
}
```

### Practical guardrails

- Use reentrancy guard or async queue to avoid recursive notify loops.
- Keep event payload immutable.
- Define observer order expectations explicitly if ordering matters.

---

## 2) Strategy

### Intent

Encapsulate interchangeable algorithms behind one interface.

### Typical use

- Compression selection (gzip/zstd/brotli)
- Authentication modes
- Load balancing rules

### Authentication strategy sketch

```java
public interface AuthStrategy {
    AuthResult authenticate(Credentials credentials);
    AuthMethod method();
}

public final class AuthenticationService {
    private final Map<AuthMethod, AuthStrategy> strategies;

    public AuthenticationService(List<AuthStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(AuthStrategy::method, s -> s));
    }

    public AuthResult authenticate(AuthMethod method, Credentials creds) {
        AuthStrategy strategy = strategies.get(method);
        if (strategy == null) throw new UnsupportedAuthMethodException(method);
        return strategy.authenticate(creds);
    }
}
```

Strategy vs State: strategy is usually externally chosen policy; state is internally transitioned lifecycle behavior.

---

## 3) Command

### Intent

Represent operations as objects so they can be queued, logged, retried, or undone.

### Typical use

- Editor undo/redo
- Job queue workers
- Migration up/down scripts

### Editor command sketch

```java
public interface EditorCommand {
    void execute();
    void undo();
}

public final class InsertTextCommand implements EditorCommand {
    private final TextBuffer buffer;
    private final int position;
    private final String text;

    public InsertTextCommand(TextBuffer buffer, int position, String text) {
        this.buffer = buffer;
        this.position = position;
        this.text = text;
    }

    @Override public void execute() { buffer.insert(position, text); }
    @Override public void undo() { buffer.delete(position, text.length()); }
}
```

Memento pairing: useful when reverse operation logic is hard; command can restore captured snapshot.

---

## 4) Iterator

### Intent

Provide sequential access without exposing internal structure.

### Typical use

- Paginated APIs
- Tree traversal
- Large log streams

### Paginated iterator sketch

```java
public final class PaginatedApiIterator<T> implements Iterator<T> {
    private final PageFetcher<T> fetcher;
    private Iterator<T> currentPage = List.<T>of().iterator();
    private int nextPageToken = 0;
    private boolean exhausted;

    public interface PageFetcher<T> {
        Page<T> fetch(int pageToken);
    }

    public record Page<T>(List<T> items, int nextToken, boolean hasMore) {}

    public PaginatedApiIterator(PageFetcher<T> fetcher) {
        this.fetcher = fetcher;
    }

    @Override
    public boolean hasNext() {
        if (currentPage.hasNext()) return true;
        if (exhausted) return false;
        loadNextPage();
        return currentPage.hasNext();
    }

    @Override
    public T next() {
        if (!hasNext()) throw new NoSuchElementException();
        return currentPage.next();
    }

    private void loadNextPage() {
        Page<T> page = fetcher.fetch(nextPageToken);
        currentPage = page.items().iterator();
        if (page.hasMore()) nextPageToken = page.nextToken();
        else exhausted = true;
    }
}
```

Caller sees plain iteration, not pagination mechanics.

---

## 5) State

### Intent

Context delegates behavior to current state object; transitions change behavior without giant enum switches.

### Typical use

- Order lifecycle
- Traffic light transitions
- Pipeline/job lifecycle

### Order state sketch

```java
public interface OrderState {
    void confirm(Order order);
    void pack(Order order);
    void ship(Order order);
    void deliver(Order order);
    String name();
}

public final class PlacedState implements OrderState {
    @Override public void confirm(Order order) { order.setState(new ConfirmedState()); }
    @Override public void pack(Order order) { throw new IllegalStateException("not confirmed"); }
    @Override public void ship(Order order) { throw new IllegalStateException(); }
    @Override public void deliver(Order order) { throw new IllegalStateException(); }
    @Override public String name() { return "PLACED"; }
}
```

State vs Strategy: in State, transition logic is part of domain lifecycle and often managed by context/state objects themselves.

---

## 6) Mixed design: Build pipeline events + routing

Pattern roles in one system:

- **Observer:** `BuildJob` -> `SlackNotifier`, `EmailNotifier`, `MetricsObserver`
- **Strategy:** `ArtifactUploadStrategy` with `S3UploadStrategy` / `GcsUploadStrategy`
- **Command:** `DeployCommand`, `RunTestsCommand`, `ScanCommand` queued to worker
- **State (stretch):** `QueuedState`, `RunningState`, `FailedState`, `PassedState`

This gives event fanout, pluggable cloud behavior, queue-friendly workflow steps, and explicit lifecycle control.

---

## 7) Self-check with answers

1. **How to prevent infinite Observer notify loops?**  
   Use reentrancy guards, async event queues, immutable events, and clear callback constraints.

2. **Where does Memento help in Command undo?**  
   When inverse logic is complex, restore pre-execution snapshot instead of computing exact reverse operations.

3. **State vs Strategy: who chooses active object?**  
   Strategy is typically selected externally (config/client); State is typically transitioned by context/domain events.

---

## 8) Interview quick lines

- Observer: one subject, many listeners.
- Strategy: replace conditional algorithm selection with polymorphism.
- Command: operation object supports queue/history/undo.
- Iterator: traversal abstraction without leaking structure.
- State: lifecycle behavior encoded in state classes.

---

## Day 5 checkpoint

- [x] I can model observer subscriptions and safe notifications.
- [x] I can design strategy registries for runtime algorithm selection.
- [x] I can use command objects for undo and queue processing.
- [x] I can hide paging/traversal internals with iterators.
- [x] I can implement state-driven lifecycle transitions cleanly.
