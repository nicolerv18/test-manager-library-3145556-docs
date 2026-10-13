# ADR-001 · The schema belongs to the application's migrations

- **Status:** accepted
- **Context:** the `infra` repository starts a database engine. Someone could put the DDL in a container startup script and someone else
  in the application's migrations; with two owners, nobody knows which schema runs in production.
- **Decision:** **all DDL lives in the application's migrations** (tables, indexes, constraints and category seed).
  `infra` only creates the empty engine, the user and the volume.
- **Rejected alternatives:** SQL scripts in `infra/db/init/` (a second source of truth); ORM in "create tables on startup" mode
  (leaves no history and no rollback).
- **Consequences:** a Postgres extension or a new index is created in the **same** migration that uses it. A schema changed by hand in the
  engine is a defect, not a variant.
- **How it is verified:** `infra` contains no `CREATE TABLE`; the queries in `data-model.md` §10 match the migrations.
