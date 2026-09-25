# ADR-0008: pg-boss for provisioning, delivery, and maintenance jobs

Status: Accepted

## Context

Provisioning, projection pushes, event delivery, health checks, connection
syncs, and purges must be durable, retried with backoff, idempotent, and
observable. The blueprint asks for background jobs with durable retries.

## Decision

- pg-boss on the existing PostgreSQL (`pgboss` schema). Handlers live in
  `packages/jobs` and are registered by the `worker` role.
- Idempotency through singleton keys listed in `docs/api-contracts.md`
  section 6 (`tenant:<id>:provision`, `event:<event_id>`, and so on) plus
  precondition re-checks inside each handler.
- Job payloads carry identifiers only; handlers load state inside a tenant
  transaction. No secrets or message bodies in payloads.
- Retry schedules are defined per job name; exhausted jobs move to
  pg-boss's failed state and also update the domain row (`runtime.status =
  failed`, `slack_event_deliveries.status = expired`) so the dashboard and
  operator views do not depend on queue internals.
- Scheduled jobs (`runtime.health_check`, `maintenance.purge`) use pg-boss
  cron with singleton execution.

## Consequences

- One fewer service to operate; queue throughput is bounded by PostgreSQL,
  which is sufficient for pilot volumes.
- Job visibility comes from `pgboss` tables plus domain status columns; an
  operator view lists failed jobs by tenant.

## Alternatives rejected

- BullMQ/Redis: additional infrastructure for the pilot.
- Provider-native queues: ties the control plane to a cloud vendor.
- In-process timers: not durable across restarts or replicas.
