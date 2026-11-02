---
title: The PostgreSQL Database
summary: "Applying a schema to an environment's PostgreSQL, locking out client roles, reaching it through the gateway and the CLI, and copying data in."
references:
  envoverview: "Tencent CloudBase, *云开发环境介绍* (environment overview), [docs.cloudbase.net/quick-start/env-overview](https://docs.cloudbase.net/quick-start/env-overview)."
  roles: "Tencent CloudBase, *角色与权限* (roles and privileges), [docs.cloudbase.net/database/postgresql/roles](https://docs.cloudbase.net/database/postgresql/roles)."
  execute: "Tencent CloudBase, *tcb db execute* (执行 SQL), [docs.cloudbase.net/cli-v1/db/postgresql/execute](https://docs.cloudbase.net/cli-v1/db/postgresql/execute)."
  rpc: "Tencent CloudBase, *调用 RPC* (calling database functions over HTTP), [docs.cloudbase.net/database/postgresql/http/rpc](https://docs.cloudbase.net/database/postgresql/http/rpc)."
  backup: "Tencent CloudBase, *备份与回档* (backup and restore), [docs.cloudbase.net/database/postgresql/backup](https://docs.cloudbase.net/database/postgresql/backup)."
  pgjson: "The PostgreSQL Global Development Group, *JSON Functions and Operators*, PostgreSQL 17 documentation, [postgresql.org/docs/17/functions-json.html](https://www.postgresql.org/docs/17/functions-json.html)."
  pgreplica: "The PostgreSQL Global Development Group, *Client Connection Defaults: session_replication_role*, PostgreSQL 17 documentation, [postgresql.org/docs/17/runtime-config-client.html](https://www.postgresql.org/docs/17/runtime-config-client.html)."
---

## The database an environment holds

A PG-mode environment (Chapter 9) holds one PostgreSQL database. CloudBase's
own features, its authentication and storage permissions, use the same
database and the same role model as the application [@envoverview]. The
application owns its schema: tables, functions and triggers created by its
migrations (Chapter 4).

The database distinguishes callers by role [@roles]:

| Role | Who acts as it | Use in an application with its own API |
|---|---|---|
| `anon` | Unauthenticated callers of the gateway | None; revoke every application object from it |
| `authenticated` | Callers signed in through CloudBase authentication | None when the application runs its own accounts; revoke |
| `service_role` | Trusted server processes; bypasses row-level security | The API, through the server API key |
| Owner role | The CLI and the console's administrator session | Applies migrations; owns the application's objects |

Table: Database roles of a CloudBase environment and their place in an
application whose API is the only client of the database.

The owner role is created per environment and named
`cloudbase_postgres_pgdb_<suffix>`; the CLI runs SQL as that role, and objects
it creates belong to it. The server API key of Chapter 10 acts with the
service role's rights: in the deployments behind this book, revoking `anon`
and `authenticated` from every application object left the API fully working.

!!! warning "Client roles are granted by default"
    CloudBase grants new tables and functions to the client roles. An
    application that keeps all access behind its own API must revoke `PUBLIC`,
    `anon` and `authenticated` from every object it creates, in the same
    script that creates it. A migration that forgets the revoke opens that
    table or function to anyone who knows the environment ID.

The documentation describes automatic backups, manual backups and
point-in-time restore, with availability and retention as the console shows
them [@backup]. On the shared database cluster of the environments behind
this book, the console offered none of them. Treat the logical export of
Section 12.5 as the backup that can be relied on, and take one before every
migration that changes tables.

## Three ways in

The database is reached in three ways, each suited to a different job.

| Route | Acts as | Transactions | Use for |
|---|---|---|---|
| Console SQL editor (`#/db/postgres/sql-editor`) | Owner (管理员 session) | Per execution | One-off inspection; applying a migration by hand |
| CLI `tcb db execute` | Owner | Per call | Migrations, exports and loads that must be repeatable |
| HTTP gateway (REST and RPC) | Role of the presented credential | Per request; none held by the client | The application at run time |

Table: Routes into a CloudBase PostgreSQL database.

**The CLI** runs SQL from a terminal against a named environment and returns
the result as JSON with `--json` [@execute]:

```bash
tcb db execute -e <env-id> --sql 'SELECT count(*) FROM pg_tables' --json
```

The documentation describes the command as running one statement. In
practice, with CLI 3.8, a call accepted a whole multi-statement script and
returned the rows of its last statement, and a script that failed part-way
left nothing behind, with or without its own `BEGIN` and `COMMIT`. Settings
made with `SET LOCAL` did not carry over to the next call, since each call is
its own transaction. A 500 KB script ran; at about 900 KB the CLI itself failed
before sending, so scripts are best kept under 350 KB. `--role` selects a
client role (`anon`, `authenticated` or a custom role) for testing what that
role can see; reserved platform roles are refused with error `42501`
[@execute].

**The gateway** exposes tables and functions over HTTPS in the PostgREST
style. A function is called by name [@rpc]:

```http
POST /v1/rdb/rest/rpc/<function> HTTP/1.1
Host: <env-id>.api.tcloudbasegateway.com
Authorization: Bearer <server-api-key>
Content-Type: application/json

{"p_event_id": "…", "p_expected_version": 3}
```

Arguments travel as a JSON object keyed by parameter name. Functions declared
`STABLE` or `IMMUTABLE` can also be called with GET and query parameters. The
client holds no transaction across requests, which is why an application on
this route puts each multi-step operation into one SQL function (Chapter 4).

!!! remark "Errors from functions"
    The gateway follows PostgREST conventions for errors. A function that
    raises with a SQLSTATE of the form `PTnnn` produces HTTP status `nnn`;
    the gateway reported such an error with the code `DATABASE_PTnnn`, for
    example `DATABASE_PT409` for a version conflict. A call to a function the
    gateway does not know returns `PGRST202`.

## Applying a schema

The schema arrives as the application's numbered migration files (Chapter 4),
applied in order, each recorded in a ledger table with its checksum. On
CloudBase the files are applied by the CLI or pasted into the SQL editor; in
both cases every script also revokes the client roles from what it creates.

!!! recipe "Applying migrations to a new environment"
    1. Generate the scripts from the migration files: create the ledger table
       first, then append each migration followed by its ledger row
       (file name and SHA-256 checksum), and end each script with the revokes
       below. Split them so that no script exceeds 350 KB, at migration
       boundaries.
    2. Rehearse the scripts on an empty local PostgreSQL of the same major
       version, with the client roles and default grants recreated, and
       confirm that a second run is refused because the first ledger row
       already exists.
    3. Apply them in order:

        ```bash
        tcb db execute -e <env-id> --sql "$(cat schema-1.sql)" --json
        ```

    4. Check the ledger: the row count equals the number of migration files,
       and the last row names the last file.

    **Check:** compare the new environment's catalog with a reference (the
    local rehearsal, or an existing environment at the same migration):
    columns, constraints, indexes, function definitions, triggers, extensions
    and privileges. Compare parsed rows, not the text of a dump.

The revokes for application objects, assuming they live in `public` and the
application's functions share a prefix:

```sql
REVOKE ALL ON ALL TABLES IN SCHEMA public FROM PUBLIC, anon, authenticated;
REVOKE ALL ON ALL SEQUENCES IN SCHEMA public FROM PUBLIC, anon, authenticated;
DO $$
DECLARE fn regprocedure;
BEGIN
  FOR fn IN
    SELECT p.oid::regprocedure FROM pg_proc p
    JOIN pg_namespace n ON n.oid = p.pronamespace
    WHERE n.nspname = 'public' AND p.proname LIKE 'myapp\_%'
  LOOP
    EXECUTE format('REVOKE ALL ON FUNCTION %s FROM PUBLIC, anon, authenticated', fn);
  END LOOP;
END $$;
```

and the check that nothing was missed, which must return no rows:

```sql
SELECT 'table' AS kind, table_name AS name, grantee
FROM information_schema.role_table_grants
WHERE table_schema = 'public' AND grantee IN ('PUBLIC', 'anon', 'authenticated')
UNION ALL
SELECT 'function', p.proname, r.rolname
FROM pg_proc p
JOIN pg_namespace n ON n.oid = p.pronamespace
CROSS JOIN (VALUES ('anon'), ('authenticated')) AS r(rolname)
WHERE n.nspname = 'public' AND p.proname LIKE 'myapp\_%'
  AND has_function_privilege(r.rolname, p.oid, 'EXECUTE');
```

A later migration is applied the same way, one file with its ledger row and
its revokes. A migration that changes tables runs with the API stopped or at
zero instances; a migration that only adds functions can run first and be
followed by the release that uses them (Chapter 8). The API should check at
startup that the functions it needs exist and refuse to start otherwise, so
that a missing migration shows up as a clear startup failure rather than as
errors in use.

## How the gateway sees the schema

The gateway reads the database's functions and tables into a cache. A newly
created function is usually callable at once; if a call to it returns
`PGRST202`, the standard PostgREST command asks the gateway to reload its
cache:

```sql
NOTIFY pgrst, 'reload schema';
```

The CloudBase documentation does not describe this command, but it is
harmless when the cache is already current. A `PGRST202` that survives it
means the function's name or argument names differ from the call, since the
gateway matches functions by name and by the set of argument names.

## Copying data into an environment

When an application moves between environments (Chapter 15), its data moves
as SQL through the CLI: CloudBase offers no copy between environments, and
the console's 迁移 (migration) page manages the CLI's own schema migrations,
not data. The procedure below implements the method of Chapter 4: export
each table as JSON, load everything in one transaction, verify by comparing
parsed values, and refuse to load twice.

!!! recipe "Copying an environment's data"
    1. **Freeze** the source: set both of its services to a minimum of 0
       and wait until no instance runs, so nothing writes during the export
       (Chapter 13).
    2. **Export** each table with one read-only call, ordered by primary key
       [@pgjson]:

        ```bash
        tcb db execute -e <source-env-id> --json --sql \
          "SELECT coalesce(json_agg(t ORDER BY id), '[]') FROM public.accounts t"
        ```

        Take a checksum of every table before and after the export; they
        must agree, which proves that nothing changed in between, since each
        CLI call is its own transaction.
    3. **Build** the load. Create a holding schema in the target, revoked
       from every client role before any data arrives, and a holding table
       of `(table_name text, part int, payload text)`. Split the exported JSON
       into scripts of at most 350 KB that insert into it.
    4. **Load**: one final script that, in a single call, runs
       `SET LOCAL session_replication_role = replica` (which skips user
       triggers and foreign-key checks for the transaction [@pgreplica]),
       inserts every table from the holding table, checks that every target
       table matches its exported checksum, raises an error otherwise, and
       drops the holding schema. Each insert names its columns and omits
       generated ones:

        ```sql
        INSERT INTO public.accounts (id, email, created_at)
        SELECT id, email, created_at
        FROM json_populate_recordset(NULL::public.accounts,
          (SELECT string_agg(payload, '' ORDER BY part)
           FROM copy_holding.parts WHERE table_name = 'accounts')::json);
        ```

    5. **Refuse a second load**: the final script begins by checking that
       every target table is empty and raises otherwise.
    6. **Verify**: export the target the same way and compare it with the
       source export table by table as parsed JSON values, not as text.

    **Check:** the comparison reports zero differences, and the application,
    started against the target, signs in an existing account and opens its
    data.

A dry run of step 4 that ends by raising a fixed message, instead of
committing, proves the whole load against the real target and leaves nothing
behind. The exports contain password hashes, session digests and email
addresses: keep them in a private directory and delete them once the move is
verified.

!!! summary "Key points"
    - The environment's PostgreSQL serves CloudBase's own features and the
      application alike. The API acts with the service role through the
      server API key; the CLI and the console act as the owner.
    - CloudBase grants new objects to `anon` and `authenticated`. Revoke them
      in every migration script and check with a query that returns no rows.
    - Use the CLI for anything repeatable: one call runs a multi-statement
      script atomically; keep scripts under 350 KB.
    - The gateway follows PostgREST: functions by name over RPC, `PTnnn`
      SQLSTATEs as HTTP statuses, `PGRST202` for an unknown function.
    - Do not rely on platform backups being available; a logical export is
      the backup, and the same procedure copies data between environments.
