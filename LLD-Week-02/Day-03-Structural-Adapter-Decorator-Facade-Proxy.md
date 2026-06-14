# Day 3 - Structural Patterns: Adapter, Decorator, Facade, Proxy

**Theme:** Compose and wrap objects so systems stay flexible without rewriting core logic.

---

## Learning goals

By the end of this document, you should be able to:

- Translate incompatible APIs with Adapter.
- Layer runtime behavior with Decorator.
- Simplify subsystem usage with Facade.
- Control access transparently with Proxy.

---

## 1) Adapter

### Intent

Convert one interface into another expected by client code.

### Typical use

- Legacy SDK does not match your internal port.
- Vendor API shape is map/string-heavy but your domain needs typed methods.

### Logger adapter example

```java
public interface ILogger {
    void log(Level level, String message);
}

public final class LegacyLogWriter {
    public void write(int severity, String text) {
        System.out.println("[" + severity + "] " + text);
    }
}

public final class LegacyLogWriterAdapter implements ILogger {
    private final LegacyLogWriter delegate;

    public LegacyLogWriterAdapter(LegacyLogWriter delegate) {
        this.delegate = delegate;
    }

    @Override
    public void log(Level level, String message) {
        int severity = switch (level) {
            case DEBUG -> 10;
            case INFO -> 20;
            case WARN -> 30;
            case ERROR -> 40;
        };
        delegate.write(severity, message);
    }
}
```

Key idea: client code depends on `ILogger`, never on legacy type.

---

## 2) Decorator

### Intent

Wrap the same interface to add behavior dynamically and stackably.

### Typical use

- Retry, logging, auth, timeout wrappers for HTTP clients.
- Compression/encryption wrappers around storage streams.

### DataSource stack and order

```java
public interface DataSource {
    byte[] read(String key);
    void write(String key, byte[] data);
}
```

Use wrappers like:

- `CompressionDecorator`
- `EncryptionDecorator`

Recommended write pipeline: **compress then encrypt**, so data remains compressible before encryption.  
Order should be encoded and documented in composition root.

---

## 3) Facade

### Intent

Provide one simple entry point to a complex subsystem.

### Typical use

- `placeOrder()` over inventory, payment, shipping, notification services.
- `runPipeline()` over checkout, build, test, scan, deploy, notify.

### Order facade example

```java
public final class OrderFacade {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;
    private final NotificationService notifications;

    public OrderReceipt placeOrder(PlaceOrderCommand cmd) {
        inventory.reserve(cmd.items());
        PaymentResult pay = payment.charge(cmd.invoice());
        if (!pay.isSuccess()) {
            inventory.release(cmd.items());
            throw new PaymentFailedException(pay.reason());
        }
        TrackingId tracking = shipping.dispatch(cmd.shippingAddress(), cmd.items());
        notifications.sendOrderConfirmation(cmd.customerEmail(), tracking);
        return OrderReceipt.of(cmd.orderId(), tracking, pay.transactionId());
    }
}
```

Facade simplifies caller experience but still delegates to dedicated subsystem services.

---

## 4) Proxy

### Intent

Use a surrogate with the same interface to control access to a real object.

Common proxy flavors:

- Virtual (lazy creation)
- Protection (authorization)
- Caching
- Remote

### Caching repository proxy example

```java
public final class CachingUserRepositoryProxy implements IUserRepository {
    private final IUserRepository delegate;
    private final Cache<UserId, User> cache;

    public CachingUserRepositoryProxy(IUserRepository delegate, Duration ttl) {
        this.delegate = delegate;
        this.cache = Caffeine.newBuilder().expireAfterWrite(ttl).build();
    }

    @Override
    public Optional<User> findById(UserId id) {
        User cached = cache.getIfPresent(id);
        if (cached != null) return Optional.of(cached);
        Optional<User> user = delegate.findById(id);
        user.ifPresent(u -> cache.put(id, u));
        return user;
    }
}
```

Proxy and Decorator can look structurally similar; the difference is primary intent (access control vs feature layering).

---

## 5) Pattern selection drill

| Scenario | Best pattern | Why |
|----------|--------------|-----|
| Legacy CSV -> `List<User>` | Adapter | Converts incompatible interface |
| Add gzip with same callers | Decorator | Adds behavior on same interface |
| One-click deploy over many tools | Facade | Single simplified entry point |
| Transparent cache on `getUser` | Proxy | Surrogate controls access/result path |

---

## 6) Mixed example: Adapter + wrappers

Scenario: `CheckoutService` expects `IPaymentGateway`, vendor SDK exposes `processPayment(Map)`.

Design:

1. `VendorPaymentAdapter` converts `charge(...)` to vendor payload/response.
2. Wrap adapter with `LoggingProxy` and `RetryDecorator` (or proxy wrappers) as needed.

Result: `CheckoutService` stays unchanged while integration logic and cross-cutting behavior remain composable.

---

## 7) Self-check with answers

1. **Adapter vs Facade: who hides subsystem types more fully?**  
   Facade, because it can hide multiple subsystem types behind one entry interface.

2. **How do you avoid decorator ordering bugs?**  
   Centralize wrapper assembly in one factory/composition root and verify order with integration tests.

3. **When can Proxy and Decorator become hard to distinguish?**  
   When both wrap same interface and delegate; naming should follow team intent and system responsibility boundaries.

---

## 8) Interview quick lines

- Adapter changes interface; Decorator keeps interface and adds behavior.
- Facade simplifies many dependencies for callers.
- Proxy controls access while preserving contract.
- State intent first when patterns overlap structurally.

---

## Day 3 checkpoint

- [x] I can implement adapters for legacy logger/payment/data readers.
- [x] I can design decorator chains and justify wrapper order.
- [x] I can model facade orchestration without creating god classes.
- [x] I can apply caching/auth/logging proxies on service ports.
