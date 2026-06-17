# Day 6 — Rate Limiter Deep Concurrency + Week 5 Capstone

This lesson consolidates rate-limiting algorithms, lock-free/thread-safe implementation choices, distributed design, and a Week 5 capstone review.

---

## Learning objectives

- Compare token bucket, leaky bucket, fixed window, and sliding window behaviors.
- Implement token bucket using CAS on atomic state (no synchronized hot path).
- Implement sliding window with timestamp deque and correct prune/check ordering.
- Explain distributed limiter semantics with Redis atomic scripts.

---

## Algorithm comparison

| Algorithm | Burst behavior | Fairness | Memory | Typical state |
|---|---|---|---|---|
| Token bucket | Allows burst up to capacity | Good | O(1) | tokens + last refill |
| Leaky bucket | Smooth output | Very good | O(queue) or O(1) level | queue/level |
| Fixed window | Boundary spike possible | Lower | O(1) | counter per window |
| Sliding window log | Fair | High | O(requests in window) | deque of timestamps |

---

## Problem 1 — Token bucket with CAS

### Why two separate atomics can fail

If tokens and refill-time are updated independently, interleavings can overwrite debits or publish stale state.
Use one atomic state object (`AtomicReference<State>`) for paired updates.

### Implementation sketch

```java
public final class TokenBucketRateLimiter {
    private final long capacity;
    private final double refillPerNano;
    private final AtomicReference<State> state;

    public TokenBucketRateLimiter(long capacity, double permitsPerSecond) {
        this.capacity = capacity;
        this.refillPerNano = permitsPerSecond / 1_000_000_000.0;
        this.state = new AtomicReference<>(new State(capacity, System.nanoTime()));
    }

    public boolean tryAcquire(int permits) {
        while (true) {
            State current = state.get();
            State refreshed = refill(current);
            if (refreshed.tokens < permits) {
                state.compareAndSet(current, refreshed); // advance clock if possible
                return false;
            }
            State next = new State(refreshed.tokens - permits, refreshed.lastRefillNanos);
            if (state.compareAndSet(current, next)) return true;
        }
    }

    private State refill(State s) {
        long now = System.nanoTime();
        long elapsed = now - s.lastRefillNanos;
        if (elapsed <= 0) return s;
        long added = (long) (elapsed * refillPerNano);
        long newTokens = Math.min(capacity, s.tokens + added);
        return new State(newTokens, now);
    }

    private record State(long tokens, long lastRefillNanos) {}
}
```

CAS retries are expected under contention; they avoid a global lock in the hot path.

---

## Problem 2 — Sliding window limiter

Use `ConcurrentLinkedDeque<Long>` of timestamps with strict operation order:

1. Prune expired timestamps (`< now - window`)
2. If current count >= limit, reject
3. Append current timestamp and allow

```java
public final class SlidingWindowRateLimiter {
    private final int limit;
    private final long windowNanos;
    private final ConcurrentLinkedDeque<Long> timestamps = new ConcurrentLinkedDeque<>();
    private final ReentrantLock lock = new ReentrantLock(); // exactness under contention

    public SlidingWindowRateLimiter(int limit, Duration window) {
        this.limit = limit;
        this.windowNanos = window.toNanos();
    }

    public boolean allow() {
        lock.lock();
        try {
            long now = System.nanoTime();
            long cutoff = now - windowNanos;
            while (true) {
                Long head = timestamps.peekFirst();
                if (head == null || head >= cutoff) break;
                timestamps.pollFirst();
            }
            if (timestamps.size() >= limit) return false;
            timestamps.addLast(now);
            return true;
        } finally {
            lock.unlock();
        }
    }
}
```

Memory note: sliding log can be expensive at high limits/windows; sliding counter buckets are a common approximation.

---

## Problem 3 — Distributed limiter with Redis

API:

```java
public interface DistributedRateLimiter {
    boolean checkLimit(String userId, String endpoint);
}
```

### Fixed-window Redis approach

- Key: `ratelimit:{user}:{endpoint}:{window}`
- Atomic Lua script: `INCR`, set `EXPIRE` on first increment, compare with limit

### Sliding-window Redis approach

- Use sorted set per `(user, endpoint)`
- Atomic script does: `ZADD`, prune old scores, `ZCARD`, allow/reject, rollback on reject if desired

### Why this is not "ConcurrentHashMap in Redis"

1. Limits must be globally shared across app nodes.
2. Check-and-update must be atomic (single script/transaction).
3. TTL/eviction and centralized state are core correctness properties.

---

## Week 5 capstone

### Unified interface + factory

```java
public interface RateLimiter {
    boolean tryAcquire();
    boolean tryAcquire(int permits);
}

public enum RateLimitAlgorithm {
    TOKEN_BUCKET, SLIDING_WINDOW, FIXED_WINDOW
}

public final class RateLimiterFactory {
    public static RateLimiter create(RateLimitConfig c) {
        return switch (c.algorithm()) {
            case TOKEN_BUCKET -> new TokenBucketRateLimiter(c.capacity(), c.permitsPerSecond());
            case SLIDING_WINDOW -> new SlidingWindowRateLimiter(c.maxRequests(), c.window());
            case FIXED_WINDOW -> new FixedWindowRateLimiter(c.maxRequests(), c.window());
        };
    }
}
```

### Integration table

| Week 5 day | Rate-limiter tie-in |
|---|---|
| 5.1 | Atomics, visibility, CAS |
| 5.3 | Blocking throttle vs fail-fast rejection |
| 5.4 | Optional pooling of limiter objects |
| 5.5 | LRU eviction of per-key limiter states |
| 5.6 | End-to-end algorithm + distributed strategy |

### Load test plan (minimum)

- Single hot key with many threads: CAS retry rate + p99 latency
- Boundary burst scenario: verify fixed-window edge spikes
- Sliding-window fairness: reject rate under sustained limit
- Distributed multi-node test: verify global cap holds

---

## Week 5 to Week 6 bridge

Use today’s concepts in Week 6 topics:

- per-key atomic updates for idempotency and dedup
- backpressure semantics for retries/circuit breakers
- distributed atomic scripts as a pattern for cross-node correctness

---

## Self-quiz with answers

1. **Why can `AtomicLong` alone be insufficient in token bucket?**  
   Because refill time and token count must be updated consistently; split atomics can cause lost updates.

2. **Sliding window memory worst-case?**  
   O(request timestamps kept in the active window), potentially large for high limit and long window.

3. **When is fixed window good enough?**  
   When simple coarse control is acceptable and edge spikes are tolerated.

---

## First three tests

1. Token bucket burst test: up to capacity allowed immediately, then limited by refill.
2. Sliding window test: N requests allowed in window, N+1 rejected.
3. Distributed test: multiple app nodes still enforce one global per-key limit.

---

## Day 6 checkpoint

- [x] Algorithms compared with fairness trade-offs
- [x] CAS-based token bucket
- [x] Sliding window with correct prune/check/append order
- [x] Redis distributed design narrative
- [x] Week 5 capstone integration and load-test thinking

**Week 5 complete.**
