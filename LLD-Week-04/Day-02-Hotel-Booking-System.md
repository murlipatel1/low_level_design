# Day 2 - Hotel Booking System

**Theme:** Design hotel search and booking with date-range inventory, lifecycle-safe transitions, and payment consistency.

---

## Learning goals

By the end of this document, you should be able to:

- Model hotels, room types, and booking inventory by date range.
- Handle booking state transitions with payment and cancellation alignment.
- Prevent double booking under concurrency.
- Use Builder, Strategy, State, and Observer in practical places.

---

## 1) Clarifying decisions

For this design, we choose:

1. Room-type inventory pool model (physical room assignment can happen at check-in).
2. Hotel-local timezone and half-open date ranges `[checkIn, checkOut)`.
3. Tiered cancellation policy per room type/rate plan.
4. Payment hold at confirmation, capture at check-in.
5. One review per completed booking after checkout.

---

## 2) Core entities

| Entity | Responsibility |
|--------|----------------|
| `Hotel` | Property metadata and room-type offerings |
| `RoomType` | Capacity, amenities, base rate, policy |
| `Room` | Physical room unit (optional assignment stage) |
| `Guest` | Identity and payment profile |
| `Booking` | Dates, room type, status, payment linkage, price snapshot |
| `Payment` | Hold/capture/refund records |
| `Review` | Post-stay guest feedback |
| `SearchService` | Query/filter/rank hotels |
| `InventoryService` | Availability reserve/release per date range |

---

## 3) Availability model

Use per-night inventory counters by hotel + room type.

```java
public final class DateRange {
    private final LocalDate checkIn;   // inclusive
    private final LocalDate checkOut;  // exclusive

    public List<LocalDate> nights() {
        return checkIn.datesUntil(checkOut).toList();
    }
}
```

```java
public final class InventoryService {
    // key: hotelId + roomTypeId + night -> remaining units
    private final ConcurrentHashMap<String, Integer> dailyAvailability = new ConcurrentHashMap<>();

    public synchronized boolean tryReserve(String hotelId, String roomTypeId, DateRange range, int rooms) {
        for (LocalDate night : range.nights()) {
            String key = key(hotelId, roomTypeId, night);
            if (dailyAvailability.getOrDefault(key, 0) < rooms) return false;
        }
        for (LocalDate night : range.nights()) {
            String key = key(hotelId, roomTypeId, night);
            dailyAvailability.merge(key, -rooms, Integer::sum);
        }
        return true;
    }

    public synchronized void release(String hotelId, String roomTypeId, DateRange range, int rooms) {
        for (LocalDate night : range.nights()) {
            String key = key(hotelId, roomTypeId, night);
            dailyAvailability.merge(key, rooms, Integer::sum);
        }
    }
}
```

Availability pseudocode:

```text
for each date in [checkIn, checkOut):
  if availableCount(date) < requestedRooms:
    return false
return true
```

---

## 4) Search with Builder

```java
public final class HotelSearchQuery {
    private final String city;
    private final DateRange dates;
    private final int guests;
    private final Optional<Money> maxPricePerNight;
    private final Optional<Integer> minStars;
    private final Set<String> requiredAmenities;

    private HotelSearchQuery(Builder b) { /* assign */ }

    public static Builder builder(String city, LocalDate in, LocalDate out, int guests) {
        return new Builder(city, in, out, guests);
    }

    public static final class Builder {
        public Builder maxPrice(Money m) { /* ... */ return this; }
        public Builder minStars(int s) { /* ... */ return this; }
        public Builder amenity(String a) { /* ... */ return this; }
        public HotelSearchQuery build() { return new HotelSearchQuery(this); }
    }
}
```

Builder is ideal because optional filter combinations grow quickly.

---

## 5) Booking lifecycle (State)

State flow:

`Requested -> Confirmed -> CheckedIn -> CheckedOut`  
`Requested/Confirmed -> Cancelled`

```java
public enum BookingStatus {
    REQUESTED, CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED
}

public interface BookingState {
    void confirm(Booking booking);
    void checkIn(Booking booking);
    void checkOut(Booking booking);
    void cancel(Booking booking);
}
```

Transition rules should enforce valid progressions and reject invalid operations centrally.

---

## 6) Payment mapping (hold/capture/refund)

| Transition | Payment action |
|-----------|----------------|
| `Requested -> Confirmed` | Authorize/hold total amount |
| Hold failure | Booking fails or cancels; inventory release |
| `Confirmed -> CheckedIn` | Capture hold |
| `Confirmed -> Cancelled` | Void hold or refund per policy |
| `CheckedOut` | Settlement complete |

---

## 7) Booking service flow

```java
public final class BookingService {
    public Booking createBooking(CreateBookingRequest req) {
        if (!inventory.tryReserve(req.hotelId(), req.roomTypeId(), req.dates(), 1))
            throw new NotAvailableException();

        Money quote = pricing.calculate(req);
        Booking booking = Booking.requested(req, quote); // stores price snapshot

        try {
            String paymentId = paymentGateway.authorize(req.guest(), quote);
            booking.attachPayment(paymentId);
            booking.confirm();
            eventBus.publish(new BookingConfirmedEvent(booking));
            return booking;
        } catch (Exception ex) {
            inventory.release(req.hotelId(), req.roomTypeId(), req.dates(), 1);
            booking.cancel();
            throw ex;
        }
    }
}
```

Price snapshot is critical to preserve booking-time commercial agreement even if rates later change.

---

## 8) Observer for confirmations

On successful confirmation, publish `BookingConfirmedEvent` to:

- Email/SMS confirmation listeners
- Analytics listeners
- Internal workflows

---

## 9) Reviews

Allow review submission only when:

- Booking exists,
- status is `CHECKED_OUT`,
- no prior review for that booking.

---

## 10) Extensions

- Dynamic/yield pricing strategy by occupancy.
- Loyalty accrual/redemption integration.
- Reservation holds with expiry and waitlist.
- Multi-property chain search ranking.

---

## 11) Self-check with answers

1. **Why store price snapshot on booking?**  
   Protects customer contract and auditability against future rate changes.

2. **How to prevent double booking?**  
   Inventory reserve in transactional boundary with lock/atomic update; release on payment failure.

3. **When is Builder better for search?**  
   When many optional query filters make constructors/overloads unreadable.

---

## 12) First tests

1. Concurrent booking on last unit: one success, one not-available failure.
2. Cancel-from-confirmed applies correct policy and inventory release.
3. Review before checkout is rejected.

---

## Day 2 checkpoint

- [x] Availability model and date-range semantics are defined.
- [x] Booking states and payment transitions are aligned.
- [x] Search builder and observer notifications are integrated.
- [x] Concurrency and consistency risks are addressed.
