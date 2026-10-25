# Transaction Isolation Levels

## Overview

Isolation determines what a transaction may observe and how concurrent transactions may affect its decisions. It is the isolation part of ACID; atomic commit/rollback alone does not make concurrent read/decide/write logic correct.

SQL names four standard levels: **Read Uncommitted, Read Committed, Repeatable Read, and Serializable**. They describe guarantees, not a universal implementation algorithm. Databases can provide stronger behavior than the minimum suggested by a level's name.

Isolation levels and [optimistic/pessimistic locking](1_Optimistic_and_Pessimistic_Locking.md) are related but different: a level defines transaction guarantees, while validation and explicit locking are ways to coordinate particular operations. The PostgreSQL sections below refer to PostgreSQL 18.

## Foundations: Transactions and Interleaving

```sql
BEGIN;
UPDATE accounts SET balance = balance - 20 WHERE id = 1;
UPDATE accounts SET balance = balance + 20 WHERE id = 2;
COMMIT;
```

Assume both accounts exist and the debit is permitted. Other transactions must not observe a dirty half-transfer, but the observations allowed across several reader statements depend on isolation. A concurrent reader might read account 1 before this commit and account 2 after it, combining states from different moments.

**Serial execution** runs transactions one after another. **Serializable execution** may run them concurrently, but committed results must be consistent with some serial ordering. It does not require one global transaction mutex or guarantee that every read returns the newest wall-clock state.

## Anomalies to Understand

| Anomaly | Example | Why it matters |
| --- | --- | --- |
| Dirty read | Read another transaction's uncommitted change, which may roll back | Decisions depend on data that never becomes committed |
| Nonrepeatable read | Read one row twice and get different committed values | A previously checked fact changes |
| Phantom read | Repeat a predicate query and see a changed matching row set | Counts, availability checks, and ranges can change |
| Lost update | Two callers read `100`, then overwrite it with independent derived values | One committed change is erased by a stale write |
| Read skew | Read related values from different committed states | A report can combine facts that never coexisted |
| Write skew | Read a shared rule, then update different rows so both decisions violate it together | Snapshot consistency does not necessarily preserve cross-row invariants |
| Serialization anomaly | Committed observations/results fit no serial transaction ordering | The general failure Serializable prevents |

The classic dirty/nonrepeatable/phantom checklist does not fully describe lost updates or write skew. Always examine the actual read/write pattern, not only an isolation-level label.

## Standard Minimum Guarantees

“Possible” means the level does not generally exclude the phenomenon; an implementation may provide stronger guarantees.

| Level | Dirty reads | Nonrepeatable reads | Phantoms | Serialization anomalies |
| --- | --- | --- | --- | --- |
| Read Uncommitted | Possible | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible | Possible |
| Serializable | Prevented | Prevented | Prevented | Prevented |

Read Uncommitted permits viewing uncommitted work. Read Committed protects against that, but need not give successive statements one shared snapshot. Repeatable Read keeps rereads stable under its implementation's rules. Serializable additionally protects against nonserializable combinations of reads and writes. See the [standard/PostgreSQL comparison](https://www.postgresql.org/docs/18/transaction-iso.html#MVCC-ISOLEVEL-TABLE).

## PostgreSQL: MVCC Foundations

PostgreSQL uses **Multiversion Concurrency Control (MVCC)**: updates create new physical versions of logical rows. A query's snapshot determines which versions are visible, using transaction visibility rules rather than copying the entire database.

Normal reads usually do not block row updates, and row updates usually do not block normal reads: a reader can use an older visible version. This does not mean “no locks”: writers coordinate with writers, and table locks for DDL can block ordinary queries. See the [MVCC introduction](https://www.postgresql.org/docs/18/mvcc-intro.html).

Row metadata includes `xmin` for the creating transaction and `xmax` associated with deletion/update or locking state. These fields require interpretation alongside transaction status; a nonzero `xmax` alone does not prove a row is invisible. Transaction IDs wrap, and `ctid` identifies a physical version rather than a stable logical row. See [system columns](https://www.postgresql.org/docs/18/ddl-system-columns.html).

All levels see their transaction's own preceding changes. A stable snapshot therefore does not mean results can never change after the transaction itself writes.

## PostgreSQL Isolation Behavior

| Requested level | Ordinary read snapshot | Important behavior |
| --- | --- | --- |
| Read Uncommitted | Same as Read Committed | No dirty reads; no separate weaker implementation |
| Read Committed, the default | New snapshot for each statement | Later statements can observe newly committed changes |
| Repeatable Read | Transaction snapshot established by the first query/data-modification statement | Snapshot Isolation; prevents ordinary phantom reads, but permits write skew |
| Serializable | Transaction snapshot plus Serializable Snapshot Isolation checks | Rejects potentially nonserializable conflict patterns with retryable errors |

`BEGIN` alone does not pin a Repeatable Read/Serializable snapshot. Transaction-control statements do not establish it; the first relevant data statement does. See [SET TRANSACTION](https://www.postgresql.org/docs/18/sql-set-transaction.html).

### Configuring the Level

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT id, balance FROM accounts ORDER BY id;
COMMIT;

SHOW transaction_isolation;
```

Alternatively, use `SET TRANSACTION ISOLATION LEVEL ...` immediately after `BEGIN`, before data access. Changing isolation after the first data statement is not allowed. Each autocommit statement is its own transaction; several independent autocommit reads do not share a Repeatable Read snapshot. With connection pools, ensure session defaults do not accidentally leak into unrelated requests.

## Read Committed in PostgreSQL

### Successive Reads and Read Skew

Assume accounts 1 and 2 each start at `100`. Run these steps through two separate sessions:

| Step | Session A: Read Committed | Session B |
| --- | --- | --- |
| 1 | `BEGIN ISOLATION LEVEL READ COMMITTED;` | |
| 2 | `SELECT balance FROM accounts WHERE id = 1;` returns `100` | |
| 3 | | `BEGIN;` |
| 4 | | Debit account 1 to `80`, credit account 2 to `120` |
| 5 | | `COMMIT;` |
| 6 | `SELECT balance FROM accounts WHERE id = 2;` returns `120` | |
| 7 | `COMMIT;` | |

A combines `100 + 120 = 220`, even though the committed total was always `200`. It performed two reads from different snapshots. One ordinary query reading both balances gets one statement snapshot; Repeatable Read can provide one snapshot across several reads.

### Updates Can Follow a Newer Row Version

An updating command initially searches with its statement snapshot. If a target row has a conflicting in-progress update, it can wait; after the other transaction commits, PostgreSQL rechecks the predicate against the updated row version.

```sql
UPDATE inventory
SET available = available - 1
WHERE product_id = 42 AND available > 0
RETURNING available;
```

With one unit left, competing decrements do not both consume that same unit at Read Committed: after waiting, the second statement sees that its row no longer qualifies. Check the affected rows. By contrast, writing a constant calculated from a stale application read can still overwrite another update. See [UPDATE](https://www.postgresql.org/docs/18/sql-update.html).

This recheck is row-oriented, not a fresh whole-query snapshot. Complex updating statements can combine an updated target row with older observations elsewhere. A statement-level snapshot is not a complete business-invariant guarantee.

### Uniqueness and UPSERT Are Not Just Snapshot Reads

At Read Committed, `INSERT ... ON CONFLICT DO UPDATE` can update a conflicting row that was not visible in the statement's original snapshot. `DO NOTHING` can also skip insertion because of a concurrent row not visible in that snapshot. Do not treat a prior “not found” query as permission to insert without a constraint. `MERGE` is not a drop-in substitute for the same concurrent insert-or-update guarantee. See [INSERT / ON CONFLICT](https://www.postgresql.org/docs/18/sql-insert.html).

## Repeatable Read: Snapshot Isolation

PostgreSQL keeps one snapshot across ordinary reads in the transaction. Other transactions' later commits do not appear, so repeating a predicate query does not acquire new phantom rows from their inserts. This is stronger than the standard minimum for Repeatable Read.

However, overlapping writers can fail when trying to update or lock a row changed since their snapshot was established. PostgreSQL raises `40001` rather than letting a Repeatable Read transaction silently move to a newer version. Taking `SELECT FOR UPDATE` does not refresh the existing transaction snapshot. See [row-lock behavior](https://www.postgresql.org/docs/18/explicit-locking.html#LOCKING-ROWS).

### Write Skew: Different Rows, One Invariant

Suppose at least one doctor must remain on call:

```sql
CREATE TABLE doctors (
    id integer PRIMARY KEY,
    on_call boolean NOT NULL
);
INSERT INTO doctors VALUES (1, true), (2, true);
```

The application allows a doctor to leave only if its count query returns more than one. Interleave two sessions as follows; both initial reads must happen before either update:

| Step | Session A | Session B |
| --- | --- | --- |
| 1 | `BEGIN ISOLATION LEVEL REPEATABLE READ;` | |
| 2 | `SELECT count(*) FROM doctors WHERE on_call;` returns `2` | |
| 3 | | `BEGIN ISOLATION LEVEL REPEATABLE READ;` |
| 4 | | Same count query returns `2` |
| 5 | `UPDATE doctors SET on_call = false WHERE id = 1;` | |
| 6 | `COMMIT;` | |
| 7 | | `UPDATE doctors SET on_call = false WHERE id = 2;` |
| 8 | | `COMMIT;` |

Both can commit under PostgreSQL Repeatable Read because they modify different rows. Now no doctor is on call. No dirty read or visible phantom was needed: each transaction acted on a stable, individually plausible snapshot.

Possible solutions include a serializable transaction for the complete decision, or an explicit shared guard protocol covering all relevant changes. Simply versioning each doctor's own row does not catch the conflict. See [application-level consistency](https://www.postgresql.org/docs/18/applevel-consistency.html).

## Serializable in PostgreSQL: SSI

PostgreSQL uses **Serializable Snapshot Isolation (SSI)**, rather than executing every transaction sequentially. Transactions read snapshots, while the database tracks read/write dependencies that could make their combined results nonserializable.

In the doctor example, each transaction reads the on-call status that the other changes. A dependency cycle would require A to precede B and B to precede A. If both complete decisions use Serializable, PostgreSQL prevents both conflicting outcomes from committing: a participant can fail during a statement or at commit. The application must rerun its complete decision against a fresh transaction.

### Predicate Tracking Is Not a Blocking Range Lock

SSI records reads using predicate locks exposed as `SIReadLock` in `pg_locks`. These represent read dependencies, including predicate/range observations, rather than reservations that block matching inserts. They do not themselves cause deadlocks; ordinary writer locks still can.

Tracking can cover tuples, pages, or relations depending on the plan and granularity. Coarser tracking and conservative dependency detection can cause retries even when an actual anomaly was not inevitable. Read tracking can outlive commit until overlapping transactions finish. See [Serializable isolation](https://www.postgresql.org/docs/18/transaction-iso.html#XACT-SERIALIZABLE) and [pg_locks](https://www.postgresql.org/docs/18/view-pg-locks.html).

To rely on this protection, use Serializable for all relevant transactions that participate in the invariant, or define a separate compatible protocol. Merely raising the isolation of one reader does not repair arbitrary lower-isolation writers or incorrect business logic. Serializable validates a possible serial ordering, not whether that ordering respects an unstated business rule.

### Read-Only Deferrable Transactions

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SELECT sum(balance) FROM accounts;
COMMIT;
```

This mode can wait initially for a safe snapshot, after which the transaction avoids the ordinary SSI serialization-failure risk. It is useful for long reports or consistent exports when startup waiting is acceptable. `DEFERRABLE` has this special benefit only with `SERIALIZABLE READ ONLY`; it is not a request to defer constraints. See [transaction modes](https://www.postgresql.org/docs/18/sql-set-transaction.html).

## Isolation, Explicit Locks, and Constraints

- A normal MVCC read does not reserve the rows it observes. `SELECT FOR UPDATE` adds row-locking behavior.
- Locking existing matching rows does not generally lock the absence of other matches. Inserts can change a business predicate without conflicting with those row locks.
- `UNIQUE`, exclusion constraints, foreign keys, and row-local `CHECK` constraints express different integrity rules. They still matter at Serializable. A PostgreSQL `CHECK` is not a safe mechanism for enforcing arbitrary rules over other rows.
- A shared guard row can serialize decisions if every participant follows the protocol. At Read Committed, lock it before reading the protected predicate. At Repeatable Read, a lock alone cannot replace the already selected snapshot; updating a shared guard or using another design may be necessary to force conflict detection.

See [constraints](https://www.postgresql.org/docs/18/ddl-constraints.html) and [consistency with explicit locks](https://www.postgresql.org/docs/18/applevel-consistency.html#NON-SERIALIZABLE-CONSISTENCY).

## Retryable Errors and Transaction Boundaries

| SQLSTATE | Meaning | Typical response |
| --- | --- | --- |
| `40001` | Serialization failure, including PostgreSQL Repeatable Read update conflicts | Roll back and retry the complete transaction with fresh reads |
| `40P01` | Deadlock detected | Roll back; retry if appropriate; investigate lock ordering |
| `55P03` | Lock not available, including relevant `NOWAIT` / lock-timeout failures | Roll back or recover through an established savepoint; follow a bounded waiting policy |
| `23505` | Unique violation | Usually handle as a constraint/business conflict; do not blindly retry |

After an ordinary statement error, the transaction normally remains failed until rollback or recovery to a savepoint. Retrying only the last statement with the same stale decisions is not sufficient for `40001`.

```text
repeat up to the retry limit:
    begin a fresh transaction at the chosen isolation level
    read current inputs and recompute the complete decision
    apply all database changes
    commit
    return success only after commit succeeds
on a retryable database error:
    roll back, then retry with bounded backoff and jitter
```

Keep emails, payments, and other irreversible external effects out of a retried transaction body unless their own protocol handles duplication. A transactional outbox can persist the intent with the database change. Do not publish results from a transaction whose commit later fails. See [serialization failure handling](https://www.postgresql.org/docs/18/mvcc-serialization-failure-handling.html) and [SQLSTATE codes](https://www.postgresql.org/docs/18/errcodes-appendix.html).

## Advanced Operational Details

- **Long snapshots retain old versions.** Vacuum cannot remove versions still needed by active snapshots. Long or idle-in-transaction sessions can contribute to bloat and maintenance pressure. See [vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html).
- **Isolation does not make sequences transactional.** `nextval()` allocations are not undone on rollback; gaps are normal. Sequence values are not a reliable commit-order clock. See [sequence functions](https://www.postgresql.org/docs/18/functions-sequence.html).
- **Snapshot guarantees have DDL exceptions.** `TRUNCATE` and table-rewriting operations have MVCC caveats. Hot standby also does not support Serializable in PostgreSQL 18; Repeatable Read is its strongest level. See [MVCC caveats](https://www.postgresql.org/docs/18/mvcc-caveats.html).
- **No universal performance ranking.** Stronger isolation can reduce application locking complexity but adds tracking/retry costs. Coarse pessimistic locks can be more expensive than SSI for some workloads; measure throughput, wait time, and retry rate together.

## Choosing a Level

| Requirement | Starting choice | Additional work |
| --- | --- | --- |
| Simple operations with constraints and atomic updates | Read Committed | Validate outcomes; avoid stale application replacements |
| Stable ordinary reads across multiple reporting queries | Repeatable Read | Keep the snapshot short enough; understand write-skew limits |
| Decisions over predicates or several rows that must behave serially | Serializable | Whole-transaction retries and consistent participant policies |
| Long serializable read-only report | Serializable Read Only Deferrable | Accept possible startup waiting |

The correct choice follows the invariant, query pattern, and database implementation, rather than a rule that one level is always sufficient or always faster.

## Best Practices

- State the invariant and list the reads and writes that implement it.
- Choose the transaction boundary before choosing the isolation level.
- Distinguish a stable snapshot from serializable business decisions.
- Prefer database constraints for integrity rules the database can express directly.
- Implement bounded whole-transaction retries where the selected level requires them.
- Test critical schedules through independent connections with deliberate interleaving, including commit failures, not only sequential SQL.

## Common Pitfalls

- **Assuming PostgreSQL Repeatable Read allows ordinary phantoms.** Its snapshot implementation prevents them; the standard minimum is weaker.
- **Assuming no phantoms means Serializable.** Write skew can occur without any transaction observing changed query results.
- **Treating BEGIN as the snapshot timestamp.** PostgreSQL establishes the stable snapshot when relevant data access starts.
- **Using SELECT FOR UPDATE to refresh a Repeatable Read snapshot.** It can instead fail on a changed target row.
- **Selecting an absent key before inserting without a constraint.** Other transactions can make the same decision.
- **Retrying just COMMIT or the final UPDATE after 40001.** Earlier reads and decisions must be repeated too.
- **Equating Serializable with no waits or no failures.** Ordinary locks can still block, and SSI can abort transactions.

## Interview Questions

**Q: What is the difference between atomicity and isolation?**

A: Atomicity commits or rolls back a transaction's changes together. Isolation constrains interactions between concurrent transactions.

**Q: What is PostgreSQL's default isolation level?**

A: Read Committed, with a fresh ordinary read snapshot per statement.

**Q: Does PostgreSQL support dirty reads through Read Uncommitted?**

A: No. The requested mode behaves as Read Committed.

**Q: Why can PostgreSQL Repeatable Read prevent phantoms but still allow write skew?**

A: Its snapshot keeps each reader's observations stable, but decisions based on overlapping reads can update different rows and violate a joint invariant.

**Q: How does Serializable differ from Repeatable Read in PostgreSQL?**

A: SSI adds dependency tracking and aborts potentially nonserializable executions. Both use stable snapshots, and both can require retries for writers.

**Q: Do PostgreSQL Serializable predicate locks block inserts?**

A: No. `SIReadLock` tracks dependencies; it is not a blocking gap/range reservation.

**Q: Why does an atomic increment differ from read-then-write?**

A: The database can apply the increment to the current qualifying row under write coordination. A constant computed from a stale application read can overwrite a newer value.

**Q: Does a read-only transaction always avoid serialization failures?**

A: A PostgreSQL Repeatable Read read-only transaction does not have write-conflict failures, but ordinary Serializable read-only transactions can face SSI failures. Serializable Read Only Deferrable waits for a safe snapshot.

**Q: What should be retried after SQLSTATE 40001?**

A: The entire transaction, including all reads and the logic that chose its writes, using a fresh transaction.

**Q: Does Serializable make external API calls transactional?**

A: No. Database rollback and isolation do not undo payments, messages, or other external effects.
