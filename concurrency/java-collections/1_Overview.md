# Concurrent Collections in Java

## Overview

Concurrent collections are data structures designed for safe access by multiple threads. Java provides them in `java.util.concurrent` to handle shared data and coordination between threads.

## Why They Exist

Ordinary mutable collections do not generally synchronize access. When threads share one and at least one modifies it, missing synchronization can cause lost updates, stale reads, or inconsistent internal state.

External locking can make access safe, but every caller must follow the same locking rules. A single lock can also become a bottleneck as contention grows.

Concurrent collections encapsulate synchronization and offer different trade-offs for concurrent access, iteration, and producer-consumer coordination. They can improve scalability, but are not automatically faster for every workload. See the [Java concurrency package documentation](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html#concurrent-collections).

## Synchronized Wrappers vs. Concurrent Collections

`Collections.synchronized*` wrappers protect individual operations with a shared monitor. Traversal requires holding the wrapper's monitor, and access must go through the wrapper. Compound actions also require synchronization across the entire sequence.

Concurrent implementations use strategies such as finer-grained locking, atomic operations, or copying on writes. Their guarantees depend on the implementation; thread-safe does not necessarily mean lock-free. See [synchronized wrappers](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/Collections.html#synchronizedCollection(java.util.Collection)).

## General Guarantees and Limits

- **Atomicity:** guarantees apply to operations explicitly specified as atomic; a sequence of method calls is not automatically one atomic action. Checking a condition and then modifying the collection can still race.
- **Visibility:** actions before placing an object into a concurrent collection happen-before actions following another thread's access to or removal of that element. Later mutations of the object need their own synchronization.
- **Iteration:** iterators may be weakly consistent or snapshot-based. They need not reflect the latest state or provide a single consistent view of the entire collection.
- **Coordination:** blocking operations can wait for data or capacity. Availability, timeouts, and interruption behavior depend on the API.

See [memory consistency guarantees](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html#memory-consistency-properties-heading) and the [concurrent map contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentMap.html).

## Common Collections

- [ConcurrentHashMap](ConcurrentHashMap.md)
- [ConcurrentSkipListMap](ConcurrentSkipListMap.md)
- [ConcurrentSkipListSet](ConcurrentSkipListSet.md)
- [CopyOnWriteArrayList](CopyOnWriteArrayList.md)
- [CopyOnWriteArraySet](CopyOnWriteArraySet.md)
- [ConcurrentLinkedQueue](ConcurrentLinkedQueue.md)
- [ConcurrentLinkedDeque](ConcurrentLinkedDeque.md)
- [ArrayBlockingQueue](ArrayBlockingQueue.md)
- [LinkedBlockingQueue](LinkedBlockingQueue.md)
- [LinkedBlockingDeque](LinkedBlockingDeque.md)
- [SynchronousQueue](SynchronousQueue.md)
- [PriorityBlockingQueue](PriorityBlockingQueue.md)

## Best Practices

- Choose by access patterns, ordering, capacity, and blocking requirements.
- Use documented atomic operations for compound updates when available.
- Prefer immutable elements, or synchronize their mutable state separately.
- Use ordinary collections when data is confined to one thread or already protected by a consistent locking protocol.

## Common Pitfalls

- Assuming a thread-safe container makes an entire algorithm or its elements thread-safe.
- Using `size()` or `isEmpty()` to decide whether a later operation will succeed; another thread can change the state between calls.
- Treating `ConcurrentModificationException` as a safety mechanism. Fail-fast detection is best-effort, as explained in the [iterator contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/ArrayList.html).

## Interview Questions

**Q: Why use concurrent collections instead of synchronizing everything?**

A: They encapsulate synchronization and offer access and coordination strategies suited to different workloads, potentially reducing contention.

**Q: Does a thread-safe collection make a check-then-act sequence atomic?**

A: No. Use an operation with the required atomicity guarantee or a coordination protocol followed by every participant.
