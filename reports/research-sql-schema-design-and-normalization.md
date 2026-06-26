# Research — sql-schema-design-and-normalization

Research backing the skill `sql-schema-design-and-normalization`. Each entry: source URL, section, verbatim quote, and why it matters for the skill. Accessed 2026-06-26.

---

## 1. Database normalization — what it is, objectives, and the three anomalies

**URL:** https://en.wikipedia.org/wiki/Database_normalization

**Section: lead / "What is normalization"**
> "Database normalization is the process of structuring a relational database in accordance with a series of normal forms to reduce data redundancy and improve data integrity."

**Section: Objectives**
> "To free the collection of relations from undesirable insertion, update and deletion dependencies."

**Section: Anomalies — verbatim, the core motivation for the skill's framing**
> Insertion anomaly: "There are circumstances in which certain facts cannot be recorded at all."

> Update anomaly: "The same information can be expressed on multiple rows; therefore updates may result in logical inconsistencies."

> Deletion anomaly: "The deletion of data representing certain facts necessitates the deletion of data representing completely different facts."

**Why it matters:** These three anomalies are the *reason* normalization exists. The skill's opening framing ("why normalize") is built directly on these three verbatim definitions. They give the WRONG/RIGHT walkthrough its teeth — a denormalized table is not "ugly," it is provably subject to insert/update/delete anomalies.

---

## 2. Normal form definitions 1NF → BCNF

**URL:** https://en.wikipedia.org/wiki/Database_normalization (Normal forms section)

> 1NF: "In the first normal form each field contains a single value. A field may not contain a set of values or a nested record."

> 2NF: "Every non-prime attribute has a full functional dependency on each candidate key (attributes depend on the whole of every key)."

> 3NF: "Every non-trivial functional dependency either begins with a superkey or ends with a prime attribute (attributes depend only on candidate keys)."

> BCNF: "Every non-trivial functional dependency begins with a superkey (a stricter form of 3NF)."

**Why it matters:** The 1NF→BCNF ladder is the spine of the worked example. The compact one-liners ("depend on the whole key" = 2NF; "depend only on candidate keys" / "the key, the whole key, and nothing but the key" = 3NF) drive the progression. BCNF as "every determinant is a superkey" closes the ladder.

---

## 3. First normal form — atomicity and repeating groups (the Jaywalking root)

**URL:** https://en.wikipedia.org/wiki/First_normal_form

> "A relation (or a table, in SQL) can be said to be in first normal form if each field is atomic, containing a single value rather than a set of values or a nested table."

> "Values in the domains on which each relation is defined are required to be atomic with respect to the DBMS."

> "Normalization to 1NF involves eliminating nested relations by breaking them up into separate relations associated with each other using foreign keys."

> "First normal form therefore requires all attribute domains to be simple domains, such that the data in each field is atomic and no relation has relation-valued attributes."

**Why it matters:** This is the theoretical basis for why a comma-separated list in a column (Jaywalking) is *not a stylistic choice but a 1NF violation*. The fix — "breaking them up into separate relations associated with each other using foreign keys" — is verbatim the junction-table fix. This is the skill's centerpiece antipattern.

---

## 4. Boyce–Codd Normal Form and functional dependency

**URL:** https://en.wikipedia.org/wiki/Boyce%E2%80%93Codd_normal_form

> "BCNF (or 3.5NF) is a normal form used in database normalization. It is a slightly stricter version of the third normal form (3NF)."

> "A relational schema R is in Boyce–Codd normal form if and only if for every one of its functional dependencies X → Y, at least one of the following conditions hold: X → Y is a trivial functional dependency (Y ⊆ X), X is a superkey for schema R."

> "Only S1, S2, S3 and S4 are candidate keys (that is, minimal superkeys for that relation)."

> "If a relational schema is in BCNF, then it is automatically also in 3NF because BCNF is a stricter form of 3NF. While all BCNF relations satisfy the conditions for 3NF, not all 3NF relations satisfy the stricter requirements of BCNF."

> "By using BCNF, a database will remove all redundancies based on functional dependencies."

**Why it matters:** Gives the precise, citable definition of BCNF (every non-trivial determinant is a superkey) and the candidate-key-as-minimal-superkey definition. Lets the skill state BCNF rigorously rather than hand-wave "3.5NF."

---

## 5. Codd 1970 — relations, normalization motivation

**URL:** https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf

The PDF is the original Codd paper (image/scan-heavy in places). Corroborated via the relational model framing already used in the sibling skill `sql-relational-and-null-discipline`:

> §1.3: a relation is a set of n-tuples; "the ordering of rows is immaterial" and "all rows are distinct."

Codd introduced normalization in this paper as a way to derive a collection of relations free of the undesirable dependencies later named the insert/update/delete anomalies (corroborated by the Wikipedia normalization article, §1 above).

**Why it matters:** Establishes the lineage — normalization is not a 2000s web-framework convention, it is the founding discipline of the relational model. Cited for provenance/authority.

---

## 6. Karwin, *SQL Antipatterns* — the structural antipatterns

**URL:** https://pragprog.com/titles/bksqla/sql-antipatterns/ (and Volume 1 https://pragprog.com/titles/bksap1/sql-antipatterns-volume-1/)

The pragprog page lists chapter titles; detailed problem/fix corroborated via the cliff-notes summary at http://wtfdev.blogspot.com/2011/05/sql-antipatterns-cliff-notes-1-of-3.html and search summaries.

**Logical-design antipatterns (the ones this skill covers):**

> **Jaywalking** — "Storing comma-separated lists in database columns." Problem: "Comma Separated Values stored in a column" creates issues with decoding, updates, aggregation functions, and referential integrity enforcement. **Solution: Create an Intersection Table** (junction table) to properly normalize the data.

> **Naive Trees** — "Using adjacency list model for hierarchical data." Problem: recursive relationships in a single table require multiple joins or recursive queries to reconstruct hierarchies. **Solutions (four):** Adjacency List, Path Enumeration, Nested Sets, **Closure Table**.

> **Entity-Attribute-Value (EAV)** — "Using attribute-value pairs instead of proper columns." Problem: generic attribute tables "cannot enforce integrity, query attributes efficiently, maintain proper SQL types, or pivot columns to rows easily." **Solutions:** Single Table Inheritance, Concrete Table Inheritance, Class Table Inheritance, or Semistructured Data (XML/JSON column).

> **Polymorphic Associations** — "Foreign keys referencing multiple table types." A single FK column referencing rows in multiple unrelated tables, disambiguated by a type column, "cannot carry a referential integrity constraint in standard SQL, producing unmaintainable join logic and silent integrity failures." **Solutions:** Reverse the Reference (one child→parent FK per parent type), or use Intersection Tables, or a common supertable.

> **ID Required / Keyless Entry** — relevant to the keys section: keyless tables undermine referential integrity; the fix is to apply constraints (foreign key or check). Touches natural-vs-surrogate-key discussion.

**Why it matters:** These four antipatterns are the back half of the skill, each as a WRONG/RIGHT pair. Jaywalking is the 1NF-violation centerpiece. EAV routes to `sql-json` (semistructured data as the lesser evil). Naive Trees routes subtree *queries* to `sql-cte-and-recursion`. Polymorphic Associations routes the FK-constraint fix to `sql-constraints-and-integrity`.

---

## Synthesis — what the skill must nail

1. **Why normalize** = remove the three anomalies (insert/update/delete) — verbatim framing (§1).
2. **1NF→BCNF ladder** with a worked WRONG (denormalized, anomaly-ridden) → RIGHT (normalized) example (§2).
3. **1NF = atomic columns, no repeating groups** → Jaywalking comma-list violation → junction-table fix (§3, §6). CENTERPIECE.
4. **EAV** = key-value rows instead of columns; destroys constraints/types/queries; proper columns, or JSON as lesser evil → route to `sql-json` (§6).
5. **Naive Trees** = adjacency-only can't query subtrees → nested sets / closure table; route recursive queries to `sql-cte-and-recursion` (§6).
6. **Polymorphic Associations** = promiscuous FK with no real constraint → proper per-parent FKs or supertable → route to `sql-constraints-and-integrity` (§6).
7. **M:N needs a junction table** (intersection table) (§3, §6).
8. **Natural vs surrogate keys** — deliberate choice; candidate key = minimal superkey (§4, §6).
9. **Denormalize only deliberately** with a stated reason (redundancy is an anomaly source, §1).
10. **Portability:** pure design theory is 100% engine-independent; only the constraint *syntax* that enforces it varies → route to `sql-constraints-and-integrity`.
