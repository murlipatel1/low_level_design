# Day 1 - Parking Lot System

**Theme:** Design a multi-floor parking system with spot compatibility, ticket lifecycle, pricing strategies, and gate orchestration.

---

## Learning goals

By the end of this document, you should be able to:

- Model lot, floor, spot, vehicle, ticket, and gate responsibilities.
- Enforce spot-state transitions and compatibility rules safely.
- Use Strategy for pricing and Observer for availability updates.
- Handle entry/exit concurrency without breaking invariants.

---

## 1) Clarifying decisions

For this design, we choose:

1. Single parking lot aggregate per site.
2. Drive-up flow as core; reservation as optional extension.
3. Payment at exit.
4. Hard compatibility rules for spot/vehicle assignment.
5. Assignment and release operations serialized with lock at lot level (per-floor locking as optimization).

---

## 2) Compatibility matrix

| Vehicle \ Spot | Compact | Regular | Large | Handicapped | Electric |
|----------------|---------|---------|-------|-------------|----------|
| Bike | Yes | Yes | Yes | No | No |
| Car | Yes | Yes | Yes | No* | Yes |
| Van | No | Yes | Yes | No | Yes |
| Truck | No | No | Yes | No | No |

\*Handicapped usage can be enabled with permit extension.

---

## 3) State model

`AVAILABLE -> RESERVED -> OCCUPIED -> AVAILABLE`

If reservation is out of scope, simplify to `AVAILABLE <-> OCCUPIED`.

State transitions must be guarded internally (no public raw setter).

```java
public enum SpotStatus {
    AVAILABLE, RESERVED, OCCUPIED
}
```

---

## 4) Core entities

| Entity | Responsibility |
|--------|----------------|
| `ParkingLot` | Aggregate root; assign/release spots and manage active tickets |
| `ParkingFloor` | Holds and searches compatible available spots |
| `ParkingSpot` | Spot identity, type, status, and occupancy transitions |
| `Vehicle` | Vehicle type and license details |
| `Ticket` | Entry record and billing anchor |
| `EntryGate` | Intake flow and ticket issuance |
| `ExitGate` | Exit flow, billing trigger, release |
| `PricingStrategy` | Pricing rule abstraction |

---

## 5) Design sketch

```java
public final class ParkingLot {
    private final List<ParkingFloor> floors;
    private final Map<String, Ticket> activeTickets = new ConcurrentHashMap<>();
    private final PricingStrategy pricing;
    private final List<SpotAvailabilityObserver> observers = new CopyOnWriteArrayList<>();
    private final Object assignLock = new Object();

    public Ticket parkVehicle(Vehicle vehicle) {
        synchronized (assignLock) {
            ParkingSpot spot = findAvailableSpot(vehicle).orElseThrow(ParkingFullException::new);
            spot.occupy(vehicle);
            Ticket ticket = Ticket.issue(vehicle, spot, Instant.now());
            activeTickets.put(ticket.id(), ticket);
            return ticket;
        }
    }

    public PaymentReceipt releaseVehicle(Ticket ticket) {
        synchronized (assignLock) {
            ParkingSpot spot = getSpot(ticket.spotId());
            Money fee = pricing.calculate(ticket.entryTime(), Instant.now(), ticket);
            spot.release();
            activeTickets.remove(ticket.id());
            observers.forEach(o -> o.onSpotAvailable(spot));
            return new PaymentReceipt(ticket.id(), fee);
        }
    }
}
```

---

## 6) Factory + Strategy + Observer

### Factory

`VehicleFactory` can create vehicle from gate scan payload and enforce parsing/validation.

### Strategy

Pricing variants:

- Hourly
- Flat
- Overnight cap
- Membership discount wrappers (extension)

```java
public interface PricingStrategy {
    Money calculate(Instant entry, Instant exit, Ticket ticket);
}
```

### Observer

When spot is released, notify displays/waitlists/dashboard listeners.

```java
public interface SpotAvailabilityObserver {
    void onSpotAvailable(ParkingSpot spot);
}
```

---

## 7) Entry flow sequence

1. Driver reaches entry gate.
2. Gate scans plate/type and builds `Vehicle`.
3. Gate requests `parkVehicle(vehicle)` from lot.
4. Lot finds compatible available spot.
5. Spot transitions to occupied.
6. Ticket is issued and printed.

Exit flow:

1. Ticket scanned.
2. Lot calculates fee via pricing strategy.
3. Payment is collected.
4. Spot released and observers notified.

---

## 8) Extensions

- EV charging spot/session billing.
- Monthly pass or member-specific pricing.
- Reservation hold with expiry.
- Per-floor lock striping for higher gate throughput.

---

## 9) Self-check with answers

1. **Where can factory be replaced by enum map?**  
   If creation is trivial string-to-enum conversion with no validation or extra attributes.

2. **What breaks with public `setState` on spot?**  
   Invalid transitions and inconsistent occupancy/ticket invariants (double-booking, stale counts).

3. **Observer vs polling dashboard?**  
   Observer pushes immediate updates with less overhead; polling introduces lag and repeated read load.

---

## 10) First tests

1. Incompatible vehicle/spot assignment is rejected.
2. Full lot returns clear parking-full outcome.
3. Release transitions spot to available and emits exactly one availability event.

---

## Day 1 checkpoint

- [x] Compatibility and state rules are explicit.
- [x] Aggregate root boundaries are clear.
- [x] Pricing and event updates are decoupled via patterns.
- [x] Entry/exit flow and concurrency concerns are addressed.
