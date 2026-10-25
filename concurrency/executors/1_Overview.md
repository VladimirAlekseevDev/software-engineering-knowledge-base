# Executors and ExecutorService in Java

## Overview

The executor framework separates **what a task does** from **how and where it runs**. Instead of creating and managing threads at every call site, submit tasks to an executor with an appropriate execution policy.

| Type | Responsibility |
| --- | --- |
| `Executor` | Accept `Runnable` tasks through `execute()` |
| `ExecutorService` | Add task results, cancellation, bulk execution, and lifecycle management |
| `ScheduledExecutorService` | Add delayed and periodic execution |
| `Executors` | Provide static factories and adapters; it is not an executor instance |

An executor need not run asynchronously: an implementation or rejection policy can execute work on the submitting thread. An `ExecutorService` is also **not necessarily a thread pool**; a thread-per-task executor creates a fresh thread for each task. See the [Executor contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/Executor.html).

## Tasks and Results

- `Runnable` represents work without a returned result; `run()` cannot declare checked exceptions.
- `Callable<T>` returns a value and its `call()` method may throw checked exceptions.
- `Future<T>` represents one submitted computation: wait for its result, inspect completion, or request cancellation.

Submitting a task does not mean it has started or completed. Tasks can queue, run concurrently, fail, or be rejected. Sharing a task object or captured mutable data between submissions still requires synchronization.

## Main Implementations

| Implementation / executor kind | Execution model | Typical use |
| --- | --- | --- |
| [ThreadPoolExecutor](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html) | Reuses workers; configurable sizes, queue, thread factory, and rejection policy | General task execution with explicit resource limits |
| [ScheduledThreadPoolExecutor](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ScheduledThreadPoolExecutor.html) | Extends `ThreadPoolExecutor` with delayed and periodic tasks | Timers, maintenance, polling |
| [ForkJoinPool](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ForkJoinPool.html) | Work-stealing across worker queues | Recursive CPU tasks and many small independent tasks |
| Thread-per-task executor, obtained from `Executors` | A new thread for each task, using the supplied thread factory | Independent tasks; virtual threads for blocking workloads |

`AbstractExecutorService` is a base class for implementations, rather than a ready-to-use executor. It provides submission and bulk-operation machinery, normally using `FutureTask`. Factory results may be wrappers: program against the returned interface instead of assuming a concrete class. See [AbstractExecutorService](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/AbstractExecutorService.html).

In Java 25, `ForkJoinPool` also implements `ScheduledExecutorService`; this scheduling support is absent in earlier releases. For its usual work-stealing role, unmanaged blocking I/O can reduce effective parallelism; use a suitable executor or the pool's managed-blocking facilities when necessary.

## Useful Executors Factories

| Factory | Behavior | Important trade-off |
| --- | --- | --- |
| `newFixedThreadPool(n)` | At most `n` active workers; excess work queues | Shared **unbounded queue**; backlog and memory use can grow |
| `newSingleThreadExecutor()` | One task executes at a time; failed workers can be replaced | Unbounded queue; one slow task delays later tasks |
| `newCachedThreadPool()` | Reuses idle workers, creates workers as needed, retires idle ones | No practical configured bound on worker count |
| `newScheduledThreadPool(n)` | Scheduled executor with `n` core workers | Delayed work uses an unbounded queue |
| `newSingleThreadScheduledExecutor()` | Scheduled execution with one worker | Long tasks delay other scheduled work |
| `newWorkStealingPool()` / `(parallelism)` | Work-stealing executor; default target uses available processors | No task execution-order guarantee; parallelism is a target |
| `newThreadPerTaskExecutor(factory)` | New thread per task, using that factory; Java 21+ | No built-in task/thread-count bound |
| `newVirtualThreadPerTaskExecutor()` | New virtual thread per task; Java 21+ | No built-in bound on concurrent tasks or downstream resources |

Many platform-thread factories have `ThreadFactory` overloads. A single-thread executor serializes tasks; it does not guarantee that one particular physical thread lives forever. See the [Executors API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/Executors.html).

## Submission and Coordination Methods

| Method | Behavior |
| --- | --- |
| `execute(Runnable)` | Submit without returning a result handle |
| `submit(Callable<T>)` | Return `Future<T>` for the computed result |
| `submit(Runnable)` | Return `Future<?>`; normal `get()` result is `null` |
| `submit(Runnable, result)` | Return a future containing the supplied result after successful execution |
| `invokeAll(tasks)` | Wait for all tasks to finish; return futures in input iteration order |
| `invokeAny(tasks)` | Return a result from a successfully completed task; cancel unfinished tasks |

Timed bulk overloads limit waiting. Timed `invokeAll` cancels tasks unfinished on return; timed `invokeAny` throws `TimeoutException` if no successful result is obtained in time. Cancellation is a request, not a guarantee that task code has stopped.

`invokeAll` completion includes failures: inspect each future to distinguish results, exceptions, and cancellation. `invokeAny` selects a successful result, not simply the first task to finish. See [bulk execution contracts](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/AbstractExecutorService.html).

## Basic Example: Submit Before Waiting

Examples assume imports from `java.util` and `java.util.concurrent`, and an enclosing method declaring the checked exceptions, such as `throws Exception`. Try-with-resources for `ExecutorService` requires Java 19+; virtual-thread examples require Java 21+.

```java
try (ExecutorService pool = Executors.newFixedThreadPool(2)) {
    Future<Integer> first = pool.submit(() -> 21 * 2);
    Future<Integer> second = pool.submit(() -> 10 + 5);

    System.out.println(first.get() + second.get()); // 57
}
```

Both tasks are submitted before the caller waits. Calling `submit(...).get()` on every loop iteration would serialize submission and waiting, losing the intended overlap.

### Bulk Results

```java
List<Callable<String>> tasks = List.of(() -> "east", () -> "west");
try (ExecutorService pool = Executors.newFixedThreadPool(2)) {
    List<Future<String>> results = pool.invokeAll(tasks);
    for (Future<String> result : results) {
        System.out.println(result.get()); // east, then west
    }
}
```

This prints in input order, regardless of completion order. When results should be consumed as they complete, wrap an executor in `ExecutorCompletionService` and retrieve completed futures through `take()` or `poll()`. The completion service uses the supplied executor; it is not another pool. See [ExecutorCompletionService](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ExecutorCompletionService.html).

## Failures, Timeouts, and Cancellation

- `Future.get()` waits; task failure is reported as `ExecutionException`, with the original exception in `getCause()`.
- `get(timeout, unit)` throws `TimeoutException` if waiting expires. **It does not automatically cancel the task.**
- Successful cancellation causes `get()` to throw `CancellationException`. `isDone()` includes normal completion, failure, and cancellation.
- `cancel(true)` may interrupt the executing thread; tasks must cooperate with interruption. `cancel(false)` does not interrupt a running task.
- Interruption of a caller waiting in `get()` does not itself cancel the submitted task. Propagate interruption or restore its status and follow the application's cancellation policy.

With a typical `ThreadPoolExecutor`, an unchecked exception escaping a plain task passed to `execute()` reaches worker-level uncaught-exception handling and can terminate that worker. `submit()` normally wraps the task so failure is captured in its future instead. Ignoring that future can hide the failure; `close()` does not rethrow task exceptions. See [Future](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/Future.html).

## Lifecycle and Ownership

| Method / state | Meaning |
| --- | --- |
| `shutdown()` | Reject new work; begin orderly termination; does not wait for termination |
| `shutdownNow()` | Attempt to interrupt active work and prevent queued work from starting; return never-started tasks |
| `awaitTermination(timeout, unit)` | Wait for termination after shutdown; does not initiate shutdown |
| `isShutdown()` | Shutdown has been initiated |
| `isTerminated()` | All execution has ended following shutdown |
| `close()` | Begin orderly shutdown and wait for termination; available since Java 19 |

Own the lifecycle at the component that creates the executor. A method borrowing a shared executor should not close it. Application-wide pools generally live until application shutdown; the examples use short-lived scopes for clarity.

For explicit bounded waiting: call `shutdown()`, await termination, then use `shutdownNow()` and await again if necessary. Handle interruption by requesting shutdown and propagating it or restoring interrupt status. Tasks ignoring interruption may prevent termination.

`close()` can block indefinitely on nonterminating tasks. If interrupted while waiting, its default implementation requests immediate shutdown, continues waiting for active tasks, and restores interrupt status before returning. For a scheduled executor, shutdown behavior for delayed and periodic tasks depends on its policies. See the [ExecutorService lifecycle contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ExecutorService.html).

## Bounded Queues and Rejection

For explicit overload limits, configure a `ThreadPoolExecutor` rather than relying on an unbounded factory queue:

```java
try (ThreadPoolExecutor pool = new ThreadPoolExecutor(
        2, 4, 30, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(100),
        Executors.defaultThreadFactory(),
        new ThreadPoolExecutor.AbortPolicy())) {
    Future<Integer> result = pool.submit(() -> 42);
    System.out.println(result.get());
}
```

The ordinary submission policy is: create workers up to the core size, then queue work; when the queue cannot accept it, create workers up to the maximum; then reject. Consequently, an unbounded queue usually makes `maximumPoolSize` ineffective beyond the core size.

| Rejection policy | Behavior |
| --- | --- |
| `AbortPolicy` | Throw `RejectedExecutionException`; the default |
| `CallerRunsPolicy` | Run on the submitting thread unless shut down; can slow the producer |
| `DiscardPolicy` | Silently discard the rejected task |
| `DiscardOldestPolicy` | Drop the oldest queued task and retry, unless shut down |

Silently discarding tasks submitted with futures can leave callers waiting on futures that never complete. Choose rejection behavior deliberately; a bounded queue alone does not define a complete backpressure policy. See [ThreadPoolExecutor queueing and rejection](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html).

## Scheduled Execution

- `schedule()` enables one execution after a relative delay.
- `scheduleAtFixedRate()` targets `initialDelay + n * period`; long runs can make subsequent runs late.
- `scheduleWithFixedDelay()` waits the delay **after one execution finishes** before enabling the next.

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
try {
    ScheduledFuture<?> heartbeat = scheduler.scheduleAtFixedRate(
            () -> System.out.println("tick"), 1, 1, TimeUnit.SECONDS);
    ScheduledFuture<String> ready = scheduler.schedule(
            () -> "ready", 10, TimeUnit.MILLISECONDS);

    System.out.println(ready.get());
    heartbeat.cancel(false); // Stop future repetitions.
} finally {
    scheduler.shutdownNow();
}
```

A delay is not an exact execution deadline. An exception from a periodic execution suppresses subsequent executions; inspect its future or arrange explicit failure reporting. A periodic future does not complete normally while repetitions continue. See [ScheduledExecutorService](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ScheduledExecutorService.html).

In `ScheduledThreadPoolExecutor`, executions of the same periodic task do not overlap, and one execution happens-before the next. Separate tasks can run concurrently. Its effective worker size is the core size; raising the maximum does not help its unbounded delay queue. `setRemoveOnCancelPolicy(true)` avoids retaining cancelled delayed tasks until their delays expire. See [ScheduledThreadPoolExecutor](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ScheduledThreadPoolExecutor.html).

## Virtual Threads and Workload Choice

```java
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<Boolean> result = executor.submit(() -> Thread.currentThread().isVirtual());
    System.out.println(result.get()); // true
}
```

Virtual threads make large numbers of mostly waiting tasks cheaper to represent. They do not increase CPU capacity or speed up a CPU-intensive task. Do not pool virtual threads to reuse them; create one per task, and separately bound scarce resources such as database connections with a connection pool or semaphore.

For CPU-bound work, start by considering a bounded platform-thread pool or `ForkJoinPool` with parallelism around available CPU capacity, then measure. For many independent blocking operations, a virtual-thread-per-task executor is often suitable. Neither choice removes the need for admission limits and downstream timeouts. See the [virtual threads guide](https://docs.oracle.com/en/java/javase/25/core/virtual-threads.html).

## Other Useful Executors Methods

| Utility | Purpose |
| --- | --- |
| `defaultThreadFactory()` | Create non-daemon platform threads using the default naming and priority rules |
| `callable(runnable)` | Adapt a `Runnable` to a `Callable` returning `null` |
| `callable(runnable, result)` | Adapt a `Runnable` to return the supplied value after success |
| `unconfigurableExecutorService(executor)` | Hide implementation-specific configuration methods behind a wrapper |
| `unconfigurableScheduledExecutorService(executor)` | Corresponding wrapper for a scheduled executor |

An unconfigurable wrapper still exposes lifecycle methods: it does not prevent the recipient from calling `shutdown()`.

Use a custom thread factory for meaningful names and uncaught-exception handling. For example, with Java 21+ thread builders:

```java
ThreadFactory named = Thread.ofPlatform().name("worker-", 0).factory();
try (ExecutorService pool = Executors.newFixedThreadPool(2, named)) {
    Callable<String> adapted = Executors.callable(
            () -> System.out.println("task"), "done");
    System.out.println(pool.submit(adapted).get()); // task, then done
}
```

## Java Memory Model Guarantees

Actions before submitting a task **happen-before** its execution. The task's actions happen-before the caller's actions after retrieving its result through the corresponding `Future.get()`.

```text
Caller: prepare input -> submit
                           |
                      happens-before
                           v
Worker:                run task -> produce result
                                       |
                                  happens-before
                                       v
Caller:                           get() returns -> use result
```

This publishes prepared inputs and completed results. It does not protect later mutations shared with a running task, make a shared collection thread-safe, or order independent tasks against each other. Use task results, immutable data, or a suitable synchronization protocol. See [Executor publication](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/Executor.html) and [Future result visibility](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/Future.html).

## Best Practices

- Choose the execution model according to CPU work, blocking, ordering, and resource limits.
- Define queue capacity, rejection, deadlines, and cancellation together for overload handling.
- Keep submission separate from waiting when overlap is intended; observe task failures.
- Give executors clear lifecycle owners and descriptive thread names.
- Keep tasks interruption-aware; release resources in `finally` or try-with-resources.
- Monitor active workers, queue depth, rejection counts, and task latency; tune using representative measurements.

## Common Pitfalls

- **Assuming fixed worker count bounds total work.** Queued tasks and captured data can still grow without a bound.
- **Using cached pools without considering thread growth.** Blocking tasks can consume many platform threads.
- **Waiting for child tasks in the same saturated pool.** If parents occupy every worker, queued children cannot run; barriers and latches have the same risk.
- **Ignoring futures returned by submit.** Task exceptions may go unobserved.
- **Treating timeout or cancel as forced termination.** The task may keep running.
- **Closing a borrowed executor or forgetting to shut down an owned one.** Both break lifecycle assumptions.
- **Leaving ThreadLocal state in reused workers.** Clean up task-specific values to avoid leaking context into later tasks.
- **Assuming submission order means completion order.** Multi-worker executors do not provide that guarantee.

## Interview Questions

**Q: What is the difference between Executor, ExecutorService, and Executors?**

A: `Executor` accepts tasks, `ExecutorService` adds results and lifecycle control, and `Executors` is a static factory/utility class.

**Q: Does ExecutorService always use a pool?**

A: No. A thread-per-task executor creates a fresh thread for each task, including the virtual-thread variant.

**Q: How do execute and submit differ?**

A: `execute` returns no result handle; `submit` returns a future and normally captures task failure for retrieval through it.

**Q: Does Future.get(timeout) cancel the task on timeout?**

A: No. Waiting times out; task cancellation is a separate decision.

**Q: What is the difference between shutdown and shutdownNow?**

A: `shutdown` begins orderly termination. `shutdownNow` attempts interruption and prevents queued work from starting. Neither waits for termination or guarantees running code stops.

**Q: Why can newFixedThreadPool still exhaust memory?**

A: Its worker count is bounded, but its queue is unbounded. Producers can enqueue tasks faster than workers complete them.

**Q: When does ThreadPoolExecutor create workers beyond the core size?**

A: Normally when the queue cannot accept a task, up to the maximum size. An unbounded queue usually prevents this expansion.

**Q: How do invokeAll and invokeAny differ?**

A: `invokeAll` returns futures after all finish or its timed wait expires. `invokeAny` returns one successful result and cancels unfinished tasks.

**Q: What is the difference between fixed rate and fixed delay?**

A: Fixed rate targets a recurring schedule; fixed delay measures the gap after the preceding execution finishes.

**Q: Do virtual threads make computation faster?**

A: They primarily improve scalability for many waiting tasks. CPU-bound work still competes for the available processors.

**Q: What visibility does task submission provide?**

A: Prior caller actions happen-before task execution, and task actions happen-before result consumption after the corresponding `Future.get()`.
