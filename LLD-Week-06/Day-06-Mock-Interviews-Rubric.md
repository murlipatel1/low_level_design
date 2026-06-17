# Day 6 — Mock Interview Problem Bank, Rubric, and Prompt Template

This day consolidates the Week 6 problem bank, scoring rubric, and mock-interview workflow into one practical practice guide.

---

## Part A — How to use the problem bank

### Suggested practice routine

- Run timed mocks: 45-60 minutes per problem.
- Split time into:
  1. Clarification (5 min)
  2. Entities/interfaces (10-15 min)
  3. Core flows/state machine (10 min)
  4. Pattern justification (5 min)
  5. Edge cases + tests (5-10 min)

### Session plan example

- Session 1: 2 Tier-1 problems
- Session 2: 1 Tier-2 problem
- Session 3: 1 Tier-3 problem + rubric scoring

---

## Tier 1 — Easy problems (25-30 min each)

| Problem | Key patterns | Watch for |
|---|---|---|
| Vending Machine | State, Singleton/DI | coin/change transitions, empty slot |
| Notification System | Observer, Strategy, Factory | fanout failures, retry/idempotency |
| ATM Machine | State, Facade | PIN lockout, concurrent debit safety |
| Tic-Tac-Toe | Template/Iterator | win/draw logic, invalid moves |
| URL Shortener (in-memory) | Facade, Singleton/DI | collision, expiry, fast lookup |

Drill: pick any two and complete in 30 minutes each.

---

## Tier 2 — Medium problems (40-45 min each)

| Problem | Key patterns | Watch for |
|---|---|---|
| Hotel Booking | Builder, State, Strategy | double booking, cancellation/refund |
| Shopping Cart | Composite, Strategy, Observer | coupon/tax/inventory coupling |
| Snake and Ladder | Command, Iterator | turn order, undo constraints |
| Amazon Locker | State, Factory, Observer | expiry and auto-release |
| Splitwise Clone | Strategy, Observer | debt simplification, multi-currency |
| Cricket Scorecard | Template, Observer | innings transitions and edge events |

Drill: one full 45-minute mock, then self-score.

---

## Tier 3 — Hard problems (55-60 min each)

| Problem | Key patterns | Watch for |
|---|---|---|
| Stock Exchange | Mediator, Observer, Command | matching engine correctness |
| Distributed Task Queue (in-process) | Command, Observer, Pool | retries, DLQ, priority |
| Code Review Tool | Observer, State, Composite | PR state and approvals |
| Collaborative Editor | Command, Memento, Observer | conflict/undo semantics |
| API Gateway (in-process) | CoR, Proxy, Strategy | auth, rate limit, breaker/retry |

Flagship choice: API Gateway for end-to-end cross-cutting design.

---

## Part B — Self-assessment rubric (10 dimensions)

Score each dimension 1-5 (total /50):

| Dimension | Evaluate |
|---|---|
| Requirements clarity | quality of scoping/clarifying questions |
| Entity modeling | cohesion, missing actors, value objects |
| Class hierarchy | composition vs inheritance decisions |
| Pattern application | justified usage, not pattern bingo |
| Interface design | clean contracts and SRP |
| State machines | complete states + invalid transition blocking |
| Concurrency | shared-state safety and hot-path choices |
| Extensibility | feature addition without major rewrites |
| Edge cases | failure/concurrency/timeouts/null/empty |
| Code quality | readability and modularity |

Target benchmark: **40+/50** for strong senior-level performance.

### Score calibration

- 3 = adequate mid-level
- 4 = strong senior
- 5 = staff-level trade-off depth and proactive extension thinking

Most candidates under-score on concurrency and edge-case coverage.

---

## Part C — Mock interview prompt template

```text
Act as a Staff Engineer interviewing me for a FAANG senior SDE role.

Problem: [Design an API Gateway (in-process)]

Conduct a realistic 45-minute interview:
- Minute 0-5: Ask me clarifying questions
- Minute 5-15: I'll define entities and interfaces - challenge weak spots
- Minute 15-30: I'll write core classes and key methods - ask design questions
- Minute 30-40: Walk through end-to-end flow and identify gaps
- Minute 40-45: Give one extension problem

After the session:
1. Score me on the 10-dimension rubric (1-5 each)
2. Name top 3 strengths
3. Name top 3 concrete improvements with examples
```

Replace problem line with any Tier 1/2/3 item.

---

## Part D — API Gateway flagship reference (in-process)

### Clarifying choices

- Scope: single JVM (LLD focus), route to backend adapters.
- Filters: auth, rate limit, circuit breaker, retry, routing.
- Error mapping: 401/403/429/503 explicit ownership by filter.

### Core interfaces

```java
public interface GatewayFilter {
    void filter(GatewayContext ctx, FilterChain chain) throws Exception;
}

public interface RouteResolver {
    Optional<Route> resolve(GatewayRequest request);
}

public interface Backend {
    GatewayResponse invoke(GatewayRequest request) throws Exception;
}
```

### Chain-of-responsibility flow

```java
public final class FilterChain {
    private final List<GatewayFilter> filters;
    private int index;

    public void proceed(GatewayContext ctx) throws Exception {
        if (index == filters.size()) {
            ctx.setResponse(ctx.getRoute().backend().invoke(ctx.getRequest()));
            return;
        }
        filters.get(index++).filter(ctx, this);
    }
}
```

Typical filter order:

1. Logging
2. Authentication
3. Rate limit (token bucket)
4. Circuit breaker
5. Retry (only when breaker allows)
6. Routing/backend invocation

---

## Part E — Week 6 synthesis links

### Gateway integration with prior weeks

- Chain of Responsibility from pattern week for filter pipeline.
- Circuit breaker + retry from Week 6 resilience.
- Token bucket rate limiter from Week 5 concurrency.
- Observer/event bus for async metrics and alerts.

### Task queue integration

- Blocking queue and worker pool from Week 5.
- Retry + DLQ concepts from event bus day.
- Snowflake IDs for task identifiers.

---

## Part F — Post-mock feedback template

```markdown
## Rubric Scores (/50)
[table]

## Top 3 strengths
1. ...
2. ...
3. ...

## Top 3 improvements (specific)
1. ...
2. ...
3. ...

## Extension attempted
[short note]
```

---

## Day 6 self-quiz with answers

1. **One Tier-2 problem where State is essential vs one where enum+guards can be enough?**  
   Essential: Amazon Locker or ATM session flow.  
   Enum+guards often sufficient: URL shortener lifecycle (active/expired).

2. **Which rubric dimensions are most under-scored?**  
   Concurrency reasoning and edge-case handling.

3. **Why is in-process API Gateway still a favorite interview problem?**  
   It tests architectural composition of cross-cutting concerns without requiring full distributed infra setup.

---

## 6-week capstone checklist

| Week | You should now be able to do |
|---|---|
| 1 | SOLID + modeling fundamentals |
| 2 | Pattern selection with justification |
| 3 | System components (logging/cache/DI/rate limit) |
| 4 | Consumer-domain LLD with state machines |
| 5 | Concurrency correctness patterns |
| 6 | Event-driven + resilience + distributed IDs + interview delivery |

---

## Interview sound bites

- "Clarify scope early or redesign later."
- "Entities before patterns."
- "Name explicit failure ownership (401/429/503)."
- "Show concurrency and edge-case discipline to break into 40+ rubric range."

---

## Day 6 checkpoint

- [x] Tiered problem bank practice strategy
- [x] Rubric usage with scoring calibration
- [x] API Gateway flagship walk-through
- [x] Week synthesis and post-mock loop
- [x] Self-quiz and capstone completion

**Week 6 complete.**
