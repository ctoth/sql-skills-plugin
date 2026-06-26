# Research: sql-transactions-and-isolation

Skill #23 in the SQL skills plugin. Owns the **SQL surface** of transactions only — statements,
savepoints, isolation-level *names* and how to set them. HARD boundary: all anomaly/isolation
**theory** (which level permits which phenomenon, serializability, snapshot mechanics, write skew,
SSI) routes to `~/code/mvcc-skills-plugin` (`mvcc-isolation-levels-and-anomalies` and siblings).

Sibling MVCC skill names confirmed by Glob on `C:\Users\Q\code\mvcc-skills-plugin\plugins\mvcc\skills\`:
`mvcc-isolation-levels-and-anomalies`, `mvcc-snapshot-isolation`, `mvcc-serializable-ssi`,
`mvcc-choosing-isolation`, plus `mvcc-foundations`, `mvcc-write-skew-and-conflict-materialization`, etc.

Accessed: 2026-06-26.

---

## Source 1 — PostgreSQL: A Brief Tour / Transactions (tutorial-transactions.html)

URL: https://www.postgresql.org/docs/current/tutorial-transactions.html

### §Transactions — atomicity / all-or-nothing
> "The essential point of a transaction is that it bundles multiple steps into a single,
> all-or-nothing operation. The intermediate states between the steps are not visible to other
> concurrent transactions, and if some failure occurs that prevents the transaction from
> completing, then none of the steps affect the database at all."

> "A transaction is said to be _atomic_: from the point of view of other transactions, it either
> happens completely or not at all."

**Why it matters:** This is the centerpiece guarantee. It is the reason multi-statement invariants
(debit + credit) must be wrapped — without the transaction a crash between the two writes leaves the
database in an intermediate state that "affects the database." Direct support for the WRONG/RIGHT
centerpiece.

### §Transactions — the bank transfer motivating example
> "It would certainly not do for a system failure to result in Bob receiving $100.00 that was not
> debited from Alice. Nor would Alice long remain a happy customer if she was debited without Bob
> being credited."

**Why it matters:** This IS the "Who suffers" persona (Alice debited, Bob never credited). Use it
verbatim as the motivating multi-statement invariant.

### §Transactions — BEGIN/COMMIT/ROLLBACK
> "In PostgreSQL, a transaction is set up by surrounding the SQL commands of the transaction with
> `BEGIN` and `COMMIT` commands."

```
BEGIN;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
-- etc etc
COMMIT;
```

> "If, partway through the transaction, we decide we do not want to commit ... we can issue the
> command `ROLLBACK` instead of `COMMIT`, and all our updates so far will be canceled."

**Why it matters:** Canonical statement set. Note PG uses `BEGIN`; the standard/portable spelling is
`START TRANSACTION` (per the plan's draft description). Both belong in the skill.

### §Transactions — SAVEPOINT / ROLLBACK TO
> "It's possible to control the statements in a transaction in a more granular fashion through the
> use of _savepoints_. Savepoints allow you to selectively discard parts of the transaction, while
> committing the rest. After defining a savepoint with `SAVEPOINT`, you can if needed roll back to
> the savepoint with `ROLLBACK TO`. All the transaction's database changes between defining the
> savepoint and rolling back to it are discarded, but changes earlier than the savepoint are kept."

Example (debit Alice, credit wrong person, roll back to savepoint, credit Wally):
```
BEGIN;
UPDATE accounts SET balance = balance - 100.00 WHERE name = 'Alice';
SAVEPOINT my_savepoint;
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Bob';
-- oops ... forget that and use Wally's account
ROLLBACK TO my_savepoint;
UPDATE accounts SET balance = balance + 100.00 WHERE name = 'Wally';
COMMIT;
```

**Why it matters:** Exact source for the SAVEPOINT section, with a self-contained worked example.

---

## Source 2 — PostgreSQL: SET TRANSACTION (sql-set-transaction.html)

URL: https://www.postgresql.org/docs/current/sql-set-transaction.html

### Synopsis — isolation level names
> "ISOLATION LEVEL { SERIALIZABLE | REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED }"

**Why it matters:** The four standard level *names* — exactly what this skill owns. The skill states
the names and how to set them; it does NOT explain which anomalies each permits (that routes to MVCC).

### Access mode — READ WRITE / READ ONLY
> "The transaction access mode determines whether the transaction is read/write or read-only.
> Read/write is the default."

### DEFERRABLE
> "The `DEFERRABLE` transaction property has no effect unless the transaction is also `SERIALIZABLE`
> and `READ ONLY`. When all three of these properties are selected for a transaction, the transaction
> may block when first acquiring its snapshot, after which it is able to run without the normal
> overhead of a `SERIALIZABLE` transaction and without any risk of contributing to or being canceled
> by a serialization failure."

### Default
> "In PostgreSQL the default is ordinarily `READ COMMITTED`."

**Why it matters:** Covers (d) level names + SET syntax, (g) read-only/deferrable, and one cell of
the portability defaults table (PG default READ COMMITTED).

---

## Source 3 — PostgreSQL: Transaction Isolation (transaction-iso.html)

URL: https://www.postgresql.org/docs/current/transaction-iso.html

### The four phenomena (names only — DO NOT deep-dive; that's MVCC's job)
> Dirty read: "A transaction reads data written by a concurrent uncommitted transaction."
> Nonrepeatable read: "A transaction re-reads data it has previously read and finds that data has
> been modified by another transaction (that committed since the initial read)."
> Phantom read: "A transaction re-executes a query returning a set of rows that satisfy a search
> condition and finds that the set of rows satisfying the condition has changed due to another
> recently-committed transaction."
> Serialization anomaly: "The result of successfully committing a group of transactions is
> inconsistent with all possible orderings of running those transactions one at a time."

### Levels defined in terms of phenomena
> "The SQL standard defines four levels of transaction isolation. The most strict is Serializable,
> which is defined by the standard in a paragraph which says that any concurrent execution of a set
> of Serializable transactions is guaranteed to produce the same effect as running them one at a
> time in some order. The other three levels are defined in terms of phenomena, resulting from
> interaction between concurrent transactions, which must not occur at each level."

### Default
> "_Read Committed_ is the default isolation level in PostgreSQL."

**Why it matters:** Source for the ONE-paragraph anomaly summary that NAMES dirty/non-repeatable/
phantom read (and serialization anomaly) — then ROUTES which-level-permits-what to MVCC. This is the
hard-boundary paragraph: name the phenomena, do not build the per-level matrix here.

---

## Source 4 — SQLite: BEGIN TRANSACTION (lang_transaction.html)

URL: https://www.sqlite.org/lang_transaction.html

### Autocommit (default)
> "No reads or writes occur except within a transaction. Any command that accesses the database
> (basically, any SQL command, except a few PRAGMA statements) will automatically start a
> transaction if one is not already in effect. Automatically started transactions are committed when
> the last SQL statement finishes."

**Why it matters:** Source for (e) autocommit — every bare statement is its own transaction. This is
*why* two separate autocommit UPDATEs are not atomic (the WRONG centerpiece).

### BEGIN [DEFERRED | IMMEDIATE | EXCLUSIVE]
> DEFERRED: "the transaction does not actually start until the database is first accessed.
> Internally, the BEGIN DEFERRED statement merely sets a flag on the database connection that turns
> off the automatic commit ..."
> IMMEDIATE: "causes the database connection to start a new write immediately ... The BEGIN IMMEDIATE
> might fail with SQLITE_BUSY if another write transaction is already active on another database
> connection."
> EXCLUSIVE: "similar to IMMEDIATE in that a write transaction is started immediately. EXCLUSIVE and
> IMMEDIATE are the same in WAL mode, but in other journaling modes, EXCLUSIVE prevents other
> database connections from reading the database while the transaction is underway."

**Why it matters:** SQLite's BEGIN modes — a dialect note in the transaction-control section.

### COMMIT / ROLLBACK / SAVEPOINT
> "For nested transactions, use the SAVEPOINT and RELEASE commands. The "TO SAVEPOINT name" clause of
> the ROLLBACK command ... is only applicable to SAVEPOINT transactions."

---

## Source 5 — SQLite: Isolation In SQLite (isolation.html)

URL: https://www.sqlite.org/isolation.html

> "Except in the case of shared cache database connections with PRAGMA read_uncommitted turned on,
> all transactions in SQLite show "serializable" isolation."
> "Transactions in SQLite are SERIALIZABLE." (Summary)

**Why it matters:** Source for SQLite's default = SERIALIZABLE in the portability defaults row. SQLite
gives the strongest level by default; PG gives READ COMMITTED; MySQL/InnoDB gives REPEATABLE READ.

---

## Source 6 — MySQL: Statements That Cause an Implicit Commit (implicit-commit.html)

URL: https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html

> "The statements listed in this section (and any synonyms for them) implicitly end any transaction
> active in the current session, as if you had done a `COMMIT` before executing the statement."
> "Most of these statements also cause an implicit commit after executing. The intent is to handle
> each such statement in its own special transaction."

DDL list includes `ALTER TABLE`, `CREATE TABLE`, `CREATE INDEX`, `DROP TABLE`, `RENAME TABLE`,
`TRUNCATE TABLE`, etc.

On TEMPORARY tables (the deeper trap):
> "However, although no implicit commit occurs, neither can the statement be rolled back, which means
> that the use of such statements causes transactional atomicity to be violated."

**Why it matters:** Source for (f) the DDL-in-transaction caveat. On MySQL a DDL statement inside a
transaction implicitly commits everything before it and cannot be rolled back — so a failed multi-step
migration leaves the schema half-applied. PostgreSQL, by contrast, has transactional DDL (a CREATE/
ALTER inside BEGIN…COMMIT rolls back cleanly). This is the second "Who suffers" persona.

---

## Boundary checklist (what stays here vs routes to MVCC)

STAYS (SQL surface):
- when to open a transaction (multi-statement invariant, check-then-act)
- START TRANSACTION / BEGIN / COMMIT / ROLLBACK and the all-or-nothing guarantee
- SAVEPOINT / ROLLBACK TO / RELEASE
- the four isolation-level *names* and `SET TRANSACTION ISOLATION LEVEL`
- autocommit
- READ ONLY / READ WRITE / DEFERRABLE
- DDL-in-transaction caveat (PG transactional DDL vs MySQL implicit commit)
- a ≤1-paragraph anomaly summary that NAMES the phenomena

ROUTES OUT (theory → mvcc-skills-plugin):
- which anomaly each level permits → `mvcc-isolation-levels-and-anomalies`
- snapshot isolation mechanics → `mvcc-snapshot-isolation`
- "forbidding 3 phenomena ≠ serializable", SSI, predicate locks → `mvcc-serializable-ssi`
- write skew → `mvcc-write-skew-and-conflict-materialization`
- choosing a level by workload/anomaly → `mvcc-choosing-isolation`

ROUTES OUT (SQL plugin):
- upsert/merge concurrency → `sql-merge-and-upsert`
- the full default-level differences table → `sql-standard-vs-dialect-map`
- transaction-scoped privileges → `sql-privileges-and-access-control`
