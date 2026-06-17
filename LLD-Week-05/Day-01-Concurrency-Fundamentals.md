# Day 1 — Concurrency Fundamentals for LLD

This lesson builds the thread-safety foundations reused in rate limiters, caches, pools, queues, and repositories.

---

## Learning objectives

- Explain race conditions, visibility, and atomicity with concrete examples.
- Choose between `synchronized`, `ReentrantLock`, `ReadWriteLock`, and `StampedLock`.
- Know when `volatile` is enough and when it is not.
- Use `Atomic*` types and concurrent collections correctly.

---

## Race, visibility, and atomicity

```java
// BUG: lost updates
public class BrokenCounter {
    private int count = 0;
    public void increment() { count++; }  // read -> add -> write (not atomic)
    public int get() { return count; }
}
```

If two threads both read `5` and both write `6`, one increment is lost.

| Term | Meaning | Typical fix |
|---|---|---|
| Race | Behavior depends on thread interleaving | Lock or CAS |
| Visibility | One thread's write not seen by another in time | `volatile`, lock, concurrent classes |
| Atomicity | Compound operation should appear indivisible | `Atomic*` or lock |

```java
// Fix A: atomic counter
private final AtomicInteger count = new AtomicInteger();
public void increment() { count.incrementAndGet(); }

// Fix B: lock-based
public synchronized void increment() { count++; }
```

---

## Synchronization mechanisms

| Mechanism | Best use |
|---|---|
| `synchronized` | Simple critical section and single lock |
| `ReentrantLock` | Need `tryLock`, timed waits, fairness, interruptible lock |
| `ReadWriteLock` | Read-heavy repo/config with occasional writes |
| `StampedLock` | Read-mostly flows with optimistic read + validation |
| `volatile` | Visibility-only flags or immutable reference publish |

Important: `volatile` does **not** make `counter++` atomic.

---

## Problem 1 — `RequestCounter`

Design a per-endpoint HTTP counter with high concurrency.

```java
public final class RequestCounter {
    private final ConcurrentHashMap<String, AtomicLong> counts = new ConcurrentHashMap<>();

    public void increment(String endpoint) {
        counts.computeIfAbsent(endpoint, k -> new AtomicLong(0))
              .incrementAndGet();
    }

    public long get(String endpoint) {
        AtomicLong counter = counts.get(endpoint);
        return counter == null ? 0L : counter.get();
    }

    public Map<String, Long> snapshot() {
        Map<String, Long> copy = new HashMap<>();
        counts.forEach((k, v) -> copy.put(k, v.get()));
        return Collections.unmodifiableMap(copy);
    }
}
```

Why `ConcurrentHashMap<String, AtomicLong>` over one lock + `HashMap`?

- Per-key updates scale better.
- No global lock serialization across unrelated endpoints.
- Better throughput under mixed-key traffic.

Hot key caution (`/health` dominates traffic):

- Single `AtomicLong` can suffer CAS retry contention.
- Mitigate with sharded counters or `LongAdder`.

```java
private final ConcurrentHashMap<String, LongAdder> counts = new ConcurrentHashMap<>();
public void increment(String endpoint) {
    counts.computeIfAbsent(endpoint, k -> new LongAdder()).increment();
}
```

---

## Problem 2 — `TTLCache<K, V>`

Use a concurrent map with per-entry expiry.

```java
public final class ExpiringValue<V> {
    private final V value;
    private final long expiresAtEpochMs;

    public ExpiringValue(V value, long expiresAtEpochMs) {
        this.value = value;
        this.expiresAtEpochMs = expiresAtEpochMs;
    }

    public V value() { return value; }
    public boolean isExpired(long now) { return now >= expiresAtEpochMs; }
}
```

```java
public final class TTLCache<K, V> {
    private final ConcurrentHashMap<K, ExpiringValue<V>> map = new ConcurrentHashMap<>();
    private final long ttlMs;
    private final int maxSize; // 0 means unbounded

    public TTLCache(long ttlMs, int maxSize) {
        this.ttlMs = ttlMs;
        this.maxSize = maxSize;
    }

    public void put(K key, V value) {
        long expiresAt = System.currentTimeMillis() + ttlMs;
        map.put(key, new ExpiringValue<>(value, expiresAt));
        evictIfNeeded();
    }

    public V get(K key) {
        ExpiringValue<V> wrapped = map.get(key);
        if (wrapped == null) return null;
        if (wrapped.isExpired(System.currentTimeMillis())) {
            map.remove(key, wrapped); // remove only if unchanged
            return null;
        }
        return wrapped.value();
    }

    public void startSweeper(Duration period, ScheduledExecutorService exec) {
        exec.scheduleAtFixedRate(
            this::sweepExpired,
            period.toMillis(),
            period.toMillis(),
            TimeUnit.MILLISECONDS
        );
    }

    private void evictIfNeeded() {
        if (maxSize <= 0 || map.size() <= maxSize) return;
        sweepExpired();
    }

    private void sweepExpired() {
        long now = System.currentTimeMillis();
        map.entrySet().removeIf(e -> e.getValue().isExpired(now));
    }
}
```

Lazy vs sweeper:

- Lazy expiry (`get`/`put`) keeps runtime simple.
- Sweeper prevents stale cold keys from lingering forever.
- In interviews, a practical answer is **both**.

Happens-before note:

- `ConcurrentHashMap.put` safely publishes entries.
- Subsequent `get` sees either null or a fully constructed value.

---

## Problem 3 — `UserRepository` with `ReadWriteLock`

Allow many reads, but exclusive writes.

```java
public final class UserRepository {
    private final Map<String, User> store = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock readLock = lock.readLock();
    private final Lock writeLock = lock.writeLock();

    public Optional<User> findById(String id) {
        readLock.lock();
        try {
            return Optional.ofNullable(store.get(id));
        } finally {
            readLock.unlock();
        }
    }

    public List<User> findAll() {
        readLock.lock();
        try {
            return List.copyOf(store.values());
        } finally {
            readLock.unlock();
        }
    }

    public void save(User user) {
        writeLock.lock();
        try {
            store.put(user.id(), user);
        } finally {
            writeLock.unlock();
        }
    }

    public void delete(String id) {
        writeLock.lock();
        try {
            store.remove(id);
        } finally {
            writeLock.unlock();
        }
    }
}
```

Avoid read-to-write upgrade while holding read lock:

```java
// Deadlock risk pattern
readLock.lock();
try {
    if (!store.containsKey(id)) {
        writeLock.lock(); // can block forever in some interleavings
        try { store.put(id, user); } finally { writeLock.unlock(); }
    }
} finally {
    readLock.unlock();
}
```

Use write lock directly for compound read-then-write methods.

---

## Lost update demo

```java
public static void main(String[] args) throws Exception {
    BrokenCounter c = new BrokenCounter();
    Runnable task = () -> {
        for (int i = 0; i < 1_000_000; i++) c.increment();
    };
    Thread t1 = new Thread(task);
    Thread t2 = new Thread(task);
    t1.start();
    t2.start();
    t1.join();
    t2.join();
    System.out.println(c.get()); // often < 2_000_000
}
```

Replace with `AtomicInteger` or `synchronized` to stabilize at `2_000_000`.

---

## `volatile` quick guide

| Scenario | `volatile` enough? |
|---|---|
| `boolean shutdown` flag | Yes |
| Shared `counter++` | No |
| Publish immutable config reference | Yes |
| Check-then-act map mutation | No |

---

## Concurrent collection selection

| Type | Good for | Bad for |
|---|---|---|
| `ConcurrentHashMap` | Shared caches/counters | Strictly consistent iteration under heavy writes |
| `CopyOnWriteArrayList` | Read-mostly listener/config lists | High-write append logs |
| `BlockingQueue` | Producer-consumer boundaries | Random-access workloads |

---

## Self-quiz with answers

1. **Why can both threads pass `containsKey` then both `put`?**  
   This is a check-then-act race. Both threads observe "absent" before either write is visible, so one update can overwrite another. Use `putIfAbsent` on `ConcurrentHashMap` or lock the compound operation.

2. **When is `CopyOnWriteArrayList` a poor request-log choice?**  
   When write rate is high. Every write copies the underlying array, causing O(n) append cost and memory churn.

3. **What can `ReentrantLock` do that `synchronized` cannot?**  
   `tryLock(timeout)` and `lockInterruptibly()` are key capabilities; fair lock mode is another.

---

## First three tests

1. `RequestCounter`: 100 threads x 1000 increments on same endpoint => exact final count.
2. `TTLCache`: put with TTL 100ms, sleep 150ms, `get` returns null.
3. `UserRepository`: concurrent reads during writes produce no `ConcurrentModificationException`; each read returns a valid snapshot.

---

## Interview sound bites

- "`i++` is read-modify-write, not atomic."
- "`volatile` provides visibility, not compound-operation safety."
- "Per-key concurrency (`CHM + Atomic/Adder`) scales better than one global lock."
- "Do not upgrade read lock to write lock while holding read lock."
- "Use lock choice to match read/write ratio, not habit."

---

## Day 1 checkpoint

- [x] Race, visibility, atomicity foundations
- [x] Thread-safe `RequestCounter` and hot-key tradeoff
- [x] `TTLCache` with lazy+sweeper expiry model
- [x] Read/write separated repository model
- [x] Self-quiz and practical tests

**Next:** `Day-02-Thread-Safe-Singleton.md`
