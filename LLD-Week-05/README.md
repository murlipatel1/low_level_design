# Week 5 — Advanced LLD Part 1: Concurrency, Thread Safety & Performance (6-Day Track)

This folder breaks **Week 5** of the main plan into **six structured study days**: fundamentals → singleton → queues → pools → LRU/LFU → rate limiting deep dive + synthesis.

| Day | File | Topic (plan §) |
|-----|------|----------------|
| 1 | [Day-01-Concurrency-Fundamentals.md](./Day-01-Concurrency-Fundamentals.md) · [Answers](./Day-01-Concurrency-Fundamentals-Answers.md) | 5.1 Concurrency fundamentals |
| 2 | [Day-02-Thread-Safe-Singleton.md](./Day-02-Thread-Safe-Singleton.md) · [Answers](./Day-02-Thread-Safe-Singleton-Answers.md) | 5.2 Thread-safe singleton (all approaches) |
| 3 | [Day-03-Producer-Consumer-BlockingQueue.md](./Day-03-Producer-Consumer-BlockingQueue.md) · [Answers](./Day-03-Producer-Consumer-BlockingQueue-Answers.md) | 5.3 Producer–consumer & blocking queue |
| 4 | [Day-04-Object-Pool-Pattern.md](./Day-04-Object-Pool-Pattern.md) · [Answers](./Day-04-Object-Pool-Pattern-Answers.md) | 5.4 Object pool pattern |
| 5 | [Day-05-Thread-Safe-LRU-LFU-Cache.md](./Day-05-Thread-Safe-LRU-LFU-Cache.md) · [Answers](./Day-05-Thread-Safe-LRU-LFU-Cache-Answers.md) | 5.5 LRU cache (O(1)) + LFU extension |
| 6 | [Day-06-Rate-Limiter-Deep-Concurrency.md](./Day-06-Rate-Limiter-Deep-Concurrency.md) · [Answers](./Day-06-Rate-Limiter-Deep-Concurrency-Answers.md) | 5.6 Rate limiter deep concurrency + Week 5 capstone + bridge to Week 6 |

**Goal (from main plan):** Design **correct**, **high-performance**, **thread-safe** systems from first principles.

**Languages:** The plan references Java primitives (`synchronized`, `ReentrantLock`, etc.); each day notes **Python** equivalents where useful (`threading`, `queue.Queue`, `asyncio` boundaries).

**How to use:** Read concepts → reproduce problems on paper → implement in one language → compare throughput vs correctness. The main plan’s **Week 5 prompt template** (LRU thread-safety walkthrough + LFU follow-up + timed `BlockingQueue` drill) is embedded in **Day 5** for copy-paste AI practice.
