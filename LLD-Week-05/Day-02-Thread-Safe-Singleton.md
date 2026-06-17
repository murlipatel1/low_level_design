# Day 2 — Thread-Safe Singleton (All Approaches)

This lesson covers all major singleton implementations, their concurrency correctness, and practical production guidance.

---

## Learning objectives

- Explain six singleton styles and their trade-offs.
- Identify why naive lazy singleton breaks under concurrency.
- Explain reflection and serialization risks.
- Design read-heavy feature-flag lookups with lock-free reads.

---

## Interview framing first

A strong framing is:

> In production, prefer DI-scoped singleton managed by the container.  
> In LLD, show Bill Pugh holder or enum when true JVM-wide singleton semantics are required.

Why this works:

- Better testability with DI.
- Lifecycle and environment control from framework.
- Avoids static global hidden dependencies.

---

## The six singleton approaches

| # | Approach | Thread-safe | After-init performance | Notes |
|---|---|---|---|---|
| 1 | Eager | Yes | Fast | Initializes at class load |
| 2 | Lazy (broken) | No | Fast | Check-then-act race |
| 3 | Synchronized method | Yes | Slower | Lock on every call |
| 4 | DCL + `volatile` | Yes | Fast | Correct but subtle |
| 5 | Bill Pugh holder | Yes | Fast | Preferred Java LLD option |
| 6 | Enum | Yes | Fast | Reflection + serialization resilient |

### 1) Eager singleton

```java
public final class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();

    private EagerSingleton() {}

    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}
```

Correct because class initialization is JVM-synchronized.

### 2) Lazy singleton (broken)

```java
public final class BrokenLazySingleton {
    private static BrokenLazySingleton instance;

    public static BrokenLazySingleton getInstance() {
        if (instance == null) {
            instance = new BrokenLazySingleton();
        }
        return instance;
    }
}
```

Broken because two threads can both see `null` and create separate instances.

### 3) Synchronized method singleton

```java
public final class SyncMethodSingleton {
    private static SyncMethodSingleton instance;

    private SyncMethodSingleton() {}

    public static synchronized SyncMethodSingleton getInstance() {
        if (instance == null) {
            instance = new SyncMethodSingleton();
        }
        return instance;
    }
}
```

Correct, but every call acquires class monitor lock.

### 4) Double-checked locking (correct)

```java
public final class DclSingleton {
    private static volatile DclSingleton instance;

    private DclSingleton() {}

    public static DclSingleton getInstance() {
        if (instance == null) {
            synchronized (DclSingleton.class) {
                if (instance == null) {
                    instance = new DclSingleton();
                }
            }
        }
        return instance;
    }
}
```

Why `volatile` matters:

- Prevents publishing partially constructed objects.
- Ensures visibility of fully initialized object to other threads.

### 5) Bill Pugh holder idiom

```java
public final class BillPughSingleton {
    private BillPughSingleton() {}

    private static class Holder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

Lazy + thread-safe through JVM class initialization guarantees, without explicit locking.

### 6) Enum singleton

```java
public enum EnumSingleton {
    INSTANCE;

    public void doWork() {
        // work
    }
}
```

Usually the most robust for reflection and serialization safety.

---

## Reflection and singleton breakage

Classic singleton can be broken:

```java
Constructor<VulnerableSingleton> c = VulnerableSingleton.class.getDeclaredConstructor();
c.setAccessible(true);
VulnerableSingleton second = c.newInstance();
```

Mitigations:

- Enum singleton (best built-in protection).
- Constructor guard (`if (instance != null) throw`), though not fully bulletproof in all sequences.
- Prefer DI container singleton scope instead of static singleton where possible.

Serialization note for non-enum singleton:

```java
private Object readResolve() {
    return getInstance();
}
```

---

## FeatureFlagService (60s reload, fast reads)

Goal: many `isEnabled(flag)` calls, periodic config refresh, minimal read overhead.

Use immutable snapshot + atomic publish:

```java
public final class FeatureFlagService {
    private static final FeatureFlagService INSTANCE = new FeatureFlagService();

    private final AtomicReference<ImmutableFlagMap> snapshot =
        new AtomicReference<>(ImmutableFlagMap.empty());
    private final ScheduledExecutorService scheduler =
        Executors.newSingleThreadScheduledExecutor();

    private FeatureFlagService() {
        reloadNow();
        scheduler.scheduleAtFixedRate(this::reloadNow, 60, 60, TimeUnit.SECONDS);
    }

    public static FeatureFlagService getInstance() {
        return INSTANCE;
    }

    public boolean isEnabled(String flag) {
        return snapshot.get().isEnabled(flag);
    }

    private void reloadNow() {
        try {
            Map<String, Boolean> loaded = loadFromFile(Path.of("flags.properties"));
            snapshot.set(new ImmutableFlagMap(loaded));
        } catch (IOException ignored) {
            // retain last good snapshot
        }
    }
}
```

```java
public final class ImmutableFlagMap {
    private final Map<String, Boolean> flags;

    public ImmutableFlagMap(Map<String, Boolean> source) {
        this.flags = Map.copyOf(source);
    }

    public boolean isEnabled(String flag) {
        return flags.getOrDefault(flag, false);
    }

    public static ImmutableFlagMap empty() {
        return new ImmutableFlagMap(Map.of());
    }
}
```

Why it scales:

- Reads do one atomic reference read + map lookup (no lock).
- Writer builds new immutable map and swaps atomically.
- Readers never observe partial updates.

Avoid:

- Global read lock on every `isEnabled`.

---

## Self-quiz with answers

1. **What does `volatile` fix in DCL?**  
   It prevents unsafe publication (partially initialized instance visibility) and ensures proper visibility across threads.

2. **Why does enum singleton resist reflection attacks?**  
   JVM disallows reflective construction of enum constants through normal constructor reflection APIs.

3. **When is eager better than lazy?**  
   When object is always needed at startup, construction is cheap/predictable, and you want fail-fast initialization.

---

## First three tests

1. 100 concurrent calls to `BillPughSingleton.getInstance()` return same reference.
2. Feature flag reload flips `beta=true` and readers observe new value without lock contention.
3. Reflection second-instance attempt on guarded/enum approach fails as expected.

---

## Interview sound bites

- "Holder idiom or enum by default; DCL if explicitly asked."
- "`volatile` in DCL is about safe publication, not only visibility."
- "Immutable snapshot + atomic swap beats read-lock-per-request."
- "Production code usually prefers DI container singleton scope."

---

## Day 2 checkpoint

- [x] Six singleton approaches with correctness and trade-offs
- [x] Reflection and serialization concerns
- [x] FeatureFlagService lock-free read path design
- [x] Self-quiz and practical test checklist

**Next:** `Day-03-Producer-Consumer-BlockingQueue.md`
