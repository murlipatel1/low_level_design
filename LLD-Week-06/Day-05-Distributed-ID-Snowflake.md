# Day 5 — Distributed ID Generator (Snowflake)

This lesson covers time-ordered distributed IDs using Snowflake, clock-skew handling, and practical comparison with UUIDv4 and UUIDv7.

---

## Learning objectives

- Explain Snowflake 64-bit layout and why it is sortable.
- Implement thread-safe ID generation with per-millisecond sequence control.
- Handle clock rollback safely.
- Compare Snowflake and UUIDv7 trade-offs for real systems.

---

## Snowflake bit layout

```text
| 1 sign bit | 41 timestamp bits | 10 machine bits | 12 sequence bits |
```

Implications:

- 41-bit timestamp (ms): ~69 years from custom epoch.
- 10-bit machine id: up to 1024 nodes.
- 12-bit sequence: up to 4096 IDs/ms per node.

Common custom epoch reduces timestamp magnitude and extends practical lifecycle.

---

## Pack and unpack helpers

```java
public final class SnowflakeLayout {
    public static final long EPOCH_MS = 1_577_836_800_000L; // 2020-01-01 example
    private static final int MACHINE_BITS = 10;
    private static final int SEQUENCE_BITS = 12;

    public static final long MAX_SEQUENCE = (1L << SEQUENCE_BITS) - 1; // 4095
    private static final int MACHINE_SHIFT = SEQUENCE_BITS;
    private static final int TIMESTAMP_SHIFT = MACHINE_BITS + SEQUENCE_BITS;

    public static long pack(long timestampMs, long machineId, long sequence) {
        return ((timestampMs - EPOCH_MS) << TIMESTAMP_SHIFT)
            | (machineId << MACHINE_SHIFT)
            | sequence;
    }

    public static long timestampMs(long id) {
        return (id >>> TIMESTAMP_SHIFT) + EPOCH_MS;
    }

    public static long machineId(long id) {
        return (id >>> MACHINE_SHIFT) & 0x3FFL;
    }

    public static long sequence(long id) {
        return id & MAX_SEQUENCE;
    }
}
```

Pseudocode:

```text
pack = ((ts - epoch) << 22) | (machine << 12) | seq
seq = id & 0xFFF
machine = (id >> 12) & 0x3FF
timestamp = (id >> 22) + epoch
```

---

## Throughput math

With 12-bit sequence:

- 4096 IDs per ms per machine
- 4,096,000 IDs per second theoretical max per machine

When sequence overflows in same millisecond, generator waits for next millisecond.

---

## `SnowflakeIdGenerator` (thread-safe)

```java
public final class SnowflakeIdGenerator {
    private final long machineId;
    private final long epochMs;

    private long lastTimestamp = -1L;
    private long sequence = 0L;

    public SnowflakeIdGenerator(long machineId, long epochMs) {
        if (machineId < 0 || machineId > 1023) {
            throw new IllegalArgumentException("machineId must be 0..1023");
        }
        this.machineId = machineId;
        this.epochMs = epochMs;
    }

    public synchronized long nextId() {
        long now = System.currentTimeMillis();

        if (now < lastTimestamp) {
            handleClockBackward(now);
            now = System.currentTimeMillis();
        }

        if (now == lastTimestamp) {
            sequence = (sequence + 1) & SnowflakeLayout.MAX_SEQUENCE;
            if (sequence == 0) {
                now = waitNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0;
        }

        lastTimestamp = now;
        return ((now - epochMs) << 22) | (machineId << 12) | sequence;
    }

    private long waitNextMillis(long lastTs) {
        long now = System.currentTimeMillis();
        while (now <= lastTs) now = System.currentTimeMillis();
        return now;
    }

    private void handleClockBackward(long now) {
        long offset = lastTimestamp - now;
        if (offset <= 5) {
            waitNextMillis(lastTimestamp);
            return;
        }
        throw new ClockMovedBackwardsException("Clock moved backward by " + offset + "ms");
    }
}
```

You can also implement with packed `AtomicLong` CAS loops, but synchronized version is interview-friendly and correct.

---

## Clock moved backward handling

Detection:

`if (currentTime < lastTimestamp) -> rollback`

Policy options:

- Wait briefly for small skew (NTP jitter).
- Throw on large rollback (fail closed).
- Advanced borrowed-future strategies (rarely needed in interviews).

Recommended LLD policy: wait small skew, throw if large.

---

## Machine ID uniqueness

Two processes with same machine id can generate duplicate IDs for same timestamp/sequence.

Machine id assignment options:

- Orchestrator-controlled environment variable
- StatefulSet ordinal + datacenter prefix
- Central config/lease registry

Uniqueness of machine id is an operational requirement, not just code detail.

---

## UUIDv7 design overview

UUIDv7 is time-ordered 128-bit UUID with timestamp in high bits and randomness/counter in lower bits.

Benefits:

- Standard UUID tooling and storage compatibility
- Better index locality than UUIDv4
- No explicit machine-id coordination like Snowflake

---

## Snowflake vs UUIDv4 vs UUIDv7

| Aspect | Snowflake | UUIDv4 | UUIDv7 |
|---|---|---|---|
| Size | 64-bit | 128-bit | 128-bit |
| Sortable | Yes | No | Yes |
| Coordination needed | Machine id uniqueness | None | None |
| DB index locality | Excellent | Poor | Good |
| Ecosystem standard | Custom | Universal | Growing standard |
| Entropy focus | Lower | High | Mixed (time + random) |

Important: sortable IDs are not inherently secure random tokens.

---

## Why sortable IDs help databases

- Clustered/B-tree indexes handle append-like inserts better than random insertion.
- Time-range scans become simpler with ID boundaries.
- Reduced fragmentation compared with UUIDv4-heavy write paths.

Caveat: monotonic writes can hotspot last page at extreme throughput; shard/partition when needed.

---

## Self-quiz with answers

1. **Why sortable ID for PK/time queries?**  
   Better index locality and efficient time-ordered scans.

2. **What if two JVMs share machineId?**  
   Duplicate IDs and data integrity failures are possible.

3. **Is UUIDv7 randomness always equivalent to UUIDv4?**  
   No; v7 dedicates high bits to timestamp, so entropy profile differs by design.

---

## First three tests

1. Round-trip pack/unpack for timestamp, machine id, and sequence.
2. Generate >4096 IDs in one ms and confirm overflow waits to next ms without duplicates.
3. Simulate clock rollback and verify configured wait/throw behavior.

---

## Interview sound bites

- "4096 IDs/ms per node from 12-bit sequence."
- "Clock rollback must be handled explicitly."
- "Machine-id uniqueness is operationally critical."
- "Snowflake is compact and sortable; UUIDv7 is standard and sortable."

---

## Day 5 checkpoint

- [x] Snowflake bit allocation and math
- [x] Thread-safe generator logic
- [x] Clock-skew safeguards
- [x] UUIDv7 comparison and selection criteria
- [x] Self-quiz and tests

**Next:** `Day-06-Mock-Interviews-Rubric.md`
