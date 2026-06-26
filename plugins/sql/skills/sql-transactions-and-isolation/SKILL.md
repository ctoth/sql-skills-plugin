---
name: sql-transactions-and-isolation
description: Guides the SQL-statement surface of transactions — wrap any multi-statement invariant or check-then-act sequence in `START TRANSACTION`/`COMMIT`/`ROLLBACK` so it is atomic (all-or-nothing), use `SAVEPOINT`/`ROLLBACK TO` for partial rollback, set the isolation level with `SET TRANSACTION ISOLATION LEVEL`, and know the four standard level names (READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE), autocommit, READ ONLY/DEFERRABLE, and that DDL is not transactional everywhere (MySQL implicitly commits on DDL; PostgreSQL does not). Names the standard anomaly catalog (dirty read, non-repeatable read, phantom read, serialization anomaly) in one paragraph and routes ALL isolation theory — which level permits which anomaly, snapshot isolation, serializability, write skew — to the MVCC plugin. Auto-invokes when writing or editing `BEGIN`/`START TRANSACTION`/`COMMIT`/`ROLLBACK`/`SAVEPOINT`, `SET TRANSACTION`, a multi-statement write sequence, a debit/credit or check-then-act flow, or on "should this be in a transaction" / "which isolation level" / "why did half my migration apply" requests.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL Transactions and Isolation

> "The essential point of a transaction is that it bundles multiple steps into a single, all-or-nothing operation. ... if some failure occurs that prevents the transaction from completing, then none of the steps affect the database at all."
> — [PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)

> "It would certainly not do for a system failure to result in Bob receiving $100.00 that was not debited from Alice. Nor would Alice long remain a happy customer if she was debited without Bob being credited."
> — [PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)

A transaction is the unit of *atomicity*: a group of statements that must all happen or none happen. This skill owns the **SQL surface** of transactions — the statements you write (`START TRANSACTION`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`), how to set the isolation level, and the cross-engine caveats. It assumes the set semantics and three-valued logic from the policy root `sql-relational-and-null-discipline`. It deliberately does **not** teach isolation *theory* — which anomaly each level permits, how snapshots work, what serializability costs. That is a separate domain with its own plugin; see the routing callout in §4.

---

## 1. When to Open a Transaction — Multi-Statement Invariants and Check-Then-Act (the centerpiece)

Open a transaction whenever **two or more statements must succeed or fail together**, or whenever you **read a value, decide on it, then write based on that decision** ("check-then-act"). The canonical case is a money transfer: the debit and the credit are a single business fact, so they must be a single atomic unit.

In autocommit mode (§5) each statement commits on its own. So this:

```sql
-- WRONG — two independent auto-committed statements.
-- A crash, error, or disconnect between them leaves Alice debited and Bob never credited.
-- The $100 has simply vanished; there is no intermediate state to recover.
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Bob';
```

is not a transfer — it is two unrelated writes that happen to be adjacent. Wrap them so they are atomic:

```sql
-- RIGHT — one all-or-nothing unit. Either both rows change or neither does.
START TRANSACTION;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Bob';
COMMIT;
```

The guarantee is exact: a transaction "bundles multiple steps into a single, all-or-nothing operation ... if some failure occurs that prevents the transaction from completing, then none of the steps affect the database at all" ([PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)). Check-then-act (read a seat as free, then book it; read a balance, then debit it) needs the same wrapping *and* often an isolation level above the default or explicit row locking — but the **concurrency** correctness of check-then-act is exactly where this skill hands off to MVCC (§4).

---

## 2. Transaction Control Statements and the Atomicity Guarantee

The control statements are small and standard:

- `START TRANSACTION` (SQL-standard spelling) or `BEGIN` (PostgreSQL/SQLite/MySQL accept `BEGIN`) opens a transaction.
- `COMMIT` makes every change in the transaction durable and visible as one unit.
- `ROLLBACK` discards the entire transaction: "we can issue the command `ROLLBACK` instead of `COMMIT`, and all our updates so far will be canceled" ([PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)).

The whole point is the all-or-nothing property: a transaction "is said to be _atomic_: from the point of view of other transactions, it either happens completely or not at all" ([PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)).

```sql
-- ROLLBACK on a detected violation — the canceled debit leaves no trace.
START TRANSACTION;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
-- application checks: Alice's balance went negative -> abort
ROLLBACK;   -- the UPDATE above is undone; the database is untouched
```

Always pair an opened transaction with exactly one terminator. A transaction left open holds resources and (depending on isolation) blocks others; on most drivers a dropped connection rolls it back, but relying on that is a bug.

---

## 3. SAVEPOINT and ROLLBACK TO — Partial Rollback

A `SAVEPOINT` is a named marker inside an open transaction. `ROLLBACK TO savepoint` undoes everything after the marker while keeping the work before it and **keeping the transaction open**: "Savepoints allow you to selectively discard parts of the transaction, while committing the rest. ... All the transaction's database changes between defining the savepoint and rolling back to it are discarded, but changes earlier than the savepoint are kept" ([PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)).

```sql
-- Debit Alice, tentatively credit Bob, discover it should be Wally, fix it — without losing the debit.
START TRANSACTION;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
SAVEPOINT my_savepoint;
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Bob';
ROLLBACK TO my_savepoint;   -- undo only the Bob credit; the Alice debit stays
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Wally';
COMMIT;
```

`RELEASE SAVEPOINT` forgets a savepoint you no longer need. SQLite uses the same `SAVEPOINT`/`RELEASE`/`ROLLBACK TO` family for nested transactions ([SQLite — Transactions](https://www.sqlite.org/lang_transaction.html)). Savepoints are also how robust application code recovers from a single failed statement (e.g. a duplicate-key insert) without throwing away the whole transaction.

---

## 4. Isolation-Level Names and `SET TRANSACTION`

The SQL standard defines **four** isolation levels, set with `SET TRANSACTION ISOLATION LEVEL { SERIALIZABLE | REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED }` ([PostgreSQL — SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html)). Set it on the transaction you are about to run:

```sql
-- Per-transaction: raise this transaction to the strongest level.
START TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ... statements ...
COMMIT;

-- Or set the mode before the work begins.
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

The four levels exist to control **anomalies** — interactions between concurrent transactions. The standard names four phenomena: a **dirty read** ("a transaction reads data written by a concurrent uncommitted transaction"), a **non-repeatable read** (re-reading data and finding it changed by a committed transaction), a **phantom read** (re-running a query and finding the set of matching rows changed), and a **serialization anomaly** (a committed group of transactions producing a result "inconsistent with all possible orderings of running those transactions one at a time") ([PostgreSQL — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)). That one paragraph is the entire anomaly catalog this skill states.

> **⚠ MVCC boundary — route all isolation theory out.** *Which* level permits *which* phenomenon, why "forbidding the three phenomena does not make a level truly serializable," how snapshot isolation actually works, write skew, and how to choose a level for a workload are **not** in this skill. They belong to the MVCC plugin (`~/code/mvcc-skills-plugin`):
> - `mvcc-isolation-levels-and-anomalies` — the per-level / per-anomaly matrix and the phenomena in depth.
> - `mvcc-snapshot-isolation` — how MVCC snapshots implement REPEATABLE READ.
> - `mvcc-serializable-ssi` — true serializability, SSI, predicate locks, serialization failures.
> - `mvcc-write-skew-and-conflict-materialization` — the write-skew anomaly and fixes.
> - `mvcc-choosing-isolation` — picking a level by workload and anomaly tolerance.
>
> This skill teaches the **names and the `SET` syntax**; that plugin teaches **what the levels mean**.

---

## 5. Autocommit

Outside an explicit transaction, every statement is its own transaction. SQLite states it plainly: "Any command that accesses the database ... will automatically start a transaction if one is not already in effect. Automatically started transactions are committed when the last SQL statement finishes" ([SQLite — Transactions](https://www.sqlite.org/lang_transaction.html)). This is why the two-`UPDATE` transfer in §1 loses money: in autocommit each `UPDATE` commits independently, so there is no unit that contains both.

The practical rule: if a unit of work spans more than one statement and must be all-or-nothing, you must turn autocommit off or open an explicit transaction. (Client drivers and ORMs vary in whether they autocommit by default; verify your driver rather than assuming.)

---

## 6. DDL in a Transaction Is Not Portable

Whether you can wrap schema changes (`CREATE`/`ALTER`/`DROP TABLE`, `CREATE INDEX`, …) in a transaction and roll them back **differs sharply by engine** — this is one of the most damaging cross-engine surprises.

- **PostgreSQL** and **SQLite** have *transactional DDL*: a `CREATE`/`ALTER` inside `BEGIN … COMMIT` participates in the transaction and rolls back cleanly on `ROLLBACK`.
- **MySQL/InnoDB** does **not**. A DDL statement causes an implicit commit: such statements "implicitly end any transaction active in the current session, as if you had done a `COMMIT` before executing the statement," and "most of these statements also cause an implicit commit after executing" ([MySQL — Statements That Cause an Implicit Commit](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)). DDL cannot be rolled back. Even the `TEMPORARY`-table exception is a trap: "no implicit commit occurs, neither can the statement be rolled back, which means that the use of such statements causes transactional atomicity to be violated" ([MySQL](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)).

```sql
-- On PostgreSQL/SQLite this multi-step migration is atomic: a failure rolls ALL of it back.
-- On MySQL each DDL implicitly commits, so a failure mid-way leaves the schema HALF-APPLIED.
START TRANSACTION;
ALTER TABLE orders ADD COLUMN status TEXT;
CREATE INDEX idx_orders_status ON orders(status);   -- if this fails on MySQL, the ALTER already committed
COMMIT;
```

Consequence: migration tooling that assumes a failed migration rolls back is only safe on transactional-DDL engines. On MySQL, plan migrations to be idempotent / forward-only. The full default-level and DDL-transactionality matrix lives in `sql-standard-vs-dialect-map`.

---

## 7. Read-Only and Deferrable Transactions

A transaction can declare it will not write, and PostgreSQL can use that to relax overhead. The access mode "determines whether the transaction is read/write or read-only. Read/write is the default" ([PostgreSQL — SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html)). Declaring `READ ONLY` documents intent and lets the engine reject accidental writes.

```sql
-- A consistent, low-overhead read snapshot for a report.
START TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SELECT ...;
COMMIT;
```

`DEFERRABLE` is a PostgreSQL optimization for long read-only reporting: it "has no effect unless the transaction is also `SERIALIZABLE` and `READ ONLY`. When all three of these properties are selected ... the transaction may block when first acquiring its snapshot, after which it is able to run without the normal overhead of a `SERIALIZABLE` transaction and without any risk of contributing to or being canceled by a serialization failure" ([PostgreSQL — SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html)).

---

## 8. Portability Snapshot

The four **level names** are standard and accepted everywhere. What differs is the **default level** each engine gives an unconfigured transaction — so the same application code runs at a different isolation level on each engine unless you set it explicitly.

| Engine | Default isolation level | DDL inside a transaction |
|---|---|---|
| PostgreSQL | `READ COMMITTED` ([PG](https://www.postgresql.org/docs/current/transaction-iso.html)) | transactional — rolls back |
| MySQL / InnoDB | `REPEATABLE READ` | **implicit commit — cannot roll back** ([MySQL](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)) |
| SQLite | `SERIALIZABLE` ([SQLite](https://www.sqlite.org/isolation.html)) | transactional — rolls back |

Two more dialect notes: PostgreSQL treats `READ UNCOMMITTED` as `READ COMMITTED` (it never returns dirty reads) ([PG — SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html)); and SQLite's `BEGIN` takes a locking modifier — `BEGIN DEFERRED` (default, lazy lock), `BEGIN IMMEDIATE` (acquire a write lock now), `BEGIN EXCLUSIVE` — that has no standard equivalent ([SQLite — Transactions](https://www.sqlite.org/lang_transaction.html)). For the complete defaults-and-anomalies matrix, route to **`sql-standard-vs-dialect-map`**.

---

## 9. Who Suffers When Atomicity Is Missing

Skipping a transaction never errors at write time — the failure surfaces later, as corrupt data or a wedged deploy, and someone downstream pays:

- The **customer** whose account was debited but whose payee was never credited (§1), because the two `UPDATE`s ran as separate autocommit statements and the process crashed between them. There is no error in the log — just $100 that left one account and arrived nowhere. The transaction Alice's bank didn't open is the money she lost.
- The **on-call engineer** woken at 3 a.m. because a deploy's migration failed halfway on **MySQL** and left the schema half-applied (§6) — the new column exists but its index doesn't, the app is throwing, and `ROLLBACK` does nothing because the DDL already implicitly committed. The same migration on PostgreSQL would have rolled back as one unit.
- The **next maintainer** who finds a transaction opened with no matching `COMMIT`/`ROLLBACK` on one code path (§2), silently holding locks and bloating the database until connections pile up.

A transaction is empathy in code: the `START TRANSACTION … COMMIT` that turns two risky writes into one safe fact is a gift to whoever operates this system under load.

---

## 10. Routing to Related Skills

This skill is a **technique** built on the foundation and routes theory outward:

- `sql-relational-and-null-discipline` — the policy root (set semantics, three-valued logic) this skill assumes.
- `sql-merge-and-upsert` — concurrency-safe insert-or-update; the check-then-act pattern (§1) in its `MERGE`/`INSERT … ON CONFLICT` form.
- `sql-privileges-and-access-control` — who is allowed to run these statements; least-privilege app roles.
- `sql-standard-vs-dialect-map` — the full default-isolation and DDL-transactionality matrix, `BEGIN` dialect modifiers, and `START TRANSACTION` vs `BEGIN` spellings.
- **MVCC plugin** (`~/code/mvcc-skills-plugin`) — all isolation *theory*: `mvcc-isolation-levels-and-anomalies`, `mvcc-snapshot-isolation`, `mvcc-serializable-ssi`, `mvcc-write-skew-and-conflict-materialization`, `mvcc-choosing-isolation`. If the question is "which level permits which anomaly" or "is this serializable," it belongs there, not here.

---

## 11. Reference Files

High-frequency transaction/isolation anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
