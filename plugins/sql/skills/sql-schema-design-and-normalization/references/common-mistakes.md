# Common SQL Schema-Design Mistakes
## Contents

- [1. Storing a list in a column (Jaywalking — a 1NF violation)](#1-storing-a-list-in-a-column-jaywalking-a-1nf-violation)
- [2. Modeling many-to-many with repeated columns or a list](#2-modeling-many-to-many-with-repeated-columns-or-a-list)
- [3. EAV — attributes as rows instead of typed columns](#3-eav-attributes-as-rows-instead-of-typed-columns)
- [4. Adjacency-only tree that can't query a subtree (Naive Trees)](#4-adjacency-only-tree-that-cant-query-a-subtree-naive-trees)
- [5. Polymorphic / promiscuous foreign key](#5-polymorphic-promiscuous-foreign-key)
- [6. Surrogate id with no UNIQUE on the natural key](#6-surrogate-id-with-no-unique-on-the-natural-key)
- [7. Redundant columns that create an update anomaly](#7-redundant-columns-that-create-an-update-anomaly)
- [8. Denormalizing with no stated reason](#8-denormalizing-with-no-stated-reason)


Structural anti-patterns in LLM-generated SQL schemas, each with wrong/right code and a primary-source
citation. The skill (`sql-schema-design-and-normalization`) states the design theory; this file holds the
high-frequency failure modes. All RIGHT examples are portable relational design; the constraint *syntax*
that enforces them is owned by `sql-constraints-and-integrity` (one helper, `num_nonnulls`, is flagged as
PostgreSQL-specific).

---

## 1. Storing a list in a column (Jaywalking — a 1NF violation)

**The problem:** The model stores multiple values — often foreign keys — as a delimited string. This violates First Normal Form, which "requires all attribute domains to be simple domains, such that the data in each field is atomic" ([Wikipedia — 1NF](https://en.wikipedia.org/wiki/First_normal_form)). No FK can validate the references, membership tests degrade to fragile `LIKE`, and aggregates need string parsing. Karwin calls it "Jaywalking" ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)).

```sql
-- WRONG — a list of tag ids in one column; LIKE '%17%' also matches 170, 1700
CREATE TABLE articles (article_id INT PRIMARY KEY, tag_ids TEXT);  -- '3,17,42'

-- RIGHT — a junction (intersection) table: one (article, tag) pair per row
CREATE TABLE article_tags (
  article_id INT NOT NULL REFERENCES articles(article_id),
  tag_id     INT NOT NULL REFERENCES tags(tag_id),
  PRIMARY KEY (article_id, tag_id)
);
```

*Source: [Wikipedia — 1NF](https://en.wikipedia.org/wiki/First_normal_form); [Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §3.*

---

## 2. Modeling many-to-many with repeated columns or a list

**The problem:** The model represents M:N with `tag1, tag2, tag3` columns (Karwin's "Multi-Column Attributes") or a list, capping cardinality and reintroducing anomalies. M:N has exactly one faithful relational form — a junction table.

```sql
-- WRONG — fixed slots; a 4th tag has nowhere to go, and "find tag X" must check every column
CREATE TABLE articles (article_id INT PRIMARY KEY, tag1 INT, tag2 INT, tag3 INT);

-- RIGHT — a junction table scales to any cardinality and is indexable
CREATE TABLE article_tags (
  article_id INT NOT NULL REFERENCES articles(article_id),
  tag_id     INT NOT NULL REFERENCES tags(tag_id),
  PRIMARY KEY (article_id, tag_id)
);
```

*Source: [Wikipedia — 1NF](https://en.wikipedia.org/wiki/First_normal_form); [Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §3–§4.*

---

## 3. EAV — attributes as rows instead of typed columns

**The problem:** To look "flexible," the model stores `(entity, attribute_name, value)` rows. A generic value table "cannot enforce integrity, query attributes efficiently, maintain proper SQL types, or pivot columns to rows easily" ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)). `price` becomes untyped text with no `NOT NULL` and no `CHECK`.

```sql
-- WRONG — stringly-typed, constraint-free
CREATE TABLE product_attrs (product_id INT, attr_name TEXT, attr_value TEXT);

-- RIGHT — typed columns with real constraints
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  price      NUMERIC(10,2) NOT NULL CHECK (price >= 0),
  color      TEXT
);
-- For genuinely sparse, open-ended custom fields, a typed JSON/JSONB column is the
-- deliberate lesser evil -> route to sql-json (NOT the default schema).
```

*Source: [Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §5; JSON tradeoff owned by `sql-json`.*

---

## 4. Adjacency-only tree that can't query a subtree (Naive Trees)

**The problem:** The model stores a hierarchy with only `parent_id`. That handles direct parent/child but cannot return "the whole subtree under X" or "all ancestors of X" with a fixed number of joins — Karwin's "Naive Trees" ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)).

```sql
-- WRONG (when subtree queries are needed) — adjacency list alone
CREATE TABLE comments (comment_id INT PRIMARY KEY, parent_id INT REFERENCES comments(comment_id));

-- RIGHT (option A) — a closure table makes subtree/ancestor queries indexed lookups
CREATE TABLE comment_tree (
  ancestor_id   INT NOT NULL REFERENCES comments(comment_id),
  descendant_id INT NOT NULL REFERENCES comments(comment_id),
  depth         INT NOT NULL,
  PRIMARY KEY (ancestor_id, descendant_id)
);
-- RIGHT (option B) — keep the adjacency list, query it with WITH RECURSIVE -> sql-cte-and-recursion
```

*Source: [Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §6; recursive query mechanics owned by `sql-cte-and-recursion`.*

---

## 5. Polymorphic / promiscuous foreign key

**The problem:** One FK column points at multiple unrelated tables, disambiguated by a `*_type` string. It "cannot carry a referential integrity constraint in standard SQL, producing unmaintainable join logic and silent integrity failures" ([Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/)). Orphans and wrong-table ids are undetectable.

```sql
-- WRONG — no FK can reference "articles OR photos depending on a string"
CREATE TABLE comments (
  comment_id INT PRIMARY KEY,
  commentable_id INT NOT NULL, commentable_type TEXT NOT NULL  -- 'article' | 'photo'
);

-- RIGHT — one real FK per concrete parent, exactly one non-null
CREATE TABLE comments (
  comment_id INT PRIMARY KEY,
  article_id INT REFERENCES articles(article_id),
  photo_id   INT REFERENCES photos(photo_id),
  CHECK (num_nonnulls(article_id, photo_id) = 1)  -- num_nonnulls is PostgreSQL-specific
);
```

*Source: [Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §7; constraint syntax owned by `sql-constraints-and-integrity`.*

---

## 6. Surrogate `id` with no UNIQUE on the natural key

**The problem:** The model adds an auto-increment `id` and assumes that prevents duplicates. A surrogate key only guarantees the *surrogate* is unique — without a `UNIQUE` constraint on the real natural key, the table stores the same logical row twice. A candidate key is a "minimal superkey" ([Wikipedia — BCNF](https://en.wikipedia.org/wiki/Boyce%E2%80%93Codd_normal_form)); the natural key is still one even when a surrogate is the PK.

```sql
-- WRONG — two rows with the same email are both accepted
CREATE TABLE users (user_id BIGINT PRIMARY KEY, email TEXT NOT NULL);

-- RIGHT — surrogate PK for stable joins, natural key enforced UNIQUE
CREATE TABLE users (
  user_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email   TEXT NOT NULL UNIQUE
);
```

*Source: [Wikipedia — BCNF](https://en.wikipedia.org/wiki/Boyce%E2%80%93Codd_normal_form); [Karwin, *SQL Antipatterns*](https://pragprog.com/titles/bksqla/sql-antipatterns/). Depth: this skill, §8.*

---

## 7. Redundant columns that create an update anomaly

**The problem:** The model copies a fact (customer email, product name) onto every related row "to avoid a join." Now "the same information can be expressed on multiple rows; therefore updates may result in logical inconsistencies" ([Wikipedia — Normalization](https://en.wikipedia.org/wiki/Database_normalization)) — the update anomaly. The copies drift; no constraint keeps them in sync.

```sql
-- WRONG — customer_email duplicated on every order; update one, miss the rest
CREATE TABLE orders (order_id INT PRIMARY KEY, customer_id INT, customer_email TEXT);

-- RIGHT — email lives once in customers; join (or a view) to read it
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  customer_id INT NOT NULL REFERENCES customers(customer_id)
);
```

*Source: [Wikipedia — Database Normalization](https://en.wikipedia.org/wiki/Database_normalization). Depth: this skill, §1, §9.*

---

## 8. Denormalizing with no stated reason

**The problem:** The model duplicates data for "performance" before any measurement, taking on the update anomaly with no plan to service it. Normalize first; reach for an index or materialized view before duplicating; and if you do denormalize, write down which anomaly you accept and what keeps the copies consistent.

```sql
-- WRONG — a hand-maintained running total that nothing keeps correct
CREATE TABLE accounts (account_id INT PRIMARY KEY, cached_balance NUMERIC);  -- drifts silently

-- RIGHT — derive it, or maintain it in the database (generated column / materialized view)
--   ...measure first; prefer an index -> sql-indexing-and-sargability
```

*Source: [Wikipedia — Database Normalization](https://en.wikipedia.org/wiki/Database_normalization). Depth: this skill, §9; indexing owned by `sql-indexing-and-sargability`.*
