# Day 2 - Rate Limiter Service

**Theme:** Design a configurable, thread-safe, extensible limiter for API gateways and backend services.

---

## Learning goals

By the end of this document, you should be able to:

- Design one limiter interface with multiple algorithm implementations.
- Handle keys by endpoint and identity (API key/user/IP).
- Explain concurrency strategy and distributed extension path.
- Return useful deny semantics (`429` + `Retry-After`).

---

## 1) Clarifying decisions

For this design, we choose:

1. In-process core implementation; Redis as distributed extension.
2. Deny responses include retry-after duration.
3. Composite key uses endpoint + identity dimensions.
4. Burst and sustained rate are separately configurable.
5. Metrics/logging are optional wrappers, not embedded in algorithms.

---

## 2) Core model and contracts

```java
public enum RateLimitAlgorithm {
    TOKEN_BUCKET, LEAKY_BUCKET, FIXED_WINDOW, SLIDING_WINDOW
}

public final class RateLimitRule {
    private final RateLimitAlgorithm algorithm;
    private final long limit;
    private final Duration window;
    private final long burstCapacity;
    private final double refillPerSecond;
    private final String endpointPattern;
    // constructor/getters
}

public final class ClientIdentifier {
    public enum Type { USER_ID, API_KEY, IP }
    private final Type type;
    private final String value;
    // constructor/getters
}

public final class RateLimitContext {
    private final ClientIdentifier client;
    private final String endpoint;
    private final Instant now;
    private final int cost;

    public String compositeKey() {
        return endpoint + "|" + client.type() + ":" + client.value();
    }
}

public final class RateLimitResult {
    private final boolean allowed;
    private final long remaining;
    private final long retryAfterMs;

    public static RateLimitResult allow(long remaining) {
        return new RateLimitResult(true, remaining, 0);
    }

    public static RateLimitResult deny(long retryAfterMs) {
        return new RateLimitResult(false, 0, retryAfterMs);
    }
}

public interface IRateLimiter {
    RateLimitResult tryAcquire(RateLimitContext ctx);
}
```

---

## 3) Pattern mapping

| Pattern | Role |
|---------|------|
| Strategy | Algorithm selection (`TokenBucket`, `SlidingWindow`, etc.) |
| Factory | Build limiter from rule config |
| Singleton/DI singleton | Registry lifecycle and shared rule/state hub |
| Decorator | Add logging/metrics without touching limiter math |
| Facade (optional) | HTTP-friendly `RateLimitService` wrapper |

---

## 4) Token bucket (full implementation)

```java
public final class TokenBucketLimiter implements IRateLimiter {
    private final long capacity;
    private final double refillPerSecond;
    private final Object lock = new Object();

    private double tokens;
    private long lastRefillEpochMs;

    public TokenBucketLimiter(long capacity, double refillPerSecond) {
        this.capacity = capacity;
        this.refillPerSecond = refillPerSecond;
        this.tokens = capacity;
        this.lastRefillEpochMs = System.currentTimeMillis();
    }

    @Override
    public RateLimitResult tryAcquire(RateLimitContext ctx) {
        int cost = ctx.cost();
        synchronized (lock) {
            refill(ctx.now().toEpochMilli());
            if (tokens >= cost) {
                tokens -= cost;
                return RateLimitResult.allow((long) tokens);
            }
            double deficit = cost - tokens;
            long retryMs = (long) Math.ceil(deficit / refillPerSecond * 1000);
            return RateLimitResult.deny(retryMs);
        }
    }

    private void refill(long nowMs) {
        double elapsedSec = (nowMs - lastRefillEpochMs) / 1000.0;
        tokens = Math.min(capacity, tokens + elapsedSec * refillPerSecond);
        lastRefillEpochMs = nowMs;
    }
}
```

Burst behavior: `capacity` controls burst size, `refillPerSecond` controls sustained throughput.

---

## 5) Sliding and fixed window notes

### Sliding window (practical)

- Use fixed number of small sub-windows.
- Sum active buckets for decision.
- Better fairness near boundaries than fixed window.

### Fixed window (simple)

- Counter resets each window.
- Very easy but boundary burst issue exists (double burst across adjacent windows).

---

## 6) Factory, registry, and service layer

```java
public final class RateLimiterFactory {
    public IRateLimiter create(RateLimitRule rule) {
        return switch (rule.algorithm()) {
            case TOKEN_BUCKET -> new TokenBucketLimiter(rule.burstCapacity(), rule.refillPerSecond());
            case SLIDING_WINDOW -> new SlidingWindowLimiter((int) rule.limit(), rule.window(), 10);
            case FIXED_WINDOW -> new FixedWindowLimiter((int) rule.limit(), rule.window());
            case LEAKY_BUCKET -> new LeakyBucketLimiter(rule);
        };
    }
}

public final class RateLimiterRegistry {
    private static final RateLimiterRegistry INSTANCE = new RateLimiterRegistry();
    private final ConcurrentHashMap<String, IRateLimiter> byKey = new ConcurrentHashMap<>();
    private final RateLimiterFactory factory = new RateLimiterFactory();

    public static RateLimiterRegistry getInstance() { return INSTANCE; }

    public IRateLimiter getOrCreate(String compositeKey, RateLimitRule rule) {
        return byKey.computeIfAbsent(compositeKey, k -> factory.create(rule));
    }
}
```

Add TTL eviction for inactive keys in real systems to avoid unbounded memory growth.

---

## 7) HTTP integration flow

`RateLimitFilter`:

1. Extract identity and endpoint.
2. Build context with current timestamp and cost.
3. Resolve rule and limiter.
4. Call `tryAcquire`.
5. On deny: return `429` and `Retry-After`.

---

## 8) Decorator examples

```java
public final class LoggingRateLimiter implements IRateLimiter {
    private final IRateLimiter delegate;
    private final ILogger log;

    public LoggingRateLimiter(IRateLimiter delegate, ILogger log) {
        this.delegate = delegate;
        this.log = log;
    }

    @Override
    public RateLimitResult tryAcquire(RateLimitContext ctx) {
        RateLimitResult r = delegate.tryAcquire(ctx);
        if (!r.allowed()) log.warn("Rate limit denied key=" + ctx.compositeKey());
        return r;
    }
}
```

Use decorator for observability concerns without polluting core algorithm code.

---

## 9) Concurrency walkthrough

Scenario: same key, token bucket capacity=1, two concurrent requests.

- Thread A acquires lock, sees token available, consumes token, returns allow.
- Thread B then acquires lock, sees zero tokens, returns deny with retry-after.

Without synchronization/CAS discipline, both could incorrectly allow.

---

## 10) Distributed extension (Redis)

In multi-instance deployments:

- Local counters are per-process and drift globally.
- Use Redis atomic scripts (Lua) for shared counters/timestamps.
- Keep same `IRateLimiter` contract; switch implementation via strategy/factory.

---

## 11) Self-check with answers

1. **Why is per-key state hard at scale?**  
   Key cardinality can explode memory usage; hot keys create contention; needs eviction and/or centralized distributed store.

2. **One fairness issue in fixed window?**  
   Boundary burst allows roughly 2x instantaneous traffic across window edges.

3. **Where does Decorator help?**  
   Add metrics/logging/tracing around any algorithm without modifying algorithm implementation classes.

---

## 12) First tests

1. Token bucket capacity and refill behavior.
2. Fixed/sliding window allow-deny transitions.
3. Decorator increments metrics or emits logs correctly on deny.

---

## Day 2 checkpoint

- [x] Multi-algorithm limiter abstraction is defined.
- [x] Token bucket flow and retry-after behavior are implemented.
- [x] Registry/factory/service integration is modeled.
- [x] Concurrency, fairness, and distributed extension are explained.
