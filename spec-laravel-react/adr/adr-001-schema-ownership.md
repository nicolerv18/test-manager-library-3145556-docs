# ADR-001 · The schema belongs to the application's migrations

- **Status:** accepted
- **Context:** the `infra` repository starts a database engine. Someone could put the DDL in a container startup script and someone else
  in the application's migrations; with two owners, nobody knows which schema runs in production.
- **Decision:** **all DDL lives in the application's migrations** (tables, indexes, constraints, extensions and the category seed). `infra`
  only creates the empty engine, the user and the volume. In this Laravel version the migrations are Laravel migrations under
  `database/migrations/`, and any extension (e.g. `pg_trgm`) is created in the **same** migration that uses it.
- **Rejected alternatives:** SQL scripts in `infra/db/init/` (a second source of truth); an ORM mode that "creates tables on startup"
  (leaves no history and no rollback).
- **Consequences:** a new Postgres extension or index is created in the same migration that needs it. A schema changed by hand in the
  engine is a defect, not a variant. Framework bookkeeping (Laravel's `migrations` table) is owned by the framework and is not part of the
  domain schema (see `data-model.md` §0.1).
- **How it is verified:** `infra` contains no `CREATE TABLE`; the queries in `data-model.md` §10 match the migrations.
