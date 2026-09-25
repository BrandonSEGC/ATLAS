# ADR-0001: One runtime and volume per tenant; tenant-scoped control plane

Status: Accepted

## Context

Jarvis is a single-workspace agent: one process reads one workspace
directory, one `settings.json`, one `.jarvis/secrets.json`, one Slack token
set, and one MCP connector list (see `docs/architecture.md` section 6.1). Its
memory, skills, attachments, and logs live in that directory. Nothing inside
the agent, its tools, or its prompt assembly carries a tenant identifier.

ATLAS must serve many contractor companies whose data, tools, Slack
workspaces, and conversations must never mix.

## Decision

1. Each tenant gets a dedicated Jarvis runtime (container/service) with its own
   persistent volume, its own set of credentials, and its own private ingress.
   The control plane never runs agent code and never mounts or reads a tenant
   volume.
2. Inside the control plane, every tenant-owned row carries `tenant_id`, all
   repository functions for tenant-owned tables require a `TenantContext`,
   composite foreign keys keep child rows inside the parent's tenant, and
   PostgreSQL row-level security with `FORCE ROW LEVEL SECURITY` is enabled
   on tenant-owned tables as defense in depth (`docs/data-model.md`).
3. Tenant identity is derived only from a server-side session (dashboard), a
   runtime service token (runtime calls), or a verified Slack signature plus
   installation lookup (webhooks). It is never accepted from a request body,
   query string, header, hostname, or connection name.
4. Ambiguity fails closed: if an installation, runtime, or connection cannot
   be resolved to exactly one tenant, the request is rejected.
5. Suspension is a first-class state on `tenants`, `runtime_instances`, and
   `subscriptions`; the gateway, connection mutations, and the MCP broker all
   check it.

## Consequences

- Cost per tenant is a running service plus a volume even when idle. That is
  acceptable for the pilot; scale-to-zero is a provider capability to revisit
  later without changing the model.
- Automatic fleet-wide upgrades are out of scope for the first milestone;
  `release` is recorded per runtime and `updateRelease` exists on the
  provisioner so a rolling upgrade job can be added later.
- Jarvis needs no tenant checks added to its agent loop. Required Jarvis
  changes are limited to the runtime contract (`docs/architecture.md` 6.6).
- Operators cannot inspect a tenant's workspace through ATLAS. Support
  workflows that need it must go through the customer.

## Alternatives rejected

- Shared multi-tenant Jarvis process with a per-request tenant context:
  requires auditing every file, memory, tool, and prompt path for tenant
  leakage; one bug leaks across all customers.
- One process per tenant on a shared host with shared filesystem: weaker
  isolation, harder resource limits, and shared secrets on disk.
