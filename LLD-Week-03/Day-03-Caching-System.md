# Day 3 - Caching System (Redis/Guava-style Concepts)

**Theme:** Build a thread-safe in-memory cache with eviction, TTL, loading policies, and observability.

---

## Learning goals

By the end of this document, you should be able to:

- Design cache abstractions and metadata entities.
- Implement/justify LRU and describe LFU efficiently.
- Explain cache loading and write modes.
- Apply patterns for extensibility and operational concerns.

---

## 1) Clarifying decisions

For this design, we choose:

1. Capacity by max entries (with optional weighted extension).
2. Single-process implementation; Redis as distributed extension.
3. LRU recency based on access (read/write touches).
4. Write-behind consistency is eventual.
5. Generic key/value API with namespaced key conventions if needed.

---

## 2) Core contracts

```java
public interface Cache<K, V> {
    Optional<V> getIfPresent(K key);
    V get(K key, CacheLoader<K, V> loader);
    void put(K key, V value);
    void put(K key, V value, Duration ttl);
    void invalidate(K key);
    void invalidateAll();
    CacheStats stats();
}

public interface CacheLoader<K, V> {
    V load(K key) throws Exception;
}

public interface EvictionPolicy<K> {
    void onAccess(K key);
    void onInsert(K key);
    K evictCandidate();
    void onRemove(K key);
}
```

---

## 3) Pattern mapping

| Pattern | Role in design |
|---------|----------------|
| Strategy | `EvictionPolicy` variants (`LRU`, `LFU`, `FIFO`) |
| Template Method | Shared `AbstractCache.get()` flow hooks |
| Decorator | Stats/logging wrappers around cache |
| Proxy | `CachingRepository` in front of DB repository |
| Observer | Eviction/expiry event listeners |

---

## 4) LRU implementation (deep)

### Data structures

- `HashMap<K, Node<K,V>>` for O(1) lookup.
- Doubly linked list for recency order.
- Head = most recently used, tail = least recently used.

### LRU behavior

- `get(key)` hit -> move node to head.
- `put(key)` new/update -> move node to head.
- If size exceeds capacity -> remove tail node and map entry.

This gives O(1) average `get` and `put`.

---

## 5) LFU design (practical sketch)

Use:

- `Map<K, Node>` where `Node` has `frequency`.
- `Map<Integer, LinkedHashSet<K>>` from frequency to keys.
- Track minimum frequency.

On `get`: increment frequency and move key between buckets.  
Eviction: remove LRU key from minimum-frequency bucket.

---

## 6) TTL and loading modes

### TTL policy

- Store `expiresAt` in each `CacheEntry`.
- On read:
  - if expired -> treat as miss, remove entry.
  - else return value and update recency/frequency metadata.

Optional background reaper cleans expired entries proactively.

### Loading modes

| Mode | Miss handling | Write behavior |
|------|---------------|----------------|
| Cache-aside | Application loads DB then `put`s cache | App writes DB and invalidates/updates cache |
| Read-through | Cache calls loader internally | App writes DB separately |
| Write-through | N/A | Cache + DB sync write |
| Write-behind | N/A | Cache write immediate, DB async queue |

---

## 7) Template Method and decorators

`AbstractCache.get()` can define:

1. Lookup
2. Expiry check
3. Hit/miss hooks
4. Load-on-miss
5. Stats hooks

Decorators (for example `StatsCollectingCache`) keep observability separate from eviction logic and avoid subclass combinations like `LruStatsCache`, `LfuStatsCache`, etc.

---

## 8) Repository proxy integration

`CachingUserRepository` implements `UserRepository`:

- On read: `cache.get(id, loader -> db.findById(id))`
- On write: persist DB then cache update/invalidate

Transaction boundary should remain in service/application layer; cache updates ideally happen after successful DB commit.

---

## 9) Concurrency and race conditions

Common race:

- Two threads miss same key and both load from DB (stampede).

Mitigations:

- Single-flight/inflight map per key.
- Key-level locking or striped locks.
- `ConcurrentHashMap.compute` patterns for controlled updates.

Use lock granularity carefully to avoid one global bottleneck.

---

## 10) Extension topics

- Write-behind retry with DLQ.
- Stampede protection for hot keys.
- Segmented locking for high throughput.
- Redis/distributed cache for cross-node consistency.

---

## 11) Hands-on walkthrough

### Example LRU order (MRU -> LRU)

`put(A), put(B), put(C), get(A), get(B)` yields:

- after first three puts: `[C, B, A]`
- after `get(A)`: `[A, C, B]`
- after `get(B)`: `[B, A, C]`

If capacity is 2, `put(C)` after `[B, A]` evicts `A` and list becomes `[C, B]`.

---

## 12) Self-check with answers

1. **Why Decorator for stats?**  
   Reusable cross-cutting concern without subclass explosion across eviction policies.

2. **Cache-aside vs read-through: who loads on miss?**  
   Cache-aside: application code. Read-through: cache (via loader).

3. **One race on same key put/load and fix?**  
   Stampede or stale overwrite; use per-key synchronization/single-flight.

---

## 13) First tests

1. LRU eviction order with capacity constraints.
2. TTL expiry returns miss after deadline.
3. Stats counters/hit-rate update correctly.

---

## Day 3 checkpoint

- [x] Core interfaces and entities are defined.
- [x] LRU deeply understood; LFU modeled.
- [x] TTL and loading/write modes are clear.
- [x] Decorator/proxy/observer usage is justified.
- [x] Concurrency risks and mitigations are documented.
