# Optimistic and Pessimistic Locking

## Overview

These are strategies for coordinating operations that may conflict over shared database state:

- **Optimistic locking:** read and prepare a change without reserving the row, then atomically validate that the state has not changed before writing.
- **Pessimistic locking:** acquire a database lock before making a decision that depends on the protected state, and retain it through the change.

The choice concerns conflict handling, not a transaction's entire isolation contract. Both can be used alongside different [isolation levels](2_Isolation_Levels.md). PostgreSQL-specific examples below follow PostgreSQL 18 behavior.

## The Problem: Lost Updates

Suppose an account balance starts at `100`:

| Step | Transaction A | Transaction B |
| --- | --- | --- |
| 1 | Read `100`; calculate `110` | |
| 2 | | Read `100`; calculate `120` |
| 3 | Write `110`; commit | |
| 4 | | Write `120`; commit |

The intended combined result was `130`, but B overwrote A's change using stale input. A transaction around each read/write sequence does not automatically prevent this at every isolation level.

An atomic SQL expression such as `SET balance = balance + 10` avoids this particular stale read-modify-write pattern. More complicated edits may need validation or explicit locks.

## Optimistic Locking

### Version-Based Validation

Use a version column as a token identifying the state that was read:

```sql
CREATE TABLE documents (
    id bigint PRIMARY KEY,
    body text NOT NULL,
    version bigint NOT NULL DEFAULT 0
);

INSERT INTO documents (id, body, version) VALUES (1, 'Original', 7);
```

Read the document and retain its version. Later, perform the validation and replacement **in one statement**:

```sql
SELECT body, version FROM documents WHERE id = 1;
-- Application retains version 7, without keeping a transaction open while editing.

UPDATE documents
SET body = 'Edited by Alice', version = version + 1
WHERE id = 1 AND version = 7
RETURNING version;
```

One returned row means this statement applied the change. Zero means no row matched: it may have changed, been deleted, or failed another predicate. A successful statement inside an explicit transaction still needs a successful commit. See [UPDATE and RETURNING](https://www.postgresql.org/docs/18/sql-update.html).

If Alice and Bob both read version `7`, Alice's committed update produces `8`; Bob's subsequent version-`7` update affects zero rows. At PostgreSQL Read Committed, a competing updater can wait and then recheck the predicate. At higher isolation levels, a conflict may instead surface as a serialization failure.

Never split the version check and write into separate unprotected statements. All update paths must maintain the token, including administrative SQL and bulk updates. Otherwise a writer can bypass conflict detection.

This is the same broad principle as compare-and-set: replace only the state you expected. It does not imply the database implements the operation using a CPU CAS instruction. The approach also supports business edits spanning several short database transactions; see [Optimistic Offline Lock](https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html).

### What to Do After a Conflict

- For an interactive edit, return a conflict and let the user reload or merge changes.
- For a safe automated operation, reread the latest state and recompute the decision in a fresh transaction.
- If other statements belong to the same failed business operation, roll back those changes too.
- Bound retries and use backoff under contention. Blindly retrying the same stale replacement can erase the newer edit's intent.

A dedicated integer version is usually clearer than a timestamp, whose precision or generation rules may miss changes. PostgreSQL transaction IDs such as `xmin` are implementation metadata and wrap over time, so they are not a general-purpose long-lived application version token. See [system columns](https://www.postgresql.org/docs/18/ddl-system-columns.html).

### Advantages and Costs

| Advantages | Costs / limitations |
| --- | --- |
| No row reservation during user think time | Conflicts are discovered after some work has already been done |
| Independent readers and editors can proceed | Hot rows can cause repeated failures and wasted computation |
| Well suited to request/response editing | Every relevant writer must follow the version protocol |
| Detects stale overwrites explicitly | One row's version does not protect an invariant across unrelated rows |

**Optimistic does not mean lock-free.** The final `UPDATE` still takes database write locks and can wait, time out, or participate in a deadlock. The optimistic property is that no reservation is held throughout the preceding read/decision interval.

## Pessimistic Locking

### Lock Before the Dependent Decision

Suppose inventory is stored as follows:

```sql
CREATE TABLE inventory (
    product_id bigint PRIMARY KEY,
    available integer NOT NULL CHECK (available >= 0)
);

INSERT INTO inventory VALUES (42, 10);
```

Acquire a row lock and retain the transaction while deciding and updating:

```sql
BEGIN;
SET LOCAL lock_timeout = '2s';

SELECT available
FROM inventory
WHERE product_id = 42
FOR UPDATE;

-- Application checks the returned row before proceeding.
UPDATE inventory
SET available = available - 1
WHERE product_id = 42 AND available > 0
RETURNING available;

COMMIT;
```

Competing updates and conflicting row-lock requests must wait or fail according to their timeout/locking options. Normal MVCC `SELECT` queries can still read a visible version; `FOR UPDATE` does not block every reader. Locks normally last until transaction end, with savepoint-rollback exceptions. In autocommit mode, a standalone locking `SELECT` releases its locks before a later statement, defeating this read/decide/write pattern. See [PostgreSQL explicit locking](https://www.postgresql.org/docs/18/explicit-locking.html).

Do not retain this transaction while waiting for a user, payment provider, or other remote service. For a reservation spanning requests, store an explicit reservation with an expiry and define its lifecycle instead of holding a database connection and row lock open.

### PostgreSQL Row-Lock Modes

| Mode | Intended protection |
| --- | --- |
| `FOR UPDATE` | Strongest row-lock mode; conflicts with all other row-lock modes |
| `FOR NO KEY UPDATE` | Protect ordinary updates while permitting key-share locks |
| `FOR SHARE` | Allow compatible readers that explicitly lock; prevent updates/deletion |
| `FOR KEY SHARE` | Protect against deletion and key-changing updates; allow ordinary non-key updates |

Use the weakest sufficient mode and consult the database's conflict matrix. Row locks coexist with table-level locks: despite its name, a table-level `ROW EXCLUSIVE` lock is not the same thing as locking individual rows.

### Waiting Options and Queue Consumers

- `NOWAIT` fails rather than waiting for a conflicting row lock.
- `SKIP LOCKED` omits rows that cannot immediately be row-locked. It is useful for competing queue consumers, not for an accurate general-purpose report.
- `lock_timeout` limits lock-acquisition waiting; `statement_timeout` limits statement execution more broadly. Neither is a lease that releases a successfully acquired lock after a fixed duration. See [timeout settings](https://www.postgresql.org/docs/18/runtime-config-client.html#GUC-LOCK-TIMEOUT).

An atomic claim can select and update a queue item in the same transaction:

```sql
CREATE TABLE jobs (
    id bigint PRIMARY KEY,
    status text NOT NULL
);
INSERT INTO jobs VALUES (1, 'READY'), (2, 'READY');

BEGIN;
WITH candidate AS (
    SELECT id
    FROM jobs
    WHERE status = 'READY'
    ORDER BY id
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE jobs AS j
SET status = 'PROCESSING'
FROM candidate AS c
WHERE j.id = c.id
RETURNING j.id;
COMMIT;
```

Process the claimed job after commit. A crash after claiming needs a recovery/lease policy; skipping locks does not provide exactly-once external processing or strict FIFO order. `NOWAIT` and `SKIP LOCKED` apply to row locking, not every table-lock wait. See [SELECT locking clauses](https://www.postgresql.org/docs/18/sql-select.html#SQL-FOR-UPDATE-SHARE).

### Advantages and Costs

| Advantages | Costs / limitations |
| --- | --- |
| Coordinates short competing decisions before work proceeds | Waits, connection occupancy, and reduced concurrency |
| Can avoid repeated application-level optimistic conflicts on hot rows | Deadlocks and timeouts still require handling |
| Gives an explicit protocol for shared resource allocation | Long transactions amplify contention |
| Multiple rows can be coordinated when all writers follow the protocol | Existing-row locks alone do not protect missing rows or an arbitrary predicate |

## Choosing a Strategy

| Situation | Starting approach | Why |
| --- | --- | --- |
| User edits a document over minutes | Optimistic version check | Avoid holding locks while the user thinks; surface stale edits |
| Low-conflict profile/configuration changes | Optimistic validation | Conflicts are uncommon and inexpensive to handle |
| Short operation on a highly contested existing resource | Pessimistic row lock or atomic conditional update | Serialize the dependent decision or encode it directly |
| Independent queue consumers | `FOR UPDATE SKIP LOCKED` claim | Consumers can claim different available rows |
| Simple stock decrement | One conditional `UPDATE` | Avoid an unnecessary preliminary read and lock round trip |
| Rule spanning a changing set of rows | Constraint, shared guard protocol, or Serializable | One row's token or lock may not cover the invariant |

For the inventory case, this single statement is often sufficient:

```sql
UPDATE inventory
SET available = available - 1
WHERE product_id = 42 AND available >= 1
RETURNING available;
```

Check whether a row was returned. Do not infer success from an earlier availability query. This statement still uses database locks; it avoids requiring a separate application read/lock sequence.

## Invariants Beyond One Row

A row lock cannot lock a row that does not exist. Locking all currently matching bookings does not generally stop another transaction inserting a new matching booking in PostgreSQL.

Prefer a suitable `UNIQUE` or exclusion constraint where possible. Alternatively, serialize operations through an existing parent/guard row that every participant locks, or use Serializable isolation with retries. A guard protocol must account for snapshots: taking a lock does not refresh an already established Repeatable Read snapshot. Transaction-scoped advisory locks can coordinate logical resources, but only cooperating code honors them. See [constraints](https://www.postgresql.org/docs/18/ddl-constraints.html), [application consistency](https://www.postgresql.org/docs/18/applevel-consistency.html), and [advisory locks](https://www.postgresql.org/docs/18/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS).

## Best Practices

- Identify the business invariant and every operation that can change it before choosing a locking mechanism.
- Prefer constraints and atomic conditional SQL when they directly express the requirement.
- Keep transactions short and acquire multiple locks in a consistent order.
- Treat zero-row optimistic updates, database errors, and failed commits as distinct outcomes.
- Retry retryable database failures from the beginning of the transaction; keep external effects idempotent or coordinate them through an outbox.
- Monitor lock waits, transaction age, deadlocks, and conflict/retry rates. `pg_locks` and `pg_blocking_pids()` help diagnose PostgreSQL blocking; row-lock waits often appear as transaction-ID waits. See [pg_locks](https://www.postgresql.org/docs/18/view-pg-locks.html).

## Common Pitfalls

- **Checking a version, then writing without that predicate.** Another writer can intervene.
- **Calling optimistic locking nonblocking.** Its database write can wait on another transaction.
- **Releasing the pessimistic lock before the dependent write.** The decision loses its protection.
- **Assuming FOR UPDATE protects an entire query predicate.** Missing and future rows need a different invariant strategy.
- **Assuming row locks hide data from ordinary readers.** PostgreSQL MVCC readers normally see an older visible version.
- **Retrying external side effects with the transaction.** A repeated payment or message can survive database rollback.

## Interview Questions

**Q: What is the main difference between optimistic and pessimistic locking?**

A: Optimistic locking validates the expected state when applying a change; pessimistic locking reserves conflicting access before the dependent decision.

**Q: Does optimistic locking avoid database locks?**

A: No. It avoids holding a reservation throughout the read/decision interval, but the final write still uses database concurrency control.

**Q: Why must version validation and update be atomic?**

A: Otherwise another writer can change the row between validation and replacement, allowing a stale overwrite.

**Q: When is pessimistic locking preferable?**

A: For short, contested operations where retrying optimistic work is costly, provided the lock covers the invariant and the transaction stays short.

**Q: Does SELECT FOR UPDATE block ordinary PostgreSQL SELECT queries?**

A: Not through its row lock. Ordinary readers use visible MVCC versions; incompatible table-level locks can still block reads.

**Q: Can either strategy guarantee a rule spanning several rows?**

A: Only with a protocol covering all relevant changes. A single-row lock or version check alone is insufficient.

**Q: Is a version conflict always safe to retry automatically?**

A: No. Automated operations can recompute when semantically safe, but a user's stale edit may require an explicit merge or conflict response.
