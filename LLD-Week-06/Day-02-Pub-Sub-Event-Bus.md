# Day 2 — Pub-Sub Event Bus (In-Memory)

This lesson builds a typed in-memory event bus with async dispatch, retry with backoff, dead-letter handling, predicate subscriptions, and ordering trade-offs.

---

## Learning objectives

- Define type-safe event bus contracts.
- Dispatch handlers asynchronously with clear isolation semantics.
- Add retry and DLQ for poison handlers.
- Support filtered subscriptions and reason about in-flight unsubscribe races.

---

## Core interfaces

```java
public interface Event {
    String eventId();
    Instant occurredAt();
    String correlationId();
}

public interface EventHandler<T extends Event> {
    void handle(T event) throws Exception;
}

public interface IEventBus {
    <T extends Event> void publish(T event);
    <T extends Event> void subscribe(Class<T> type, EventHandler<T> handler);
    <T extends Event> void subscribe(Class<T> type, Predicate<T> filter, EventHandler<T> handler);
    <T extends Event> void unsubscribe(Class<T> type, EventHandler<T> handler);
    void shutdown(long timeout, TimeUnit unit);
}
```

Design choices for this version:

- Exact runtime event class matching (no inheritance walk).
- Per-handler task isolation (one handler failure does not block others).
- No strict ordering guarantee unless explicitly partitioned.
- Bounded executor queue for backpressure safety.

---

## Async `InMemoryEventBus`

```java
public final class InMemoryEventBus implements IEventBus {
    private final ConcurrentHashMap<Class<?>, CopyOnWriteArrayList<Registration<?>>> subscribers =
        new ConcurrentHashMap<>();
    private final ExecutorService executor;
    private final HandlerRetryPolicy retryPolicy;
    private final AtomicBoolean closed = new AtomicBoolean(false);

    public InMemoryEventBus(int poolSize, int queueCapacity, HandlerRetryPolicy retryPolicy) {
        this.executor = new ThreadPoolExecutor(
            poolSize, poolSize,
            0L, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<>(queueCapacity),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
        this.retryPolicy = retryPolicy;
    }

    @Override
    public <T extends Event> void subscribe(Class<T> type, EventHandler<T> handler) {
        subscribe(type, e -> true, handler);
    }

    @Override
    public <T extends Event> void subscribe(
            Class<T> type,
            Predicate<T> filter,
            EventHandler<T> handler) {
        subscribers.computeIfAbsent(type, k -> new CopyOnWriteArrayList<>())
            .add(new Registration<>(handler, filter));
    }

    @Override
    public <T extends Event> void unsubscribe(Class<T> type, EventHandler<T> handler) {
        List<Registration<?>> regs = subscribers.get(type);
        if (regs != null) regs.removeIf(r -> r.handler == handler);
    }

    @Override
    public <T extends Event> void publish(T event) {
        if (closed.get()) throw new IllegalStateException("event bus is shutdown");
        List<Registration<?>> regs = subscribers.getOrDefault(event.getClass(), new CopyOnWriteArrayList<>());
        for (Registration<?> reg : regs) {
            executor.submit(() -> dispatchOne(event, reg));
        }
    }

    @SuppressWarnings("unchecked")
    private <T extends Event> void dispatchOne(T event, Registration<?> reg) {
        Registration<T> typed = (Registration<T>) reg;
        if (!typed.filter.test(event)) return;
        retryPolicy.execute(() -> typed.handler.handle(event), event);
    }

    @Override
    public void shutdown(long timeout, TimeUnit unit) {
        closed.set(true);
        executor.shutdown();
        try {
            if (!executor.awaitTermination(timeout, unit)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }

    private record Registration<T extends Event>(EventHandler<T> handler, Predicate<T> filter) {}
}
```

---

## Retry with backoff and DLQ

```java
public final class HandlerRetryPolicy {
    private final int maxAttempts = 3;
    private final DeadLetterQueue dlq;

    public HandlerRetryPolicy(DeadLetterQueue dlq) {
        this.dlq = dlq;
    }

    public void execute(HandlerTask task, Event event) {
        int attempt = 0;
        while (true) {
            try {
                task.run();
                return;
            } catch (Exception e) {
                attempt++;
                if (attempt >= maxAttempts) {
                    dlq.enqueue(new DeadLetterEvent(
                        event,
                        event.getClass().getName(),
                        e.getMessage(),
                        stackTraceOf(e),
                        event.correlationId(),
                        Instant.now(),
                        attempt
                    ));
                    return;
                }
                backoffSleep(attempt);
            }
        }
    }

    private void backoffSleep(int attempt) {
        long delayMs = (long) (100 * Math.pow(2, attempt - 1));
        try {
            Thread.sleep(delayMs);
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt();
        }
    }
}
```

```java
public record DeadLetterEvent(
    Event originalEvent,
    String eventType,
    String errorMessage,
    String stackTrace,
    String correlationId,
    Instant failedAt,
    int attempts
) implements Event {
    @Override public String eventId() { return "dlq-" + originalEvent.eventId(); }
    @Override public Instant occurredAt() { return failedAt; }
}
```

DLQ metadata should include event id, correlation id, handler context, attempts, and failure details for safe replay and debugging.

---

## Predicate subscriptions

```java
bus.subscribe(
    OrderCreated.class,
    e -> e.total().compareTo(Money.of(10_000)) > 0,
    e -> fraudService.review(e)
);

bus.subscribe(OrderCreated.class, e -> inventoryService.reserve(e));
```

Use lightweight predicates only; complex filtering belongs in dedicated handlers.

---

## Thread model (mental model)

- Publisher thread calls `publish` and returns after enqueue.
- Worker pool executes handlers concurrently.
- Retry logic runs inside worker task.
- Final failures are written to DLQ.
- Optional replay worker consumes DLQ and republishes after fixes.

---

## Ordering and partitioning

Naive shared thread pools can reorder same-aggregate events (for example `OrderCancelled` before `OrderPlaced` handling completion).

When ordering matters:

- Partition dispatch by aggregate key (`hash(aggregateId) % N`).
- Use single-threaded shard executors.

```java
int shard = Math.floorMod(aggregateId.hashCode(), shardCount);
shardExecutors[shard].submit(() -> handler.handle(event));
```

---

## Unsubscribe and in-flight events

Using `CopyOnWriteArrayList` gives safe iteration snapshots:

- Unsubscribe prevents future dispatch selection.
- Already queued/running tasks may still execute.
- Hard stop requires bus shutdown and executor drain.

---

## Idempotency expectations for subscribers

Because retry means at-least-once delivery:

1. Deduplicate by `eventId`.
2. Prefer naturally idempotent side effects.
3. Guard non-idempotent effects (email/payment) with dedupe keys.

---

## Self-quiz with answers

1. **Why does ordering per aggregate matter, and how can pool dispatch break it?**  
   Many domains require sequential state transitions; parallel worker execution can reorder completion unless partitioned.

2. **What metadata is required in DLQ for safe replay?**  
   Original payload, event id/correlation id, event type, handler identity, attempts, failure time, and error trace.

3. **How to avoid unsubscribe races?**  
   Snapshot-safe subscriber structures plus the rule that in-flight tasks may finish; unsubscribe blocks only future dispatches.

---

## First three tests

1. Handler isolation: handler A fails, handler B still runs for same event.
2. Predicate routing: high-value order triggers fraud handler, low-value does not.
3. Retry + DLQ: always-failing handler lands exactly one DLQ entry after max attempts.

---

## Interview sound bites

- "Per-handler async tasks isolate failures."
- "Retry + DLQ implies at-least-once semantics, so handlers must be idempotent."
- "Ordering is not free; use key partitioning when aggregate sequence matters."
- "Bounded executor queues are part of backpressure design."

---

## Day 2 checkpoint

- [x] Typed async event bus structure
- [x] Retry/backoff and DLQ flow
- [x] Predicate subscription API
- [x] Ordering, unsubscribe, and idempotency trade-offs
- [x] Self-quiz and baseline tests

**Next:** `Day-03-Circuit-Breaker.md`
