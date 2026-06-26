# Research: sql-property-graph-queries (SQL:2023 SQL/PGQ)

Skill #30 in `reports/skill-plan-sql.md` (lines 416-424). Type: reference. Bleeding-edge,
VERY LOW PORTABILITY. Each finding: URL, section, verbatim quote, why it matters.

Fetch status notes:
- `https://modern-sql.com/standard/2023` — fetched, but the rendered page did NOT surface any
  SQL/PGQ / Part 16 / GRAPH_TABLE content (model reported "no information about SQL/PGQ"). Not
  cited for PGQ claims; PGQ standardization sourced from the more specific pages below.
- `https://blogs.oracle.com/database/...sql-pgq-standard` — HTTP 403 on direct WebFetch; content
  recovered via WebSearch snippet of the same Oracle blog. ORACLE-BASE used as the verbatim-syntax
  backstop for the same product (Oracle 23ai).
- `https://arxiv.org/pdf/2409.01102` — PDF was binary/password-protected; abstract recovered from
  the arXiv HTML abstract page `https://arxiv.org/abs/2409.01102`.

---

## 1. SQL/PGQ defines a property graph as a VIEW/metadata overlay over existing relational tables

URL: https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23 (section "Key Concept")

Verbatim: "the vertices and edges are schema objects such as tables, external tables,
materialized views or synonyms to these objects." And: "There is no data materialized by the SQL
property graph, it is just metadata. All the actual data comes from the referenced objects."

Verbatim (definition): "A property graph is a model that describes nodes (vertices), and the
relationships between them (edges)."

Why it matters: This is the central framing of the skill. SQL/PGQ does NOT create a new graph
store — `CREATE PROPERTY GRAPH` is a logical overlay (like a view) over rows already living in
ordinary relational tables. This is the hook that says "graph queries without a graph database"
and the reason a recursive CTE over the SAME tables is the portable equivalent.

---

## 2. CREATE PROPERTY GRAPH — verbatim syntax (VERTEX TABLES / EDGE TABLES / SOURCE/DESTINATION KEY)

URL: https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23 (section "CREATE PROPERTY GRAPH Syntax")

Verbatim code:
```sql
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
      SOURCE KEY (person_id_1) REFERENCES people (person_id)
      DESTINATION KEY (person_id_2) REFERENCES people (person_id)
      LABEL connection
      PROPERTIES ALL COLUMNS
  );
```

Why it matters: Verbatim, Oracle-validated centerpiece syntax. Shows the structure: VERTEX TABLES
each get a KEY + LABEL + PROPERTIES; EDGE TABLES additionally declare SOURCE KEY ... REFERENCES and
DESTINATION KEY ... REFERENCES that wire the edge's foreign keys to vertex keys. This is how an
ordinary join table (`connections`) becomes graph edges.

---

## 3. GRAPH_TABLE / MATCH / COLUMNS — verbatim query syntax + "operator in the FROM clause"

URL: https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23 (section "GRAPH_TABLE Query Syntax")

Verbatim code:
```sql
SELECT person1, person2
FROM GRAPH_TABLE (connections_pg
  MATCH
  (p1 IS person) -[c IS connection]-> (p2 IS person)
  COLUMNS (p1.name AS person1,
           p2.name AS person2)
)
ORDER BY 1;
```

Verbatim (same article): "The `GRAPH_TABLE` operator enables querying property graphs using SQL/PGQ
patterns in the `FROM` clause with `MATCH` conditions defining vertex and edge relationships."

URL: https://blogs.oracle.com/database/property-graphs-in-oracle-database-23ai-the-sql-pgq-standard
(recovered via WebSearch snippet of the Oracle blog)

Verbatim: "GRAPH_TABLE is a new operator in the FROM clause which executes the graph query (pattern)
against a given graph and returns matches in tabular form for further processing with regular SQL."

Verbatim: "WHERE and COLUMNS clauses inside GRAPH_TABLE use the same operators, functions and
predicates as are available elsewhere in SQL."

Why it matters: `GRAPH_TABLE(...)` sits in the FROM clause and returns a normal relational table
(the columns named in COLUMNS), so the rest of the SQL statement (WHERE, ORDER BY, joins) is
ordinary SQL. This is the bridge between the graph pattern and relational results, and the reason
the feature integrates with the rest of standard SQL.

---

## 4. ASCII-art vertex/edge pattern syntax — verbatim, with direction semantics

URL: https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq

Verbatim (CREATE PROPERTY GRAPH):
```sql
CREATE PROPERTY GRAPH characters
  VERTEX TABLES (
    nodes LABEL node PROPERTIES ( id, name, details )
  )
  EDGE TABLES (
   edges
    SOURCE KEY ( from_id ) REFERENCES nodes ( id )
    DESTINATION KEY ( to_id ) REFERENCES nodes ( id )
    LABEL relationship PROPERTIES (type)
  );
```

Verbatim (friend-of-friend, multi-hop pattern):
```sql
SELECT DISTINCT id, fof_parents, details FROM GRAPH_TABLE(characters
 MATCH (a IS node WHERE a.name='Samwise Gamgee')-[x IS relationship WHERE x.type='friend']->(b IS node)-[y IS relationship WHERE y.type='friend']->(c IS node)<-[z IS relationship WHERE z.type='parent']-(d IS node)
 COLUMNS (d.id, d.name as fof_parents, d.details as details)
);
```

Verbatim (direction): the arrow directions indicate traversal: "SOURCE is RELATIONSHIP to
DESTINATION."

Why it matters: This is the verbatim ASCII-art grammar:
- `(v)` = a vertex pattern; `(a IS node)` binds variable `a` with label `node`.
- `-[e]->` = a directed edge from left vertex to right vertex; `<-[e]-` reverses direction.
- `-[e IS relationship WHERE ...]->` attaches a label + an inline WHERE predicate to the edge.
- chaining `(a)-[x]->(b)-[y]->(c)<-[z]-(d)` expresses a multi-hop path in one pattern.
This is the lineage borrowed from Cypher's visual syntax, standardized.

---

## 5. A second worked example — recommendation / reachability over relational tables

URL: https://gavinray97.github.io/blog/postgres-sql-property-graphs

Verbatim code:
```sql
CREATE PROPERTY GRAPH recommender_graph
VERTEX TABLES (
    users LABEL users PROPERTIES (id, name),
    products LABEL product PROPERTIES (id, name, price)
)
EDGE TABLES (
    purchases
    SOURCE KEY (user_id) REFERENCES users (id)
    DESTINATION KEY (product_id) REFERENCES products (id)
    LABEL BOUGHT
);
```
```sql
SELECT DISTINCT g.rec_id, g.rec_name
FROM GRAPH_TABLE (
       recommender_graph
       MATCH   (me  IS users    WHERE me.name = 'Alice')
               -[:BOUGHT]->(p   IS product)<-[:BOUGHT]-(sim IS users)
               -[:BOUGHT]->(rec IS product)
       COLUMNS (me.id  AS uid,
                rec.id AS rec_id,
                rec.name AS rec_name)
) AS g
WHERE NOT EXISTS (
        SELECT 1 FROM purchases p
        WHERE p.user_id = g.uid AND p.product_id = g.rec_id
);
```

Why it matters: Demonstrates "people who bought what Alice bought also bought..." collaborative
filtering as a graph pattern over the ordinary `users`/`products`/`purchases` tables — and shows
GRAPH_TABLE composing with an ordinary `NOT EXISTS` (null-safe anti-join). Shows the `-[:LABEL]->`
anonymous-edge shorthand (no edge variable).

---

## 6. Relationship to GQL — shared pattern sub-language (GPML); GQL is standalone

URL: https://arxiv.org/abs/2409.01102 ("GQL and SQL/PGQ: Theoretical Models and Expressive Power",
Gheerbrant, Libkin, Peterfreund, Rogova)

Verbatim (abstract): "SQL/PGQ and GQL are very recent international standards for querying property
graphs: SQL/PGQ specifies how to query relational representations of property graphs in SQL, while
GQL is a standalone language for graph databases."

Verbatim (difference): "PGQ's bottom up evaluation versus GQL's linear, or pipelined approach."

URL: https://jtc1info.org/wp-content/uploads/2024/04/2024-Article-39075-Database-Language-GQL.docx.pdf
(ISO/IEC JTC1 article on ISO/IEC 39075, recovered via WebSearch snippet)

Verbatim: "GQL supports the same graph pattern matching syntax as SQL Property Graph Queries,
ISO/IEC 9075-16, Information technology — Database languages SQL — Part 16: Property Graph Queries
(SQL/PGQ)."

Verbatim: GQL "defines a runtime environment that is initialized from a persistent and extensible
catalog ... supporting insertion, updating, deletion and reading of property graphs."

Supporting (the informal name GPML): WebSearch result summary — "Both GQL and SQL/PGQ share a
pattern-matching sub-language (informally called GPML)."

Why it matters: Establishes the (c) relationship requirement. SQL/PGQ (ISO/IEC 9075-16:2023, a part
OF the SQL standard) is read-only pattern matching embedded in SQL; GQL (ISO/IEC 39075:2024, a
SEPARATE standard) is a full standalone graph database language (it can write graphs). They share
the graph-pattern-matching sub-language (GPML) — same ASCII-art `(a)-[e]->(b)` syntax lineage —
which is why the visual pattern syntax looks the same in both, and both descend from Cypher.

GQL significance: ISO/IEC 39075:2024 is the first brand-new ISO database language standard since SQL
itself (1987). (Confirmed across Neo4j/ISO/Wikipedia coverage of the April 2024 publication.)

---

## 7. VERY LOW PORTABILITY — Oracle 23ai ships it; Postgres experimental; SQLite/MySQL absent

URL: https://gavinray97.github.io/blog/postgres-sql-property-graphs

Verbatim: "To date, only Oracle 23 has released support for it, but I want to show you how you can
experiment with SQL/PGQ today in Postgres." (status as of mid-2025) — implementation uses "patches
applied to Postgres 18beta2 ... through a custom Docker image rather than official PostgreSQL
releases."

URL: https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq

Verbatim: "While still very much a work-in-progress with no release date, it is functional and can be
patched into Postgres for you to explore!" (requires building PostgreSQL from source + patches from
the pgsql-hackers mailing list)

URL: https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23 — Availability: "Oracle
Database 23ai and later versions."

Why it matters: This is the (d) portability reality. As of 2026: Oracle Database 23ai is the only
GA/shipping implementation. PostgreSQL support exists only as out-of-tree patches (not merged, no
release date). SQLite and MySQL have nothing. Therefore the PORTABLE way to do graph traversal /
reachability on relational data is a recursive CTE (`WITH RECURSIVE`) — route to
`sql-cte-and-recursion`. Recommending SQL/PGQ in a Postgres/SQLite/MySQL app today is a bug.

---

## 8. The LLM failure mode to prevent — hallucinating Cypher/Gremlin in a SQL context

Synthesis (grounded in #4, #6): the ASCII-art lineage means Cypher and SQL/PGQ LOOK similar, so an
LLM asked for "graph queries in my database" will often emit raw Cypher (`MATCH (a)-[:R]->(b)
RETURN ...`) or Gremlin (`g.V().has(...)`) into a relational SQL context where neither parses.

Key distinctions to encode:
- Cypher uses `RETURN`; SQL/PGQ wraps the pattern in `GRAPH_TABLE(... COLUMNS (...))` inside a SQL
  `FROM` clause and projects with the outer `SELECT`.
- Cypher/Gremlin require a graph database (Neo4j / TinkerPop); they do not run in Oracle/Postgres
  SQL engines. SQL/PGQ runs inside the SQL engine over relational tables.
- If the target engine lacks SQL/PGQ (Postgres/SQLite/MySQL today), the right answer is a recursive
  CTE, NOT Cypher.

---

## Routing targets (verified to exist in plugin)
- foundation: `sql-relational-and-null-discipline` (set semantics, null-safe anti-join used in #5)
- portable fallback: `sql-cte-and-recursion` (WITH RECURSIVE graph traversal)
- engine support: `sql-standard-vs-dialect-map`

## Source URL list (accessed 2026-06-26)
1. https://oracle-base.com/articles/23/sql-property-graphs-and-sql-pgq-23
2. https://blogs.oracle.com/database/property-graphs-in-oracle-database-23ai-the-sql-pgq-standard
3. https://www.enterprisedb.com/blog/representing-graphs-postgresql-sqlpgq
4. https://gavinray97.github.io/blog/postgres-sql-property-graphs
5. https://arxiv.org/abs/2409.01102
6. https://jtc1info.org/wp-content/uploads/2024/04/2024-Article-39075-Database-Language-GQL.docx.pdf
7. https://modern-sql.com/standard/2023 (general SQL:2023 overview; PGQ not surfaced)
