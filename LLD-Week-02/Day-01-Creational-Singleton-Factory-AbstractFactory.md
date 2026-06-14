# Day 1 - Creational Patterns: Singleton, Factory Method, Abstract Factory

**Theme:** Control object creation to improve consistency, testability, and flexibility.

---

## Learning goals

By the end of this document, you should be able to:

- Use Singleton only when one process-local instance is truly required.
- Apply Factory Method to hide concrete creation from client code.
- Apply Abstract Factory to build consistent families of related products.

---

## 1) Singleton

### Intent

Ensure one instance per process and provide controlled access to it.

### Good use cases

- Config cache loaded once per process
- Metrics registry in one runtime
- Feature-flag snapshot manager

### Common pitfalls

- Unsafe lazy initialization under concurrency
- Hidden global state hurting test isolation
- Mistaken assumption that singleton means cluster-wide uniqueness

### Thread-safe Java sketch (holder idiom)

```java
public final class FileLogger {
    private final Path logFile;
    private final Object lock = new Object();

    private FileLogger(Path logFile) {
        this.logFile = logFile;
    }

    private static class Holder {
        static final FileLogger INSTANCE = new FileLogger(Path.of("app.log"));
    }

    public static FileLogger getInstance() {
        return Holder.INSTANCE;
    }

    public void log(Level level, String message) {
        String line = Instant.now() + " [" + level + "] " + message + System.lineSeparator();
        synchronized (lock) {
            Files.writeString(logFile, line, StandardOpenOption.CREATE, StandardOpenOption.APPEND);
        }
    }
}
```

### Better testing posture

Even when using one runtime instance, prefer injecting that single instance from composition root instead of calling `getInstance()` everywhere.

---

## 2) Factory Method

### Intent

Centralize and abstract object creation so clients depend on product interfaces, not concrete classes.

### Good use cases

- Runtime format/vendor/profile selection
- Config-driven parser/connector creation
- Avoiding scattered `new ConcreteType()` across domain code

### Parser factory example

```java
public interface Parser {
    ParsedDocument parse(InputStream in);
}

public final class ParserFactory {
    private static final Map<String, Supplier<Parser>> BY_EXT = Map.of(
        ".json", JsonParser::new,
        ".xml", XmlParser::new,
        ".csv", CsvParser::new,
        ".yaml", YamlParser::new,
        ".yml", YamlParser::new
    );

    public Parser createParser(Path path) {
        String ext = extension(path);
        Supplier<Parser> supplier = BY_EXT.get(ext);
        if (supplier == null) throw new UnsupportedFormatException(ext);
        return supplier.get();
    }

    private static String extension(Path path) {
        String name = path.getFileName().toString();
        int i = name.lastIndexOf('.');
        return i < 0 ? "" : name.substring(i).toLowerCase();
    }
}
```

Client services should depend on `Parser` abstraction only.

---

## 3) Abstract Factory

### Intent

Create consistent families of related objects without hard-coding concrete family types in client logic.

### Good use cases

- Cloud bundles (compute + storage + load balancer)
- Cross-platform UI kit families
- Test toolkit families (unit profile vs integration profile)

### Cross-cloud example

```java
public interface ComputeInstance { void start(); void stop(); }
public interface ObjectStorage { void put(String key, byte[] data); byte[] get(String key); }
public interface LoadBalancer { void registerBackend(String instanceId); }

public interface CloudFactory {
    ComputeInstance createCompute();
    ObjectStorage createStorage();
    LoadBalancer createLoadBalancer();
}
```

```java
public final class AwsCloudFactory implements CloudFactory {
    @Override public ComputeInstance createCompute() { return new Ec2Instance(); }
    @Override public ObjectStorage createStorage() { return new S3Storage(); }
    @Override public LoadBalancer createLoadBalancer() { return new AwsAlb(); }
}

public final class GcpCloudFactory implements CloudFactory {
    @Override public ComputeInstance createCompute() { return new GceInstance(); }
    @Override public ObjectStorage createStorage() { return new GcsStorage(); }
    @Override public LoadBalancer createLoadBalancer() { return new GcpLb(); }
}
```

### DIP connection

Application services depend on `CloudFactory` and product interfaces; concrete cloud details are chosen at bootstrap.

---

## 4) Pattern comparison quick guide

- **Singleton:** one instance access policy
- **Factory Method:** one product creation abstraction
- **Abstract Factory:** coordinated family creation abstraction

Common confusion:

- Factory Method vs Abstract Factory: one product vs product family
- Singleton vs Flyweight: one service instance vs many shared intrinsic states

---

## 5) Integration exercise: Multi-tenant CLI

Scenario: CLI targets AWS or GCP per active tenant.

Suggested design:

- `TenantConfigRegistry` as process-wide registry (singleton or DI singleton)
- `CloudClientFactory` abstraction with `AwsClientFactory` and `GcpClientFactory`
- Product interfaces: `BlobStore`, `Queue`, `SecretsReader`
- Optional `CloudClientFactoryProvider` (factory method) to select factory from profile

Example flow:

1. User selects tenant profile
2. Registry exposes active profile
3. Factory provider returns correct cloud family
4. CLI command uses product interfaces without cloud-specific branching

---

## 6) Self-check with answers

1. **Singleton vs static utility: what does instance buy you?**  
   Interface-based substitution, stateful behavior, lifecycle control, and easier testing/mocking.

2. **How to avoid large `switch` in factory?**  
   Use a registry map (`Map<String, Supplier<T>>`), SPI discovery, or DI-based registration.

3. **Why per-product interfaces in Abstract Factory instead of `Object` returns?**  
   Strong typing, no widespread casts/`instanceof`, and better ISP compliance.

---

## 7) Interview quick lines

- Singleton is per process, not globally distributed.
- Prefer wiring one shared instance through DI over global static access.
- Factory Method removes concrete `new` from business logic.
- Abstract Factory protects consistency when products must come in matching families.

---

## Day 1 checkpoint

- [x] I can implement or explain thread-safe singleton trade-offs.
- [x] I can design a registry-driven factory method.
- [x] I can model cloud/test product families with abstract factory.
- [x] I can explain where DIP appears in all three patterns.
