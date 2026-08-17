# Common SQL Privilege & Access-Control Mistakes
## Contents

- [1. The application connects as a superuser](#1-the-application-connects-as-a-superuser)
- [2. GRANT ALL to the application role](#2-grant-all-to-the-application-role)
- [3. The application role owns its tables](#3-the-application-role-owns-its-tables)
- [4. Granting privileges to PUBLIC](#4-granting-privileges-to-public)
- [5. Handing out WITH GRANT OPTION casually](#5-handing-out-with-grant-option-casually)
- [6. Granting privileges to each user instead of a group role](#6-granting-privileges-to-each-user-instead-of-a-group-role)
- [7. Writing GRANT/CREATE ROLE for SQLite](#7-writing-grantcreate-role-for-sqlite)
- [8. Leaving default PUBLIC privileges in place](#8-leaving-default-public-privileges-in-place)


Anti-patterns in LLM-generated SQL and database-setup code around `GRANT`/`REVOKE`, roles, and least
privilege, each with wrong/right code and a primary-source citation. The policy skill
(`sql-privileges-and-access-control`) states the model; this file holds the high-frequency failure
modes. All RIGHT examples are standard/portable SQL where a privilege model exists; engine-specific
spellings (and SQLite's *absence* of any privilege model) are flagged and routed to
`sql-standard-vs-dialect-map`.

---

## 1. The application connects as a superuser

**The problem:** The generated connection string uses the cluster superuser (`postgres`, `root`, `sa`). Because "a database superuser bypasses all permission checks" ([PostgreSQL — Role Attributes](https://www.postgresql.org/docs/current/role-attributes.html)), `GRANT`/`REVOKE` are inert on that connection — every privilege check is skipped. A single SQL injection or logic bug is then total compromise.

```sql
-- WRONG — app uses superuser credentials; least privilege is impossible here
--   postgres://postgres:secret@db/appdb

-- RIGHT — a dedicated, non-superuser LOGIN role with only what the app needs
CREATE ROLE app_rw LOGIN PASSWORD '...';
GRANT CONNECT ON DATABASE appdb TO app_rw;
GRANT USAGE ON SCHEMA public TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders, customers TO app_rw;
```

*Source: [PostgreSQL — Role Attributes](https://www.postgresql.org/docs/current/role-attributes.html). Depth: this skill, §2.*

---

## 2. `GRANT ALL` to the application role

**The problem:** A dedicated role is created, then handed `ALL PRIVILEGES` on everything "to make it work." That includes `DROP`/`TRUNCATE`/`ALTER`-class powers the app never issues, so a bug or injection can destroy the schema. Grants are additive and should name the exact verbs ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)).

```sql
-- WRONG — over-broad; the app can now wreck the schema
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_rw;

-- RIGHT — only the data-manipulation verbs the app actually uses
GRANT SELECT, INSERT, UPDATE, DELETE ON orders, customers, line_items TO app_rw;
```

*Source: [PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html). Depth: this skill, §1, §2.*

---

## 3. The application role owns its tables

**The problem:** Migrations and the running app share one identity, so the app role *owns* the tables. Ownership's destroy right "is inherent in being the object's owner, and cannot be granted or revoked in itself" ([PostgreSQL — §5.8](https://www.postgresql.org/docs/current/ddl-priv.html)) — no `REVOKE` can stop an owner from `DROP`-ing. Trying to revoke the owner's own grants fails too, because "owners are always treated as holding all grant options."

```sql
-- WRONG — app role owns the table; it can DROP/ALTER no matter what you REVOKE

-- RIGHT — separate the DDL/migration owner from the runtime app role
ALTER TABLE orders OWNER TO schema_owner;                 -- migrations only
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO app_rw; -- runtime: data only
```

*Source: [PostgreSQL — Privileges (§5.8)](https://www.postgresql.org/docs/current/ddl-priv.html). Depth: this skill, §5.*

---

## 4. Granting privileges to `PUBLIC`

**The problem:** To "make sure everything can read it," code grants to `PUBLIC`, which is "an implicitly defined group that always includes all roles," including future ones ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)). The grant silently applies to every account that will ever exist.

```sql
-- WRONG — every present and future role can read this, forever
GRANT SELECT ON salaries TO PUBLIC;

-- RIGHT — name the role that needs it
GRANT SELECT ON salaries TO payroll_role;
```

*Source: [PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html). Depth: this skill, §6.*

---

## 5. Handing out `WITH GRANT OPTION` casually

**The problem:** Adding `WITH GRANT OPTION` to an ordinary grant lets "the recipient of the privilege ... in turn grant it to others" ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)), so privilege distribution is no longer centrally controlled — and revoking it later cascades to everyone downstream ([PostgreSQL — §5.8](https://www.postgresql.org/docs/current/ddl-priv.html)).

```sql
-- WRONG — the app can now re-grant SELECT to anyone
GRANT SELECT ON orders TO app_rw WITH GRANT OPTION;

-- RIGHT — the app just uses the privilege; granting stays an admin act
GRANT SELECT ON orders TO app_rw;
```

*Source: [PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html); [PostgreSQL — §5.8](https://www.postgresql.org/docs/current/ddl-priv.html). Depth: this skill, §4.*

---

## 6. Granting privileges to each user instead of a group role

**The problem:** Privileges are granted directly to every individual login, so adding or removing a person means re-running grants everywhere and access drifts. Group roles fix this: "privileges can be granted to, or revoked from, a group as a whole" ([PostgreSQL — Role Membership](https://www.postgresql.org/docs/current/role-membership.html)).

```sql
-- WRONG — privileges scattered across individuals; unmanageable over time
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO alice;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO bob;

-- RIGHT — grant once to a group role, then add members
CREATE ROLE writers;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO writers;
GRANT writers TO alice, bob;        -- GRANT role TO role
```

*Source: [PostgreSQL — Role Membership](https://www.postgresql.org/docs/current/role-membership.html). Depth: this skill, §3.*

---

## 7. Writing `GRANT`/`CREATE ROLE` for SQLite

**The problem:** A SQLite setup script includes `CREATE ROLE`/`GRANT`. SQLite has no privilege model: "The GRANT and REVOKE commands ... are not implemented because they would be meaningless for an embedded database engine," and "the only access permissions that can be applied are the normal file access permissions of the underlying operating system" ([SQLite — Omitted Features](https://www.sqlite.org/omitted.html)).

```sql
-- WRONG — these statements are not implemented in SQLite and will error
CREATE ROLE app_rw;
GRANT SELECT ON orders TO app_rw;
```

```text
-- RIGHT — control access with OS file permissions on the database file
chmod 600 app.db          # only the owning OS user can read/write
chown appuser app.db
```

*Source: [SQLite — Omitted Features](https://www.sqlite.org/omitted.html). Depth: this skill, §7; dialect map owned by `sql-standard-vs-dialect-map`.*

---

## 8. Leaving default `PUBLIC` privileges in place

**The problem:** Code grants nothing to `PUBLIC` but assumes that means nothing is exposed. Some privileges are granted to `PUBLIC` *by default* at creation — including `CONNECT` and `TEMPORARY` on databases and `EXECUTE` on functions/procedures ([PostgreSQL — §5.8](https://www.postgresql.org/docs/current/ddl-priv.html)). Tightening access means actively `REVOKE`-ing those defaults.

```sql
-- WRONG (assumption) — "I never granted CONNECT to PUBLIC, so the DB is private"
-- ...but every role can still CONNECT and EXECUTE functions by default.

-- RIGHT — revoke the default PUBLIC privileges, then grant deliberately
REVOKE CONNECT ON DATABASE appdb FROM PUBLIC;
REVOKE EXECUTE ON ALL FUNCTIONS IN SCHEMA public FROM PUBLIC;
GRANT CONNECT ON DATABASE appdb TO app_rw;
```

*Source: [PostgreSQL — Privileges (§5.8)](https://www.postgresql.org/docs/current/ddl-priv.html). Depth: this skill, §6.*
