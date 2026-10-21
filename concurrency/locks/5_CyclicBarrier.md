# CyclicBarrier

## Overview

`java.util.concurrent.CyclicBarrier` coordinates a fixed number of participants at repeated synchronization points. Each participant calls `await()` after its phase work; once all arrive, the barrier trips and they can continue into the next phase.

Successful cycles automatically start a fresh generation. This is useful for iterative calculations, simulations, and parallel processing with a common phase boundary. The barrier coordinates progress; it does not implement `Lock` or provide mutual exclusion within a phase.

## CyclicBarrier vs. CountDownLatch

| Aspect | `CyclicBarrier` | [CountDownLatch](4_CountDownLatch.md) |
| --- | --- | --- |
| Arrival | `await()` both arrives and waits | `countDown()` signals without waiting |
| Participants | Fixed number of arrivals per generation | Fixed number of signals, independent of waiter count |
| Reuse | Automatic after each successful cycle | One-time gate; stays open after zero |
| Completion action | Optional action before releasing participants | No built-in action |
| Timeout / interruption | Can break the current generation for peers | Does not invalidate the latch for other waiters |

Use a latch when a coordinator waits for independent work; use a barrier when participants must repeatedly wait for each other. For phases with changing participation, consider `Phaser`. See the [concurrency package overview](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html).

## Core API

| Operation | Purpose |
| --- | --- |
| `new CyclicBarrier(parties)` | Set the required arrivals; `parties` must be at least `1` |
| `new CyclicBarrier(parties, action)` | Also run an action when each generation completes |
| `await()` | Arrive and wait; return an arrival index |
| `await(timeout, unit)` | Arrive with limited waiting; timeout throws `TimeoutException` |
| `getParties()` | Return the fixed required arrival count |
| `getNumberWaiting()` | Observe how many participants are currently waiting |
| `isBroken()` | Observe whether the current generation is broken |
| `reset()` | Break waiting participants out and start a fresh generation |

The last arrival receives index `0`, the first receives `parties - 1`. Indices describe arrival order, not stable worker identities. A barrier counts arrivals and does not register or verify thread identities. See the [CyclicBarrier API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/CyclicBarrier.html).

## Example: Two Calculation Phases

Each worker owns one array slot. The barrier action aggregates that phase before workers can overwrite their slots for the next phase:

```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

final class BarrierExample {
    public static void main(String[] args) throws Exception {
        int[] partial = new int[3];
        int[] completedPhases = {0};
        CyclicBarrier barrier = new CyclicBarrier(3, () -> {
            int total = Arrays.stream(partial).sum();
            System.out.println("Phase " + ++completedPhases[0] + ": " + total);
        });

        var pool = Executors.newFixedThreadPool(3);
        try {
            List<Future<?>> workers = new ArrayList<>();
            for (int i = 0; i < partial.length; i++) {
                final int slot = i;
                workers.add(pool.submit(() -> {
                    for (int phase = 1; phase <= 2; phase++) {
                        partial[slot] = (slot + 1) * phase;
                        barrier.await(5, TimeUnit.SECONDS);
                    }
                    return null;
                }));
            }
            for (Future<?> worker : workers) {
                worker.get(); // Propagate worker and barrier failures.
            }
        } finally {
            pool.shutdownNow();
        }
    }
}
```

Output is `Phase 1: 6`, then `Phase 2: 12`. The pool has enough workers for every participant to reach the barrier. Checked exceptions leave the callable and are reported through its `Future`.

## Barrier Action

The last arriving thread runs the optional action synchronously, before releasing its peers. Keep it short and avoid waiting for work that those peers must perform.

The action can aggregate results or prepare shared state for the next phase. It may run on a different participant each cycle. Using `if (barrier.await() == 0)` instead elects a participant **after release**; other participants can already be proceeding, so this does not replace a pre-release action.

## Java Memory Model Guarantees

Phase writes before `await()` **happen-before** the barrier action, whose actions happen-before code following successful returns from the corresponding `await()` in other participants. See the [barrier memory consistency guarantees](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html#memory-consistency-properties-heading).

```text
Workers: write phase data -> await()
                                |
                           happens-before
                                v
                         barrier action
                                |
                           happens-before
                                v
Workers:                  return -> next phase
```

The example needs no `volatile` array elements: writers own separate slots, and the action reads after all arrivals. The barrier does not protect conflicting writes within a phase or publish arbitrary writes performed after arrival.

After release, a fast worker can start modifying next-phase data before a slow peer finishes reading previous-phase data. Aggregate in the action, use separate buffers, or add another phase boundary when readers must finish before reuse. Failed waits do not authorize treating incomplete phase data as a completed result.

## Failure, Interruption, and Reset

The barrier uses a shared failure model:

- Interruption while waiting breaks the generation. The interrupted participant receives `InterruptedException`; its interrupt status is cleared.
- A participant timing out receives `TimeoutException`; the generation is broken.
- Peers waiting in that broken generation, and later arrivals before recovery, receive `BrokenBarrierException` unless independently interrupted.
- If the action throws, the executing participant receives that failure; peers are released exceptionally.
- `reset()` releases current waiters exceptionally and initializes a fresh generation. Successful cycles need no explicit reset.

See [BrokenBarrierException](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/BrokenBarrierException.html) and the [await/reset contracts](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/CyclicBarrier.html#reset()).

Failure **before reaching `await()`** does not automatically notify the barrier. Remaining participants need a timeout or an external cancellation policy. A broken barrier does not itself stop worker computations or roll back their writes. Coordinate recovery before restarting; replacing the barrier is often simpler than racing to reset it while old participants are still active.

The timeout limits waiting for the phase rendezvous; it does not time-limit the barrier action or guarantee a method-return deadline.

## Best Practices

- Make every participant follow the same phase protocol and reach the required boundaries.
- Ensure the executor can run enough participants concurrently; queued participants cannot arrive while every worker is blocked at the barrier.
- Use timed waits and propagate failures when a participant may disappear.
- Keep action work bounded and independent of blocked participants.
- Preserve interruption when handling it locally, and define a group cancellation policy.
- Use monitoring methods for diagnostics, rather than as permission to access phase data.

## Common Pitfalls

- **Treating await as a nonblocking signal.** It arrives and waits; use a latch when signalling must proceed independently.
- **Continuing after BrokenBarrierException as if synchronization succeeded.** The phase did not complete normally.
- **Resetting after every successful phase.** Reuse is automatic; reset can break peers already waiting in the next generation.
- **Assuming a barrier prevents all races.** It orders phases, not arbitrary accesses within them.
- **Waiting while holding a lock needed by another participant.** Barrier waiting does not release that lock.

## Interview Questions

**Q: Why is the barrier cyclic?**

A: Each successful rendezvous automatically prepares the barrier for another generation with the same required arrival count.

**Q: Who runs the barrier action?**

A: The last arriving participant, before the waiting participants are released.

**Q: What happens when one participant times out?**

A: It receives `TimeoutException`, and the generation breaks; its waiting peers normally receive `BrokenBarrierException`.

**Q: Does a worker failure before await automatically break the barrier?**

A: No. Peers can remain blocked unless a timeout or coordinated cancellation handles that failure.

**Q: When are phase results visible to other participants?**

A: Writes before arrival are published through the successful phase synchronization; the action can aggregate them before participants continue.
