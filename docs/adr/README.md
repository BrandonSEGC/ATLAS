# Architecture decision records

One file per decision, numbered, never edited after acceptance except to
change status (superseded decisions link forward). New ADRs are required for
any change that affects tenancy, identity, credential ownership, or data
deletion.

| ADR | Title | Status |
| --- | --- | --- |
| [0001](0001-tenant-isolation.md) | One runtime and volume per tenant; tenant-scoped control plane | Accepted |
| [0002](0002-slack-event-routing.md) | Central Slack Events API gateway with durable dedup and re-signed forwarding | Accepted |
| [0003](0003-pipedream-identity-and-broker.md) | Platform-owned Pipedream Connect, opaque external user IDs, ATLAS MCP broker | Accepted |
| [0004](0004-web-framework-and-repository-shape.md) | Fastify, npm workspaces, single deployable | Accepted |
| [0005](0005-database-and-migrations.md) | PostgreSQL 16, `pg`, SQL migrations with node-pg-migrate | Accepted |
| [0006](0006-sessions-and-roles.md) | Server-side sessions in PostgreSQL; roles; operator flag | Accepted |
| [0007](0007-secret-encryption-and-rotation.md) | Envelope encryption with a master key ring and per-tenant DEKs | Accepted |
| [0008](0008-job-queue.md) | pg-boss for provisioning, delivery, and maintenance jobs | Accepted |
| [0009](0009-railway-provisioning.md) | Railway GraphQL API adapter behind `RuntimeProvisioner` | Accepted |
| [0010](0010-private-runtime-authentication.md) | Per-runtime credentials: HMAC forwarding secret, admin token, service token | Accepted |
| [0011](0011-jarvis-release-registry.md) | Jarvis images published to GHCR and pinned by digest | Accepted |
| [0012](0012-slack-scopes.md) | Initial Slack bot and sign-in scopes | Accepted |
| [0013](0013-tenant-deletion-and-backups.md) | Deletion state machine, grace window, backup retention | Proposed (needs product confirmation, OD-1) |
