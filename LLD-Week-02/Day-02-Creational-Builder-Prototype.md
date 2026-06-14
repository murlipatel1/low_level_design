# Day 2 - Creational Patterns: Builder and Prototype

**Theme:** Build complex objects clearly, and duplicate templates safely with minimal changes.

---

## Learning goals

By the end of this document, you should be able to:

- Use Builder to create immutable objects with clear required/optional fields.
- Use Prototype to clone configuration/templates while avoiding shared mutable state bugs.
- Combine creational patterns pragmatically without over-design.

---

## 1) Builder pattern

### Intent

Construct one complex object step-by-step and validate once at `build()`.

### Use when

- Objects have many optional fields.
- Constructor signatures become unreadable.
- You want a fluent API and immutable final object.

### Avoid when

Object has very few required fields and little variation.

### SQL query builder example

```java
public final class SqlQuery {
    private final String sql;

    private SqlQuery(String sql) {
        this.sql = sql;
    }

    public String sql() { return sql; }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private final List<String> selectCols = new ArrayList<>();
        private String fromTable;
        private final List<String> whereClauses = new ArrayList<>();
        private Integer limit;

        public Builder select(String... columns) {
            selectCols.addAll(Arrays.asList(columns));
            return this;
        }

        public Builder from(String table) {
            this.fromTable = table;
            return this;
        }

        public Builder where(String predicate) {
            whereClauses.add(predicate);
            return this;
        }

        public Builder limit(int limit) {
            this.limit = limit;
            return this;
        }

        public SqlQuery build() {
            if (selectCols.isEmpty()) throw new IllegalStateException("SELECT required");
            if (fromTable == null || fromTable.isBlank()) throw new IllegalStateException("FROM required");

            StringBuilder sb = new StringBuilder("SELECT ");
            sb.append(String.join(", ", selectCols));
            sb.append(" FROM ").append(fromTable);
            if (!whereClauses.isEmpty()) sb.append(" WHERE ").append(String.join(" AND ", whereClauses));
            if (limit != null) sb.append(" LIMIT ").append(limit);
            return new SqlQuery(sb.toString());
        }
    }
}
```

### HTTP request builder guidance

Typical required fields: `url`, `method` (or default `GET`).  
Typical optional fields: headers, body, timeout, retry count.

Validation at `build()`:

- `GET` should not carry body (if this rule is enforced in your API style)
- `POST/PUT` may require body
- retry count should be non-negative

Immutability in built object helps thread-safe reuse across async workers.

---

## 2) Prototype pattern

### Intent

Create objects by copying existing templates and applying small overrides.

### Use when

- Base object creation is expensive or verbose.
- Many objects share mostly same configuration.
- You need fast per-environment or per-service variants.

### Main risk

Shallow copy of nested mutable state can cause cross-object side effects.

### Safer Java approach: copy constructor

```java
public final class ServerConfig {
    private final int port;
    private final Duration timeout;
    private final LogLevel logLevel;
    private final Map<String, String> tags;

    public ServerConfig(int port, Duration timeout, LogLevel logLevel, Map<String, String> tags) {
        this.port = port;
        this.timeout = timeout;
        this.logLevel = logLevel;
        this.tags = Map.copyOf(tags);
    }

    public ServerConfig(ServerConfig prototype) {
        this.port = prototype.port;
        this.timeout = prototype.timeout;
        this.logLevel = prototype.logLevel;
        this.tags = new HashMap<>(prototype.tags);
    }

    public ServerConfig withPort(int port) {
        return new ServerConfig(port, timeout, logLevel, tags);
    }

    public ServerConfig withServiceTag(String serviceName) {
        Map<String, String> copy = new HashMap<>(tags);
        copy.put("service", serviceName);
        return new ServerConfig(port, timeout, logLevel, copy);
    }
}
```

This makes copy semantics explicit and avoids `Cloneable` surprises.

---

## 3) Builder vs Prototype vs Abstract Factory

| Pattern | Primary purpose |
|--------|------------------|
| Builder | Assemble one complex object stepwise |
| Prototype | Clone an existing template and tweak deltas |
| Abstract Factory | Create consistent families of related products |

Common combo in real systems:

- Use Builder for initial config creation.
- Use Prototype to derive many variants from that config.
- Use Abstract Factory to select environment-specific runtime family.

---

## 4) Mixed design: Config + Connection

Target:

- Abstract Factory creates `IDbConnection` + `IQueryRunner` family for dev/prod.
- Builder creates `ConnectionConfig`.
- Prototype duplicates `ConnectionConfig` for another service name.

Class responsibilities:

- `ConnectionConfig.Builder` -> fluent assembly + invariant checks
- `ConnectionConfig(ConnectionConfig other)` -> copy semantics
- `DatabaseToolkitFactory` -> family abstraction
- `DevDatabaseToolkitFactory` / `ProdDatabaseToolkitFactory` -> concrete families
- `ServiceBootstrap` -> calls `build()`, `clone/copy`, and selects concrete factory

---

## 5) Self-check with answers

1. **Why does immutable built object help concurrency?**  
   Built instances can be shared safely across threads without synchronization.

2. **When is copy constructor safer than `Cloneable`?**  
   When you need explicit deep-copy behavior and clearer invariant enforcement.

3. **One natural Factory Method + Builder combo?**  
   `HttpClientFactory.createClient()` plus `HttpRequest.builder(...).build()` for each request.

---

## 6) Interview quick lines

- Builder prevents telescoping constructors and centralizes validation.
- Prototype is about fast safe duplication, not global uniqueness.
- Prefer explicit copy semantics over implicit shallow clone behavior.
- Builder creates first version; Prototype creates many close variants.

---

## Day 2 checkpoint

- [x] I can design immutable builders with `build()` validation.
- [x] I can implement safe prototype copying for nested mutable fields.
- [x] I can combine Builder, Prototype, and Abstract Factory in one design.
- [x] I can explain when each creational pattern is the right fit.
