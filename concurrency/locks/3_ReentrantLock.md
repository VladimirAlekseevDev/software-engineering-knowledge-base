# ReentrantLock

## Overview

`java.util.concurrent.locks.ReentrantLock` is an exclusive, reentrant implementation of `Lock`. Use it to protect shared-state invariants when timed or interruptible acquisition, condition variables, or configurable fairness are needed.

Only the owning thread can release it. Reacquisition by that thread increments its hold count; the lock becomes available to another thread only after every hold is released. An unmatched release throws `IllegalMonitorStateException`.

See the [Lock overview](1_Overview.md#basic-usage) for a complete usage example, API summary, and Java Memory Model guarantees.

## Fairness and Conditions

- The default constructor uses a nonfair policy. `new ReentrantLock(true)` favors the longest-waiting thread under contention, potentially reducing throughput.
- Untimed `tryLock()` can bypass queued waiters even in fair mode. The timed form respects fairness for non-reentrant acquisition.
- A fair lock does not make thread scheduling fair or establish a maximum waiting time.
- `newCondition()` supports separate waiting predicates. Hold the lock when awaiting or signalling; check predicates in a loop.

See the [ReentrantLock API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html).

## Best Practices

- Acquire immediately before `try` and release immediately inside `finally`.
- Guard related reads and writes with the same stable lock instance.
- Use interruptible or timed acquisition when the operation requires cancellation or bounded waiting.

## Common Pitfalls

- Forgetting a release after reentrant acquisition, leaving an outstanding hold.
- Treating `isLocked()` or a queue-length estimate as permission to access protected state.
- Assuming an explicit lock always outperforms `synchronized`.

## Interview Questions

**Q: What does reentrant mean?**

A: The owning thread may acquire the same lock again without blocking itself, but must release each acquisition.

**Q: Can another thread unlock it?**

A: No. Release by a non-owner throws `IllegalMonitorStateException`.
