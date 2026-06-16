# Day 1 - Logging Framework (Log4j/SLF4J-style Implementation)

**Theme:** Design a production-grade logging subsystem with levels, appenders, formatters, hierarchy, async support, and failure isolation.

---

## Learning goals

By the end of this document, you should be able to:

- Design core logging entities and contracts.
- Explain pattern usage in a logging framework.
- Walk through `logger.info()` end-to-end.
- Handle practical concerns: async overflow, appender failures, MDC tracing.

---

## 1) Clarifying decisions

For this design, we choose:

1. Sync core path + optional async via wrapper/decorator.
2. Hierarchical logger names with parent level inheritance.
3. One log event can fan out to multiple appenders.
4. Logging failures must not break business request path.
5. Structured fields + MDC included in record shape.

---

## 2) Core entities

| Entity | Role |
|-------|------|
| `LogLevel` | Severity ordering (`TRACE`...`FATAL`) |
| `LogRecord` | Immutable log snapshot |
| `ILogger` / `Logger` | Public API and enabled-level check |
| `IAppender` | Sink contract (`console`, `file`, `remote`) |
| `LogFormatter` | Formatting strategy (`plain`, `json`) |
| `LogFilter` | Per-appender or global pass/reject checks |
| `LogManager` | Logger registry and hierarchy resolution |
| `MdcContext` | Thread-scoped trace/context map |

---

## 3) Pattern mapping

| Pattern | Concrete role in this design |
|---------|------------------------------|
| Singleton | `LogManager` registry (`getInstance`) or DI singleton |
| Observer | Logger fans out records to appenders |
| Chain of Responsibility | Filter chain decides if record continues to sink |
| Decorator | `AsyncAppender` / `FilteringAppender` wrappers |
| Template Method | `AbstractAppender.append -> format -> write -> flush` |
| Strategy | `LogFormatter` implementations (`JsonFormatter`, `PlainTextFormatter`) |

---

## 4) Core contracts and model

```java
public enum LogLevel {
    TRACE(0), DEBUG(10), INFO(20), WARN(30), ERROR(40), FATAL(50);
    private final int severity;
    LogLevel(int severity) { this.severity = severity; }
    public boolean isEnabled(LogLevel configured) { return this.severity >= configured.severity; }
}
```

```java
public interface ILogger {
    void trace(String msg);
    void debug(String msg);
    void info(String msg);
    void warn(String msg);
    void error(String msg, Throwable t);
    boolean isEnabled(LogLevel level);
}

public interface IAppender {
    void append(LogRecord record);
    void flush();
    void close();
}

public interface LogFormatter {
    String format(LogRecord record);
}

public interface LogFilter {
    boolean accept(LogRecord record);
}
```

`LogRecord` should be immutable and copy MDC/fields at creation time.

---

## 5) Key implementation pieces

### `Logger` flow

```java
public final class Logger implements ILogger {
    private final String name;
    private final LogManager manager;

    Logger(String name, LogManager manager) {
        this.name = name;
        this.manager = manager;
    }

    @Override
    public boolean isEnabled(LogLevel level) {
        return level.isEnabled(manager.effectiveLevel(name));
    }

    @Override
    public void info(String message) {
        log(LogLevel.INFO, message, null);
    }

    @Override
    public void error(String message, Throwable t) {
        log(LogLevel.ERROR, message, t);
    }

    private void log(LogLevel level, String message, Throwable t) {
        if (!isEnabled(level)) return;
        LogRecord record = LogRecord.of(level, name, message, t, MdcContext.snapshot());
        for (IAppender appender : manager.appendersFor(name)) {
            try {
                appender.append(record);
            } catch (Exception e) {
                FallbackErrorHandler.handle(e, record);
            }
        }
    }
}
```

### `AbstractAppender` template

```java
public abstract class AbstractAppender implements IAppender {
    protected final LogFormatter formatter;
    private final LogFilter filter;

    protected AbstractAppender(LogFormatter formatter, LogFilter filter) {
        this.formatter = formatter;
        this.filter = filter != null ? filter : r -> true;
    }

    @Override
    public final void append(LogRecord record) {
        if (!filter.accept(record)) return;
        String formatted = formatter.format(record);
        doWrite(formatted.getBytes(StandardCharsets.UTF_8), record);
        flush();
    }

    protected abstract void doWrite(byte[] bytes, LogRecord record);
    @Override public void flush() {}
    @Override public void close() { flush(); }
}
```

### `AsyncAppender` decorator

```java
public final class AsyncAppender implements IAppender {
    private final IAppender delegate;
    private final BlockingQueue<LogRecord> queue;
    private final Thread worker;
    private volatile boolean running = true;

    public AsyncAppender(IAppender delegate, int queueCapacity) {
        this.delegate = delegate;
        this.queue = new ArrayBlockingQueue<>(queueCapacity);
        this.worker = new Thread(this::drain, "async-log-worker");
        this.worker.setDaemon(true);
        this.worker.start();
    }

    @Override
    public void append(LogRecord record) {
        if (!queue.offer(record)) FallbackErrorHandler.drop(record); // drop policy
    }

    private void drain() {
        while (running) {
            try {
                delegate.append(queue.take());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } catch (Exception e) {
                FallbackErrorHandler.handle(e, null);
            }
        }
    }

    @Override public void flush() { delegate.flush(); }
    @Override public void close() { running = false; worker.interrupt(); delegate.close(); }
}
```

---

## 6) End-to-end call path (`info`)

1. Caller requests logger by name from `LogManager`.
2. `Logger.info()` checks `isEnabled(INFO)` using effective level.
3. If enabled, logger creates immutable `LogRecord` with MDC snapshot.
4. Logger fans out record to configured appenders.
5. Each appender applies filter, formatter, and sink write.
6. Appender exceptions are swallowed and routed to fallback handler.

---

## 7) Failure behavior decisions

| Scenario | Behavior |
|----------|----------|
| Disk full in `FileAppender` | Catch exception, report via fallback channel, do not throw to business code |
| Remote sink timeout | Catch and continue; in async mode use drop/block policy and metrics on dropped logs |

Logging should fail safe (degrade observability, not application correctness).

---

## 8) Extensions

- Per-appender thresholds (`ERROR` to remote, `DEBUG` to file).
- Async queue policies: drop, block, or drop+counter.
- Rolling file policy strategy.
- MDC propagation across async boundaries.

---

## 9) Self-check with answers

1. **Why Observer and CoR together?**  
   Observer handles fanout (who receives); CoR handles filtering/short-circuiting (whether a sink processes).

2. **Why Decorator over many appender subclasses?**  
   Wrappers compose behaviors (`Async(Filtering(File))`) without subclass explosion.

3. **Why immutable `LogRecord`?**  
   Prevents race conditions and cross-appender/cross-request data corruption.

---

## 10) First tests to write

1. Disabled level does not invoke appenders.
2. Same event reaches multiple appenders.
3. Async overflow follows chosen drop/block policy without breaking caller.

---

## Day 1 checkpoint

- [x] Core contracts and entities are defined.
- [x] Pattern-to-class mapping is explicit.
- [x] Sync + async appender paths are modeled.
- [x] Failure handling and MDC concerns are covered.
