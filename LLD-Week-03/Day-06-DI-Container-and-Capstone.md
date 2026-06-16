# Day 6 - DI Container and Week 3 Capstone

**Theme:** Build a minimal IoC container and consolidate Week 3 system-design understanding into reusable capstone frameworks.

---

## Part A - DI Container (Problem 3.6)

## Learning goals

By the end of this section, you should be able to:

- Model bean definitions, scopes, and lookup APIs.
- Support constructor-based dependency resolution.
- Detect circular dependencies with clear diagnostics.
- Explain singleton/prototype lifecycle and thread safety.

---

## 1) Clarifying decisions

For this DI design, we choose:

1. Reflection + explicit factory registration both supported.
2. Singleton as default scope, prototype explicitly opt-in.
3. Circular dependencies fail fast (no lazy proxy workaround in core design).
4. Thread-safe singleton caching.
5. AOP proxies mentioned as extension, not required core feature.

---

## 2) Core entities and contracts

```java
public enum Scope {
    SINGLETON, PROTOTYPE
}

public final class BeanDefinition {
    private final String name;
    private final Class<?> type;
    private final Scope scope;
    private final List<Class<?>> constructorDependencies;
    private final Supplier<Object> factory;
    // constructor/getters
}

public interface BeanFactory {
    <T> T getBean(Class<T> type);
    Object getBean(String name);
    boolean containsBean(String name);
}

public interface ApplicationContext extends BeanFactory {
    void refresh();
    void close();
}
```

---

## 3) Pattern mapping

| Pattern | Role |
|---------|------|
| Factory | `getBean` creates/resolves instances |
| Singleton scope | One instance per bean definition in container |
| Template Method | Standard bean creation flow (`resolve -> create -> init`) |
| Proxy (extension) | Transaction/logging wrappers after creation |

---

## 4) `getBean` with cycle detection

```java
public abstract class AbstractBeanFactory implements BeanFactory {
    private final Map<String, BeanDefinition> definitions = new HashMap<>();
    private final Map<Class<?>, String> primaryByType = new HashMap<>();
    private final Map<String, Object> singletonCache = new ConcurrentHashMap<>();
    private final Set<String> creating = Collections.synchronizedSet(new LinkedHashSet<>());

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getBean(Class<T> type) {
        String name = primaryByType.get(type);
        if (name == null) throw new NoSuchBeanDefinitionException(type.getName());
        return (T) getBean(name);
    }

    @Override
    public Object getBean(String name) {
        BeanDefinition def = definitions.get(name);
        if (def == null) throw new NoSuchBeanDefinitionException(name);

        if (def.scope() == Scope.SINGLETON) {
            return singletonCache.computeIfAbsent(name, n -> createBean(n, def));
        }
        return createBean(name, def);
    }

    private Object createBean(String name, BeanDefinition def) {
        if (!creating.add(name)) {
            throw new CircularDependencyException("Cycle detected: " + String.join(" -> ", creating) + " -> " + name);
        }
        try {
            Object[] deps = def.constructorDependencies().stream()
                .map(this::getBean)
                .toArray();
            Object bean = instantiate(def, deps);
            invokeInit(bean);
            return bean;
        } finally {
            creating.remove(name);
        }
    }

    protected abstract Object instantiate(BeanDefinition def, Object[] deps);
    protected void invokeInit(Object bean) {}
}
```

Cycle example `A -> B -> C -> A` should produce deterministic dependency path in exception message.

---

## 5) Lifecycle and scope behavior

- **Singleton:** created once (eager at `refresh` or lazy at first `getBean`).
- **Prototype:** new instance per `getBean`.
- **Init hooks:** `afterPropertiesSet` or annotation-based post-construct.
- **Destroy hooks:** run on `context.close()` for singleton beans.

---

## 6) Useful extensions

- Qualifier/named injection for multiple same-type beans.
- Lazy singleton wrappers.
- Optional proxy post-processors (transactions/logging).
- Better cycle diagnostics with full graph output.

---

## Part B - Week 3 Capstone Review

## 7) Cross-system summary table

| # | System | Primary abstraction | Hardest concurrency/scale issue |
|---|--------|----------------------|----------------------------------|
| 3.1 | Logging | `ILogger` / `IAppender` | Async queue backpressure and sink failure isolation |
| 3.2 | Rate limiter | `IRateLimiter` | Hot-key contention and distributed quota consistency |
| 3.3 | Cache | `Cache<K,V>` | Stampede, eviction contention, bounded memory |
| 3.4 | Plugins | `IPlugin` + `ExtensionRegistry` | Dependency ordering, isolation, activation failure safety |
| 3.5 | CI/CD | `IExecutable` / `PipelineCommand` | Parallel execution coordination and artifact ordering |
| 3.6 | DI | `BeanFactory` | Thread-safe singleton creation and cycle detection |

---

## 8) Deep capstone example A: Observability stack

Combine:

- Logging framework
- Rate limiter middleware
- Metrics decorator

Core path:

1. Request enters rate-limiter filter.
2. Decision emitted with metrics + logs.
3. Business handler executes with trace context in MDC.
4. Completion/failure logged consistently.

---

## 9) Deep capstone example B: CI + DI boundaries

Use DI container to wire:

- `PipelineExecutor`
- `RunnerPool`
- `ArtifactStore`
- `PipelineEventBus`

Keep pipeline definitions as data/config, not container lookups in random runtime logic.

This avoids service-locator anti-pattern and keeps dependencies explicit.

---

## 10) Self-check with answers

1. **Why topological sort in both DI and CI DAGs?**  
   Both require dependency-respecting order; cycles mean no valid execution/creation order.

2. **DI singleton scope vs static singleton?**  
   DI singleton is container-scoped and replaceable in tests; static singleton is global and harder to isolate.

3. **Service locator anti-pattern and fix?**  
   Don’t call container everywhere; use constructor injection from composition root.

---

## Part C - Week 3 to Week 4 bridge

Week 4 moves to domain-heavy systems (parking, hotel, delivery), but the same process remains:

1. Clarify requirements.
2. Model entities and interfaces.
3. Choose patterns intentionally.
4. Validate flows, edge cases, and failure modes.

---

## Day 6 checkpoint

- [x] DI container entities and contracts modeled.
- [x] `getBean` cycle detection approach documented.
- [x] Scope/lifecycle behavior clarified.
- [x] Week 3 six-system review completed.
- [x] Capstone integration patterns prepared for Week 4.
