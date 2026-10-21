# ReentrantReadWriteLock

## Overview

`java.util.concurrent.locks.ReentrantReadWriteLock` implements `ReadWriteLock`. It coordinates two associated locks over the **same shared state**:

- `readLock()` returns a shared `Lock` for read-only operations.
- `writeLock()` returns an exclusive `Lock` for mutations.

The class itself does not implement `Lock`; its nested `ReadLock` and `WriteLock` classes do. See the [Lock overview](1_Overview.md) for acquisition methods and general memory guarantees.

## Why It Is Useful

An exclusive mutex serializes readers even when they only observe data. A read/write lock can allow expensive reads to overlap while keeping updates exclusive. Examples include searching a shared index, inspecting a registry, and reading a consistent snapshot spanning several entries.

It is most promising when reads substantially outnumber writes and protected read work is large enough to offset coordination costs. Tiny lookups, frequent writes, or low contention may perform better with a simple mutex. A concurrent collection may already provide the required operations without a separate lock. Measure the actual workload; see the [ReadWriteLock performance considerations](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReadWriteLock.html).

## Access Rules

| Requested access | Other readers hold the lock | Another thread holds the write lock |
| --- | --- | --- |
| Read | Compatible; admission also depends on queueing policy | Must wait or fail the attempt |
| Write | Must wait or fail the attempt | Must wait or fail the attempt |

One writer can hold the write lock, or multiple readers can hold read locks. The writing thread may also acquire the read lock itself. A queued writer can delay new readers even when no writer currently holds the lock; availability is not determined solely by the compatibility table.

Readers do not exclude other readers. **Writing under a read lock is unsafe**, including hidden mutations such as lazy initialization or an access-order map updating its order during `get()`.

## Basic Usage

All methods below coordinate through one lock pair. Values are immutable strings, and `snapshot()` returns an immutable copy rather than exposing the protected map:

```java
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

final class Catalog {
    private final ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
    private final Lock read = rw.readLock();
    private final Lock write = rw.writeLock();
    private final Map<String, String> entries = new HashMap<>();

    String get(String key) {
        read.lock();
        try {
            return entries.get(key);
        } finally {
            read.unlock();
        }
    }

    void put(String key, String value) {
        Objects.requireNonNull(key);
        Objects.requireNonNull(value);
        write.lock();
        try {
            entries.put(key, value);
        } finally {
            write.unlock();
        }
    }

    Map<String, String> snapshot() {
        read.lock();
        try {
            return Map.copyOf(entries);
        } finally {
            read.unlock();
        }
    }
}
```

The reader lock prevents writers from changing the map throughout the copy. Returning the original map, a live view, or an iterator to use after unlocking would escape that protection.

## Java Memory Model Guarantees

The read and write locks are associated synchronization modes, rather than unrelated locks:

```text
Writer: update fields -> write.unlock()
                               |
                          happens-before
                               v
Reader:                    read.lock() -> observe fields
```

A reader successfully acquiring after a write release sees the writes published by it. Write-to-write handoff also provides the ordinary exclusive-lock guarantees. Ordinary fields protected by this protocol do not additionally need `volatile`.

The guarantee covers memory writes before release, including state in referenced objects; it does not make mutations after release safe. Concurrent readers cannot publish arbitrary mutations to each other merely by holding read locks. See the [ReadWriteLock memory contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReadWriteLock.html) and [JLS happens-before rules](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html#jls-17.4.5).

## Reentrancy and Ownership

- A reader may reacquire the read lock; a writer may reacquire the write lock.
- A writer may acquire the read lock while retaining write ownership.
- Every successful acquisition adds a hold that must be released. Another writer cannot enter until all relevant holds are released.
- Holds belong to the acquiring thread. Releasing a read or write hold not owned by the current thread throws `IllegalMonitorStateException`.

The [ReadLock](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantReadWriteLock.ReadLock.html) and [WriteLock](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantReadWriteLock.WriteLock.html) APIs define these operations.

## Upgrading and Downgrading

### Read to Write: Release, Acquire, Recheck

A thread holding only a read lock cannot acquire the write lock without releasing **all its read holds**. Waiting in `write.lock()` while retaining a read hold can block indefinitely because the read hold itself prevents exclusive acquisition.

Release the read lock, acquire the write lock, then **recheck the predicate**. Other threads may update the state during the gap. For example, add this method to `Catalog`:

```java
String getOrInsert(String key, String candidate) {
    Objects.requireNonNull(key);
    Objects.requireNonNull(candidate);
    read.lock();
    try {
        String current = entries.get(key);
        if (current != null) {
            return current;
        }
    } finally {
        read.unlock();
    }

    write.lock();
    try {
        // Recheck: another thread may have inserted after our read.
        return entries.computeIfAbsent(key, ignored -> candidate);
    } finally {
        write.unlock();
    }
}
```

This method assumes its caller does not already hold a read lock. Releasing the single hold acquired here cannot remove any outer read holds. For a simple insertion, taking the write lock directly is often clearer; this example demonstrates the required recheck.

### Write to Read: Downgrade Without a Gap

Acquire the read lock **before** releasing the write lock. This keeps another writer from intervening while allowing readers to join after write release. Add this method to `Catalog`:

```java
Map<String, String> putAndSnapshot(String key, String value) {
    Objects.requireNonNull(key);
    Objects.requireNonNull(value);
    write.lock();
    try {
        entries.put(key, value);
        read.lock();
    } finally {
        write.unlock();
    }

    try {
        // Still protected from writers, but other readers may proceed.
        return Map.copyOf(entries);
    } finally {
        read.unlock();
    }
}
```

This assumes no outer write hold; otherwise releasing one hold does not fully downgrade. The order matters: releasing write before acquiring read leaves a gap for another writer. See the [class contract on downgrading](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantReadWriteLock.html).

## Fairness and Waiting

```java
ReentrantReadWriteLock defaultPolicy = new ReentrantReadWriteLock();
ReentrantReadWriteLock fairPolicy = new ReentrantReadWriteLock(true);
```

Default nonfair mode does not promise acquisition order; sustained contention can indefinitely postpone readers or writers. Fair mode uses approximate arrival ordering, allowing eligible reader groups to acquire together. Fairness can reduce throughput and does not guarantee thread scheduling or a maximum waiting time.

Both modes support interruptible acquisition and timed attempts. **Untimed `tryLock()` ignores fairness**, whereas timed `tryLock(timeout, unit)` honors it for non-reentrant acquisition. `tryLock(0, TimeUnit.SECONDS)` requests a no-wait, fairness-respecting attempt and also detects interruption. See the [read](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantReadWriteLock.ReadLock.html#tryLock(long,java.util.concurrent.TimeUnit)) and [write](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantReadWriteLock.WriteLock.html#tryLock(long,java.util.concurrent.TimeUnit)) acquisition contracts.

## Conditions

Only the write lock supports conditions; `read.newCondition()` throws `UnsupportedOperationException`. A writer-side condition can wait for a predicate that needs exclusive inspection or modification.

For example, extend `Catalog` with this field and method, and call `nonEmpty.signalAll()` immediately after `entries.put()` in `put()`, while still holding `write`:

```java
private final java.util.concurrent.locks.Condition nonEmpty = write.newCondition();

Map<String, String> awaitNonEmpty() throws InterruptedException {
    write.lockInterruptibly();
    try {
        while (entries.isEmpty()) {
            nonEmpty.await();
        }
        return Map.copyOf(entries);
    } finally {
        write.unlock();
    }
}
```

For this extension, other methods that can make the map nonempty must signal too. `await()` releases the write holds and restores them when reacquiring before returning. **Do not retain a read hold while awaiting a write condition**: it can prevent another writer from making the predicate true. The loop handles spurious wakeups and state changes before reacquisition. See [write-lock conditions](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantReadWriteLock.WriteLock.html#newCondition()) and [Condition](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/Condition.html).

## Best Practices

- Use one lock pair for the entire protected invariant; name read-only and mutating operations clearly.
- Use the write lock for every mutation, even if the method is called a getter.
- Return immutable values or detached snapshots when callers outlive the critical section.
- Keep writes short; long reads also delay writers.
- Use interruption or timeouts where waiting must be cancellable or limited.
- Compare measured performance with a mutex and suitable concurrent collections before choosing this design.

## Common Pitfalls

- **Trying to upgrade while still holding a read lock.** Release every read hold first, then acquire write and recheck.
- **Treating the two lock objects as independent.** They coordinate through one `ReentrantReadWriteLock`; two separately created instances do not protect each other.
- **Calling read-protected code that secretly mutates state.** Multiple readers may run it concurrently.
- **Exposing mutable values or live views.** The lock does not accompany an object after it is returned.
- **Using monitoring methods to decide correctness.** `getReadLockCount()`, `isWriteLocked()`, and queue estimates are observations, not reservations; use acquisition operations.
- **Assuming nonfair means permanent reader preference.** It promises neither reader-first nor writer-first ordering.

## Interview Questions

**Q: Does ReentrantReadWriteLock implement Lock?**

A: It implements `ReadWriteLock`. The associated `ReadLock` and `WriteLock` implement `Lock`.

**Q: Why can readers run concurrently?**

A: Read-only operations do not conflict with each other. The shared read mode excludes other threads' writers, not other readers.

**Q: Does write unlock publish updates to readers?**

A: Yes. The associated lock pair guarantees visibility after a subsequent successful read acquisition.

**Q: Can a reader upgrade to write?**

A: Not while retaining its read holds. It must release them, acquire write, and recheck state that could have changed during the gap.

**Q: How is a write lock downgraded?**

A: Acquire read while owning write, then fully release write. Retain the read hold for the remaining read-only work.

**Q: Does fair mode make every acquisition FIFO?**

A: No. It uses approximate arrival ordering with reader grouping and reentrancy; untimed `tryLock()` can bypass waiters.

**Q: When is this class a poor fit?**

A: When coordination overhead outweighs overlapping read work, writes dominate, or a simpler concurrency primitive already provides the required guarantees.
