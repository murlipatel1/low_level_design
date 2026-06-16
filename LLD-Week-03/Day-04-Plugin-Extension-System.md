# Day 4 - Plugin / Extension System (VS Code-style)

**Theme:** Enable runtime extensibility through plugin lifecycle control, typed extension points, and event-driven plugin collaboration.

---

## Learning goals

By the end of this document, you should be able to:

- Design plugin discovery, load, activate, deactivate, unload lifecycle.
- Build typed extension-point registration and retrieval.
- Use EventBus mediator to avoid plugin-to-plugin direct coupling.
- Handle plugin dependencies, permissions, and failure isolation.

---

## 1) Clarifying decisions

For this design, we choose:

1. Disk-based plugins (`plugin.json` + JAR/wheel style loading).
2. API boundary + permission model in scope; deep sandboxing as extension.
3. Manifest-declared extension points and dependencies.
4. Dependency graph must be acyclic (topological ordering).
5. Backend command/formatter/parser contributions; UI widgets out of core scope.

---

## 2) Core interfaces and entities

```java
public interface IPlugin {
    PluginMetadata metadata();
    void activate(PluginContext context) throws PluginActivationException;
    void deactivate();
}

public final class PluginMetadata {
    private final String id;
    private final String version;
    private final List<String> dependencies;
    private final Set<String> permissions;
    // constructor/getters
}

public interface PluginContext {
    EventBus eventBus();
    ExtensionRegistry extensions();
    PluginLogger logger();
    WorkspaceService workspace();
}

public interface EventBus {
    <T extends DomainEvent> void subscribe(Class<T> type, EventHandler<T> handler);
    <T extends DomainEvent> void publish(T event);
}
```

Example extension strategies:

- `IFormatter`
- `ILanguageParser`
- `ICommand`

---

## 3) Pattern mapping

| Pattern | Role |
|---------|------|
| Factory | `PluginLoader` creates plugin instances from manifest metadata |
| Mediator | `EventBus` coordinates plugin communication |
| Observer | Event handlers subscribe to bus events |
| Strategy | Formatter/parser/command implementations |
| Composite | Command palette tree with groups and leaves |
| Registry | Plugin and extension lookup by id/type |

---

## 4) Lifecycle design

```text
discover -> load -> resolve dependencies -> instantiate -> activate -> running
-> deactivate -> unload
```

### `PluginLoader` (factory sketch)

```java
public final class PluginLoader {
    public IPlugin load(Path pluginDir) {
        PluginManifest manifest = PluginManifest.read(pluginDir.resolve("plugin.json"));
        validate(manifest);
        ClassLoader cl = createPluginClassLoader(pluginDir, manifest);
        Class<?> main = Class.forName(manifest.mainClass(), true, cl);
        return (IPlugin) main.getDeclaredConstructor().newInstance();
    }
}
```

### `PluginManager` orchestration sketch

```java
public final class PluginManager {
    private final PluginRegistry registry = new PluginRegistry();
    private final ExtensionRegistry extensions = new ExtensionRegistry();
    private final EventBus eventBus = new InMemoryEventBus();
    private final PluginLoader loader = new PluginLoader();
    private final DependencyResolver resolver = new DependencyResolver();

    public void startup(Path pluginsRoot) {
        List<PluginManifest> manifests = discover(pluginsRoot);
        List<PluginManifest> ordered = resolver.resolveOrder(manifests);

        for (PluginManifest m : ordered) {
            try {
                IPlugin plugin = loader.load(pluginsRoot.resolve(m.id()));
                enforcePermissions(m);
                PluginContext ctx = new DefaultPluginContext(eventBus, extensions, m.id());
                plugin.activate(ctx);
                registry.register(m.id(), plugin);
            } catch (Exception e) {
                registry.markFailed(m.id(), e);
            }
        }
    }
}
```

---

## 5) Extension registry and host usage

```java
public final class ExtensionRegistry {
    private final Map<String, List<Object>> contributions = new ConcurrentHashMap<>();

    public <T> void register(String extensionPointId, T contribution) {
        contributions.computeIfAbsent(extensionPointId, k -> new CopyOnWriteArrayList<>()).add(contribution);
    }

    public <T> List<T> getContributions(String extensionPointId, Class<T> type) {
        return contributions.getOrDefault(extensionPointId, List.of()).stream()
            .filter(type::isInstance)
            .map(type::cast)
            .toList();
    }
}
```

Host example: choose formatter by language from registered formatter strategies.

---

## 6) EventBus as mediator

```java
public final class InMemoryEventBus implements EventBus {
    private final Map<Class<?>, List<EventHandler<?>>> handlers = new ConcurrentHashMap<>();

    @Override
    public <T extends DomainEvent> void subscribe(Class<T> type, EventHandler<T> handler) {
        handlers.computeIfAbsent(type, k -> new CopyOnWriteArrayList<>()).add(handler);
    }

    @SuppressWarnings("unchecked")
    @Override
    public <T extends DomainEvent> void publish(T event) {
        for (EventHandler<?> h : handlers.getOrDefault(event.getClass(), List.of())) {
            try {
                ((EventHandler<T>) h).onEvent(event);
            } catch (Exception e) {
                // isolate plugin failure
            }
        }
    }
}
```

Plugins communicate by publishing/subscribing events, not by direct references.

---

## 7) Dependency and permission extensions

### Dependency resolver

- Build plugin DAG from `dependencies`.
- Topological sort for activation order.
- Detect cycles and mark involved plugins failed.

### Permission model

Manifest contains declared permissions (`workspace.read`, `workspace.write`, `network`).  
Host grants restricted context capabilities based on permission checks.

---

## 8) Composite command tree

Represent command palette as tree:

- `CommandGroup` (composite)
- `CommandLeaf` (leaf)

Multiple plugins contribute their command nodes; host merges into one global tree.

---

## 9) File-save sequence (formatter + linter)

1. User saves file in host.
2. Host persists file and publishes `FileSavedEvent`.
3. Formatter plugin receives event and formats.
4. Linter plugin receives event and publishes diagnostics event.
5. Host/diagnostics component consumes diagnostics event.

---

## 10) What not to put in plugin API (ISP)

Avoid:

- Fat `IPlugin` methods for every capability (`format`, `lint`, `parse`, `build`, etc.).
- Direct references to other plugins.
- Full host internals in one massive context object.

Prefer narrow interfaces and typed extension points.

---

## 11) Self-check with answers

1. **Mediator vs Observer: who knows peers?**  
   Mediator (`EventBus`) knows handler routing; plugins know only bus, not peer plugins.

2. **Why Factory loader instead of host direct `new`?**  
   Runtime plugin discovery and decoupling from concrete plugin classes require manifest-driven instantiation.

3. **Activate failure handling rule?**  
   Roll back partial registrations, mark plugin failed, and keep system running with remaining healthy plugins.

---

## 12) First tests

1. Dependency ordering and cycle detection behavior.
2. EventBus isolates failing handler and still calls other handlers.
3. Plugin activation failure does not leak partial extension registrations.

---

## Day 4 checkpoint

- [x] Lifecycle and loader design are defined.
- [x] Extension and event contracts are separated cleanly.
- [x] Dependency and permission extensions are modeled.
- [x] Composite command contribution model is included.
- [x] Failure isolation and test plan are documented.
