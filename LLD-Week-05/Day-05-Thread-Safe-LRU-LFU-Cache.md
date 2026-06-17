# Day 5 — Thread-Safe LRU Cache (O(1)) + LFU Extension

This lesson covers practical cache internals for interviews: O(1) LRU, concurrency correctness, and LFU evolution.

---

## Learning objectives

- Build O(1) LRU using `HashMap<K, Node>` + doubly linked list.
- Explain why strict LRU `get` is a write operation.
- Make LRU thread-safe with a clear locking strategy.
- Extend to LFU using frequency buckets with recency tie-break.

---

## Why a plain `HashMap` is insufficient

`HashMap` solves lookup, but not eviction order or thread safety.

| Need | Plain HashMap |
|---|---|
| O(1) key lookup | Yes |
| Track recency/frequency | No |
| Safe concurrent mutation | No |

For LRU, we need:

- Map for O(1) key → node lookup
- Doubly linked list for O(1) detach/attach and tail eviction

---

## Problem 1 — LRU cache with O(1) `get` and `put`

### Data structure shape

- `head` sentinel = most recently used side
- `tail` sentinel = least recently used side
- On `get`, move node to head
- On `put`, insert/update at head; if over capacity, evict tail previous node

```java
public final class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null);
    private final Node<K, V> tail = new Node<>(null, null);
    private int size;

    public LRUCache(int capacity) {
        if (capacity < 1) throw new IllegalArgumentException();
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        moveToHead(node);
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> existing = map.get(key);
        if (existing != null) {
            existing.value = value;
            moveToHead(existing);
            return;
        }
        Node<K, V> node = new Node<>(key, value);
        map.put(key, node);
        addToHead(node);
        size++;
        if (size > capacity) {
            Node<K, V> lru = removeTail();
            map.remove(lru.key);
            size--;
        }
    }

    private void moveToHead(Node<K, V> node) {
        remove(node);
        addToHead(node);
    }

    private void addToHead(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void remove(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private Node<K, V> removeTail() {
        Node<K, V> node = tail.prev;
        remove(node);
        return node;
    }

    private static final class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }
}
```

Capacity=1 edge case is naturally handled by tail eviction on second distinct insert.

---

## Operation trace (capacity = 3)

```text
put(A): head <-> A <-> tail
put(B): head <-> B <-> A <-> tail
put(C): head <-> C <-> B <-> A <-> tail
get(A): head <-> A <-> C <-> B <-> tail
```

Next insert of new key evicts `B` (tail-side LRU).

---

## Atomic fields that must move together on `put`

These must be protected as one critical section:

1. Map insert/remove
2. Node link rewiring (`prev`/`next`)
3. Head/tail adjacency updates
4. Size counter updates

Partial visibility creates list corruption and ghost entries.

---

## Problem 2 — Thread-safe LRU

### Key concurrency nuance

Strict LRU `get` changes recency order (`moveToHead`), so it mutates structure.
That means `get` is effectively a write.

### Recommended interview implementation

Use one `ReentrantLock` for both `get` and `put`:

```java
public final class ThreadSafeLRUCache<K, V> {
    private final LRUCache<K, V> delegate;
    private final ReentrantLock lock = new ReentrantLock();

    public ThreadSafeLRUCache(int capacity) {
        this.delegate = new LRUCache<>(capacity);
    }

    public V get(K key) {
        lock.lock();
        try {
            return delegate.get(key);
        } finally {
            lock.unlock();
        }
    }

    public void put(K key, V value) {
        lock.lock();
        try {
            delegate.put(key, value);
        } finally {
            lock.unlock();
        }
    }
}
```

Why not read lock on `get` for strict LRU:

- Concurrent `get` calls would race while rewiring the same list.
- Read lock is only valid if `get` does not mutate recency (approximate LRU design).

### Throughput tradeoff

One big lock is simple and correct, but contention grows with concurrency.
Typical bottleneck is lock wait and p99 latency under hot keys.

Scale-up mention:

- Sharded caches by key hash
- More advanced policies/buffers (e.g., production libraries)

---

## Problem 3 — LFU extension

### Core structure

- `map`: key → node
- `freqBuckets`: frequency → ordered set of nodes
- `minFreq`: current minimum frequency in cache

Use `LinkedHashSet` per frequency to break ties by recency within same frequency.

```java
public final class LFUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Map<Integer, LinkedHashSet<Node<K, V>>> freqBuckets = new HashMap<>();
    private int minFreq;
    private int size;

    public LFUCache(int capacity) {
        this.capacity = capacity;
    }

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        increaseFreq(node);
        return node.value;
    }

    public void put(K key, V value) {
        if (capacity == 0) return;
        if (map.containsKey(key)) {
            Node<K, V> node = map.get(key);
            node.value = value;
            increaseFreq(node);
            return;
        }
        if (size >= capacity) {
            LinkedHashSet<Node<K, V>> bucket = freqBuckets.get(minFreq);
            Node<K, V> evict = bucket.iterator().next();
            bucket.remove(evict);
            map.remove(evict.key);
            size--;
        }
        Node<K, V> node = new Node<>(key, value);
        node.freq = 1;
        map.put(key, node);
        freqBuckets.computeIfAbsent(1, f -> new LinkedHashSet<>()).add(node);
        minFreq = 1;
        size++;
    }

    private void increaseFreq(Node<K, V> node) {
        int f = node.freq;
        LinkedHashSet<Node<K, V>> bucket = freqBuckets.get(f);
        bucket.remove(node);
        if (bucket.isEmpty() && f == minFreq) minFreq++;
        node.freq++;
        freqBuckets.computeIfAbsent(node.freq, k -> new LinkedHashSet<>()).add(node);
    }

    private static final class Node<K, V> {
        final K key;
        V value;
        int freq;
        Node(K key, V value) { this.key = key; this.value = value; }
    }
}
```

Frequency increment flow:

1. Remove node from old frequency bucket.
2. If old bucket empty and was `minFreq`, increment `minFreq`.
3. Increment node frequency.
4. Add node to new frequency bucket.

---

## LRU vs LFU

| Aspect | LRU | LFU |
|---|---|---|
| Eviction basis | Least recent | Least frequent |
| Tie-break detail | List order | Recency inside same freq bucket |
| Wins when | Temporal locality dominates | Long-term hot keys dominate |

LFU clearly wins in heavily skewed workloads where a small set of keys stays hot over long windows.

---

## Self-quiz with answers

1. **Why not `HashMap + ArrayList` by recency?**  
   Updating recency requires list movement/removal in middle, which is O(n), so eviction/update is not O(1).

2. **Under one big lock, what scales poorly?**  
   Lock contention and p99 latency under high concurrency/hot keys.

3. **Workload where LFU beats LRU?**  
   Highly skewed repeated-access workloads (e.g., hot static assets/config keys).

---

## First three tests

1. Capacity test: `put(A), put(B), put(C), put(D)` (capacity 3) evicts `A` if no reads.
2. Recency test: `put(A), put(B), put(C), get(A), put(D)` evicts `B`.
3. Concurrency test: parallel `get/put` never corrupts list and never exceeds capacity.

---

## Interview sound bites

- "Strict LRU `get` is a write because it mutates recency."
- "Use one lock first for correctness, then discuss sharding."
- "LFU needs `minFreq` + per-frequency ordered buckets."
- "ArrayList cannot provide O(1) recency updates."

---

## Day 5 checkpoint

- [x] O(1) LRU with map + doubly linked list
- [x] Correct strict-LRU locking model
- [x] LFU with frequency buckets and tie-break recency
- [x] Self-quiz and tests

**Next:** `Day-06-Rate-Limiter-Deep-Concurrency.md`
