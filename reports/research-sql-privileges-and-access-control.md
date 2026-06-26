# Research: sql-privileges-and-access-control

Research backing the skill `sql-privileges-and-access-control` (skill #24 in `reports/skill-plan-sql.md`).
Each entry: source URL, section, verbatim quote, and why it matters for the skill.

Accessed: 2026-06-26.

---

## Source 1 — PostgreSQL: GRANT
URL: https://www.postgresql.org/docs/current/sql-grant.html

### 1a. GRANT on database objects (the object-privilege model)
Section: "GRANT on Database Objects"
> "This variant of the `GRANT` command gives specific privileges on a database object to one or more roles. These privileges are added to those already granted, if any."

Why it matters: establishes that `GRANT` is *additive* — grants accumulate. This is the core of the standard model and the reason `REVOKE` is needed to take privileges away; the skill's "standard model" section opens with this.

### 1b. The object privilege types
Section: "The possible privileges are:"
> "`SELECT` `INSERT` `UPDATE` `DELETE` `TRUNCATE` `REFERENCES` `TRIGGER` `CREATE` `CONNECT` `TEMPORARY` `EXECUTE` `USAGE` `SET` `ALTER SYSTEM` `MAINTAIN`"

> "`ALL PRIVILEGES` — Grant all of the privileges available for the object's type. The `PRIVILEGES` key word is optional in PostgreSQL, though it is required by strict SQL."

Why it matters: the named privilege list is the vocabulary the app-role section uses (`SELECT, INSERT, UPDATE, DELETE` on specific tables). `ALL PRIVILEGES` is the anti-pattern centerpiece — it grants more than an app ever needs. The note "required by strict SQL" supports the portability claim that standard SQL spells it `ALL PRIVILEGES`.

### 1c. WITH GRANT OPTION
Section: "GRANT on Database Objects"
> "If `WITH GRANT OPTION` is specified, the recipient of the privilege can in turn grant it to others. Without a grant option, the recipient cannot do that. Grant options cannot be granted to `PUBLIC`."

Why it matters: directly defines the WITH GRANT OPTION section. The "cannot be granted to PUBLIC" clause is a precise, citable fact for the PUBLIC pitfalls section.

### 1d. GRANT on roles (granting a role to a role)
Section: "GRANT on Roles"
> "This variant of the `GRANT` command grants membership in a role to one or more other roles, and the modification of membership options `SET`, `INHERIT`, and `ADMIN`..."

> "GRANT `role_name` [, ...] TO `role_specification` [, ...]"

> "Unlike the case with privileges, membership in a role cannot be granted to `PUBLIC`."

Why it matters: this is the syntactic basis for "roles & hierarchies" — `GRANT role TO role`. Shows the same keyword (`GRANT`) does two distinct jobs: object privileges vs role membership.

### 1e. GRANTED BY
Section: "GRANT on Database Objects"
> "If `GRANTED BY` is specified, the specified grantor must be the current user. This clause is currently present in this form only for SQL compatibility."

Why it matters: minor, supports the SQL-standard framing. Not central.

---

## Source 2 — PostgreSQL: Privileges (DDL)
URL: https://www.postgresql.org/docs/current/ddl-priv.html

### 2a. Ownership and the default-deny posture
Section: "5.8. Privileges"
> "When an object is created, it is assigned an owner. The owner is normally the role that executed the creation statement. For most kinds of objects, the initial state is that only the owner (or a superuser) can do anything with the object. To allow other roles to use it, _privileges_ must be granted."

Why it matters: this is the foundation for "ownership vs grants" and for least privilege. Default state = nobody but owner/superuser can touch the object; access is opt-in via GRANT. The skill leans on this to motivate the scoped app role.

### 2b. The right to modify/destroy is inherent in ownership
Section: "5.8. Privileges"
> "The right to modify or destroy an object is inherent in being the object's owner, and cannot be granted or revoked in itself."

Why it matters: THE load-bearing quote for "ownership ≠ grants." An owner can `DROP`/`ALTER` the object regardless of any GRANT/REVOKE. This is exactly why the app must NOT connect as the table owner — you cannot REVOKE `DROP` from an owner. Centerpiece support.

### 2c. Owner can revoke their own ordinary privileges but keeps grant options
Section: "5.8. Privileges"
> "An object's owner can choose to revoke their own ordinary privileges, for example to make a table read-only for themselves as well as others. But owners are always treated as holding all grant options, so they can always re-grant their own privileges."

Why it matters: reinforces 2b — even revoking your own SELECT doesn't remove the inherent owner power; you can always re-grant. So "make the owner safe by revoking from it" is a false sense of security. Use a separate non-owner role instead.

### 2d. PUBLIC default privileges
Section: "5.8. Privileges"
> "PostgreSQL grants privileges on some types of objects to `PUBLIC` by default when the objects are created. No privileges are granted to `PUBLIC` by default on tables, table columns, sequences, foreign data wrappers, foreign servers, large objects, schemas, tablespaces, or configuration parameters. For other types of objects, the default privileges granted to `PUBLIC` are as follows: `CONNECT` and `TEMPORARY` (create temporary tables) privileges for databases; `EXECUTE` privilege for functions and procedures; and `USAGE` privilege for languages and data types (including domains)."

Why it matters: PUBLIC pitfalls section. By default PUBLIC can `CONNECT` to any database and `EXECUTE` any function — a real over-exposure. `GRANT ... TO PUBLIC` hands a privilege to every current and future role. Citable basis for "revoke PUBLIC defaults / never grant to PUBLIC casually."

### 2e. WITH GRANT OPTION cascade on revoke
Section: "5.8. Privileges"
> "However, it is possible to grant a privilege \"with grant option\", which gives the recipient the right to grant it in turn to others. If the grant option is subsequently revoked then all who received the privilege from that recipient (directly or through a chain of grants) will lose the privilege."

Why it matters: the RISK half of the WITH GRANT OPTION section — grant chains. Once you hand out grant option, privilege propagation is no longer centrally controlled, and revoking cascades unpredictably.

### 2f. Only owner/superuser grants
Section: "5.8. Privileges"
> "Ordinarily, only the object's owner (or a superuser) can grant or revoke privileges on an object."

Why it matters: ties grant authority back to ownership; supports the model section.

---

## Source 3 — PostgreSQL: Database Roles / Role Attributes / Role Membership
URLs:
- https://www.postgresql.org/docs/current/user-manag.html (chapter intro)
- https://www.postgresql.org/docs/current/database-roles.html
- https://www.postgresql.org/docs/current/role-attributes.html
- https://www.postgresql.org/docs/current/role-membership.html

### 3a. Roles subsume users and groups
Section: "21. Database Roles" (intro)
> "PostgreSQL manages database access permissions using the concept of _roles_. A role can be thought of as either a database user, or a group of database users, depending on how the role is set up."

> "The concept of roles subsumes the concepts of 'users' and 'groups'. In PostgreSQL versions before 8.1, users and groups were distinct kinds of entities, but now there are only roles."

Why it matters: a "role" is the single primitive — user OR group. This is the standard-SQL role concept; the skill frames everything around roles, not "users."

### 3b. Roles separate from OS users
Section: "21.1. Database Roles"
> "Database roles are conceptually completely separate from operating system users. In practice it might be convenient to maintain a correspondence, but this is not required."

Why it matters: the DB privilege model is independent of the filesystem — directly contrasts with SQLite, where access control IS filesystem permissions. Sets up the SQLite portability flag.

### 3c. CREATE ROLE
Section: "21.1. Database Roles"
> "To create a role use the `CREATE ROLE` SQL command: CREATE ROLE `name`;"

(Plus: "you will usually want to add additional options, such as `LOGIN`, to the command.")

Why it matters: `CREATE ROLE` is the standard role-creation statement the skill demonstrates.

### 3d. LOGIN attribute
Section: "21.2. Role Attributes"
> "Only roles that have the `LOGIN` attribute can be used as the initial role name for a database connection."

Why it matters: distinguishes login roles (connect with credentials) from group roles (privilege containers). The app connects as a LOGIN role; that role is a member of a group role holding the privileges.

### 3e. Superuser bypasses all checks
Section: "21.2. Role Attributes"
> "A database superuser bypasses all permission checks, except the right to log in."

Why it matters: THE quote for the "never connect as superuser" anti-pattern. A superuser connection means GRANT/REVOKE are irrelevant — every privilege check is skipped. An injection on a superuser connection is total compromise. Centerpiece + "Who suffers" support.

### 3f. CREATEDB / CREATEROLE are explicit, separate powers
Section: "21.2. Role Attributes"
> "A role must be explicitly given permission to create databases (except for superusers, since those bypass all permission checks)."

> "A role must be explicitly given permission to create more roles (except for superusers...)."

Why it matters: shows privileges are granular and opt-in — you grant exactly the administrative powers needed, never the blanket superuser bit.

### 3g. Group roles ease privilege management
Section: "21.3. Role Membership"
> "It is frequently convenient to group users together to ease management of privileges: that way, privileges can be granted to, or revoked from, a group as a whole."

> "In PostgreSQL this is done by creating a role that represents the group, and then granting _membership_ in the group role to individual user roles."

> "GRANT `group_role` TO `role1`, ... ;"

Why it matters: the practical motivation for role hierarchies — grant privileges to a group role once, add/remove members. The skill's "roles & hierarchies" section uses this exact pattern.

### 3h. How members use group privileges — SET ROLE vs INHERIT
Section: "21.3. Role Membership"
> "First, member roles that have been granted membership with the `SET` option can do `SET ROLE` to temporarily \"become\" the group role."

> "Second, member roles that have been granted membership with the `INHERIT` option automatically have use of the privileges of those directly or indirectly a member of, though the chain stops at memberships lacking the inherit option."

> "However, PostgreSQL defaults to giving all roles the `INHERIT` attribute..."

Why it matters: explains how a login role actually exercises a group role's privileges (inherit automatically, or SET ROLE explicitly). Supporting depth for the hierarchies section.

---

## Synthesis — what the skill must nail (from the spec)

- (a) **GRANT/REVOKE = the standard SQL security model.** Additive grants (1a), named privilege types (1b), `ALL PRIVILEGES` required-word in strict SQL (1b). REVOKE undoes. SQL-92/99 feature.
- (b) **Least privilege CENTERPIECE.** Default-deny (2a) means access is opt-in. WRONG: app connects as superuser (3e — bypasses ALL checks) or with `GRANT ALL`. RIGHT: dedicated LOGIN role (3d) holding only `SELECT/INSERT/UPDATE/DELETE` (1b) on specific tables, never the owner.
- (c) **Roles & hierarchies** (3a, 3g, 3h): `GRANT role TO role` (1d), group roles hold privileges, members inherit.
- (d) **WITH GRANT OPTION + risk** (1c, 2e): recipient can re-grant; revoke cascades down the chain.
- (e) **Ownership ≠ grants** (2a, 2b, 2c): owner has inherent modify/destroy right that cannot be revoked — so the app must not be the owner.
- (f) **PUBLIC pitfalls** (1c, 2d): grants to PUBLIC hit every current/future role; default PUBLIC CONNECT/EXECUTE; grant options can't go to PUBLIC.
- **SQLite has NO privilege model** (3b contrast): access control is filesystem permissions on the DB file. Flag prominently; route to standard-vs-dialect-map.
