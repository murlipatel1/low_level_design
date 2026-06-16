# Day 5 - Online Food Ordering and Library Management

Two systems in one day:

- **Problem 4.5:** Online food ordering
- **Problem 4.8:** Library management

Shared design skills: state transitions, inventory/availability control, strategy-based policies, and event-driven notifications.

---

# Part A - Online Food Ordering

## Learning goals

By the end of this section, you should be able to:

- Model restaurant/menu/cart/order/delivery entities.
- Design order lifecycle and payment alignment.
- Use Facade for order placement while preserving explicit failure semantics.
- Use Observer for tracking and notifications.

---

## 1) Clarifying decisions

For this design, we choose:

1. One restaurant per cart.
2. Nearest idle partner assignment.
3. Explicit payment-failure and refund-pending paths.
4. Session-based carts (guest checkout optional).
5. Push updates through event bus; polling as fallback.

---

## 2) Core entities

| Entity | Responsibility |
|--------|----------------|
| `User` | Addresses, payment instruments, profile |
| `Restaurant` | Menu, hours, delivery zone |
| `MenuItem` | Price, availability, modifiers |
| `Cart` | Items, quantities, totals, restaurant scope |
| `Order` | Lifecycle + assignment + payment refs |
| `DeliveryPartner` | Availability and location |
| `Payment` | Authorization/capture/refund outcomes |
| `OrderPlacementFacade` | Coordinates place-order transaction |

---

## 3) Order state machine

`PLACED -> CONFIRMED -> PREPARING -> PICKED_UP -> OUT_FOR_DELIVERY -> DELIVERED`

Error/alternate states:

- `CANCELLED`
- `PAYMENT_FAILED`
- `REFUND_PENDING`

Transitions should be validated centrally.

---

## 4) Facade sequence (deliverable)

`OrderPlacementFacade.placeOrder(cart, paymentDetails)` flow:

1. Validate restaurant open + serviceability.
2. Reserve menu inventory.
3. Persist order in placed state.
4. Execute payment strategy.
5. On success: confirm order, notify kitchen, trigger dispatch, clear cart.
6. On failure: release inventory, set failure state, return explicit failure result.

Facade should not hide meaningful failures (`NotServiceable`, `OutOfStock`, `PaymentFailed`).

---

## 5) Strategy + Factory for payment

```java
public interface PaymentStrategy {
    PaymentResult pay(Order order, PaymentDetails details);
    PaymentResult refund(String paymentId, Money amount);
}

public final class PaymentStrategyFactory {
    public PaymentStrategy create(PaymentMethodType type) {
        return switch (type) {
            case UPI -> new UpiPaymentStrategy();
            case CARD -> new CardPaymentStrategy();
            case WALLET -> new WalletPaymentStrategy();
        };
    }
}
```

---

## 6) Observer subscriptions

| Transition | Subscribers |
|-----------|-------------|
| `CONFIRMED` | User app, restaurant tablet, analytics |
| `PREPARING` | User tracking UI |
| `PICKED_UP` / `OUT_FOR_DELIVERY` | User app, partner app |
| `DELIVERED` | User receipt, review prompt, payout pipeline |
| `CANCELLED` / `REFUND_PENDING` | User app, support console |

---

## 7) Failure path: confirmed then capture fails

If order already confirmed/preparing and payment capture fails:

1. Transition to `REFUND_PENDING` or `PAYMENT_FAILED`.
2. Notify support + user.
3. Trigger refund/void workflow.
4. Resolve dispatch behavior (continue/cancel) by business policy.

---

## 8) Extension ideas

- Surge fee strategy by demand/weather/zone.
- Group ordering + split payments.
- Partner batching route optimization.

---

# Part B - Library Management

## Learning goals

By the end of this section, you should be able to:

- Separate title metadata from physical copies.
- Model issue/return/reservation/fine workflows.
- Apply Builder for search and Strategy for fine policy.
- Use Observer for reservation notifications.

---

## 1) Clarifying decisions

For this design, we choose:

1. `Book` (metadata) and `BookItem` (physical copy) are distinct.
2. Member-tier-based loan periods and limits.
3. FIFO reservation queue per `Book` title.
4. Max renewal policy with reservation-aware checks.
5. Reviews/recommendations optional extensions.

---

## 2) Core entities

| Entity | Responsibility |
|--------|----------------|
| `Book` | ISBN/title/author/genre metadata |
| `BookItem` | Barcode copy and lendable status |
| `Member` | Borrow limits and active loans |
| `Loan` | Due date, renewals, return state |
| `Reservation` | Queue position for title |
| `Fine` | Overdue amount and payment state |
| `CatalogService` | Search over book metadata |
| `CirculationService` | Issue/return/reservation orchestration |

---

## 3) Book vs BookItem relationship

- `Book`: searchable concept.
- `BookItem`: physical inventory unit.
- Loans attach to `BookItem`.
- Reservation queues attach to `Book`.

This prevents mixing catalog concerns with copy-level circulation.

---

## 4) Search with Builder

```java
public final class BookSearchQuery {
    private final Optional<String> title;
    private final Optional<String> author;
    private final Optional<String> isbn;
    private final Optional<String> genre;

    public static Builder builder() { return new Builder(); }
}
```

Builder handles many optional search filters cleanly.

---

## 5) Issue/return flow

Issue flow:

1. Validate member borrow limits.
2. Validate `BookItem` availability.
3. Create loan with due date based on member tier.
4. Mark item loaned.

Return flow:

1. Find active loan.
2. Compute fine using `FinePolicy` strategy.
3. Mark item available.
4. Pop next reservation (if any) and notify via observers.

---

## 6) Fine strategy

```java
public interface FinePolicy {
    Money compute(Loan loan, LocalDate returnDate);
}

public final class PerDayFinePolicy implements FinePolicy {
    private final Money perDay;
    private final int graceDays;
    private final Money cap;

    @Override
    public Money compute(Loan loan, LocalDate returnDate) {
        long overdueDays = ChronoUnit.DAYS.between(loan.dueDate(), returnDate) - graceDays;
        if (overdueDays <= 0) return Money.ZERO;
        Money raw = perDay.multiply((int) overdueDays);
        return raw.min(cap);
    }
}
```

---

## 7) Observer on return (deliverable)

On successful return:

- Mark item available.
- Check reservation queue for the book.
- Notify next member (email/SMS/app) with hold window.

---

## 8) Shared self-check (answers)

1. **Facade in food ordering: what should not be hidden?**  
   Explicit business failures and partial-commit states must remain visible to caller.

2. **Why separate `Book` and `BookItem`?**  
   Catalog/search metadata differs fundamentally from lendable copy inventory.

3. **Observer risks in both systems?**  
   Duplicate events, missed unsubscription, ordering issues, and handler failure isolation.

---

## 9) First tests

### Food Ordering

1. Multi-restaurant cart rejected.
2. Payment failure triggers inventory release and failure state.
3. Invalid status transition rejected.

### Library

1. Borrow limit enforcement.
2. Fine calculation with grace and cap.
3. Reservation notification triggered exactly once on return.

---

## Day 5 checkpoint

- [x] Food ordering lifecycle, facade flow, and payment handling are modeled.
- [x] Library metadata vs copy inventory separation is explicit.
- [x] Search builder and policy strategies are integrated.
- [x] Observer-driven notifications are covered in both domains.
