# Day 4 - Structural Patterns: Composite, Bridge, Flyweight

**Theme:** Handle trees, dual-axis variation, and large object counts without complexity explosion.

---

## Learning goals

By the end of this document, you should be able to:

- Model part-whole trees with a uniform interface (Composite).
- Separate two independent variation axes cleanly (Bridge).
- Share immutable intrinsic state for memory efficiency (Flyweight).

---

## 1) Composite

### Intent

Treat single objects and containers uniformly in tree structures.

### Best fit

- File/folder systems
- UI trees
- Organization hierarchies
- Pipeline step groups

### File system example

```java
public interface FileSystemNode {
    String getName();
    long getSize();
    void print(String indent);
}

public final class File implements FileSystemNode {
    private final String name;
    private final long sizeBytes;

    public File(String name, long sizeBytes) {
        this.name = name;
        this.sizeBytes = sizeBytes;
    }

    @Override public String getName() { return name; }
    @Override public long getSize() { return sizeBytes; }
    @Override public void print(String indent) {
        System.out.println(indent + "FILE " + name + " (" + sizeBytes + " B)");
    }
}

public final class Directory implements FileSystemNode {
    private final String name;
    private final List<FileSystemNode> children = new ArrayList<>();

    public Directory(String name) { this.name = name; }

    public void add(FileSystemNode node) { children.add(node); }

    @Override public String getName() { return name; }
    @Override public long getSize() { return children.stream().mapToLong(FileSystemNode::getSize).sum(); }
    @Override public void print(String indent) {
        System.out.println(indent + "DIR " + name + "/");
        String next = indent + "  ";
        children.forEach(c -> c.print(next));
    }
}
```

Design note: keep mutator methods like `add` on composite type (`Directory`) only, not leaf type (`File`), to avoid ISP/LSP pressure.

---

## 2) Bridge

### Intent

Split abstraction and implementation hierarchies so both can evolve independently.

### Best fit

When two independent axes exist, for example:

- Notification type × delivery channel
- Export format × data source
- Device abstraction × driver implementation

### Messaging bridge example

```java
public interface MessageChannel {
    void deliver(String recipient, String body);
}

public final class EmailChannel implements MessageChannel {
    @Override public void deliver(String recipient, String body) { /* SMTP */ }
}

public final class SmsChannel implements MessageChannel {
    @Override public void deliver(String recipient, String body) { /* SMS API */ }
}

public abstract class Notification {
    protected final MessageChannel channel;

    protected Notification(MessageChannel channel) {
        this.channel = channel;
    }

    public abstract void send(String recipient, String message);
}

public final class StandardNotification extends Notification {
    public StandardNotification(MessageChannel channel) { super(channel); }
    @Override public void send(String recipient, String message) { channel.deliver(recipient, message); }
}

public final class UrgentNotification extends Notification {
    public UrgentNotification(MessageChannel channel) { super(channel); }
    @Override public void send(String recipient, String message) {
        channel.deliver(recipient, "URGENT: " + message.toUpperCase());
    }
}
```

This avoids subclass explosion like `UrgentEmailNotification`, `UrgentSmsNotification`, etc.

---

## 3) Flyweight

### Intent

Share common intrinsic state across many objects and keep per-instance state extrinsic.

### Best fit

- Text editor style objects
- Game object type metadata
- Large repeated visual/config descriptors

### Intrinsic vs extrinsic

- **Intrinsic:** shared immutable data (font family, tree texture)
- **Extrinsic:** per-object context (x/y position, char location)

### Character style flyweight sketch

```java
public final class CharacterStyle {
    private final String fontFamily;
    private final int fontSize;
    private final boolean bold;
    private final int rgb;

    public CharacterStyle(String fontFamily, int fontSize, boolean bold, int rgb) {
        this.fontFamily = fontFamily;
        this.fontSize = fontSize;
        this.bold = bold;
        this.rgb = rgb;
    }

    public String key() {
        return fontFamily + "|" + fontSize + "|" + bold + "|" + rgb;
    }
}

public final class CharacterStyleFactory {
    private final Map<String, CharacterStyle> cache = new ConcurrentHashMap<>();

    public CharacterStyle get(String font, int size, boolean bold, int rgb) {
        CharacterStyle probe = new CharacterStyle(font, size, bold, rgb);
        return cache.computeIfAbsent(probe.key(), k -> probe);
    }
}
```

Flyweight safety rule: intrinsic state must be immutable or effectively read-only.

---

## 4) Mixed problem: Org chart + export

Design goal:

- Use Composite for `Department` + `Employee` hierarchy.
- Use Bridge for rendering same org report into HTML or PDF.

Suggested split:

- `OrgNode` (composite interface): `getHeadcount()`, `getCost()`
- `Department`, `Employee`
- `OrgReportRenderer` (bridge implementor): `HtmlOrgRenderer`, `PdfOrgRenderer`
- `OrgReportExporter` (bridge abstraction): one traversal, pluggable renderer

Result: traversal logic is written once; output format remains swappable.

---

## 5) Pattern comparison quick view

- Composite: tree structure with uniform node API.
- Bridge: two independent hierarchies connected by composition.
- Flyweight: memory optimization via shared immutable core state.

Common confusions:

- Composite vs Decorator: tree of children vs wrapper chain.
- Bridge vs Strategy: two axes vs one algorithm axis.
- Flyweight vs Singleton: many keyed shared objects vs one global instance.

---

## 6) Self-check with answers

1. **Where does ISP pressure appear in Composite?**  
   Composite-only methods (`addChild`) are meaningless for leaves; keep interfaces narrow or split mutator contracts.

2. **Example of two true Bridge axes?**  
   `ReportExporter` format axis (PDF/CSV) and data-source axis (SQL/API).

3. **Why must intrinsic Flyweight state be safe to share?**  
   Shared mutable intrinsic state causes cross-instance corruption and race conditions.

---

## 7) Interview quick lines

- Composite handles part-whole recursion cleanly.
- Bridge prevents N x M subclass growth.
- Flyweight trades complexity for memory efficiency at scale.
- State intrinsic/extrinsic separation explicitly when explaining Flyweight.

---

## Day 4 checkpoint

- [x] I can design file/org/pipeline trees with Composite.
- [x] I can identify two variation axes and model them with Bridge.
- [x] I can implement Flyweight factories and explain intrinsic/extrinsic state.
- [x] I can combine Composite and Bridge in a capstone-style design.
