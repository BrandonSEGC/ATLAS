# ADR-0013: Deletion state machine, grace window, backup retention

Status: Proposed (defaults adopted for implementation; product confirmation
requested in `docs/open-decisions.md` OD-1)

## Context

A tenant's data lives in the control-plane database, its runtime volume, its
Pipedream external users and accounts, and provider or database backups.
Deletion must be complete, auditable, safe against accidental triggering, and
retry-safe.

## Decision

Deletion is triggered by a tenant `owner` from the dashboard (with typed
confirmation), by an operator, or by `app_uninstalled` after a 14 day
unreinstalled period.

```text
active|suspended --request--> deleting (purge_after = now() + 30 days)
  immediate: suspend runtime, revoke runtime tokens and sessions, revoke
             Slack installation record, stop event delivery, block logins,
             send confirmation with the purge date
  during grace: owner or operator may cancel -> suspended (then resume)
deleting --purge job after purge_after--> deleted
  purge: destroy runtime service and volume; delete Pipedream accounts and
         external users for the tenant; delete secrets, sessions, deliveries,
         connections, members' emails and display names; keep tenants row
         (status deleted, slug, slack_team_id hash, timestamps), audit_events,
         and usage_events aggregates
```

`runtime.destroy` requires `tenants.status = deleting AND purge_after <
now()` and is idempotent at the provider (lookup by name; absent means done).

Backups:

- Control-plane database: managed point-in-time recovery with 7 day
  retention plus daily snapshots retained 30 days. Deleted tenants can
  therefore appear in backups for up to 30 days after purge; this is
  disclosed in customer terms.
- Runtime volumes: whatever the provider offers (Railway volume backups) at
  the platform's configured schedule; retention capped at 30 days. Restores
  are operator-initiated and audited.
- Secrets in backups are ciphertext; a restored database without the master
  key ring cannot reveal them.

Suspension is distinct from deletion: it stops the runtime and blocks access
but retains everything indefinitely until resumed or deleted.

## Consequences

- Customers have a 30 day undo window; storage costs continue during it.
- `tenants` rows are never physically deleted so audit references stay valid.

## Alternatives rejected

- Immediate hard delete: no recovery from mistakes or disputes.
- Indefinite soft delete: unbounded liability and cost.
