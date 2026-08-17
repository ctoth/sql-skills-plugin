---
name: sql-property-graph-queries
description: >-
  Guides SQL:2023 SQL/PGQ (ISO/IEC 9075-16, Part 16) — define a property graph as a metadata overlay over existing relational tables with `CREATE PROPERTY GRAPH ... VERTEX TABLES (...) EDGE TABLES (...)`, then query it with `GRAPH_TABLE (graph MATCH (a)-[e]- greater than (b) WHERE ... COLUMNS (...))` using ASCII-art vertex/edge patterns in the FROM clause — and explains its relationship to the standalone GQL language (ISO/IEC 39075:2024), with which it shares the graph-pattern sub-language (GPML). Auto-invokes when writing or editing graph-pattern queries over relational data, `GRAPH_TABLE`/`CREATE PROPERTY GRAPH`, friend-of-friend / reachability / recommendation traversals, or on "can I do graph / Cypher-style queries in SQL" requests. Prevents the LLM failure of hallucinating Cypher or Gremlin syntax where SQL/PGQ (or a recursive CTE) is what's meant. Confirm engine support before recommending.
allowed-tools: Read, Glob, Grep
---

# SQL Property Graph Queries (SQL/PGQ)

> "There is no data materialized by the SQL property graph, it is just metadata. All the actual data comes from the referenced objects."
> — [ORACLE-BASE — SQL Property Graphs and SQL/PGQ in Oracle Database 23ai](https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23)

> "SQL/PGQ specifies how to query relational representations of property graphs in SQL, while GQL is a standalone language for graph databases."
> — [Gheerbrant et al. — *GQL and SQL/PGQ: Theoretical Models and Expressive Power*](https://arxiv.org/abs/2409.01102)

SQL/PGQ (ISO/IEC 9075-16:2023 — Part 16 of the SQL:2023 standard) lets you lay a **property-graph view over tables you already have** and query it with visual `(a)-[e]->(b)` patterns. No new store, no ETL, no separate graph database: "the vertices and edges are schema objects such as tables, external tables, materialized views or synonyms to these objects" ([ORACLE-BASE](https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23)). It assumes the set semantics and three-valued logic of `sql-relational-and-null-discipline` (the policy root) and composes with ordinary SQL. This is a **bleeding-edge** reference: read §6 before you recommend it to anyone, because almost nothing ships it yet.

---

## 1. What SQL/PGQ Is — a Graph View Over Relational Tables

A property graph "is a model that describes nodes (vertices), and the relationships between them (edges)" ([ORACLE-BASE](https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23)). In SQL/PGQ the graph is **logical**: `CREATE PROPERTY GRAPH` declares which existing tables are vertices, which are edges, and how the edge foreign keys wire vertices together. Nothing is copied — the graph is metadata, exactly like a `VIEW`.

Two statements make up the feature:

- **`CREATE PROPERTY GRAPH`** — the definition (the overlay).
- **`GRAPH_TABLE (... MATCH ... COLUMNS (...))`** — a FROM-clause operator that runs a pattern against the graph and returns a normal relational table for the rest of your SQL to consume.

That second point is the whole design: "GRAPH_TABLE is a new operator in the FROM clause which executes the graph query (pattern) against a given graph and returns matches in tabular form for further processing with regular SQL" ([Oracle — Property Graphs in 23ai](https://blogs.oracle.com/database/property-graphs-in-oracle-database-23ai-the-sql-pgq-standard)).

---

## 2. `CREATE PROPERTY GRAPH` + `GRAPH_TABLE` — the Centerpiece

Start from ordinary tables: `people(person_id, name, …)` and a join table `connections(connection_id, person_id_1, person_id_2, …)`. Declare the graph:

```sql
-- The graph is a metadata overlay; it materializes NO data of its own.
CREATE PROPERTY GRAPH connections_pg
  VERTEX TABLES (
    people
      KEY (person_id)
      LABEL person
      PROPERTIES ALL COLUMNS
  )
  EDGE TABLES (
    connections
      KEY (connection_id)
      SOURCE      KEY (person_id_1) REFERENCES people (person_id)
      DESTINATION KEY (person_id_2) REFERENCES people (person_id)
      LABEL connection
      PROPERTIES ALL COLUMNS
  );
```

Each `VERTEX TABLE` gets a `KEY` and a `LABEL`; each `EDGE TABLE` additionally declares `SOURCE KEY … REFERENCES` and `DESTINATION KEY … REFERENCES`, which is how a plain join table becomes directed edges ([ORACLE-BASE](https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23)). Now query it:

```sql
-- GRAPH_TABLE lives in the FROM clause and yields the COLUMNS as a relational table.
SELECT person1, person2
FROM GRAPH_TABLE (connections_pg
  MATCH
  (p1 IS person) -[c IS connection]-> (p2 IS person)
  COLUMNS (p1.name AS person1,
           p2.name AS person2)
)
ORDER BY 1;
```

The outer `SELECT … ORDER BY` is plain SQL: "WHERE and COLUMNS clauses inside GRAPH_TABLE use the same operators, functions and predicates as are available elsewhere in SQL" ([Oracle](https://blogs.oracle.com/database/property-graphs-in-oracle-database-23ai-the-sql-pgq-standard)).

### Worked example — friend-of-friend (multi-hop)

A path of several edges is one chained pattern. This finds the parents of Samwise's friends' friends ([EnterpriseDB — Representing Graphs in PostgreSQL with SQL/PGQ](https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq)):

```sql
SELECT DISTINCT id, fof_parents, details
FROM GRAPH_TABLE (characters
  MATCH (a IS node WHERE a.name='Samwise Gamgee')
        -[x IS relationship WHERE x.type='friend']->(b IS node)
        -[y IS relationship WHERE y.type='friend']->(c IS node)
        <-[z IS relationship WHERE z.type='parent']-(d IS node)
  COLUMNS (d.id, d.name AS fof_parents, d.details AS details)
);
```

---

## 3. The ASCII-Art Vertex/Edge Pattern Syntax

The `MATCH` pattern is the GPML pattern sub-language — the same visual grammar Cypher popularized, now standardized:

- `(v)` — a **vertex** pattern. `(a IS node WHERE a.name='…')` binds variable `a`, requires label `node`, and applies an inline predicate.
- `-[e]->` — a **directed edge** from the left vertex to the right. `<-[e]-` reverses it. The arrows are traversal direction: "SOURCE is RELATIONSHIP to DESTINATION" ([EnterpriseDB](https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq)).
- `-[e IS relationship WHERE …]->` — a labelled edge with its own predicate; `-[:BOUGHT]->` is the anonymous-edge shorthand (no variable bound).
- Chaining `(a)-[x]->(b)-[y]->(c)<-[z]-(d)` expresses a **multi-hop path** in a single pattern.

```sql
-- "users who bought what Alice bought also bought rec" — a recommendation traversal.
SELECT DISTINCT g.rec_id, g.rec_name
FROM GRAPH_TABLE (recommender_graph
  MATCH (me IS users WHERE me.name='Alice')
        -[:BOUGHT]->(p IS product)<-[:BOUGHT]-(sim IS users)
        -[:BOUGHT]->(rec IS product)
  COLUMNS (me.id AS uid, rec.id AS rec_id, rec.name AS rec_name)
) AS g
WHERE NOT EXISTS (                                  -- null-safe anti-join (foundation §5)
  SELECT 1 FROM purchases pu
  WHERE pu.user_id = g.uid AND pu.product_id = g.rec_id
);
```

`GRAPH_TABLE` returns a table, so it composes with a plain `NOT EXISTS` ([gavinray — Experimenting with SQL/PGQ in Postgres 18](https://gavinray97.github.io/blog/postgres-sql-property-graphs)). The anti-join semantics are owned by `sql-relational-and-null-discipline`.

---

## 4. Relationship to GQL (ISO/IEC 39075:2024)

SQL/PGQ and **GQL** are two different 2023/2024 standards that share one sub-language:

- **SQL/PGQ** (ISO/IEC 9075-16:2023) is *part of SQL*: read-only graph pattern matching embedded in SQL via `GRAPH_TABLE`, over relational tables ([arXiv 2409.01102](https://arxiv.org/abs/2409.01102)).
- **GQL** (ISO/IEC 39075:2024) is a *standalone* graph-database language that can also create and mutate graphs — it "defines a runtime environment … supporting insertion, updating, deletion and reading of property graphs" ([ISO/IEC JTC1 — Database Language GQL](https://jtc1info.org/wp-content/uploads/2024/04/2024-Article-39075-Database-Language-GQL.docx.pdf)).

They are kin because "GQL supports the same graph pattern matching syntax as SQL Property Graph Queries, ISO/IEC 9075-16 … (SQL/PGQ)" ([ISO/IEC JTC1](https://jtc1info.org/wp-content/uploads/2024/04/2024-Article-39075-Database-Language-GQL.docx.pdf)) — the shared pattern sub-language informally called **GPML**. The split: "SQL/PGQ specifies how to query relational representations of property graphs in SQL, while GQL is a standalone language for graph databases" ([arXiv](https://arxiv.org/abs/2409.01102)). GQL is the first brand-new ISO database language since SQL itself. If your data lives in relational tables and you want to stay in SQL, SQL/PGQ is the relevant standard; GQL targets dedicated graph stores.

---

## 5. Don't Hallucinate Cypher / Gremlin Into a SQL Context

Because the ASCII-art lineage makes Cypher and SQL/PGQ *look* alike, the standing LLM failure is to emit raw Cypher (or Gremlin) into a relational SQL session where neither parses.

```sql
-- WRONG — Cypher. Will NOT parse in Oracle/PostgreSQL/SQLite/MySQL; RETURN is not SQL,
-- and there is no FROM. This needs Neo4j, not a SQL engine.
MATCH (a:Person)-[:FRIEND]->(b:Person)
WHERE a.name = 'Alice'
RETURN b.name;
```
```sql
-- WRONG — Gremlin (Apache TinkerPop). A graph-DB traversal API, not SQL at all.
g.V().has('name','Alice').out('friend').values('name')
```
```sql
-- RIGHT (engine HAS SQL/PGQ, e.g. Oracle 23ai) — pattern goes inside GRAPH_TABLE,
-- projection comes from the outer SELECT via COLUMNS.
SELECT friend
FROM GRAPH_TABLE (social
  MATCH (a IS person WHERE a.name='Alice')-[f IS friend]->(b IS person)
  COLUMNS (b.name AS friend)
);
```

The tells: Cypher uses `RETURN`; SQL/PGQ wraps the pattern in `GRAPH_TABLE(... COLUMNS (...))` inside a SQL `FROM` and projects with the outer `SELECT`. Cypher/Gremlin require a graph database (Neo4j, TinkerPop); SQL/PGQ runs inside the SQL engine. And if the engine lacks SQL/PGQ, the answer is a recursive CTE — not Cypher (§6).

---

## 6. Portability — VERY LOW (read before recommending)

This is **bleeding-edge**. As of 2026 there is essentially one shipping implementation:

| Engine | SQL/PGQ status | What to do |
|---|---|---|
| **Oracle Database 23ai** | **Ships it (GA).** Availability: "Oracle Database 23ai and later versions" ([ORACLE-BASE](https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23)) | Use SQL/PGQ |
| **PostgreSQL** | **Experimental, out-of-tree only.** "still very much a work-in-progress with no release date … can be patched into Postgres" ([EnterpriseDB](https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq)); needs patches on 18beta2 ([gavinray](https://gavinray97.github.io/blog/postgres-sql-property-graphs)) | Recursive CTE |
| **SQLite** | None | Recursive CTE |
| **MySQL / MariaDB** | None | Recursive CTE |

"To date, only Oracle 23 has released support for it" ([gavinray](https://gavinray97.github.io/blog/postgres-sql-property-graphs)). So unless you are on Oracle 23ai, **the portable way to traverse graph-shaped relational data is a recursive CTE** (`WITH RECURSIVE`) over the same tables — reachability, shortest-path-by-depth, friend-of-friend all express cleanly there. That is owned by `sql-cte-and-recursion`; route there for the portable spelling. For the engine-support matrix and dialect spellings, route to `sql-standard-vs-dialect-map`. **Confirm engine support before recommending SQL/PGQ in any application code.**

---

## 7. Who Suffers When This Goes Wrong

- The **developer** who hand-wrote Cypher (`MATCH … RETURN`) into a PostgreSQL or MySQL app because an LLM emitted it confidently — it failed at parse time, and they lost an afternoon discovering their engine has no graph language at all. A recursive CTE was always available (§6).
- The **team** that stood up a *separate graph database* (new service, new sync pipeline, new on-call surface) to answer a friend-of-friend question — when SQL/PGQ on Oracle 23ai, or a recursive CTE over their existing tables on anything else, would have answered it in place with no new infrastructure (§1, §6).
- The **maintainer** who shipped `GRAPH_TABLE` queries against Oracle 23ai and then could not port the app to PostgreSQL, because the graph layer had no equivalent there and the traversals had to be rewritten as recursive CTEs under deadline (§6).

---

## 8. Routing to Related Skills

- `sql-relational-and-null-discipline` — the foundation: set semantics, three-valued logic, and the null-safe `NOT EXISTS` anti-join used to filter `GRAPH_TABLE` output (§3).
- `sql-cte-and-recursion` — **the portable fallback.** `WITH RECURSIVE` is how you do graph traversal / reachability on every engine that lacks SQL/PGQ, i.e. almost all of them (§6).
- `sql-standard-vs-dialect-map` — engine support matrix and dialect spellings; the home for "which engines ship SQL/PGQ."

---

## 9. Reference Files

High-frequency SQL/PGQ and graph-query anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
