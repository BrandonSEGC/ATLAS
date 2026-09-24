# ATLAS

Automated Task, Logistics & Activity System.

ATLAS is the commercial control plane that sells and operates isolated
[Jarvis](https://github.com/BrandonSEGC/Jarvis) AI-agent runtimes for
contractor companies. ATLAS owns signup, Slack installation and login, tenant
and member identity, Pipedream Connect, runtime provisioning, central Slack
event routing, usage metering, and audit. Each customer receives a dedicated
Jarvis runtime and persistent volume; ATLAS never runs a shared multi-tenant
agent.

## Status

Planning pass complete; implementation has not started. The documents below
are the source of truth for the first milestone and must stay consistent with
each other. Read them in this order:

| Document | Purpose |
| --- | --- |
| [`ATLAS-BLUEPRINT.md`](ATLAS-BLUEPRINT.md) | Product and architecture handoff from the product owner |
| [`docs/architecture.md`](docs/architecture.md) | System architecture, module boundaries, runtime contract, findings from the Jarvis codebase |
| [`docs/repository-structure.md`](docs/repository-structure.md) | Proposed monorepo layout and dependency rules |
| [`docs/data-model.md`](docs/data-model.md) | PostgreSQL schema, tenancy invariants, and migration conventions |
| [`docs/api-contracts.md`](docs/api-contracts.md) | HTTP API, webhook, runtime, and job/event contracts |
| [`docs/threat-model.md`](docs/threat-model.md) | Threats, mitigations, and pre-pilot requirements |
| [`docs/test-plan.md`](docs/test-plan.md) | Required automated tests mapped to modules |
| [`docs/implementation-plan.md`](docs/implementation-plan.md) | Ordered vertical slices for the first milestone |
| [`docs/open-decisions.md`](docs/open-decisions.md) | Product decisions that need an owner answer |
| [`docs/operations.md`](docs/operations.md) | Operator runbook outline |
| [`docs/adr/`](docs/adr/) | Architecture decision records |

## Repository rules

See [`AGENTS.md`](AGENTS.md). In short: this repository is treated as if it
were public. No customer data, live infrastructure identifiers, credentials,
or production-derived fixtures are ever committed.
