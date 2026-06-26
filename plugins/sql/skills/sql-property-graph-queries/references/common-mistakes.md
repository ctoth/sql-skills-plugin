# Common SQL/PGQ & Graph-Query Mistakes

Anti-patterns in LLM-generated SQL around property-graph queries, each with wrong/right code and a
primary-source citation. The skill (`sql-property-graph-queries`) states the model; this file holds
the high-frequency failure modes. Because SQL/PGQ has **very low portability** (Oracle 23ai only as
of 2026), most RIGHT examples come in two forms: the SQL/PGQ spelling *and* the portable
recursive-CTE fallback, routed to `sql-cte-and-recursion`.

---

## 1. Emitting raw Cypher into a SQL session

**The problem:** Asked for "graph queries in my database," the model writes Cypher — `MATCH (a)-[:R]->(b) RETURN b`. Cypher's `RETURN` is not SQL, there is no `FROM`, and it requires Neo4j; it will not parse in Oracle, PostgreSQL, SQLite, or MySQL. "GRAPH_TABLE is a new operator in the FROM clause" ([Oracle](https://blogs.oracle.com/database/property-graphs-in-oracle-database-23ai-the-sql-pgq-standard)) — the pattern must be wrapped in it.

```sql
-- WRONG — Cypher; no FROM, RETURN is not SQL, needs a graph database
MATCH (a:Person)-[:FRIEND]->(b:Person) WHERE a.name='Alice' RETURN b.name;

-- RIGHT (SQL/PGQ, engine permitting) — pattern inside GRAPH_TABLE, project via outer SELECT
SELECT friend FROM GRAPH_TABLE (social
  MATCH (a IS person WHERE a.name='Alice')-[f IS friend]->(b IS person)
  COLUMNS (b.name AS friend)
);
```

*Source: [Oracle — Property Graphs in 23ai](https://blogs.oracle.com/database/property-graphs-in-oracle-database-23ai-the-sql-pgq-standard). Depth: this skill, §5.*

---

## 2. Emitting Gremlin (a traversal API, not a query language)

**The problem:** The model produces `g.V().has('name','Alice').out('friend')…` — Apache TinkerPop Gremlin, a graph-DB traversal API. It is not SQL and runs in no SQL engine.

```sql
-- WRONG — Gremlin; not SQL at all
g.V().has('name','Alice').out('friend').values('name')

-- RIGHT (portable, any engine) — recursive CTE over the existing tables
WITH RECURSIVE reachable(person_id) AS (
  SELECT person_id FROM people WHERE name = 'Alice'
  UNION
  SELECT c.person_id_2 FROM reachable r JOIN connections c ON c.person_id_1 = r.person_id
)
SELECT p.name FROM reachable r JOIN people p ON p.person_id = r.person_id;
```

*Source: [gavinray — SQL/PGQ in Postgres 18](https://gavinray97.github.io/blog/postgres-sql-property-graphs). Depth: this skill, §5, §6; recursive form owned by `sql-cte-and-recursion`.*

---

## 3. Recommending SQL/PGQ on an engine that doesn't have it

**The problem:** The model writes `GRAPH_TABLE`/`CREATE PROPERTY GRAPH` for a PostgreSQL, SQLite, or MySQL app. As of 2026, "only Oracle 23 has released support for it" ([gavinray](https://gavinray97.github.io/blog/postgres-sql-property-graphs)); PostgreSQL is "still very much a work-in-progress with no release date" ([EnterpriseDB](https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq)). The statement is a syntax error everywhere but Oracle 23ai.

```sql
-- WRONG (on PostgreSQL/SQLite/MySQL) — GRAPH_TABLE is not implemented; parse error
SELECT * FROM GRAPH_TABLE (g MATCH (a)-[e]->(b) COLUMNS (a.id, b.id));

-- RIGHT (portable) — friend-of-friend as a recursive CTE, depth-bounded
WITH RECURSIVE foaf(start_id, person_id, depth) AS (
  SELECT person_id, person_id, 0 FROM people WHERE name = 'Alice'
  UNION ALL
  SELECT f.start_id, c.person_id_2, f.depth + 1
  FROM foaf f JOIN connections c ON c.person_id_1 = f.person_id
  WHERE f.depth < 2
)
SELECT DISTINCT p.name FROM foaf f JOIN people p ON p.person_id = f.person_id
WHERE f.depth = 2;
```

*Source: [gavinray](https://gavinray97.github.io/blog/postgres-sql-property-graphs); [EnterpriseDB](https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq). Depth: this skill, §6; portable form owned by `sql-cte-and-recursion`.*

---

## 4. Reaching for a separate graph database when the data is already relational

**The problem:** To answer one reachability/recommendation question, the model proposes standing up Neo4j (new service, sync pipeline, on-call surface). But the data is already in relational tables, and SQL/PGQ overlays them with no copy: "There is no data materialized by the SQL property graph, it is just metadata. All the actual data comes from the referenced objects" ([ORACLE-BASE](https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23)). On non-Oracle engines a recursive CTE does the same in place.

```sql
-- RIGHT — overlay the EXISTING tables; no new datastore (Oracle 23ai)
CREATE PROPERTY GRAPH connections_pg
  VERTEX TABLES ( people KEY (person_id) LABEL person PROPERTIES ALL COLUMNS )
  EDGE TABLES (
    connections KEY (connection_id)
      SOURCE KEY (person_id_1) REFERENCES people (person_id)
      DESTINATION KEY (person_id_2) REFERENCES people (person_id)
      LABEL connection PROPERTIES ALL COLUMNS
  );
-- ...or a recursive CTE over the same tables on every other engine (see #3).
```

*Source: [ORACLE-BASE](https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23). Depth: this skill, §1, §7.*

---

## 5. Confusing SQL/PGQ with GQL — using GQL/write syntax in SQL/PGQ

**The problem:** The model treats SQL/PGQ as a full graph database language and tries to *create* or *mutate* graph data through it, or mixes GQL's standalone syntax into `GRAPH_TABLE`. SQL/PGQ is read-only pattern matching; GQL is the separate standard that mutates graphs. "SQL/PGQ specifies how to query relational representations of property graphs in SQL, while GQL is a standalone language for graph databases" ([arXiv 2409.01102](https://arxiv.org/abs/2409.01102)); GQL (ISO/IEC 39075:2024) is what "support[s] insertion, updating, deletion and reading of property graphs" ([ISO/IEC JTC1](https://jtc1info.org/wp-content/uploads/2024/04/2024-Article-39075-Database-Language-GQL.docx.pdf)).

```sql
-- WRONG — there is no graph "INSERT" in SQL/PGQ; you mutate the underlying TABLES
INSERT INTO GRAPH connections_pg (a)-[:connection]->(b) ...;

-- RIGHT — insert into the base table; the graph overlay reflects it automatically
INSERT INTO connections (connection_id, person_id_1, person_id_2)
VALUES (101, 1, 2);
```

*Source: [arXiv 2409.01102](https://arxiv.org/abs/2409.01102); [ISO/IEC JTC1 — GQL](https://jtc1info.org/wp-content/uploads/2024/04/2024-Article-39075-Database-Language-GQL.docx.pdf). Depth: this skill, §4.*

---

## 6. Getting the edge direction backwards in the pattern

**The problem:** The model writes `(a)-[e]->(b)` when the edge's `SOURCE KEY`/`DESTINATION KEY` actually run the other way, and silently gets the wrong rows (or none). The arrow is traversal direction: "SOURCE is RELATIONSHIP to DESTINATION" ([EnterpriseDB](https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq)). `->` follows SOURCE→DESTINATION as declared in `CREATE PROPERTY GRAPH`; `<-` reverses it.

```sql
-- WRONG — if connections.SOURCE is the followee, this finds followees, not followers
... MATCH (a IS person WHERE a.name='Alice')-[f IS connection]->(b IS person) ...

-- RIGHT — reverse the arrow to traverse against the declared direction
... MATCH (a IS person WHERE a.name='Alice')<-[f IS connection]-(b IS person) ...
```

*Source: [EnterpriseDB](https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq). Depth: this skill, §3.*
