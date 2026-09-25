# ADR-0005: PostgreSQL 16, `pg`, SQL migrations with node-pg-migrate

Status: Accepted

## Context

The control plane needs transactional tenant state, durable dedup, sessions,
a job queue, and row-level security. One database keeps the pilot simple.

## Decision

- PostgreSQL 16 is the only datastore for the first milestone: application
  tables, sessions, pg-boss queue (`pgboss` schema), dedup receipts, audit and
  usage ledgers.
- Driver: `pg` with a small typed query helper. No ORM. Repositories are
  hand-written functions per aggregate that take a context first
  (`docs/data-model.md`). A lint rule limits `pool.query` and SQL template
  usage to `packages/database`.
- Migrations: plain SQL files run by `node-pg-migrate`, applied by a
  release step before the new version starts (`npm run migrate`). The app
  refuses to start if pending migrations exist.
- Two database roles: `atlas_migrator` (owner, runs migrations) and
  `atlas_app` (no `BYPASSRLS`, used at runtime). RLS policies read
  `current_setting('app.tenant_id', true)`.
- Tests run against a disposable PostgreSQL (docker) with the real
  migrations; no in-memory substitute.

## Consequences

- Schema changes are reviewable SQL. The design document must be updated
  with each migration.
- RLS adds a per-transaction `SET LOCAL`; the helper makes it impossible to
  run a tenant query without it.

## Alternatives rejected

- Prisma/Drizzle: adds a schema DSL and generated client between the code and
  RLS/`SET LOCAL` semantics; not needed for this table count.
- Redis for sessions or queue: another service to operate for the pilot;
  PostgreSQL throughput is sufficient.
