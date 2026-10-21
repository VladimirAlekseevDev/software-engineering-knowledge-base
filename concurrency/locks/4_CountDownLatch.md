# CountDownLatch

## Overview

`java.util.concurrent.CountDownLatch` lets threads wait for a fixed number of signals. Each `countDown()` reduces the count; reaching zero opens the gate for every waiter. The gate stays open permanently.

It coordinates progress, rather than exclusive ownership of a resource: **CountDownLatch does not implement Lock**. See the [Lock overview](1_Overview.md) for that distinction.

## When to Use It

- Wait for independent initialization steps before serving requests.
- Gather results from a fixed batch of tasks.
- Release workers from a common start gate initialized to `1`.

The count represents signals, not distinct threads. One thread can provide multiple signals; signalling threads need not wait themselves. A start gate releases workers to proceed, but does not guarantee simultaneous execution.

## Core API

| Operation | Behavior |
| --- | --- |
| `new CountDownLatch(n)` | Set the initial count; zero starts open, negative values throw `IllegalArgumentException` |
| `countDown()` | Decrease by one; does not wait for other signals; no effect after zero |
| `await()` | Wait for zero; interruption throws `InterruptedException` and clears interrupt status |
| `await(timeout, unit)` | Return `true` after opening or `false` on timeout; also interruptible |
| `getCount()` | Inspect the current count; use `await()` for coordination |

Unlike `Condition.await()`, latch waiting needs no surrounding predicate loop. It does not release locks the waiting thread already holds. See the [CountDownLatch API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/CountDownLatch.html).

## Example: Collect Independent Results

Each worker owns a different array element. The coordinator reads only after the latch opens:

```java
import java.util.Arrays;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

final class LatchExample {
    static int[] collect() throws InterruptedException {
        int[] results = new int[3];
        CountDownLatch done = new CountDownLatch(results.length);

        for (int i = 0; i < results.length; i++) {
            final int slot = i;
            new Thread(() -> {
                try {
                    results[slot] = (slot + 1) * (slot + 1);
                } finally {
                    done.countDown();
                }
            }).start();
        }

        if (!done.await(5, TimeUnit.SECONDS)) {
            throw new IllegalStateException("Workers did not finish in time");
        }
        return results;
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println(Arrays.toString(collect())); // [1, 4, 9]
    }
}
```

The `finally` block records task termination even if work throws. For fallible work, also record failures or use `Future` results: opening the latch alone does not mean the tasks succeeded. Signalling completion does not guarantee a worker thread has terminated; it may still execute code after `countDown()`.

## Java Memory Model Guarantees

Actions before the count-reducing `countDown()` calls **happen-before** actions after another thread successfully passes the corresponding `await()`. This publishes completed worker writes to the coordinator. See [synchronizer memory consistency](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html#memory-consistency-properties-heading).

```text
Workers:     write results -> countDown() ... count reaches zero
                                                |
                                           happens-before
                                                v
Coordinator:                              await() -> read results
```

In the example, ordinary array elements are sufficient because workers write separate slots and the coordinator waits for publication. Ordering forbids observations inconsistent with this handoff; it does not impose an execution order between workers.

- A timed wait returning `false`, or an interrupted wait, does not provide the successful-wait publication guarantee. Do not then treat all results as complete.
- Writes after signalling need separate synchronization. Further calls after the count is already zero do not establish a fresh publication protocol.
- A latch does not make concurrent `ArrayList.add()` calls or increments of one shared variable safe. Worker-to-worker conflicts still require coordination.

## Reuse and Alternatives

There is no reset or increment operation. Use a new latch for a new batch. For repeated rendezvous, consider [CyclicBarrier](5_CyclicBarrier.md); for repeated phases with changing participation, consider `Phaser`. Use `Future` or `CompletableFuture` when the main requirement is obtaining task results and failures. See the [Java concurrency package overview](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html).

## Best Practices

- Define exactly what each signal means: completion, readiness, or another event.
- When counting task completion, signal once in `finally` for each task that actually runs, and handle work rejected or cancelled before execution separately.
- Publish results before signalling; keep failure reporting separate from completion counting.
- Check timed-wait results and choose a timeout or cancellation policy. Timing out does not cancel workers or reset the latch.
- Propagate interruption when possible; otherwise restore interrupt status and follow the operation's cancellation policy.

## Common Pitfalls

- **Missing or duplicate signals.** Too few can leave waiters blocked forever; too many can open the gate before all required work finishes. The latch does not track task identities.
- **Blocking a limited executor on its own queued work.** If waiting tasks occupy every worker, tasks that must signal cannot run.
- **Waiting while holding a lock needed by workers.** `await()` keeps that lock held, potentially preventing completion.
- **Reading partial results after timeout.** The unsuccessful wait is not permission to access unprotected worker state.
- **Reusing an open latch.** Future waiters pass without waiting for a new batch.

## Interview Questions

**Q: How does CountDownLatch differ from a Lock?**

A: A lock controls access to protected state. A latch waits for a count to reach zero and then allows all waiters through permanently.

**Q: Must each worker call countDown exactly once?**

A: That is a common application protocol, not a restriction enforced by the latch. It counts calls, not workers.

**Q: Are worker writes visible after await?**

A: Writes before the relevant countdown signals are published after successful waiting. Later writes and failed waits require separate consideration.

**Q: Does zero mean every task succeeded?**

A: No. It means the expected number of signals arrived; success must be recorded separately.

**Q: Can it be reset?**

A: No. Create another latch, or use a synchronizer designed for repeated phases.
