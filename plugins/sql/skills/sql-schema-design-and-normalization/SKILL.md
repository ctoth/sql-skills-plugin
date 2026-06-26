---
name: sql-schema-design-and-normalization
description: Guides portable relational schema design — normalize to remove the insertion, update, and deletion anomalies (the 1NF→BCNF ladder), choose natural vs surrogate keys deliberately, model many-to-many with a junction table, and denormalize only with a stated reason. Bans the structural antipatterns that no constraint can later rescue — comma-separated value lists ("Jaywalking", a First Normal Form violation), Entity-Attribute-Value (EAV) key-value rows instead of columns, adjacency-only "Naive Trees" that cannot query a subtree, and polymorphic/promiscuous foreign keys that carry no referential integrity. Auto-invokes when designing tables or schemas, writing CREATE TABLE for a new model, modeling a hierarchy/tree or a many-to-many relationship, storing a list or "flexible attributes" in a column, choosing a primary key, or on "is this schema/data model good", "review my schema", or "how should I store X" requests. Pure design theory — engine-independent and fully portable.
allowed-tools: Read, Glob, Grep
compatibility: "Claude Code, Codex CLI, Gemini CLI"
---

# SQL Schema Design and Normalization

> "Database normalization is the process of structuring a relational database in accordance with a series of normal forms to reduce data redundancy and improve data integrity."
> — [Wikipedia — Database Normalization](https://en.wikipedia.org/wiki/Database_normalization)

> "First normal form ... each field is atomic, containing a single value rather than a set of values or a nested table."
> — [Wikipedia — First Normal Form](https://en.wikipedia.org/wiki/First_normal_form)

Schema design decides correctness before a single query runs. A normalized schema makes wrong data *impossible to represent*; a denormalized one makes it *inevitable* and asks every query to clean up after it. This is pure relational design theory — **engine-independent and 100% portable**; only the constraint *syntax* that enforces a design varies (route to `sql-constraints-and-integrity`). It builds on `sql-relational-and-null-discipline`: a table is a set of rows, and a design that admits duplicate or contradictory facts has already lost.

---

## 1. Why Normalize — the Three Anomalies (the framing)

Normalization has one objective: "to free the collection of relations from undesirable insertion, update and deletion dependencies" ([Wikipedia — Normalization](https://en.wikipedia.org/wiki/Database_normalization)). Redundancy is not merely wasteful — it makes three classes of bug *structurally possible*. Storing the same fact in many places means:

- **Update anomaly** — "The same information can be expressed on multiple rows; therefore updates may result in logical inconsistencies" ([Wikipedia — Normalization](https://en.wikipedia.org/wiki/Database_normalization)). Update one copy, miss another, and the database now holds two contradictory truths.
- **Insertion anomaly** — "There are circumstances in which certain facts cannot be recorded at all" ([Wikipedia — Normalization](https://en.wikipedia.org/wiki/Database_normalization)). You cannot add a product the warehouse hasn't stocked yet, because the only table couples product and stock.
- **Deletion anomaly** — "The deletion of data representing certain facts necessitates the deletion of data representing completely different facts" ([Wikipedia — Normalization](https://en.wikipedia.org/wiki/Database_normalization)). Delete the last order for a customer and you lose the customer's existence too.

Every WRONG example below is an instance of one of these three. When you justify a design, name the anomaly it prevents — that is the language of the discipline.

---

## 2. The 1NF→BCNF Ladder, with a Worked Example

Take a denormalized "orders" table that crams customer, product, and line data into one relation:

```sql
-- WRONG — one wide table; the same customer/product facts repeat on every row
CREATE TABLE orders (
  order_id     INT,
  customer_id  INT,
  customer_email TEXT,        -- repeats for every order the customer places
  product_ids  TEXT,          -- '17,42,88'  <- a list in one column
  product_name TEXT,          -- depends on product, not on the order
  category     TEXT,          -- depends on product_name, not on the key
  PRIMARY KEY (order_id)
);
```

This table is subject to all three anomalies: change `customer_email` and you must hunt every order row (update); you cannot record a product with no order (insertion); delete a customer's last order and the email vanishes (deletion). Climb the ladder:

- **1NF — each field is atomic.** "In the first normal form each field contains a single value. A field may not contain a set of values or a nested record" ([Wikipedia — Normalization](https://en.wikipedia.org/wiki/Database_normalization)). `product_ids = '17,42,88'` violates this outright (see §3). Fix: break repeating groups into rows, related "using foreign keys" ([Wikipedia — 1NF](https://en.wikipedia.org/wiki/First_normal_form)).
- **2NF — no partial-key dependency.** "Every non-prime attribute has a full functional dependency on each candidate key (attributes depend on the whole of every key)" ([Wikipedia — Normalization](https://en.wikipedia.org/wiki/Database_normalization)). On a `(order_id, product_id)` line table, `product_name` depends on `product_id` *alone* — a partial dependency. Move it to a `products` table.
- **3NF — no transitive dependency.** "Every non-trivial functional dependency either begins with a superkey or ends with a prime attribute (attributes depend only on candidate keys)" ([Wikipedia — Normalization](https://en.wikipedia.org/wiki/Database_normalization)). `category` depends on `product_name`, which depends on the key — a transitive chain. Move `category` to where it belongs (a `categories` or `products` table). The mnemonic: every non-key attribute depends on *the key, the whole key, and nothing but the key.*
- **BCNF — every determinant is a superkey.** "A relational schema R is in Boyce–Codd normal form if and only if for every one of its functional dependencies X → Y ... X is a superkey for schema R" ([Wikipedia — BCNF](https://en.wikipedia.org/wiki/Boyce%E2%80%93Codd_normal_form)). BCNF is "a slightly stricter version of the third normal form" — "By using BCNF, a database will remove all redundancies based on functional dependencies" ([Wikipedia — BCNF](https://en.wikipedia.org/wiki/Boyce%E2%80%93Codd_normal_form)).

```sql
-- RIGHT — one fact in one place; FKs relate them
CREATE TABLE customers  (customer_id INT PRIMARY KEY, email TEXT NOT NULL UNIQUE);
CREATE TABLE categories (category_id INT PRIMARY KEY, name  TEXT NOT NULL UNIQUE);
CREATE TABLE products (
  product_id  INT PRIMARY KEY,
  name        TEXT NOT NULL,
  category_id INT NOT NULL REFERENCES categories(category_id)  -- category moved out (3NF)
);
CREATE TABLE orders (
  order_id    INT PRIMARY KEY,
  customer_id INT NOT NULL REFERENCES customers(customer_id)   -- email moved out (no dupes)
);
CREATE TABLE order_lines (                                     -- the M:N line items (2NF)
  order_id   INT NOT NULL REFERENCES orders(order_id),
  product_id INT NOT NULL REFERENCES products(product_id),
  quantity   INT NOT NULL CHECK (quantity > 0),
  PRIMARY KEY (order_id, product_id)
);
```

Now `email` lives once, a product can exist with zero orders, and deleting an order never erases a customer. **A practical target is 3NF/BCNF for transactional (OLTP) schemas.** (FK/`CHECK`/`UNIQUE` syntax is owned by `sql-constraints-and-integrity`; it is shown here only to make the shape concrete.)

---

## 3. Jaywalking — the Comma-Separated List (centerpiece)

The single most common structural mistake an LLM makes: storing a *list* in one column. It is a direct 1NF violation — 1NF "requires all attribute domains to be simple domains, such that the data in each field is atomic and no relation has relation-valued attributes" ([Wikipedia — 1NF](https://en.wikipedia.org/wiki/First_normal_form)). Karwin names it **Jaywalking**: "Storing comma-separated lists in database columns" ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)).

```sql
-- WRONG — a list of foreign keys jammed into one text column
CREATE TABLE articles (
  article_id INT PRIMARY KEY,
  title      TEXT NOT NULL,
  tag_ids    TEXT       -- '3,17,42'  <-- Jaywalking
);
-- "find every article tagged 17" becomes a fragile LIKE '%17%' that also matches 170, 1700...
-- no FK can validate that 42 is a real tag; aggregates ("count per tag") need string parsing
```

The fix is exactly what 1NF prescribes — "breaking them up into separate relations associated with each other using foreign keys" ([Wikipedia — 1NF](https://en.wikipedia.org/wiki/First_normal_form)) — i.e. an **intersection (junction) table** ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)):

```sql
-- RIGHT — a junction table; one (article, tag) pair per row
CREATE TABLE tags (
  tag_id INT PRIMARY KEY,
  name   TEXT NOT NULL UNIQUE
);
CREATE TABLE article_tags (
  article_id INT NOT NULL REFERENCES articles(article_id),
  tag_id     INT NOT NULL REFERENCES tags(tag_id),
  PRIMARY KEY (article_id, tag_id)
);
-- "find every article tagged 17": WHERE tag_id = 17 (sargable, indexable)
-- a real FK guarantees every tag_id exists; COUNT(*) GROUP BY tag_id just works
```

A comma-separated FK list is the design that **no constraint can protect** — the engine sees opaque text, not references.

---

## 4. M:N Relationships Need a Junction Table

The general rule behind §3: a many-to-many relationship cannot live in either parent table — it requires a third table whose rows are the pairings, with a composite primary key on the two foreign keys. This is the *only* faithful relational representation of M:N; anything else (a list column, repeated `tag1/tag2/tag3` columns — Karwin's "Multi-Column Attributes") reintroduces anomalies. Add relationship attributes (e.g. `added_at`, `role`) to the junction row itself.

```sql
-- RIGHT — students <-> courses, with the enrollment date on the pairing
CREATE TABLE enrollments (
  student_id INT NOT NULL REFERENCES students(student_id),
  course_id  INT NOT NULL REFERENCES courses(course_id),
  enrolled_on DATE NOT NULL,
  PRIMARY KEY (student_id, course_id)
);
```

---

## 5. EAV — Key-Value Rows Instead of Columns

The **Entity-Attribute-Value** antipattern stores attributes as rows in a generic `(entity, attribute_name, value)` table instead of as typed columns ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)). It looks "flexible" but it disables the relational engine: a generic value table "cannot enforce integrity, query attributes efficiently, maintain proper SQL types, or pivot columns to rows easily" ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)).

```sql
-- WRONG — every attribute is a row; value is stringly-typed
CREATE TABLE product_attrs (
  product_id  INT,
  attr_name   TEXT,     -- 'price', 'weight', 'color', 'in_stock'...
  attr_value  TEXT      -- '19.99', '2.5', 'red', 'true'  <- no type, no NOT NULL, no CHECK
);
-- can't make price NOT NULL; can't CHECK (price >= 0); can't sort numerically;
-- "price and weight of product 7" is a self-join per attribute
```

```sql
-- RIGHT — attributes are typed columns with real constraints
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  price      NUMERIC(10,2) NOT NULL CHECK (price >= 0),
  weight_kg  NUMERIC(6,3),
  color      TEXT,
  in_stock   BOOLEAN NOT NULL DEFAULT FALSE
);
```

When attributes are *genuinely* open-ended and sparse (user-defined custom fields), a typed **JSON/JSONB column is the lesser evil** — it keeps one row per entity and lets the engine index expressions — but you trade away column constraints and must validate in the application. That tradeoff (and its query/indexing cost) is owned by `sql-json`; reach for it deliberately, never as the default schema.

---

## 6. Naive Trees — Adjacency-Only Can't Query a Subtree

Modeling a hierarchy with only a `parent_id` is the **Naive Trees** antipattern ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)). The adjacency list is fine for "who is my direct parent/children," but "find the entire subtree under node X" or "all ancestors of X" needs an unbounded number of self-joins.

```sql
-- The adjacency list — correct shape, but limited
CREATE TABLE comments (
  comment_id INT PRIMARY KEY,
  parent_id  INT REFERENCES comments(comment_id),  -- NULL at the root
  body       TEXT NOT NULL
);
-- "the whole thread under comment 5" cannot be expressed with a fixed number of joins
```

Two complementary fixes — Karwin lists Adjacency List, Path Enumeration, Nested Sets, and Closure Table ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)):

```sql
-- RIGHT (option A) — a closure table: one row per ancestor/descendant pair (incl. self)
CREATE TABLE comment_tree (
  ancestor_id   INT NOT NULL REFERENCES comments(comment_id),
  descendant_id INT NOT NULL REFERENCES comments(comment_id),
  depth         INT NOT NULL,
  PRIMARY KEY (ancestor_id, descendant_id)
);
-- whole subtree of 5:   WHERE ancestor_id   = 5
-- all ancestors of 5:   WHERE descendant_id = 5
```

The closure table makes subtree and ancestor queries plain indexed lookups at the cost of write-time bookkeeping; nested sets favor reads even more but make inserts expensive. **The other modern answer is to keep the simple adjacency list and query it with a recursive CTE (`WITH RECURSIVE`)** — that is often the right call, and the recursive-query mechanics are owned by `sql-cte-and-recursion`. Route subtree/ancestor *queries* there; choose the *storage model* here.

---

## 7. Polymorphic Associations — the Promiscuous Foreign Key

A **Polymorphic Association** is one FK column that points into multiple unrelated parent tables, disambiguated by a sibling `*_type` column ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)). It "cannot carry a referential integrity constraint in standard SQL, producing unmaintainable join logic and silent integrity failures."

```sql
-- WRONG — comments on either articles OR photos via a promiscuous FK
CREATE TABLE comments (
  comment_id    INT PRIMARY KEY,
  commentable_id   INT  NOT NULL,   -- points at articles.id OR photos.id ...
  commentable_type TEXT NOT NULL,   -- 'article' | 'photo'  <- which table?
  body          TEXT NOT NULL
);
-- no FK possible: the engine can't reference "articles OR photos depending on a string";
-- a typo'd type, or an id valid in the wrong table, is undetectable
```

The fixes — "Reverse the Reference" or use intersection tables ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)) — restore real foreign keys:

```sql
-- RIGHT (option A) — one nullable FK per concrete parent, each a real REFERENCES
CREATE TABLE comments (
  comment_id INT PRIMARY KEY,
  article_id INT REFERENCES articles(article_id),
  photo_id   INT REFERENCES photos(photo_id),
  body       TEXT NOT NULL,
  CHECK (num_nonnulls(article_id, photo_id) = 1)  -- exactly one parent
);

-- RIGHT (option B) — a common supertable both parents extend, FK targets the supertable
CREATE TABLE commentables (commentable_id INT PRIMARY KEY);
-- articles & photos each reference commentables; comments.commentable_id -> commentables
```

Both restore the integrity guarantee the polymorphic column threw away. The FK/`CHECK` enforcement syntax is owned by `sql-constraints-and-integrity`.

---

## 8. Natural vs Surrogate Keys — Choose Deliberately

Every table needs a key (a missing one is Karwin's "ID Required / Keyless Entry"). A **candidate key** is a "minimal superkey" ([Wikipedia — BCNF](https://en.wikipedia.org/wiki/Boyce%E2%80%93Codd_normal_form)) — a minimal set of columns that uniquely identifies a row. The choice:

- **Natural key** — a real-world identifier already in the data (ISBN, country code, email). Pros: no extra column, the uniqueness is meaningful, joins read well. Cons: it can change (people change email), it can be wide, and "immutable in the real world" is often a lie.
- **Surrogate key** — a synthetic, meaningless `id` (auto-increment, identity, UUID). Pros: stable, narrow, never needs to change. Cons: adds a column, and a surrogate `id` is *not a uniqueness guarantee* — you still need a `UNIQUE` constraint on the real natural key, or you will store the same logical row twice.

The common discipline: surrogate primary key **plus** a `UNIQUE` constraint on the natural key. The mistake is a surrogate `id` and *no* natural-key uniqueness — that is a table that cheerfully stores duplicates.

```sql
-- RIGHT — surrogate PK for stable joins, natural key still enforced UNIQUE
CREATE TABLE users (
  user_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email   TEXT NOT NULL UNIQUE          -- the real identity; without this, dupes slip in
);
```

---

## 9. Denormalize Only Deliberately, With a Stated Reason

Normalization is the default; denormalization is a *measured* exception for a proven read-path bottleneck — never a starting point. A redundant copy reintroduces exactly the update anomaly of §1, so it is a debt you must service: every duplicated value now needs a trigger, job, or application code to stay consistent, and a `CHECK`/FK can no longer protect you. Rules:

- Normalize first; denormalize only after a real, measured performance need (and consider an index or materialized view *before* duplicating data — route to `sql-indexing-and-sargability`).
- Write down *which* anomaly you are accepting and *what* keeps the copies in sync.
- Prefer a derived/cached column maintained by the database (materialized view, generated column) over hand-maintained duplication.

A denormalization with no written justification is just a normalization bug.

---

## 10. Portability

This skill is **pure relational design theory and is 100% portable** — 1NF through BCNF, functional dependencies, anomalies, junction tables, key choice, and every antipattern here are engine-independent and predate any specific product. What varies between PostgreSQL, MySQL, SQLite, and Oracle is only the *syntax that enforces a design* — `PRIMARY KEY`, `FOREIGN KEY ... REFERENCES`, `UNIQUE`, `CHECK`, identity/auto-increment spelling — owned by `sql-constraints-and-integrity`. The `num_nonnulls(...)` helper in §7 is PostgreSQL-specific, but the *idea* ("exactly one parent FK is non-null") is expressible everywhere. Design the schema engine-independently; spell the constraints per dialect.

---

## 11. Who Suffers When the Schema Is Wrong

A bad schema doesn't error at `CREATE TABLE` — it ships, fills with data, and then bills the interest:

- The **on-call engineer** debugging "article 17 shows tag 170's data," who finds the tags are a comma-string and `LIKE '%17%'` has matched `170`/`1700` for months (§3) — a data-integrity failure no FK could prevent, because the engine only ever saw text.
- The **analyst** asked for "average price by category" against an EAV blob, where `price` is untyped text with no `NOT NULL`, spelled three ways across rows (§5). The number does not exist in any queryable form.
- The **product engineer** who needs "every comment in this thread" and learns the tree is adjacency-only, so *no* fixed-join query returns a subtree of unknown depth (§6) — the feature is blocked on a migration.
- The **next maintainer** chasing orphaned `comments` pointing at a deleted article, because the polymorphic `commentable_id` never had — could never have — a foreign key (§7).

A normalized schema is empathy in structure: the junction table, the typed column, the closure table, and the real foreign key are each a bug the next person never has to find.

---

## 12. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: set semantics, three-valued logic, and NULL discipline this skill assumes.
- `sql-constraints-and-integrity` — the syntax that *enforces* a design: `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `NOT NULL`, and the "exactly one parent" constraint of §7.
- `sql-cte-and-recursion` — querying the hierarchies of §6 with `WITH RECURSIVE` (subtree/ancestor traversal over an adjacency list).
- `sql-json` — the EAV escape hatch of §5: when a typed JSON/JSONB column is the deliberate lesser evil for sparse, open-ended attributes.
- `sql-indexing-and-sargability` — the read-path tool to reach for *before* denormalizing (§9), and indexing the junction/closure tables above.

---

## 13. Reference Files

High-frequency schema-design anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

`${CLAUDE_SKILL_DIR}/references/common-mistakes.md`

Source provenance for every claim in this skill:

`${CLAUDE_SKILL_DIR}/references/sources.yaml`
