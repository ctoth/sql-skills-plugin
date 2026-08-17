# Common SQL Transaction & Isolation Mistakes
## Contents

- [1. Multi-statement invariant left in autocommit](#1-multi-statement-invariant-left-in-autocommit)
- [2. Opening a transaction with no matching COMMIT/ROLLBACK](#2-opening-a-transaction-with-no-matching-commitrollback)
- [3. Assuming a failed migration rolls back on MySQL](#3-assuming-a-failed-migration-rolls-back-on-mysql)
- [4. ROLLBACK (whole transaction) where ROLLBACK TO (savepoint) was meant](#4-rollback-whole-transaction-where-rollback-to-savepoint-was-meant)
- [5. Relying on the default isolation level being the same across engines](#5-relying-on-the-default-isolation-level-being-the-same-across-engines)
- [6. Treating READ UNCOMMITTED as a portable performance knob](#6-treating-read-uncommitted-as-a-portable-performance-knob)
- [7. Check-then-act treated as safe just because it's in a transaction](#7-check-then-act-treated-as-safe-just-because-its-in-a-transaction)


Anti-patterns in LLM-generated SQL around transactions, savepoints, and isolation levels, each with
wrong/right code and a primary-source citation. The skill (`sql-transactions-and-isolation`) owns the
SQL surface; this file holds the high-frequency failure modes. All isolation **theory** (which level
permits which anomaly) routes to the MVCC plugin (`mvcc-isolation-levels-and-anomalies` and siblings);
dialect spellings route to `sql-standard-vs-dialect-map`.

---

## 1. Multi-statement invariant left in autocommit

**The problem:** The model emits a debit and a credit (or any "both or neither" pair) as two bare
statements. In autocommit each commits independently — "Automatically started transactions are
committed when the last SQL statement finishes" ([SQLite — Transactions](https://www.sqlite.org/lang_transaction.html))
— so a crash between them leaves the database in a state that should be impossible. "It would
certainly not do for a system failure to result in Bob receiving $100.00 that was not debited from
Alice" ([PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)).

```sql
-- WRONG — two independent commits; a failure between them loses the $100
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Bob';

-- RIGHT — one atomic, all-or-nothing unit
START TRANSACTION;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Bob';
COMMIT;
```

*Source: [PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html); [SQLite — Transactions](https://www.sqlite.org/lang_transaction.html). Depth: this skill, §1, §5.*

---

## 2. Opening a transaction with no matching COMMIT/ROLLBACK

**The problem:** The model wraps work in `START TRANSACTION` but a branch (an early return, an error
path) leaves without terminating it. The transaction stays open, holding locks and (by isolation)
blocking others, until something forces a rollback. A transaction is "all-or-nothing" only once it is
*terminated* ([PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)).

```sql
-- WRONG — the error path leaves the transaction dangling, holding locks
START TRANSACTION;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
-- if a check fails here, code returns WITHOUT issuing COMMIT or ROLLBACK

-- RIGHT — every path ends in exactly one terminator
START TRANSACTION;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
-- on failure:
ROLLBACK;
-- on success:
COMMIT;
```

*Source: [PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html). Depth: this skill, §2.*

---

## 3. Assuming a failed migration rolls back on MySQL

**The problem:** The model writes a multi-step migration inside `START TRANSACTION … COMMIT` and
assumes a mid-way failure rolls the whole thing back. True on PostgreSQL/SQLite; **false on MySQL**,
where DDL "implicitly end[s] any transaction active in the current session, as if you had done a
`COMMIT` before executing the statement" and "cause[s] an implicit commit after executing"
([MySQL — Implicit Commit](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)). DDL cannot
be rolled back, so the schema is left half-applied.

```sql
-- WRONG (assumption) — on MySQL the ALTER already committed when CREATE INDEX fails;
-- ROLLBACK cannot undo it, leaving the schema half-migrated
START TRANSACTION;
ALTER TABLE orders ADD COLUMN status TEXT;
CREATE INDEX idx_orders_status ON orders(status);
COMMIT;

-- RIGHT (MySQL) — treat each DDL as independent and make the migration idempotent/forward-only
ALTER TABLE orders ADD COLUMN status TEXT;
CREATE INDEX idx_orders_status ON orders(status);
-- (on PostgreSQL/SQLite the wrapped version IS atomic and is preferred)
```

*Source: [MySQL — Statements That Cause an Implicit Commit](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html). Depth: this skill, §6; full matrix owned by `sql-standard-vs-dialect-map`.*

---

## 4. ROLLBACK (whole transaction) where ROLLBACK TO (savepoint) was meant

**The problem:** The model wants to undo one failed statement (a duplicate insert) but issues a bare
`ROLLBACK`, throwing away the entire transaction's earlier work. `ROLLBACK TO` a savepoint discards
only what came after it: "changes earlier than the savepoint are kept"
([PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)).

```sql
-- WRONG — a bare ROLLBACK discards the Alice debit too
START TRANSACTION;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
INSERT INTO ledger(...) VALUES (...);   -- fails (duplicate key)
ROLLBACK;   -- throws away EVERYTHING, including the valid debit

-- RIGHT — undo only the failed step, keep the rest
START TRANSACTION;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
SAVEPOINT before_ledger;
INSERT INTO ledger(...) VALUES (...);   -- fails
ROLLBACK TO before_ledger;              -- only the insert is undone
INSERT INTO ledger(...) VALUES (...);   -- retry with corrected data
COMMIT;
```

*Source: [PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html). Depth: this skill, §3.*

---

## 5. Relying on the default isolation level being the same across engines

**The problem:** The model writes concurrency-sensitive code and assumes a fixed default isolation
level. Defaults differ: PostgreSQL defaults to `READ COMMITTED`
([PG](https://www.postgresql.org/docs/current/transaction-iso.html)), MySQL/InnoDB to
`REPEATABLE READ`, SQLite to `SERIALIZABLE` ([SQLite](https://www.sqlite.org/isolation.html)). The
same code therefore sees different anomalies on each engine.

```sql
-- WRONG (assumption) — "the default protects me from re-read changes" is engine-specific
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;   -- behavior on re-read depends on the engine default
-- ...

-- RIGHT — state the level you actually need; don't depend on the default
START TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;
-- ...
COMMIT;
```

*Source: [PostgreSQL — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html); [SQLite — Isolation](https://www.sqlite.org/isolation.html). Depth: this skill, §4, §8; which level prevents which anomaly is owned by the MVCC plugin (`mvcc-isolation-levels-and-anomalies`).*

---

## 6. Treating READ UNCOMMITTED as a portable performance knob

**The problem:** The model reaches for `READ UNCOMMITTED` to "go faster," assuming it behaves the same
everywhere. PostgreSQL silently treats it as `READ COMMITTED`: "In PostgreSQL `READ UNCOMMITTED` is
treated as `READ COMMITTED`" ([PG — SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html)).
So the requested level is not what runs, and the dirty-read behavior the model expected never happens.

```sql
-- WRONG — assumes dirty reads are enabled; on PostgreSQL this silently runs as READ COMMITTED
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- RIGHT — pick a level for the anomaly you actually need to avoid (route the decision to MVCC)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

*Source: [PostgreSQL — SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html). Depth: this skill, §4, §8; choosing a level by anomaly is owned by `mvcc-choosing-isolation`.*

---

## 7. Check-then-act treated as safe just because it's in a transaction

**The problem:** The model wraps a read-decide-write sequence (read seat as free, then book it) in a
transaction and assumes atomicity alone makes it concurrency-safe. Atomicity is necessary but not
sufficient — at the default isolation level a concurrent transaction can book the same seat between
the read and the write. Wrapping is this skill's job; the concurrency correctness is **not**.

```sql
-- WRONG — atomic, but at READ COMMITTED two sessions can both see the seat free and both book it
START TRANSACTION;
SELECT booked FROM seats WHERE id = 7;       -- reads: free
UPDATE seats SET booked = true WHERE id = 7; -- both concurrent txns do this
COMMIT;

-- RIGHT (the SQL surface) — lock the row you checked, or raise the isolation level...
START TRANSACTION;
SELECT booked FROM seats WHERE id = 7 FOR UPDATE;  -- lock the row under examination
UPDATE seats SET booked = true WHERE id = 7;
COMMIT;
-- ...but WHICH level/lock is correct for which anomaly is owned by the MVCC plugin.
```

*Source: [PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html). Depth: this skill, §1; the concurrency theory (lost update, write skew, level choice) is owned by `mvcc-isolation-levels-and-anomalies` / `mvcc-write-skew-and-conflict-materialization` / `mvcc-choosing-isolation`.*
