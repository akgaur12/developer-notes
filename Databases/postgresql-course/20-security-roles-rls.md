# Chapter 20 — Security: Roles, Privileges & Row-Level Security

## Where you are
Your database works and performs. Now you make it **safe**: control who can connect, what they can read and write, and even which *rows* they can see. Security in Postgres is layered and enforced by the engine itself — done right, no application bug can leak data the database forbids. This connects to authentication (Chapter 02), views-for-security (Chapter 10), and least-privilege operations (Chapter 24).

> **The "why":** Defense in depth. Application code is the first gate, but it has bugs and there's often more than one client touching a database. Privileges and row-level policies enforced *in the database* are a backstop that holds even when the app is compromised or wrong. The principle throughout is **least privilege** — grant the minimum needed, nothing more.

---

## 1. Roles — the unified concept of users and groups
In Postgres, **everything is a role.** A "user" is just a role with `LOGIN`; a "group" is a role without it that you grant membership to.
```sql
CREATE ROLE app_read NOLOGIN;                          -- a group role (no direct login)
CREATE ROLE analytics_user LOGIN PASSWORD 'secret';    -- a login role (a "user")
GRANT app_read TO analytics_user;                      -- analytics_user inherits app_read's privileges
```
Key role attributes: `LOGIN`, `SUPERUSER` (bypasses all permission checks — give to almost no one), `CREATEDB`, `CREATEROLE`, `REPLICATION` (Chapter 21), `PASSWORD`. The bootstrap `postgres` superuser should be used sparingly; create purpose-specific roles instead.

## 2. Privileges — GRANT and REVOKE
Privileges are granted per object (table, view, sequence, schema, database, function):
```sql
GRANT SELECT ON employees TO app_read;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO app_write;
GRANT USAGE ON SCHEMA app TO app_read;                 -- needed to "see into" a schema
REVOKE INSERT ON orders FROM app_write;
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_read;  -- bulk grant on existing tables
```
> **The gotcha everyone hits: future tables aren't covered.** `GRANT ... ON ALL TABLES` only affects tables that exist *right now*. New tables created later have no grant. Fix it with **default privileges**, which apply automatically to objects created in future:
> ```sql
> ALTER DEFAULT PRIVILEGES IN SCHEMA app
>   GRANT SELECT ON TABLES TO app_read;   -- every future table is auto-granted
> ```

**The `public` schema caution:** historically every user could create objects in the `public` schema. Modern Postgres (v15+) tightened this default, but on older clusters you should `REVOKE CREATE ON SCHEMA public FROM PUBLIC;` as a hardening step. (`PUBLIC` here is a special pseudo-role meaning "everyone.")

## 3. A least-privilege pattern for applications
```sql
-- 1. a group role with exactly the app's needs:
CREATE ROLE app_rw NOLOGIN;
GRANT USAGE ON SCHEMA app TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_rw;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app TO app_rw;   -- needed for IDENTITY/serial inserts
ALTER DEFAULT PRIVILEGES IN SCHEMA app GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;

-- 2. the actual login the app connects as, owning nothing, member of the group:
CREATE ROLE app_login LOGIN PASSWORD '...';
GRANT app_rw TO app_login;
```
The app connects as a role that **cannot** create/drop tables, alter schema, or read other schemas — so a SQL injection or app bug is contained to data operations on intended tables. Migrations run as a separate, more-privileged role.

## 4. Authentication recap (Chapter 02) and encryption
- `pg_hba.conf` controls *who can connect from where and how* — use **`scram-sha-256`** (the modern default; never `trust` or legacy `md5` on anything real).
- Require **TLS/SSL** for connections over a network (`ssl = on`, and `hostssl` rules in `pg_hba.conf`) so credentials and data aren't sent in clear text.
- Don't store secrets in the DB in plaintext; the `pgcrypto` extension provides hashing/encryption functions when you must.

## 5. Row-Level Security (RLS) — per-row access control
Privileges control *which tables* a role can touch. **RLS** controls *which rows within a table* a role can see or modify — enforced by the engine on every query. This is how you build multi-tenant systems where each tenant sees only their own rows, with the guarantee living in the database, not scattered through app code.
```sql
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;        -- now NO rows are visible until a policy allows them

CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.current_tenant')::bigint);
-- USING  → which existing rows are visible/affected (for SELECT/UPDATE/DELETE)
-- WITH CHECK → which new rows may be written (for INSERT/UPDATE)
```
The app sets the context per connection/transaction, and Postgres silently filters:
```sql
SET app.current_tenant = '42';
SELECT * FROM documents;     -- returns ONLY tenant 42's rows, automatically and unavoidably
```
```
Without RLS: every query must remember `WHERE tenant_id = ?` — miss it once = data leak.
With RLS:    the engine appends the policy to EVERY query. A forgotten filter can't leak data.
```
Notes: table **owners and superusers bypass RLS by default** (use `FORCE ROW LEVEL SECURITY` to apply it to the owner too); you can write separate policies per command (`FOR SELECT`, `FOR INSERT`, …) and per role. RLS is powerful but adds a predicate to every query — index the policy column (e.g. `tenant_id`) so it stays fast.

## 6. Auditing and visibility
- `\du` lists roles and their attributes; `\dp` (or `\z`) shows table privileges; `\drg` shows role memberships.
- Log connections/statements via `log_connections`, `log_statement` for an audit trail; the `pgaudit` extension gives detailed, compliance-grade auditing.

---

## Summary
- **Everything is a role**; a user = role with `LOGIN`, a group = role you `GRANT` to others. Avoid `SUPERUSER`; follow **least privilege**.
- **GRANT/REVOKE** per object; **`ALTER DEFAULT PRIVILEGES`** is required to cover *future* tables (a near-universal gotcha). Harden the `public` schema.
- Build apps to connect as a **low-privilege role** that can't alter schema — containing injection/bugs; run migrations as a separate privileged role.
- Authenticate with **`scram-sha-256`**, require **TLS** over networks, and never store plaintext secrets.
- **Row-Level Security** filters *rows* per role via `USING`/`WITH CHECK` policies — ideal for multi-tenant isolation enforced by the engine; owners bypass it unless `FORCE`d, and you should index the policy column.

## Test your understanding
1. In Postgres, what's the real difference between a "user" and a "group"?
2. You ran `GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_read`, but a table created next week isn't readable. Why, and what's the fix?
3. Describe a least-privilege setup so that a SQL-injection bug in the app can't drop tables or read other schemas.
4. What does enabling RLS with no policy do to a table's visibility, and what's the difference between `USING` and `WITH CHECK`?
5. Why do RLS policies need an index on the policy column, and who bypasses RLS by default?

## Hands-on exercise
1. Create a `NOLOGIN` group role with `SELECT`-only on a schema, a `LOGIN` role that's a member, and connect as the login role to confirm it can read but not write.
2. Set `ALTER DEFAULT PRIVILEGES`, create a new table, and confirm the read role can see it without an explicit grant.
3. Try a forbidden action (e.g. `DROP TABLE`) as the low-privilege role and confirm it's denied.
4. Enable RLS on a table with a `tenant_id`, write a `USING` policy keyed off `current_setting('app.current_tenant')`, set the tenant, and prove queries return only that tenant's rows.
5. Add a `WITH CHECK` clause and confirm an attempt to insert a row for a different tenant is rejected; then index `tenant_id` and check the plan.
