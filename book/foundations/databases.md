---
title: Databases and Migrations
summary: "Keeping a relational database in step with the code: migrations, roles and privileges, access paths, and copying data between environments."
references:
  twelve-admin: "A. Wiggins, *The Twelve-Factor App: XII. Admin processes*, <https://12factor.net/admin-processes>."
  pg-locks: "PostgreSQL Global Development Group, *Explicit Locking*, PostgreSQL documentation, section 13.3, <https://www.postgresql.org/docs/current/explicit-locking.html>."
  pg-createindex: "PostgreSQL Global Development Group, *CREATE INDEX*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/sql-createindex.html>."
  pg-admin-fn: "PostgreSQL Global Development Group, *System Administration Functions* (advisory locks), PostgreSQL documentation, <https://www.postgresql.org/docs/current/functions-admin.html>."
  pg-priv: "PostgreSQL Global Development Group, *Privileges*, PostgreSQL documentation, section 5.8, <https://www.postgresql.org/docs/current/ddl-priv.html>."
  pg-createfunction: "PostgreSQL Global Development Group, *CREATE FUNCTION*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/sql-createfunction.html>."
  supabase-api: "Supabase, *Securing your API*, Supabase documentation, <https://supabase.com/docs/guides/api/securing-your-api>."
  pg-default-priv: "PostgreSQL Global Development Group, *ALTER DEFAULT PRIVILEGES*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/sql-alterdefaultprivileges.html>."
  pg-info-fn: "PostgreSQL Global Development Group, *System Information Functions and Operators*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/functions-info.html>."
  postgrest-tx: "PostgREST, *Transactions*, PostgREST documentation, <https://docs.postgrest.org/en/stable/references/transactions.html>."
  postgrest-fn: "PostgREST, *Functions as RPC*, PostgREST documentation, <https://docs.postgrest.org/en/stable/references/api/functions.html>."
  postgrest-cache: "PostgREST, *Schema Cache*, PostgREST documentation, <https://docs.postgrest.org/en/stable/references/schema_cache.html>."
  twelve-backing: "A. Wiggins, *The Twelve-Factor App: IV. Backing services*, <https://12factor.net/backing-services>."
  pg-iso: "PostgreSQL Global Development Group, *Transaction Isolation*, PostgreSQL documentation, section 13.2, <https://www.postgresql.org/docs/current/transaction-iso.html>."
  pg-aggregate: "PostgreSQL Global Development Group, *Aggregate Functions*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/functions-aggregate.html>."
  pg-json: "PostgreSQL Global Development Group, *JSON Functions and Operators*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/functions-json.html>."
  pg-dump: "PostgreSQL Global Development Group, *pg_dump*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/app-pgdump.html>."
  pg-client-config: "PostgreSQL Global Development Group, *Client Connection Defaults* (`session_replication_role`), PostgreSQL documentation, <https://www.postgresql.org/docs/current/runtime-config-client.html>."
  pg-altertable: "PostgreSQL Global Development Group, *ALTER TABLE*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/sql-altertable.html>."
  pg-set-constraints: "PostgreSQL Global Development Group, *SET CONSTRAINTS*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/sql-set-constraints.html>."
  pg-generated: "PostgreSQL Global Development Group, *Generated Columns*, PostgreSQL documentation, section 5.4, <https://www.postgresql.org/docs/current/ddl-generated-columns.html>."
  pg-insert: "PostgreSQL Global Development Group, *INSERT*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/sql-insert.html>."
  pg-sequence-fn: "PostgreSQL Global Development Group, *Sequence Manipulation Functions*, PostgreSQL documentation, <https://www.postgresql.org/docs/current/functions-sequence.html>."
---

## Migrations

The database outlives every version of the code that uses it, so its schema has
to change in step with the code without ever being rebuilt. The established way
to do this is to express the schema as a sequence of *migrations*: SQL files,
numbered in the order they are to run, each taking the schema from one state to
the next. The files are code, kept in the repository; the database records which
of them it has applied.

!!! definition "Migration ledger"
    A table in the database itself listing every migration applied to it,
    with the file name, a checksum of the file's contents and the time it was
    applied. The migration runner compares the files in the repository with
    the ledger, applies the missing ones in order, and refuses to run if a
    recorded file's checksum no longer matches, since that means an applied
    migration was edited.

Applied migrations are immutable. A mistake in one that has run anywhere is
corrected by a new migration, never by editing the old file, because the old
file has already done its work in some database and editing it would make the
repository describe a history that did not happen. The checksum turns this
convention into a check.

PostgreSQL runs most schema changes inside transactions, so each migration
applies completely or not at all. A few statements cannot run in a transaction
block, notably `CREATE INDEX CONCURRENTLY` [@pg-createindex], and need a
migration of their own that the runner executes outside one. Migrations run as
a one-off administrative process against the release's database, with the same
code and configuration as the release, not by hand from a laptop
[@twelve-admin].

!!! recipe "A migration ledger and runner"
    The ledger:

    ```sql
    CREATE TABLE IF NOT EXISTS schema_migrations (
      filename   text        PRIMARY KEY,
      checksum   text        NOT NULL,
      applied_at timestamptz NOT NULL DEFAULT now()
    );
    ```

    For each file `NNNN_name.sql` in lexical order, the runner opens a
    transaction, takes a transaction-scoped advisory lock so that two runners
    cannot interleave [@pg-admin-fn], and then either verifies the recorded
    checksum or applies the file and records it:

    ```sql
    BEGIN;
    SELECT pg_advisory_xact_lock(4242);   -- any fixed key, the same in every run
    -- contents of 0007_add_documents.sql
    INSERT INTO schema_migrations (filename, checksum)
    VALUES ('0007_add_documents.sql', '<sha256-of-file>');
    COMMIT;
    ```

    A recorded file whose checksum differs stops the run with the file name in
    the error.

Migrations that change existing tables run with the writers stopped. Many forms
of `ALTER TABLE` take an `ACCESS EXCLUSIVE` lock, which conflicts with every
other access to the table, reads included [@pg-locks], so requests touching the
table wait until the migration commits. Old code that keeps writing after the
change writes rows in the shape it knows, which the new schema may reject or
complete with defaults the code never intended. Stopping the API service, or
switching it to a maintenance mode that refuses writes, for the duration of the
migration removes both problems. Purely additive migrations, such as a new
table or a new nullable column, can run while the old code is serving, which is
what makes the release order of [Chapter 8](releases.md) possible.

## Roles and privileges

A database role is a principal in the sense of [Chapter 3](access.md), and the
same least-privilege reasoning applies. A deployment needs at least two: an
*owner* role that runs migrations and owns the objects, and an *application*
role with which the API service connects, holding only the data privileges it
uses. The application role does not create or alter tables, so a compromised
API cannot change the schema.

PostgreSQL grants some privileges to every role by default. No privilege on a
table goes to `PUBLIC` automatically, but `EXECUTE` on every new function and
procedure does [@pg-priv]. The manual recommends revoking such a default in the
same transaction that creates the object, so that there is no moment at which it
is open to all [@pg-createfunction].

Managed PostgreSQL platforms that expose the database through an HTTP API go
further. They define client-facing roles, such as `anon` for unauthenticated
requests and `authenticated` for signed-in ones, and some grant those roles
full privileges on every new table and `EXECUTE` on every new function in the
exposed schema, which makes each new object reachable through the platform's
API as soon as it exists [@supabase-api]. An application whose API service is
the only client must undo this for every object it creates.

!!! warning "A new object is public until revoked"
    On a platform with client-facing roles, every migration that creates a
    table, view, sequence or function must revoke the client roles' privileges
    on it in the same transaction, and grant the application role what it
    needs. Changing the default privileges once with `ALTER DEFAULT PRIVILEGES`
    covers only objects later created by the role named there [@pg-default-priv],
    and a platform may reapply its own defaults, so the per-object revoke is the
    rule and the default change only a second line of defence.

!!! recipe "A migration that locks out client roles"
    ```sql
    -- 0007_add_documents.sql (the runner wraps it in one transaction)
    CREATE TABLE documents (
      id          uuid        PRIMARY KEY,
      owner_id    uuid        NOT NULL REFERENCES users (id),
      object_key  text        NOT NULL UNIQUE,
      size_bytes  bigint      NOT NULL,
      created_at  timestamptz NOT NULL DEFAULT now()
    );
    REVOKE ALL ON TABLE documents FROM PUBLIC, anon, authenticated;
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE documents TO app_rw;

    CREATE FUNCTION create_document(p_owner uuid, p_key text, p_size bigint)
    RETURNS uuid
    LANGUAGE sql
    AS $$
      INSERT INTO documents (id, owner_id, object_key, size_bytes)
      VALUES (gen_random_uuid(), p_owner, p_key, p_size)
      RETURNING id
    $$;
    REVOKE ALL ON FUNCTION create_document(uuid, text, bigint) FROM PUBLIC, anon, authenticated;
    GRANT EXECUTE ON FUNCTION create_document(uuid, text, bigint) TO app_rw;
    ```

    On plain PostgreSQL, where `anon` and `authenticated` do not exist, the
    revokes name only `PUBLIC`. After each release, a check confirms that the
    client roles hold nothing; both queries must return no rows
    [@pg-info-fn]:

    ```sql
    SELECT c.oid::regclass AS relation, r.rolname
    FROM pg_class c
    JOIN pg_namespace n ON n.oid = c.relnamespace
    CROSS JOIN pg_roles r
    WHERE n.nspname = 'public' AND c.relkind IN ('r', 'v', 'm', 'p')
      AND r.rolname IN ('anon', 'authenticated')
      AND has_table_privilege(r.oid, c.oid, 'SELECT, INSERT, UPDATE, DELETE');

    SELECT p.oid::regprocedure AS routine, r.rolname
    FROM pg_proc p
    JOIN pg_namespace n ON n.oid = p.pronamespace
    CROSS JOIN pg_roles r
    WHERE n.nspname = 'public'
      AND r.rolname IN ('anon', 'authenticated')
      AND has_function_privilege(r.oid, p.oid, 'EXECUTE');
    ```

## Access paths

An application reaches PostgreSQL in one of two ways (Table 4.1). A *driver*
speaks the PostgreSQL protocol over TCP, holds a connection, and lets the
application open a transaction, run several statements and commit. An *HTTP
gateway* in the style of PostgREST turns HTTP requests into SQL: REST requests
on tables and views, and remote procedure calls (RPC) that invoke SQL
functions by name [@postgrest-fn]. Platforms offer a gateway because HTTPS
passes through any network and needs no connection pool; the price is that
each request runs in its own transaction, which the gateway commits or rolls
back, at the database's default isolation level unless the function sets
another [@postgrest-tx]. No transaction spans two requests.

| Property | TCP driver | HTTP gateway |
|---|---|---|
| Protocol | PostgreSQL wire protocol, usually over TLS | HTTPS; REST for tables, RPC for functions |
| Transactions | Opened and committed by the client across statements | One per request, ended by the gateway |
| Where multi-step logic lives | Application code inside a transaction | SQL functions called through RPC |
| Authentication | Database role and password | A gateway key or token mapped to a database role |
| Network requirement | A route to the database port | HTTPS only |
| After a schema change | Visible at once | Visible after the gateway reloads its schema cache |

Table: The two access paths to PostgreSQL.

The gateway's model moves transactional logic into the database. An operation
that must read, check and write several rows atomically, such as creating a
document and its audit record while checking a version number, becomes one SQL
function, created by a migration and called with one RPC request; the function
body runs inside the request's transaction. The function is then part of the
schema, subject to the revoke-and-grant discipline above.

The gateway also caches the schema it serves. It learns the tables and
functions from the database's catalogue and keeps that description in memory,
so after a migration it must be told to reload; PostgREST reloads on
`NOTIFY pgrst, 'reload schema'` or on a signal [@postgrest-cache]. Until then a
newly created function is unknown to it, and a release that calls the function
fails although the migration succeeded. The reload belongs to the migration
procedure, after the last file has been applied.

### Keeping the backend switchable

An application written against one access path is tied to it, and the choice
of path is often a hosting decision that may change. The way to keep the
choice open is to put data access behind repository interfaces, with one
implementation per path, selected by configuration in one place at startup.
The business logic calls the interface and never asks which backend is in use;
both implementations run the same test suite; and the SQL functions that the
gateway path needs are ordinary migrations, applied to every database whatever
path it is reached by. A move between a TCP deployment and a gateway
deployment is then a change of configuration, which is what treating the
database as an attached backing service means [@twelve-backing].

!!! remark "The cost of two implementations"
    Two implementations of every ported repository cost more to write and to
    test, and the temptation is to let the second one decay. It pays off only
    if both stay first-class: tested on every change, and exercised by at
    least one deployed environment or an integration suite against a real
    database. A switch that has not been tested since it was written is not a
    switch.

## Copying data between environments

Moving a deployment ([Chapter 8](releases.md)) eventually means moving its
data: every row, from a source database that is still serving, into a target
whose schema was built by the same migrations. The copy has to be consistent,
complete and verifiable, and it has to be impossible to apply twice.

*Export* reads every table inside one read-only transaction at `REPEATABLE
READ`, so that all tables are read from one snapshot, taken at the
transaction's first query, and no table reflects writes that another does not
[@pg-iso]. Each table becomes one JSON array, built by `json_agg` [@pg-aggregate]:

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ READ ONLY;
SELECT coalesce(json_agg(t ORDER BY t.id), '[]'::json) FROM users t;
SELECT coalesce(json_agg(t ORDER BY t.id), '[]'::json) FROM documents t;
-- one query per table; the export script writes each result to <table>.json
COMMIT;
```

When TCP access with sufficient privileges is available, `pg_dump` is the
standard tool and makes a consistent dump while the database is in use
[@pg-dump]. A JSON export is the better fit when the target schema must come
from the application's own migrations, or when the platform's roles do not
permit a dump and restore.

*Load* runs in one transaction on the target, so a failure leaves it untouched.
Setting `session_replication_role` to `replica` for that transaction stops
ordinary triggers from firing [@pg-client-config]. Foreign keys are enforced by
internally generated triggers [@pg-altertable], so they are not checked either,
and tables can be loaded in any order. Audit or timestamp triggers stay silent
as well, which is wanted: a copy should reproduce rows, not create new history.
Changing the setting requires superuser rights or an explicit `SET` privilege
[@pg-client-config]; where neither is available, tables are loaded in
dependency order, with constraints declared `DEFERRABLE` and deferred to the
commit for any cycle [@pg-set-constraints].

```sql
BEGIN;
SET LOCAL session_replication_role = replica;

DO $$
BEGIN
  IF EXISTS (SELECT 1 FROM users) OR EXISTS (SELECT 1 FROM documents) THEN
    RAISE EXCEPTION 'target already holds data; refusing to load';
  END IF;
END
$$;

INSERT INTO users (id, email, password_hash, created_at)
SELECT id, email, password_hash, created_at
FROM json_populate_recordset(NULL::users, $1::json);

INSERT INTO documents (id, owner_id, object_key, size_bytes, created_at)
SELECT id, owner_id, object_key, size_bytes, created_at
FROM json_populate_recordset(NULL::documents, $1::json);

COMMIT;
```

The load script sends each statement separately on one connection, binding the
table's exported JSON as `$1`; `json_populate_recordset` turns the array back
into rows of the table's type [@pg-json]. Three details matter. The column
lists leave out generated columns, which cannot be written and are computed by the
database [@pg-generated]. Identity columns declared `GENERATED ALWAYS` need
`OVERRIDING SYSTEM VALUE` to accept the exported values [@pg-insert], and every
sequence behind a serial or identity column is advanced past the loaded values
afterwards with `setval(pg_get_serial_sequence('users', 'id'), max(id))`
[@pg-sequence-fn; @pg-info-fn]. And the guard at the top makes a second load
refuse to run: a load either commits completely or leaves the target empty, so
an empty target is exactly the condition under which loading is safe. Where
migrations seed rows of their own, the guard checks a marker table that the load
writes instead.

*Verification* exports the target with the same queries and compares the two
exports table by table, as parsed values keyed by primary key. Text and hashes
are the wrong comparison: the same row can serialise differently, with
timestamps written out in each session's time zone or keys in another order, so
a byte comparison reports differences that are not there, and a hash cannot
say which row differs. Timestamps compare as instants, numbers as exact
decimals. Because the load did not check foreign keys, verification also runs
one orphan query per foreign key, each of which must return no rows:

```sql
SELECT d.id FROM documents d LEFT JOIN users u ON u.id = d.owner_id
WHERE u.id IS NULL;
```

!!! warning "Exports are personal data"
    An export holds every account's email address and password hash, and
    whatever personal data the application stores. Keep it in a private
    directory outside the repository, readable only by the operator, never
    attach it to a ticket or chat, and delete it once the target has been
    verified and the old environment is no longer needed as a rollback.

[Chapter 12](../cloudbase/database.md) applies these procedures to a CloudBase
environment: applying the schema, locking out the client roles, reaching the
database through its gateway and its CLI, and loading data.

!!! summary "Key points"
    - The schema is a sequence of numbered, immutable migration files; a
      ledger with checksums records what has run. Fix forward with a new file.
    - Table-changing migrations run with writers stopped; additive ones can
      run alongside the old code.
    - Revoke default privileges in the same transaction that creates each
      object. On platforms with client-facing roles, every migration revokes
      them explicitly, and a check after each release confirms it.
    - A TCP driver gives client-held transactions; an HTTP gateway gives one
      transaction per request, so atomic operations become SQL functions, and
      the gateway's schema cache must be reloaded after migrations.
    - Keep the access path switchable behind repository interfaces selected
      by configuration, with both implementations tested.
    - Copy data as one consistent export, one guarded all-or-nothing load,
      and a verification by parsed values plus orphan checks. Treat exports as
      personal data and delete them after use.
