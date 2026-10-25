# Concurrent Collections in Java

## Overview

Concurrent collections are thread-safe data structures designed for access by multiple threads. The `java.util.concurrent` package provides concurrent maps, sets, lists, queues, and deques with different guarantees for atomicity, iteration, ordering, and coordination.

They encapsulate concurrency control, but **a thread-safe collection does not automatically make an algorithm or its stored elements thread-safe**.

## Why They Exist

Most ordinary mutable Java collections, such as `HashMap` and `ArrayList`, are not thread-safe. Concurrent access that includes modifications without appropriate synchronization can lead to lost updates, visibility problems, inconsistent observations, or corruption of internal state.

External synchronization can make ordinary collections safe, but all access must follow the same locking protocol. A single lock may also limit scalability under contention.

Concurrent collections offer specialized synchronization and coordination strategies. They are not always faster: the best choice depends on the workload and required guarantees.

## Synchronized Wrappers vs. Concurrent Collections

`Collections.synchronizedCollection()`, `synchronizedList()`, `synchronizedSet()`, and `synchronizedMap()` wrap an existing collection and synchronize individual operations on a shared monitor.

- All access must go through the wrapper, not the original collection.
- Iteration generally requires manually synchronizing on the wrapper for the entire traversal.
- Compound operations, such as `containsKey()` followed by `put()`, require an external critical section if the entire sequence must be atomic.
- A common lock can cause contention.

Concurrent implementations use techniques appropriate to their contracts, including finer-grained synchronization, atomic operations, and copy-on-write. **Concurrent does not universally mean lock-free**.

For example, `ConcurrentHashMap` permits concurrent retrievals and scalable updates, whereas `Collections.synchronizedMap(new HashMap<>())` serializes individual method calls through a shared monitor.

## General Guarantees and Limits

### Atomicity

Each collection guarantees atomicity only for the operations specified by its API. **Multiple individually thread-safe operations are not necessarily atomic as a group.**

Incorrect — read-modify-write can lose updates:

```java
ConcurrentHashMap<String, Integer> counters = new ConcurrentHashMap<>();
counters.put("orders", 0);

// Concurrent callers may read the same old value.
counters.put("orders", counters.get("orders") + 1);
```

Correct — one atomic remapping operation for the key:

```java
counters.compute("orders", (key, value) -> value + 1);
```

For a highly contended frequency counter, a `ConcurrentHashMap<String, LongAdder>` can be more scalable; create the adder atomically with `computeIfAbsent()` and then call `increment()`. Its `sum()` is not an atomic snapshot under concurrent updates.

### Memory Visibility and Safe Publication

For concurrent collections, actions in one thread **before placing an object into the collection happen-before** actions in another thread **after accessing or removing that element**.

This supports safe publication of the state written before insertion, but **does not make later modifications to a mutable element thread-safe**:

```java
ConcurrentHashMap<String, List<String>> groups = new ConcurrentHashMap<>();
groups.put("admins", new ArrayList<>());

// The map is thread-safe; the shared ArrayList is not.
groups.get("admins").add("Alice");
```

Prefer immutable values, or protect mutable values separately (for example, with synchronization or an appropriate concurrent collection).

### Iteration Models

| Model | Behavior | Example |
| --- | --- | --- |
| **Fail-fast** | May throw `ConcurrentModificationException` after structural modification; detection is best-effort, not a safety guarantee | `ArrayList` |
| **Weakly consistent** | Can traverse while updates occur; does not provide a single fixed snapshot and may reflect some concurrent changes | `ConcurrentHashMap`, `ConcurrentLinkedQueue` |
| **Snapshot** | Iterates over the array state captured when the iterator was created; does not see later updates | `CopyOnWriteArrayList`, `CopyOnWriteArraySet` |

Weakly consistent iterators of the standard concurrent collections do not throw `ConcurrentModificationException` because of concurrent updates. Iteration semantics are implementation-specific; do not assume all traversals represent one instant in time.

### Coordination, Ordering, and Capacity

- A thread-safe collection does not necessarily preserve ordering. Ordering depends on the implementation (FIFO, sorted keys, priority order, etc.).
- A `BlockingQueue` offers waiting operations such as `take()` and `put()`; whether insertion can wait for capacity depends on whether the queue is bounded.
- Other queue APIs, such as `ConcurrentLinkedQueue.poll()`, return immediately rather than waiting for a new element.
- Operations such as `size()`, `isEmpty()`, and `containsKey()` are observations, not reservations: another thread can modify the collection before a subsequent operation. Some concurrent collections also provide non-constant-time or non-snapshot size observations.

## Types of Concurrent Collections

### Concurrent Maps

| Implementation | Characteristics | Typical use |
| --- | --- | --- |
| [ConcurrentHashMap](2_ConcurrentHashMap.md) | Thread-safe hash-based map; no key ordering; atomic per-key operations such as `compute()` | Shared caches, registries, counters |
| [ConcurrentSkipListMap](ConcurrentSkipListMap.md) | Concurrent sorted/navigable map | Ordered keys, range queries |

### Concurrent Sets

| Implementation | Characteristics | Typical use |
| --- | --- | --- |
| [ConcurrentSkipListSet](ConcurrentSkipListSet.md) | Concurrent sorted/navigable set | Sorted unique elements |
| [CopyOnWriteArraySet](CopyOnWriteArraySet.md) | Copy-on-write array; snapshot iteration | Small, read-heavy sets with rare changes |
| `ConcurrentHashMap.newKeySet()` | Concurrent hash-based set view, without sorted ordering | General-purpose concurrent membership tracking |

### Copy-on-Write Lists

| Implementation | Characteristics | Typical use |
| --- | --- | --- |
| [CopyOnWriteArrayList](CopyOnWriteArrayList.md) | Every mutation creates a new backing array; snapshot iterators | Listener lists, read-mostly configurations |

Copy-on-write collections optimize stable iteration and frequent reads at the expense of allocation and copying on updates. They are usually a poor fit for write-heavy workloads or very large lists with frequent modifications.

### Non-blocking Queues and Deques

| Implementation | Characteristics | Typical use |
| --- | --- | --- |
| [ConcurrentLinkedQueue](ConcurrentLinkedQueue.md) | Unbounded, thread-safe, non-blocking FIFO queue | Concurrent task/event buffering without blocking waits |
| [ConcurrentLinkedDeque](ConcurrentLinkedDeque.md) | Unbounded, thread-safe, non-blocking double-ended queue | Access at both ends |

Here, **non-blocking** describes the queue implementations' progress-oriented operations. It is distinct from the simple fact that `poll()` returns without waiting for an element. Always check the contract of the individual implementation and operation.

### Blocking Queues and Deques

| Implementation | Capacity / ordering | Typical use |
| --- | --- | --- |
| [ArrayBlockingQueue](ArrayBlockingQueue.md) | Fixed capacity, FIFO, array-backed | Bounded producer-consumer pipelines; backpressure |
| [LinkedBlockingQueue](LinkedBlockingQueue.md) | Optional bound (default capacity is `Integer.MAX_VALUE`), FIFO, linked nodes | Producer-consumer pipelines |
| [LinkedBlockingDeque](LinkedBlockingDeque.md) | Optional bound, double-ended | Blocking access at either end |
| [SynchronousQueue](SynchronousQueue.md) | Zero storage capacity; direct handoff | Producer-consumer rendezvous |
| [PriorityBlockingQueue](PriorityBlockingQueue.md) | Logically unbounded; priority ordering | Concurrent priority scheduling |
| [DelayQueue](DelayQueue.md) | Unbounded; elements become available after their delay expires | Delayed tasks and expirations |
| [LinkedTransferQueue](LinkedTransferQueue.md) | Unbounded FIFO transfer queue; `transfer()` can wait for consumption | Producer-to-consumer handoff with optional waiting |

**Blocking vs. non-blocking operations:**

```java
BlockingQueue<String> tasks = new ArrayBlockingQueue<>(100);

tasks.put("send-email");  // May wait until capacity becomes available.
String task = tasks.take(); // May wait until an element is available.
```

```java
Queue<String> tasks = new ConcurrentLinkedQueue<>();

tasks.offer("send-email");
String task = tasks.poll(); // Returns null immediately if empty.
```

**Important exceptions:** `PriorityBlockingQueue` is logically unbounded, so insertion does not normally wait for capacity. `SynchronousQueue` does not store elements at all: an insertion must pair with a removal.

## Choosing the Right Collection

| Requirement | Good starting choice |
| --- | --- |
| High-concurrency key-value lookup | `ConcurrentHashMap` |
| Sorted keys and range queries | `ConcurrentSkipListMap` |
| Concurrent set without sorting | `ConcurrentHashMap.newKeySet()` |
| Sorted concurrent set | `ConcurrentSkipListSet` |
| Frequent reads/iterations, rare list updates | `CopyOnWriteArrayList` |
| Non-blocking FIFO queue | `ConcurrentLinkedQueue` |
| Bounded producer-consumer pipeline | `ArrayBlockingQueue` |
| Blocking FIFO queue with optional bound | `LinkedBlockingQueue` |
| Direct producer-consumer handoff, no buffering | `SynchronousQueue` |
| Priority-based removal | `PriorityBlockingQueue` |
| Delayed availability | `DelayQueue` |

These are starting points, not universal performance rankings. Consider throughput, contention, memory use, ordering, backpressure, and operational semantics; benchmark where performance matters.

## Best Practices

- Choose based on **access pattern, required ordering, boundedness, iteration semantics, and blocking behavior**.
- Use atomic APIs such as `putIfAbsent()`, `computeIfAbsent()`, `compute()`, or `merge()` when a compound action must be atomic for a map key.
- Keep remapping functions short, predictable, and free of unnecessary blocking; understand the mapping-function restrictions in the API.
- Use bounded queues when producers can outpace consumers and backpressure is required.
- Prefer immutable elements; otherwise synchronize their mutable state separately.
- Prefer ordinary collections when state is confined to one thread or already protected by a consistent synchronization protocol.
- Treat bulk operations and iteration as implementation-specific: they may not offer an atomic snapshot of the collection.

## Common Pitfalls

- **Assuming thread-safe container = thread-safe algorithm.** A check-then-act sequence can still race.
- **Assuming stored values are protected.** Mutating an `ArrayList` retrieved from a `ConcurrentHashMap` remains unsafe without separate coordination.
- **Using `size()` or `isEmpty()` as a guarantee about a later operation.** The state may change immediately afterward.
- **Treating `ConcurrentModificationException` as a concurrency-control mechanism.** Fail-fast detection is best-effort.
- **Expecting every iterator to be a snapshot.** Many concurrent iterators are weakly consistent; copy-on-write iterators are snapshots.
- **Assuming every `BlockingQueue` is bounded.** `PriorityBlockingQueue` and `DelayQueue` are logically unbounded; `LinkedBlockingQueue` defaults to a very large bound.
- **Using copy-on-write for frequent writes.** Each mutation copies the backing array.
- **Using an unbounded queue without considering producer/consumer imbalance.** Memory use can grow when producers consistently outpace consumers.

## Interview Questions

**Q: Why use concurrent collections instead of synchronizing all collection access manually?**

A: They encapsulate thread-safety guarantees and offer specialized concurrency and coordination strategies that can reduce contention or provide better semantics for a workload.

**Q: What is the difference between `ConcurrentHashMap` and `Collections.synchronizedMap()`?**

A: `synchronizedMap()` synchronizes individual operations on a shared monitor. `ConcurrentHashMap` supports concurrent retrievals and scalable updates, with documented atomic per-key operations and weakly consistent traversal.

**Q: Are all operations on `ConcurrentHashMap` atomic?**

A: Each operation has its own API guarantee. Operations such as `compute()` are atomic for the target key, but combining separate calls such as `get()` and `put()` is not automatically atomic.

**Q: What does happens-before mean for inserting an element into a concurrent collection?**

A: Actions before insertion become visible to another thread's actions following access to or removal of that element. Later unsynchronized mutations of the element are not automatically safe.

**Q: What is a weakly consistent iterator?**

A: An iterator that can tolerate concurrent modifications without throwing `ConcurrentModificationException`; it does not necessarily represent a fixed snapshot and may reflect some concurrent changes.

**Q: When would you use `CopyOnWriteArrayList`?**

A: When reads and traversals greatly outnumber writes, such as maintaining listeners or rarely changed configuration collections.

**Q: Does `ConcurrentHashMap` make its values thread-safe?**

A: No. The map protects its own concurrent operations, not arbitrary mutable state inside stored objects.

**Q: What is the difference between `ArrayBlockingQueue` and `LinkedBlockingQueue`?**

A: `ArrayBlockingQueue` uses a fixed-size array and is always bounded. `LinkedBlockingQueue` uses linked nodes and may have an explicit bound; otherwise its capacity defaults to `Integer.MAX_VALUE`.

**Q: What does `SynchronousQueue` store?**

A: Nothing. It has zero internal capacity and pairs each insertion with a corresponding removal.

**Q: Does implementing `BlockingQueue` mean the queue has a finite capacity?**

A: No. Some implementations are logically unbounded, and their `put()` operations do not ordinarily block for capacity.

**Q: What is the difference between `ConcurrentLinkedQueue` and `BlockingQueue`?**

A: `ConcurrentLinkedQueue` is a concrete non-blocking FIFO queue. `BlockingQueue` is an interface providing waiting and timed operations; its implementations have different capacity and ordering semantics.

**Q: When should a concurrent collection *not* be used?**

A: When a collection is thread-confined, immutable, or consistently protected by another synchronization mechanism and the concurrent implementation adds unnecessary overhead.

## Related Topics

- [ConcurrentHashMap](2_ConcurrentHashMap.md)
- [CopyOnWriteArrayList](CopyOnWriteArrayList.md)
- [ArrayBlockingQueue](ArrayBlockingQueue.md)
- [SynchronousQueue](SynchronousQueue.md)

## References

- [Java 25 — `java.util.concurrent` package overview](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html)
- [Java 25 — ConcurrentHashMap](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
- [Java 25 — CopyOnWriteArrayList](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/CopyOnWriteArrayList.html)
- [Java 25 — Collections synchronized wrappers](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/Collections.html#synchronizedCollection(java.util.Collection))
