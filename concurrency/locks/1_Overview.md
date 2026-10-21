# Locks in Java

## Overview

`java.util.concurrent.locks.Lock` is an interface for explicit control of access to shared state. Unlike `synchronized`, acquisition and release are method calls. The interface supports immediate, timed, and interruptible acquisition, and associated condition variables.

A lock commonly provides **mutual exclusion**, but the interface also permits shared access, such as a read lock. Reentrancy, ownership rules, fairness, and condition support depend on the implementation.

## Why Locks Exist

Concurrent code needs to preserve invariants across multiple actions, publish updates to other threads, and sometimes wait for a resource or state change. An exclusive lock can protect an entire read-modify-write sequence or an invariant spanning several fields.

`synchronized` already provides exclusion and memory synchronization. Explicit locks add control over acquisition and waiting when a lexical monitor block is insufficient. Neither mechanism automatically protects code that ignores its synchronization protocol.

## Lock vs. synchronized

| Aspect | `synchronized` | `Lock` |
| --- | --- | --- |
| Acquisition / release | Automatic on block or method entry / exit | Explicit calls; release in `finally` |
| Immediate acquisition attempt | No | `tryLock()` |
| Timed acquisition | No | `tryLock(timeout, unit)` |
| Interruptible acquisition | Monitor entry is not interruptible | `lockInterruptibly()`, where supported |
| Waiting for a state change | `wait()`, `notify()`, `notifyAll()` on the monitor | Associated `Condition` objects, where supported |
| Locking policies | Reentrant, exclusive monitor | Implementation-specific, including shared modes |
| Memory synchronization | Monitor happens-before guarantees | Equivalent acquisition / release effects |

Prefer `synchronized` when its simpler structure meets the requirements. Choose an explicit lock for capabilities the algorithm needs, rather than assuming it is always faster. See the [locks package overview](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/package-summary.html).

## Core API

| Method | Purpose |
| --- | --- |
| `lock()` | Acquire, waiting if necessary; no interruptible cancellation contract |
| `lockInterruptibly()` | Acquire or throw `InterruptedException`; check implementation support for interruption while waiting |
| `tryLock()` | Attempt immediately; `true` means the lock was acquired |
| `tryLock(long, TimeUnit)` | Attempt with a waiting limit; return `false` on timeout, or throw `InterruptedException` |
| `unlock()` | Release one acquisition, subject to the implementation's ownership rules |
| `newCondition()` | Create a condition bound to this lock; may throw `UnsupportedOperationException` |

Successful `tryLock()` acquires the lock and therefore requires `unlock()`. Calling it merely to check availability without releasing it leaks an acquisition. The [Lock API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/Lock.html) defines the contract and implementation-specific allowances.

## Basic Usage

The lock must be shared by all callers protecting the same state. Acquire **before** entering `try`, then release in `finally`:

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

final class LockedCounter {
    private final Lock lock = new ReentrantLock();
    private int value;

    void increment() {
        lock.lock();
        try {
            value++;
        } finally {
            lock.unlock();
        }
    }

    int get() {
        lock.lock();
        try {
            return value;
        } finally {
            lock.unlock();
        }
    }
}
```

Both the update and the read use the same lock. Creating a new lock inside each method invocation would provide no coordination between callers.

For a timed attempt, enter the protected block only after success. This fragment belongs in a method declaring `throws InterruptedException`:

```java
if (lock.tryLock(100, java.util.concurrent.TimeUnit.MILLISECONDS)) {
    try {
        // Access protected state.
    } finally {
        lock.unlock();
    }
} else {
    // Handle the acquisition timeout.
}
```

The timeout limits acquisition waiting, not execution of the critical section. It is not a real-time deadline for the method's return. Propagate interruption when possible; if it must be caught without propagation, normally restore the interrupt status with `Thread.currentThread().interrupt()` and follow the cancellation policy.

## Java Memory Model Guarantees

### Happens-before, Visibility, and Ordering

A successful acquisition has monitor-acquire memory effects; release has monitor-release effects. For an exclusive lock, the publication chain is:

```text
Thread A: write shared state -> release L
                                  |
                             happens-before
                                  v
Thread B:                    acquire L -> read shared state
```

By program order and transitivity, **writes before release become visible to actions after a subsequent acquisition of the same lock**. This includes ordinary non-`volatile` fields and state inside referenced objects. Later writes after release need their own synchronization.

Ordering means the runtime must preserve the observations required by this relationship. It does not require every source instruction to execute in literal order or mandate a particular CPU fence or cache-flushing procedure. See [JLS 17.4.5](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html#jls-17.4.5).

In `LockedCounter`, a `get()` acquiring after an `increment()` releases sees that increment or a later protected update. A read that acquires first may observe the earlier value. Locking does not force one application operation to run before another.

### Atomicity and Its Boundaries

- An **exclusive** lock makes a critical section indivisible relative to other participants using that lock. It can preserve compound invariants; `volatile` alone does not make `value++` atomic.
- Locking only writers leaves unsynchronized readers outside the protocol. Safely publish the containing object as well; a lock field does not fix an object escaping during construction.
- Different, unrelated locks do not provide this handoff. Associated read/write locks are a documented exception: a write release publishes to subsequent read acquisitions.
- Failed acquisition, failed release, and reentrant acquisition/release have **no additional memory synchronization effects required by the interface**. Do not use a failed `tryLock()` as a visibility barrier. For an exclusive reentrant lock, handoff occurs when the final hold is released and another thread acquires it.

These boundaries follow the [Lock memory synchronization contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/Lock.html#memory-synchronization) and the [ReadWriteLock contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReadWriteLock.html).

## Conditions

A `Condition` lets a thread wait for a predicate over protected state. With the standard reentrant exclusive locks, hold the associated lock when calling `await()`, `signal()`, or `signalAll()`.

- `await()` releases the associated lock while waiting and reacquires it before returning.
- Check the predicate in a **`while` loop**: spurious wakeups are allowed, and another thread may change the state before the waiter reacquires.
- Change state before signalling. A signal is not a stored event or an immediate transfer of lock ownership; the signalling thread still needs to release the lock.
- Multiple conditions can separate different waiting predicates. Do not mix `Condition` operations with monitor `wait()` / `notify()`.

See the [Condition contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/Condition.html). Support and detailed semantics are implementation-specific.

## Main Lock Implementations

| Implementation | Access model | Typical purpose | Note |
| --- | --- | --- | --- |
| `ReentrantLock` | Exclusive, reentrant | General shared-state invariants; configurable fairness and conditions | [ReentrantLock](3_ReentrantLock.md) |
| `ReentrantReadWriteLock.ReadLock` | Shared, reentrant | Concurrent read-only operations | [ReentrantReadWriteLock](2_ReentrantReadWriteLock.md) |
| `ReentrantReadWriteLock.WriteLock` | Exclusive, reentrant | Updates coordinated with the associated readers | [ReentrantReadWriteLock](2_ReentrantReadWriteLock.md) |

**`ReentrantReadWriteLock` itself implements `ReadWriteLock`, not `Lock`.** Its `readLock()` and `writeLock()` return the two `Lock` implementations above.

`StampedLock` is another locking tool, but implements neither interface directly. It offers `Lock` adapter views through `asReadLock()` and `asWriteLock()`; its stamp-based API and non-reentrant behavior require separate consideration. See the [StampedLock API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/StampedLock.html).

## Best Practices

- Keep locks private and stable, usually in `final` fields; document which state each protects.
- Guard every access participating in an invariant with the same protocol.
- Release in `finally`, and only after a successful acquisition. Balance every reentrant acquisition with a release.
- Keep critical sections short; avoid slow I/O and unknown callbacks while holding locks.
- Use a consistent acquisition order when multiple locks are needed to prevent circular waits.
- Choose interruptible or timed acquisition when cancellation or limited waiting is required.

## Common Pitfalls

- **Putting acquisition inside `try` with unconditional `unlock()` in `finally`.** Failed or interrupted acquisition must not trigger release.
- **Mixing `synchronized(lock)` with `lock.lock()`.** The object's monitor and its explicit lock are independent.
- **Assuming all locks are exclusive, fair, or reentrant.** Those are implementation properties.
- **Expecting rollback after an exception.** `finally` releases the lock but does not undo a partially completed update.
- **Expecting liveness from memory safety.** Correct visibility does not prevent deadlock, starvation, or livelock.

## Interview Questions

**Q: What does Lock add beyond synchronized?**

A: Explicit acquisition strategies, including immediate, timed, and interruptible attempts, plus implementation-specific policies and conditions.

**Q: Does Lock provide visibility for non-volatile fields?**

A: Yes, through a release-to-subsequent-acquisition happens-before relationship, when accesses follow the same synchronization protocol.

**Q: Are memory ordering and fairness the same guarantee?**

A: No. Memory ordering constrains observable reads and writes. Fairness concerns which waiting thread gets access; it depends on the implementation and method.

**Q: Is every Lock a mutex?**

A: No. A read lock can be held by multiple reader threads simultaneously.

**Q: Does releasing a lock restore the state after an exception?**

A: No. Code must preserve or repair its invariants; unlocking only ends the protected access.
