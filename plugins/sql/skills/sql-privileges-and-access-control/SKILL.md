---
name: sql-privileges-and-access-control
description: >-
  Guides the standard SQL access-control model — `GRANT` adds privileges, `REVOKE` removes them, roles are the grant targets (a role is a user or a group), and access is default-deny so every privilege must be explicitly granted. Drives toward least privilege — the application connects as a dedicated login role holding ONLY the privileges it needs (`SELECT/INSERT/UPDATE/DELETE` on specific tables), never as a superuser (which bypasses all permission checks) and never with `GRANT ALL` broadly. Auto-invokes when writing or editing `GRANT`/`REVOKE`/`CREATE ROLE`/`CREATE USER`, designing database user/role setup, deciding which credentials an application connects with, or on "least privilege" / "who can access this" / "set up a database user" requests.
allowed-tools: Read, Glob, Grep
---

# SQL Privileges and Access Control

> "For most kinds of objects, the initial state is that only the owner (or a superuser) can do anything with the object. To allow other roles to use it, _privileges_ must be granted."
> — [PostgreSQL — Privileges (§5.8)](https://www.postgresql.org/docs/current/ddl-priv.html)

> "A database superuser bypasses all permission checks, except the right to log in."
> — [PostgreSQL — Role Attributes](https://www.postgresql.org/docs/current/role-attributes.html)

Access control in SQL is **default-deny**: a freshly created object can be touched only by its owner or a superuser, and every other access is opt-in through an explicit `GRANT` ([PostgreSQL — §5.8](https://www.postgresql.org/docs/current/ddl-priv.html)). That single fact decides the whole discipline below. The privilege your application *doesn't* hold is the one an attacker can't abuse and a bug can't trigger. This skill builds on the set-and-null floor in `sql-relational-and-null-discipline` and is the access-control half of the security story whose other half — keeping untrusted input out of the query text — lives in `sql-injection-and-parameterization`.

---

## 1. The Standard Model: `GRANT` Adds, `REVOKE` Removes

`GRANT` "gives specific privileges on a database object to one or more roles. These privileges are added to those already granted, if any" — grants are **additive**, and `REVOKE` is the only way to take one back ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)). The standard object privileges are named, granular verbs: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER`, `EXECUTE`, `USAGE`, and more ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)). The recipient of every grant is a **role** — and "a role can be thought of as either a database user, or a group of database users" ([PostgreSQL — Database Roles](https://www.postgresql.org/docs/current/user-manag.html)).

```sql
-- Grant exactly the verbs a reader needs on exactly one table
GRANT SELECT ON orders TO reporting_role;

-- Take it back
REVOKE SELECT ON orders FROM reporting_role;
```

`GRANT ALL PRIVILEGES` exists, but note the standard spelling requires the word: "The `PRIVILEGES` key word is optional in PostgreSQL, though it is required by strict SQL" ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)). `ALL` is almost never what an application should receive — see §2.

---

## 2. Least Privilege: the Application Role (the centerpiece)

The single most consequential access-control decision is **what role your application connects as**. The default-deny model (§1) only protects you if the connecting role actually lacks dangerous privileges. Two failure modes recur in generated setups, and both hand an attacker the keys.

```sql
-- WRONG — the app connects as a superuser.
-- A superuser "bypasses all permission checks" — GRANT/REVOKE are meaningless on
-- this connection, so a single SQL injection becomes total database compromise.
--   postgres://postgres:secret@db/app      <- superuser credentials in the app

-- WRONG — a dedicated role, but granted everything on everything.
-- DROP, TRUNCATE, schema changes: an injection or a bug can now destroy the schema.
CREATE ROLE app LOGIN PASSWORD '...';
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app;
```

```sql
-- RIGHT — a dedicated LOGIN role holding ONLY the verbs the app actually issues,
-- on ONLY the tables it touches. It is NOT the owner (see §5), so it cannot DROP
-- or ALTER these tables no matter what SQL reaches the server.
CREATE ROLE app_rw LOGIN PASSWORD '...';
GRANT CONNECT ON DATABASE appdb TO app_rw;
GRANT USAGE  ON SCHEMA public  TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders, customers, line_items TO app_rw;

-- A read-only path (reports, replicas) gets even less:
CREATE ROLE app_ro LOGIN PASSWORD '...';
GRANT SELECT ON orders, customers, line_items TO app_ro;
```

"Only roles that have the `LOGIN` attribute can be used as the initial role name for a database connection" ([PostgreSQL — Role Attributes](https://www.postgresql.org/docs/current/role-attributes.html)), so the connecting role is a deliberate, minimal login role — not the owner, not a superuser. The blast radius of any injection (`sql-injection-and-parameterization`) or application bug is capped at exactly the privileges you listed.

---

## 3. Roles and Role Hierarchies

Privileges are easier to manage when granted to a **group role** once, rather than to each user individually: "privileges can be granted to, or revoked from, a group as a whole ... by creating a role that represents the group, and then granting _membership_ in the group role to individual user roles" ([PostgreSQL — Role Membership](https://www.postgresql.org/docs/current/role-membership.html)). The same keyword `GRANT` does this — but its role form "grants membership in a role to one or more other roles," and membership "cannot be granted to `PUBLIC`" ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)).

```sql
-- A privilege-holding group role (no LOGIN — it is a container, not a credential)
CREATE ROLE writers;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO writers;

-- Login roles become members; they pick up the group's privileges
CREATE ROLE alice LOGIN PASSWORD '...';
GRANT writers TO alice;          -- alice now reads/writes orders via the group
```

A member exercises the group's privileges either automatically (the `INHERIT` option, PostgreSQL's default) or by an explicit `SET ROLE` to "temporarily 'become' the group role" ([PostgreSQL — Role Membership](https://www.postgresql.org/docs/current/role-membership.html)). Re-org a permission once on `writers` and every member follows.

---

## 4. `WITH GRANT OPTION` — Delegated Granting and Its Cascade Risk

By default a grantee can *use* a privilege but not *re-grant* it. "If `WITH GRANT OPTION` is specified, the recipient of the privilege can in turn grant it to others" ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)). That delegates your authority — and the delegation propagates: "If the grant option is subsequently revoked then all who received the privilege from that recipient (directly or through a chain of grants) will lose the privilege" ([PostgreSQL — §5.8](https://www.postgresql.org/docs/current/ddl-priv.html)).

```sql
-- WRONG — handing an application or ordinary user the power to spread privileges.
-- Now who-can-see-orders is no longer something you control centrally.
GRANT SELECT ON orders TO app_rw WITH GRANT OPTION;

-- RIGHT — the app just uses the privilege; granting stays an admin act.
GRANT SELECT ON orders TO app_rw;
```

Reserve `WITH GRANT OPTION` for deliberate administrative delegation, and remember that revoking it later cascades to everyone downstream of that grant.

---

## 5. Ownership Is Not the Same as Grants

A grant gives *ordinary* privileges. Ownership gives something `GRANT`/`REVOKE` cannot touch: "The right to modify or destroy an object is inherent in being the object's owner, and cannot be granted or revoked in itself" ([PostgreSQL — §5.8](https://www.postgresql.org/docs/current/ddl-priv.html)). You cannot `REVOKE DROP` from an owner — there is no such privilege to revoke. Worse, trying to neuter the owner by revoking its own grants doesn't work: "owners are always treated as holding all grant options, so they can always re-grant their own privileges" ([PostgreSQL — §5.8](https://www.postgresql.org/docs/current/ddl-priv.html)).

```sql
-- WRONG — the app connects as the role that owns the tables.
-- It can DROP/ALTER/TRUNCATE them at will; no REVOKE can stop it.
--   migrations & app share the same owner credentials

-- RIGHT — owner (used only for migrations/DDL) is distinct from the app role.
ALTER TABLE orders OWNER TO schema_owner;   -- owns the object; runs migrations
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO app_rw;  -- app: data only
```

This is *the* reason the §2 app role must not be the object owner: ownership is an un-revokable backdoor to `DROP`. Separate the migration/DDL identity from the runtime application identity.

---

## 6. The `PUBLIC` Pseudo-Role Pitfalls

`PUBLIC` is "an implicitly defined group that always includes all roles," including ones "that might be created later" ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)). A grant to `PUBLIC` therefore leaks to every present and future role at once — the opposite of least privilege.

```sql
-- WRONG — every current and future role can now read this table, forever
GRANT SELECT ON salaries TO PUBLIC;

-- RIGHT — name the role(s) that genuinely need it
GRANT SELECT ON salaries TO payroll_role;
```

Two more `PUBLIC` facts worth holding: some privileges are granted to `PUBLIC` *by default* at creation — notably `CONNECT` and `TEMPORARY` on databases and `EXECUTE` on functions and procedures ([PostgreSQL — §5.8](https://www.postgresql.org/docs/current/ddl-priv.html)) — so tightening access can mean `REVOKE`-ing those defaults, not just refraining from granting. And "grant options cannot be granted to `PUBLIC`" ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)), so §4 delegation never applies to `PUBLIC`.

---

## 7. Portability

`GRANT`, `REVOKE`, and the role concept are core SQL — privilege grants appear in SQL-92 and roles in SQL:1999 — and the `ALL PRIVILEGES` spelling (with the word `PRIVILEGES`) "is required by strict SQL" ([PostgreSQL — GRANT](https://www.postgresql.org/docs/current/sql-grant.html)). But the model is not uniform across engines:

- **SQLite has NO privilege model at all.** "The GRANT and REVOKE commands ... are not implemented because they would be meaningless for an embedded database engine," and "since SQLite reads and writes an ordinary disk file, the only access permissions that can be applied are the normal file access permissions of the underlying operating system" ([SQLite — Omitted Features](https://www.sqlite.org/omitted.html)). With SQLite, access control = OS file permissions on the `.db` file; there are no roles, no `GRANT`, no per-table privileges. Do not write `GRANT`/`CREATE ROLE` for SQLite.
- **Privilege and role spellings vary.** MySQL ties privileges to `user@host` accounts and added SQL roles only in 8.0; Oracle and SQL Server use `CREATE USER`/`CREATE ROLE` with their own privilege names and system-privilege catalogs. The named privilege set, the `PUBLIC` semantics, and role syntax differ in the details.

For the cross-engine privilege-name and role-syntax map, route to **`sql-standard-vs-dialect-map`**.

---

## 8. Who Suffers When Access Control Is Wrong

Privilege mistakes don't error at deploy time — the app works fine. They surface as a catastrophe, and someone specific pays:

- The **incident responder** the night a single SQL injection became a full database dump and a dropped schema — because the application connected as a superuser, where (by definition) "a database superuser bypasses all permission checks" ([PostgreSQL — Role Attributes](https://www.postgresql.org/docs/current/role-attributes.html)). A scoped `app_rw` role (§2) would have capped the breach at four tables' worth of CRUD; instead the injection vector (`sql-injection-and-parameterization`) met an all-powerful credential and the two multiplied.
- The **on-call engineer** restoring from backup after a buggy code path — or an attacker — issued a `DROP TABLE`, possible only because the app role *owned* the tables and ownership's destroy right "cannot be granted or revoked" (§5). A read/write grant on a non-owner role has no `DROP`; the disaster was a privilege the app never needed but was handed anyway.
- The **auditor** who finds that `salaries` is readable by every account because someone once ran `GRANT SELECT ... TO PUBLIC` (§6), and the "later" roles the grant silently swept in now number in the hundreds.

Least privilege is empathy in infrastructure: the privilege you decline to grant is the breach that can't happen.

---

## 9. Routing to Related Skills

- `sql-relational-and-null-discipline` — the policy root (set semantics, three-valued logic) every SQL skill builds on.
- `sql-injection-and-parameterization` — the *other* half of SQL security: keeping untrusted input out of query text. Least privilege (here) caps the blast radius; parameterization removes the vector. Use both.
- `sql-views-and-introspection` — column/row-restricted access via views and the catalog tables that reveal who holds which privilege.
- `sql-standard-vs-dialect-map` — per-engine privilege names, role syntax, and the SQLite "no privileges, file permissions only" deviation.

---

## 10. Reference Files

High-frequency access-control anti-patterns in LLM-generated SQL, each with wrong/right code and a primary-source citation:

[references/common-mistakes.md](references/common-mistakes.md)

Source provenance for every claim in this skill:

[references/sources.yaml](references/sources.yaml)
