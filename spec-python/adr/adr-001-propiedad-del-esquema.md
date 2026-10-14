# ADR-001 · Application migrations own the schema

- **Status:** accepted
- **Context:** the `infra` repository starts a database engine. One person could place DDL in a
  container startup script and another in application migrations; with two owners, nobody knows which
  schema runs in production.
- **Decision:** **all DDL lives in application migrations** (tables, indexes, constraints, and the
  category seed). `infra` creates only the empty engine, user, and volume.
- **Rejected alternatives:** SQL scripts in `infra/db/init/` (a second source of truth); ORM
  “create tables at startup” mode (no history or rollback).
- **Consequences:** a PostgreSQL extension or new index is created in the **same** migration that
  uses it. A schema changed manually in the engine is a defect, not a variant.
- **Verification:** `infra` contains no `CREATE TABLE`; the queries in `data-model.md` §10 match
  the migrations.
