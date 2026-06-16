# Day 5 - CI/CD Pipeline Engine (GitHub Actions/Jenkins-style)

**Theme:** Build an execution engine for triggered pipelines with staged DAG flow, artifact passing, retries, and runner orchestration.

---

## Learning goals

By the end of this document, you should be able to:

- Model pipeline structure (pipeline -> stages -> steps).
- Execute sequential and parallel workloads with dependency guards.
- Represent steps as commands with timeout/retry/cancel policies.
- Explain trigger handling, artifact handoff, and event notifications.

---

## 1) Clarifying decisions

For this design, we choose:

1. Stage-level DAG with optional parallel steps inside a stage.
2. Local artifact storage for core design, object store as extension.
3. Unified trigger model for push/PR/cron/manual events.
4. Cooperative cancellation model.
5. Runner pool that assigns one runnable job slice per runner.

---

## 2) Core entities and contracts

```java
public enum ExecutionStatus {
    QUEUED, RUNNING, SUCCEEDED, FAILED, CANCELLED, SKIPPED
}

public final class ExecutionResult {
    private final ExecutionStatus status;
    private final String message;
    private final Map<String, ArtifactRef> artifacts;
    // factory methods
}

public interface IExecutable {
    String id();
    ExecutionResult run(ExecutionContext ctx);
}

public interface PipelineCommand {
    ExecutionResult execute(ExecutionContext ctx);
    default void cleanup(ExecutionContext ctx) {}
}
```

```java
public final class ExecutionContext {
    private final String pipelineRunId;
    private final Map<String, ArtifactRef> artifacts = new ConcurrentHashMap<>();
    private final Map<String, ExecutionResult> stageResults = new ConcurrentHashMap<>();
    private volatile boolean cancelled;

    public void publishArtifact(String name, Path path) { /* ... */ }
    public Path requireArtifact(String name) { /* ... */ }
    public void recordStageResult(String stageId, ExecutionResult result) { stageResults.put(stageId, result); }
    public boolean isStageSucceeded(String stageId) {
        ExecutionResult r = stageResults.get(stageId);
        return r != null && r.status() == ExecutionStatus.SUCCEEDED;
    }
    public boolean isCancelled() { return cancelled; }
    public void cancel() { cancelled = true; }
}
```

---

## 3) Pattern mapping

| Pattern | Role |
|---------|------|
| Composite | `Pipeline`, `Stage`, `Step` share executable abstraction |
| Command | `Step` wraps executable unit with operational metadata |
| Template Method | `BaseStep` enforces pre/execute/post flow |
| Strategy | `ExecutionStrategy` for sequential vs parallel step execution |
| Builder | Fluent pipeline definition creation |
| CoR | Pre-execution gate checks (auth/resources/concurrency) |
| Observer | Pipeline event bus for notifications/dashboard hooks |
| Decorator (optional) | Retry wrapper around step commands |

---

## 4) Composite structure and execution

```java
public final class Step implements IExecutable {
    private final String id;
    private final PipelineCommand command;
    private final RetryPolicy retryPolicy;
    private final Duration timeout;

    @Override
    public ExecutionResult run(ExecutionContext ctx) {
        if (ctx.isCancelled()) return ExecutionResult.cancelled();
        return new RetryStepDecorator(command, retryPolicy).runWithTimeout(ctx, timeout);
    }
}

public final class Stage implements IExecutable {
    private final String id;
    private final List<Step> steps;
    private final Set<String> dependsOn;
    private final ExecutionStrategy strategy;

    @Override
    public ExecutionResult run(ExecutionContext ctx) {
        for (String dep : dependsOn) {
            if (!ctx.isStageSucceeded(dep)) {
                return ExecutionResult.skipped("dependency not met: " + dep);
            }
        }
        return strategy.execute(steps, ctx);
    }
}

public final class Pipeline implements IExecutable {
    private final String name;
    private final List<Stage> stages;

    @Override
    public ExecutionResult run(ExecutionContext ctx) {
        for (Stage stage : topologicalOrder(stages)) {
            ExecutionResult r = stage.run(ctx);
            ctx.recordStageResult(stage.id(), r);
            if (r.status() == ExecutionStatus.FAILED || r.status() == ExecutionStatus.CANCELLED) {
                return r;
            }
        }
        return ExecutionResult.success(Map.of());
    }
}
```

---

## 5) Execution strategies

```java
public interface ExecutionStrategy {
    ExecutionResult execute(List<Step> steps, ExecutionContext ctx);
}

public final class SequentialStrategy implements ExecutionStrategy {
    @Override
    public ExecutionResult execute(List<Step> steps, ExecutionContext ctx) {
        for (Step step : steps) {
            ExecutionResult r = step.run(ctx);
            if (r.status() != ExecutionStatus.SUCCEEDED) return r;
        }
        return ExecutionResult.success(Map.of());
    }
}

public final class ParallelStrategy implements ExecutionStrategy {
    private final ExecutorService pool;

    @Override
    public ExecutionResult execute(List<Step> steps, ExecutionContext ctx) {
        List<Future<ExecutionResult>> futures = steps.stream()
            .map(s -> pool.submit(() -> s.run(ctx)))
            .toList();
        for (Future<ExecutionResult> f : futures) {
            try {
                ExecutionResult r = f.get();
                if (r.status() != ExecutionStatus.SUCCEEDED) {
                    ctx.cancel();
                    return r;
                }
            } catch (Exception e) {
                return ExecutionResult.failed(e.getMessage());
            }
        }
        return ExecutionResult.success(Map.of());
    }
}
```

---

## 6) Template Method for steps

```java
public abstract class BaseStep implements PipelineCommand {
    @Override
    public final ExecutionResult execute(ExecutionContext ctx) {
        preExecute(ctx);
        try {
            ExecutionResult r = doExecute(ctx);
            postExecute(ctx, r);
            return r;
        } catch (Exception e) {
            return ExecutionResult.failed(e.getMessage());
        }
    }

    protected void preExecute(ExecutionContext ctx) {}
    protected abstract ExecutionResult doExecute(ExecutionContext ctx);
    protected void postExecute(ExecutionContext ctx, ExecutionResult r) {}
}
```

---

## 7) Builder fluent API (deliverable)

```java
Pipeline pipeline = new PipelineBuilder()
    .name("ci-java")
    .stage("build")
        .step("checkout", new CheckoutStep())
        .step("compile", new CompileStep())
        .parallel()
        .step("unit-tests", new UnitTestStep())
        .step("lint", new LintStep())
        .endStage()
    .stage("release")
        .dependsOn("build")
        .step("publish", new PublishArtifactStep("app.jar"))
        .step("deploy", new DeployStep())
        .endStage()
    .build();
```

---

## 8) Executor, triggers, and events

### Executor flow

1. Pre-check CoR (`Auth -> Resource -> Concurrency`).
2. Acquire runner from pool.
3. Create execution context.
4. Run pipeline.
5. Publish run lifecycle events.
6. Release runner.

### Trigger abstraction

- `PushTrigger`
- `PullRequestTrigger`
- `CronTrigger`
- `ManualTrigger`

Trigger registry resolves matching pipelines for incoming event context.

### Observer usage

`PipelineEventBus` notifies Slack/email/dashboard listeners on queued/running/finished transitions.

---

## 9) Artifact handoff safety

Publishing step writes named artifact into execution context/store.  
Downstream step must call `requireArtifact(name)` and fail clearly if missing.

Dependency prevention:

- Stage `dependsOn` enforced before scheduling.
- Sequential ordering inside dependent step groups.
- Missing artifact errors treated as execution failures.

---

## 10) CoR boundary vs executor boundary

Pre-execution chain determines whether a run is allowed to start (authorization/quota/concurrency checks).  
Executor handles actual orchestration (scheduling, retries, cancellation, artifact flow, status tracking).

---

## 11) Self-check with answers

1. **Composite vs Builder?**  
   Composite is runtime structure; Builder is construction mechanism.

2. **Why Command for steps instead of plain `Runnable`?**  
   Command carries metadata, retry/timeout policies, optional cleanup, and traceable step identity.

3. **Downstream runs before artifact publish: prevention?**  
   Enforce stage dependencies, ordered scheduling, and artifact presence checks before consuming steps.

---

## 12) First tests

1. Dependent stage blocked when upstream fails.
2. Parallel stage failure cancels remaining work.
3. Missing required artifact causes deterministic deploy failure.

---

## Day 5 checkpoint

- [x] Pipeline/stage/step model and execution contracts are defined.
- [x] Strategy and builder usage are clear and concrete.
- [x] Triggering, pre-check chain, and event notifications are integrated.
- [x] Retry/timeout/cancel and artifact safety concerns are addressed.
