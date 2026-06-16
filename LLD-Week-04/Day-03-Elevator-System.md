# Day 3 - Elevator System

**Theme:** Coordinate multi-elevator movement safely with clear dispatch and scheduling responsibilities.

---

## Learning goals

By the end of this document, you should be able to:

- Separate building-level dispatch from per-car movement logic.
- Model elevator state transitions and enforce safety invariants.
- Apply SCAN/LOOK scheduling as strategy without changing core loop.
- Handle overload and emergency modes explicitly.

---

## 1) Clarifying decisions

For this design, we choose:

1. Central `ElevatorController` for hall-call assignment.
2. Per-elevator scheduler for local stop ordering.
3. Hall requests queued globally; cabin requests stored per elevator.
4. Overload blocks door close and movement.
5. Fire/emergency mode returns cars to ground and suspends normal calls.

---

## 2) Core entities

| Entity | Responsibility |
|--------|----------------|
| `ElevatorController` | Assign hall calls, run system tick loop |
| `Elevator` | Maintain floor, direction, state, and local queue execution |
| `RequestQueue` | Track hall and cabin stops with direction context |
| `SchedulingStrategy` | Per-elevator next-stop logic (SCAN/LOOK) |
| `DispatchStrategy` | Choose best elevator for hall requests |
| `ElevatorDoor` | Door open/close cycle and safety checks |
| `FloorArrivalObserver` | Display/chime/integration listeners |

---

## 3) State model

`IDLE -> MOVING_UP / MOVING_DOWN -> DOOR_OPENING -> DOOR_OPEN -> DOOR_CLOSING -> IDLE`

Extended statuses:

- `EMERGENCY`
- `OUT_OF_SERVICE`

Key invariant: elevator cannot move while doors are not fully closed.

---

## 4) Architecture sketch

```java
public final class ElevatorController {
    private final List<Elevator> elevators;
    private final Queue<HallRequest> pendingHall = new LinkedList<>();
    private final DispatchStrategy dispatch;

    public void submitHallRequest(HallRequest req) {
        pendingHall.offer(req);
    }

    public void tick() {
        while (!pendingHall.isEmpty()) {
            HallRequest req = pendingHall.peek();
            Optional<Elevator> chosen = dispatch.assign(elevators, req);
            if (chosen.isEmpty()) break;
            chosen.get().addHallStop(req);
            pendingHall.poll();
        }
        elevators.forEach(Elevator::tick);
    }
}
```

```java
public final class Elevator {
    private int currentFloor;
    private Direction direction = Direction.IDLE;
    private ElevatorStatus status = ElevatorStatus.IDLE;
    private final ElevatorDoor door;
    private final RequestQueue queue;
    private final SchedulingStrategy scheduler;
    private final double maxLoadKg;
    private double currentLoadKg;

    public void tick() {
        if (status == ElevatorStatus.EMERGENCY) {
            emergencyTick();
            return;
        }
        switch (status) {
            case IDLE -> pickDirectionAndMove();
            case MOVING_UP, MOVING_DOWN -> moveOneFloor();
            case DOOR_OPENING -> door.openStep();
            case DOOR_OPEN -> door.waitOpen();
            case DOOR_CLOSING -> {
                if (currentLoadKg > maxLoadKg) {
                    status = ElevatorStatus.DOOR_OPEN;
                    return;
                }
                door.closeStep();
            }
            default -> {}
        }
    }
}
```

---

## 5) SCAN scheduling (per car)

Rule:

- While moving up, serve all higher stops in ascending order.
- When none remain above, reverse direction and serve lower stops.

Example:

- Current floor: `3`, direction: `UP`, pending: `{1,2,4,7,9}`
- Service order: `4 -> 7 -> 9 -> 2 -> 1`

```java
public final class ScanSchedulingStrategy implements SchedulingStrategy {
    @Override
    public int nextStop(Elevator e, RequestQueue q) {
        int cur = e.currentFloor();
        if (e.direction() == Direction.UP) {
            return q.stopsAbove(cur).stream().min(Integer::compareTo)
                .orElseGet(() -> switchToDown(e, q));
        }
        if (e.direction() == Direction.DOWN) {
            return q.stopsBelow(cur).stream().max(Integer::compareTo)
                .orElseGet(() -> switchToUp(e, q));
        }
        return q.nearest(cur);
    }
}
```

---

## 6) Dispatch strategy (building level)

`NearestCarDispatch` can choose elevator by ETA and direction compatibility.

SCAN answers “which stop next for this car.”  
Dispatch answers “which car should get this hall request.”

---

## 7) Hall vs cabin request separation

Separate structures avoid mixing semantics:

- Hall requests include direction intent (UP/DOWN).
- Cabin requests are destination selections by passengers already inside.

This improves scheduling correctness and dispatch clarity.

---

## 8) Sequence flow (floor 3 UP, then cabin 7)

1. User presses UP at floor 3.
2. Controller receives hall request and assigns elevator.
3. Elevator moves to floor 3, opens door.
4. User enters and presses 7.
5. Cabin request added to that elevator queue.
6. Elevator continues SCAN loop to floor 7.

---

## 9) Observer and safety hooks

`FloorArrivalObserver` receives arrival events for displays/chimes/tests.

Overload behavior:

- If overweight during `DOOR_CLOSING`, return to `DOOR_OPEN`.
- Do not transition to moving state until resolved.

Emergency behavior:

- Cancel normal queue.
- Return to floor 0.
- Enter `OUT_OF_SERVICE` or emergency hold.

---

## 10) Extensions

- VIP floor priority strategy.
- Off-hours energy policy to park idle cars at ground.
- Zone-based dispatch for high-rise buildings.

---

## 11) Self-check with answers

1. **Why separate hall and cabin requests?**  
   Different source semantics and dispatch/scheduling ownership.

2. **What if movement is allowed while door open?**  
   Severe safety violation and state-machine inconsistency.

3. **SCAN vs nearest-elevator: per-car or building-level?**  
   SCAN is per-car stop ordering; nearest-elevator is building-level dispatch.

---

## 12) First tests

1. SCAN order correctness for mixed above/below requests.
2. Overload prevents movement until weight normalizes.
3. Emergency mode overrides queue and sends car to floor 0.

---

## Day 3 checkpoint

- [x] Controller vs elevator responsibilities are separated.
- [x] State/safety invariants are explicit.
- [x] SCAN and dispatch strategy boundaries are clear.
- [x] Request flow and extensions are documented.
