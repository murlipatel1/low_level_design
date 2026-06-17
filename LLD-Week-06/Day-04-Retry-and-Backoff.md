# Day 4 — Retry and Backoff Mechanism

This lesson covers resilient retry design with configurable policies, exponential backoff plus jitter, absolute deadlines, and safe composition with circuit breaker.

---

## Learning objectives

- Build a reusable `RetryExecutor` separate from business logic.
- Support fixed, exponential, and exponential-with-jitter delay strategies.
- Enforce both max attempts and global deadline.
- Compose retries with circuit breaker without creating retry storms.

---

## Retry strategy summary

| Strategy | Delay behavior |
|---|---|
| Fixed | `delay = fixedMs` |
| Exponential | `delay = base * multiplier^(attempt-1)` |
| Exponential + jitter | randomize delay within cap (recommended) |

For this design, we use **full jitter**:

`delay = random(0, min(maxDelay, base * multiplier^(attempt-1)))`

---

## Core design principles

- Retry only transient failures (timeouts, network glitches, 429, etc.).
- Never retry non-idempotent writes unless idempotency key/guard is present.
- Keep retry policy isolated as infrastructure concern.
- Apply overall deadline across all attempts, not just per-attempt timeout.

---

## `RetryExecutor` design

### Config

```java
public record RetryConfig(
    int maxAttempts,
    long baseDelayMs,
    double backoffMultiplier,
    long maxDelayMs,
    Duration deadline,
    Set<Class<? extends Throwable>> retryableExceptions,
    Predicate<Throwable> retryPredicate
) {
    public boolean isRetryable(Throwable t) {
        if (retryPredicate != null && retryPredicate.test(t)) return true;
        for (Class<? extends Throwable> c : retryableExceptions) {
            if (c.isInstance(t)) return true;
        }
        return false;
    }
}
```

### Executor

```java
public final class RetryExecutor {
    private final RetryConfig config;
    private final BackoffStrategy backoff;

    public RetryExecutor(RetryConfig config, BackoffStrategy backoff) {
        this.config = config;
        this.backoff = backoff;
    }

    public <T> T execute(Callable<T> action) throws Exception {
        long deadlineNanos = config.deadline().isZero()
            ? Long.MAX_VALUE
            : System.nanoTime() + config.deadline().toNanos();

        Throwable last = null;
        for (int attempt = 1; attempt <= config.maxAttempts(); attempt++) {
            if (System.nanoTime() > deadlineNanos) {
                throw new RetryDeadlineExceededException(last);
            }
            try {
                return action.call();
            } catch (Throwable t) {
                last = t;
                if (!config.isRetryable(t) || attempt == config.maxAttempts()) {
                    throw wrap(t);
                }
                long sleepMs = backoff.delayMs(attempt);
                sleepRespectingDeadline(sleepMs, deadlineNanos);
            }
        }
        throw new IllegalStateException("unreachable");
    }

    private void sleepRespectingDeadline(long sleepMs, long deadlineNanos)
            throws InterruptedException, RetryDeadlineExceededException {
        long remainingMs = TimeUnit.NANOSECONDS.toMillis(deadlineNanos - System.nanoTime());
        if (remainingMs <= 0) throw new RetryDeadlineExceededException(null);
        Thread.sleep(Math.min(sleepMs, remainingMs));
    }

    private Exception wrap(Throwable t) {
        return t instanceof Exception e ? e : new RuntimeException(t);
    }
}
```

### Backoff strategy implementations

```java
public interface BackoffStrategy {
    long delayMs(int attempt);
}

public final class FixedBackoff implements BackoffStrategy {
    private final long fixedMs;
    public FixedBackoff(long fixedMs) { this.fixedMs = fixedMs; }
    public long delayMs(int attempt) { return fixedMs; }
}

public final class ExponentialBackoffWithJitter implements BackoffStrategy {
    private final RetryConfig config;
    public ExponentialBackoffWithJitter(RetryConfig config) { this.config = config; }

    public long delayMs(int attempt) {
        double cap = config.baseDelayMs() * Math.pow(config.backoffMultiplier(), attempt - 1);
        cap = Math.min(config.maxDelayMs(), cap);
        return (long) (ThreadLocalRandom.current().nextDouble() * cap); // full jitter
    }
}
```

---

## Attempt table example (deliverable)

Assume `base=100ms`, `multiplier=2`, `maxDelay=5000ms`.

| Attempt | Raw exponential cap | Example jittered delay |
|---|---|---|
| 1 | 100ms | 38ms |
| 2 | 200ms | 141ms |
| 3 | 400ms | 287ms |
| 4 | 800ms | 652ms |
| 5 | 1600ms | 1201ms |

Without jitter, many clients retry in lockstep and overload recovering dependencies.

---

## Retry decision flow (deliverable)

```mermaid
flowchart TD
  A[Execute action] --> B{Success?}
  B -->|Yes| Z[Return result]
  B -->|No| C{Retryable exception?}
  C -->|No| X[Throw now]
  C -->|Yes| D{Attempts left?}
  D -->|No| X
  D -->|Yes| E{Deadline remaining?}
  E -->|No| Y[Throw deadline exceeded]
  E -->|Yes| F[Compute jittered delay]
  F --> G[Sleep min(delay, remaining)]
  G --> A
```

---

## Retry storm explanation (deliverable)

After a shared outage, synchronized retries can hammer a just-recovering dependency and cause secondary failure. Jitter spreads retry timings, while circuit breaker limits calls during unhealthy periods and probes recovery gradually.

---

## Compose with circuit breaker

Recommended layering:

`CircuitBreaker.execute(() -> RetryExecutor.execute(() -> downstream.call()))`

Policy by breaker state:

- **CLOSED**: retries allowed per retry policy.
- **OPEN**: fail immediately (no retry budget burn).
- **HALF_OPEN**: usually no retries; keep it as a controlled probe.

```java
public <T> T executeWithBreaker(Callable<T> action) throws Exception {
    if (breaker.state() == BreakerState.OPEN) throw new CircuitOpenException();
    if (breaker.state() == BreakerState.HALF_OPEN) return breaker.execute(action);
    return breaker.execute(() -> retryExecutor.execute(action));
}
```

---

## Timeout vs deadline composition

Per-attempt timeout and global deadline must be coordinated:

- Per-attempt timeout limits one call.
- Deadline caps end-to-end latency budget across all retries.

Always use `min(perAttemptTimeout, deadlineRemaining)` before each attempt.

---

## Non-idempotent retry caution

Unsafe example: payment transfer `POST` retried without idempotency key may double-charge.

Safe patterns:

- Idempotency key persisted server-side.
- Upsert/PUT semantics with stable request id.
- Outbox/saga style guaranteed processing flow.

---

## Self-quiz with answers

1. **Why careful composition of retry and timeout?**  
   Without global deadline enforcement, retries can exceed SLA and tie up resources indefinitely.

2. **Example of unsafe non-idempotent retry?**  
   Money transfer or order placement POST without dedupe key can create duplicates.

3. **When disable retries in half-open?**  
   Almost always; half-open should be a limited probe, not a mini storm.

---

## First three tests

1. Non-retryable exception path exits after first failure with no sleep.
2. Deadline test stops retries before exhausting attempts when time budget is exceeded.
3. Breaker OPEN test confirms retry executor is not invoked.

---

## Interview sound bites

- "Retry only transient and idempotent operations."
- "Use full jitter to prevent synchronized retry spikes."
- "Deadlines are end-to-end, not per-attempt only."
- "Breaker OPEN must short-circuit retries."

---

## Day 4 checkpoint

- [x] Configurable retry executor
- [x] Backoff strategy variants with jitter
- [x] Deadline and retryability rules
- [x] Circuit-breaker composition policy
- [x] Self-quiz and tests

**Next:** `Day-05-Distributed-ID-Snowflake.md`
