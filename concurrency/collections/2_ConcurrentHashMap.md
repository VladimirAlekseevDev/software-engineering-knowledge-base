# ConcurrentHashMap

## Overview

`java.util.concurrent.ConcurrentHashMap<K, V>` is a thread-safe hash-based implementation of `ConcurrentMap`. It supports concurrent lookups and updates without serializing every operation through one map-wide monitor.

Use it for shared registries, independent per-key state, lookup tables, and frequency maps. Its main properties are:

- No insertion order or sorted-key order.
- No `null` keys or values; `get(key) == null` means no mapping was found by that retrieval.
- Atomic operations for individual mappings, including conditional updates and computations.
- Weakly consistent traversal, rather than a snapshot of the whole map.
- No built-in eviction, expiration, capacity bound, or transaction across keys.

See the [Java 25 API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html) and the [concurrent collections overview](1_Overview.md).

## Basic Usage

```java
ConcurrentHashMap<String, String> registry = new ConcurrentHashMap<>(100);

registry.put("billing", "https://billing.example");
String endpoint = registry.get("billing");
String fallback = registry.getOrDefault("reports", "https://default.example");
registry.remove("billing");
```

The constructor argument is an expected mapping count used for initial sizing, not a maximum size. The map grows as needed. `getOrDefault()` supplies a return value; it does not insert that default.

Examples below use imports from `java.util`, `java.util.concurrent`, and `java.util.concurrent.atomic` as needed.

## Comparison with Other Maps

| Map | Concurrency model | Good starting use |
| --- | --- | --- |
| `HashMap` | Requires a separate synchronization protocol for shared mutation | Thread-confined state or state already protected by a lock |
| `Collections.synchronizedMap(...)` | Operations coordinate through the wrapper's monitor; traversal needs external synchronization | Code requiring one shared monitor protocol |
| `Hashtable` | Legacy synchronized map | Compatibility with existing APIs |
| `ConcurrentHashMap` | Concurrent retrievals and scalable updates; per-key atomic APIs | Shared mappings without sorted ordering |
| [ConcurrentSkipListMap](ConcurrentSkipListMap.md) | Concurrent sorted/navigable map | Ordered keys and range queries |

For a synchronized wrapper, all access must go through the wrapper, and compound sequences may need an enclosing synchronized block. `synchronized(concurrentMap)` does **not** freeze a `ConcurrentHashMap`: its ordinary operations do not acquire that monitor. See [synchronized map wrappers](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/Collections.html#synchronizedMap(java.util.Map)).

## Atomic Operations

| Operation | Guarantee for the target key |
| --- | --- |
| `putIfAbsent(key, value)` | Insert only if absent; return the previous value or `null` if inserted |
| `remove(key, expected)` | Remove only if the current value equals `expected`; return success |
| `replace(key, value)` | Replace only if present; return the previous value or `null` |
| `replace(key, expected, replacement)` | Replace only if the current value equals `expected`; return success |
| `computeIfAbsent(key, function)` | Compute and install a non-null value when absent |
| `computeIfPresent(key, function)` | Recompute only when present |
| `compute(key, function)` | Recompute whether present or absent; the old value can be `null` |
| `merge(key, value, function)` | Insert the supplied non-null value if absent; otherwise combine with the existing value |

For `compute`, `computeIfPresent`, and the combining function of `merge`, a `null` result removes the mapping or leaves it absent. A `null` result from `computeIfAbsent` leaves the key absent. Exceptions from a computation leave the mapping unchanged or unestablished; side effects on other objects are not rolled back.

These are atomic mapping operations, not a transaction over the map or external systems. See [ConcurrentMap conditional operations](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentMap.html).

### Atomic Counters

`get()` followed by `put()` can lose updates: two callers can read the same value and both write its successor. Use one atomic operation instead:

```java
ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();

counts.merge("orders", 1, Integer::sum);
counts.compute("orders", (key, current) -> current == null ? 1 : current + 1);
// With no other updates, "orders" now maps to 2.
```

Both updates cover one key. Updating two account balances with two separate `compute()` calls still permits observers to see an intermediate state. Multi-key invariants need another protocol followed by all relevant readers and writers, or a data model that stores the invariant as one mapping.

### Conditional State Transitions

```java
ConcurrentHashMap<String, String> states = new ConcurrentHashMap<>();
states.putIfAbsent("job-42", "QUEUED");

boolean claimed = states.replace("job-42", "QUEUED", "RUNNING");
boolean removed = states.remove("job-42", "DONE");
// claimed == true and removed == false without competing changes.
```

The comparison and update happen together. Conditional comparisons use equality, not an object-identity ownership token. If state can cycle back to an equal value, use a version or unique token when the intervening history matters.

### Lazy Initialization with computeIfAbsent

```java
ConcurrentHashMap<String, String> labels = new ConcurrentHashMap<>();
String label = labels.computeIfAbsent("pending-order", key -> key.replace('-', ' '));
```

The `ConcurrentHashMap` override performs the invocation atomically. It calls the function once for that invocation when the key is absent, and not when present. Concurrent callers share an installed value while it remains mapped.

This is **not once forever per key**: returning `null`, throwing, or removing the mapping permits later computation. In contrast, `putIfAbsent(key, expensiveWork())` evaluates `expensiveWork()` before calling the map, even when the key already exists. Do not generalize this override's callback guarantee to every `ConcurrentMap` implementation.

Keep mapping functions short and simple. Avoid network or database calls, blocking waits, and acquiring unrelated locks inside them. They can delay competing updates, including updates to colliding keys. Do not modify this map from a compute callback; detectable recursive updates can throw `IllegalStateException`. See [computeIfAbsent](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html#computeIfAbsent(K,java.util.function.Function)).

## Java Memory Model Guarantees

An update of a key **happens-before** a non-null retrieval for that key that reports the updated value. Initialization writes before publishing an object through the map therefore become visible to the thread retrieving it. The guarantee includes ordinary fields and referenced state; those writes do not each need `volatile`.

```text
Publisher: initialize value -> put(key, value)
                                      |
                                 happens-before
                                      v
Reader:                         get(key) -> use returned value
```

This orders observations for the relevant mapping; it does not prescribe hardware cache flushing or a global order of operations across all keys. A retrieval racing an update may observe the previous mapping or absence rather than waiting for a new value.

Later mutation of an already published object requires its own synchronization. Publishing the map reference itself must also follow a safe-publication protocol. See the [ConcurrentMap memory contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentMap.html).

### Mutable Values

`ConcurrentHashMap<String, ArrayList<String>>` does not make concurrent calls to a retrieved list's `add()` safe. Even placing list mutation inside `compute()` does not protect readers that access that list outside the same protocol.

One option is atomic replacement with immutable values:

```java
ConcurrentHashMap<String, List<String>> groups = new ConcurrentHashMap<>();
groups.compute("admins", (key, current) -> {
    List<String> updated = new ArrayList<>(current == null ? List.of() : current);
    updated.add("Alice");
    return List.copyOf(updated);
});

List<String> members = groups.get("admins"); // Stable, immutable list value.
```

This copies the list on each update. Alternatives include independently thread-safe values or explicit locking. Immutable replacement is shallow: mutable elements still require care.

## Frequency Maps with LongAdder

```java
ConcurrentHashMap<String, LongAdder> frequencies = new ConcurrentHashMap<>();
frequencies.computeIfAbsent("orders", key -> new LongAdder()).increment();
long observed = frequencies.get("orders").sum();
```

`LongAdder` spreads contended increments across internal cells, often improving throughput for statistics at the cost of space. Its `sum()` is **not an atomic snapshot during updates**. Use `AtomicLong` when compare-and-set or atomic counter transitions are needed. See [LongAdder](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/atomic/LongAdder.html).

This pattern assumes mappings are not removed or replaced during counting. Otherwise a thread can increment an adder that is no longer in the map. Installing the adder and incrementing it are separate operations, not one atomic map update.

## Iteration, Views, and Aggregate Operations

- Iterators are **weakly consistent**: traversal tolerates concurrent changes without `ConcurrentModificationException`, but need not represent one point in time. Use each iterator from one thread at a time.
- `keySet()`, `values()`, and `entrySet()` are backed views; supported removals affect the map. A returned value or entry does not reserve the mapping against later replacement.
- `size()`, `mappingCount()`, `isEmpty()`, and `containsValue()` can reflect transient state during updates. They cannot enforce an exact admission limit or predict a later operation.
- `putAll()`, `clear()`, `replaceAll()`, and traversal do not form a whole-map transaction. Other threads can observe partially completed work.
- `new HashMap<>(map)` or `Map.copyOf(map)` produces a separate map, but concurrent copying does not guarantee a snapshot of one instant. Exact snapshots require coordinated writers or another design.

For a concurrent hash-based set:

```java
Set<String> activeIds = ConcurrentHashMap.newKeySet();
activeIds.add("session-42");
boolean firstRegistration = activeIds.add("session-42"); // false
```

The ordinary `map.keySet()` view does not support adding keys because no value is supplied. `map.keySet(defaultValue)` supports additions mapped to that value. See [KeySetView](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.KeySetView.html).

## Bulk Operations

`forEach`, `search`, and `reduce` have variants that can process entries concurrently. Their results combine per-entry observations, rather than providing an atomic whole-map snapshot.

```java
ConcurrentHashMap<String, Integer> totals = new ConcurrentHashMap<>();
totals.put("east", 10);
totals.put("west", 20);
int total = totals.reduceValuesToInt(Long.MAX_VALUE, Integer::intValue, 0, Integer::sum);
// total == 30 without concurrent updates.
```

`Long.MAX_VALUE` suppresses parallelism; lower thresholds allow work in `ForkJoinPool.commonPool()`. Parallel execution has overhead and is not automatically faster. Reduction functions should be associative and commutative, with a correct identity value, and should avoid external side effects and order dependencies.

## How Modern Implementations Work

The following describes **OpenJDK 25 implementation details**, rather than a portable API contract:

- A table of bins stores mappings. Most bins use linked nodes; heavily collided bins can use balanced trees.
- An ordinary insertion into an empty bin can use CAS. Updates of populated bins generally synchronize at bin level; different keys can still contend when they share a bin.
- Retrievals traverse published table and node state without taking those update locks. The entire collection is not a lock-free data structure: update paths can block.
- Resizing can involve multiple threads helping transfer bins. Pre-sizing avoids some resizing work, not all contention.
- Java 8 and later use this table/bin approach rather than the fixed `Segment` partitioning associated with Java 7. `concurrencyLevel` is a sizing hint, not a promised segment or worker count.

With well-distributed hashes, basic lookup/update work is expected to be approximately constant time, excluding contention and resizing. Poor hashes and slow `equals()` / `hashCode()` hurt throughput; tree bins are mitigation, not a universal latency guarantee. See the [OpenJDK 25 source](https://github.com/openjdk/jdk/blob/jdk-25-ga/src/java.base/share/classes/java/util/concurrent/ConcurrentHashMap.java).

## Best Practices

- Use immutable keys with stable, well-distributed `hashCode()` and consistent `equals()`; changing a key's equality-related state while stored breaks lookup assumptions.
- Choose atomic conditional or remapping APIs for per-key compound actions.
- Prefer immutable values, or explicitly define how mutable values are protected.
- Pre-size when the expected mapping count is known; measure contention and allocation for the actual workload.
- Use a bounded cache implementation when eviction or expiration is required. A concurrent map alone can grow indefinitely.
- Use an external coordination protocol for cross-key invariants or exact snapshots, ensuring every participating access follows it.

The [Map contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/Map.html) explains equality requirements and mutable-key hazards.

## Common Pitfalls

- **Assuming several safe calls form one atomic operation.** `containsKey()` followed by `put()` still races; so does `get()` followed by `put()`.
- **Confusing null with a cached negative result.** Null values are forbidden; use an explicit non-null representation such as `Optional` when absence must be cached.
- **Treating getOrDefault as initialization.** It never installs the fallback value.
- **Running slow or recursive work in a mapping callback.** Updates can block, and callbacks must respect the map's restrictions.
- **Using size() to enforce a capacity limit.** Concurrent check-then-insert can exceed the limit.
- **Assuming removal revokes references already returned.** Callers can still hold and mutate an old value.

## Interview Questions

**Q: Is every operation atomic?**

A: Individual mapping operations have their documented guarantees. A sequence of calls, bulk update, or cross-key business operation is not automatically atomic as a whole.

**Q: How does it differ from Collections.synchronizedMap?**

A: A synchronized wrapper coordinates through one monitor. `ConcurrentHashMap` supports overlapping retrievals and updates with per-key atomic APIs; synchronizing on its object does not stop ordinary operations.

**Q: Why are null keys and values prohibited?**

A: The API reserves null as an unambiguous absence/no-result marker, including computation and reduction semantics. A separate `containsKey()` check could race with changes.

**Q: Is computeIfAbsent guaranteed to run once?**

A: Its callback runs once for that method invocation if the key is absent. A non-null installed mapping can be reused by competitors, but null results, failures, or removal allow future computations.

**Q: What does get observe during compute?**

A: Retrieval does not wait for the compute callback's update lock. It can observe an existing mapping, or absence while an absent value is being computed; it does not see the new mapping before publication.

**Q: Does putting a mutable object into the map make it thread-safe?**

A: No. Publication exposes prior initialization safely; later object mutations need their own protocol.

**Q: Is iteration a snapshot?**

A: No. It is weakly consistent and can reflect concurrent updates. It does not throw `ConcurrentModificationException` because of those updates.

**Q: Does it still use Segment locks?**

A: That describes older implementations. Modern OpenJDK uses table bins, CAS, and localized update synchronization; `concurrencyLevel` is a sizing hint.

**Q: Is it lock-free?**

A: Not as a whole. Retrievals avoid update locks, but update operations can synchronize and block; callbacks and key methods also affect progress.

**Q: Can two different keys contend?**

A: Yes. Colliding keys can share a bin, and resizing or other shared bookkeeping also creates coordination costs.

**Q: Can it support an atomic transfer between two account keys?**

A: Not through two independent updates. Use shared coordination for all participating accesses, a single immutable aggregate mapping, or transactional storage.

**Q: Why use LongAdder values instead of Integer counters?**

A: They can scale better for contended statistical increments. Their sums are not atomic snapshots, and removing/replacing adders during updates needs additional coordination.

**Q: When should another map be chosen?**

A: Use `HashMap` for confined or consistently locked state, a sorted concurrent map for range queries, and a cache implementation for managed eviction and expiration.
