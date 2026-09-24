# Proposed repository structure

Decision: npm workspaces monorepo with one deployable service for the first
milestone (ADR-0004). The blueprint's `apps/web`, `apps/api`, and
`apps/slack-gateway` are represented as modules inside `apps/control-plane`
plus the `packages/` below; splitting them into separate deployables later is
a composition change, not a code move.

```text
ATLAS/
  ATLAS-BLUEPRINT.md
  AGENTS.md
  README.md
  package.json                 # workspaces, root scripts (build, test, lint, typecheck, check:public-safety)
  tsconfig.base.json           # strict, NodeNext, isolatedModules, noUncheckedIndexedAccess
  .env.example                 # every variable read by packages/config, synthetic values only
  docker-compose.yml           # local postgres only
  apps/
    control-plane/
      package.json
      src/
        main.ts                # parse config, run migrations check, start roles per ATLAS_ROLES
        server.ts              # Fastify composition, plugins (raw body, cookies, rate limit, helmet)
        roles/
          api.ts               # registers auth, dashboard API, operator API, webhooks, internal, mcp
          gateway.ts           # registers only /webhooks/slack/events
          worker.ts            # starts pg-boss handlers
        routes/
          auth/                # /auth/slack/install, /auth/slack/login, /auth/logout
          api/                 # /api/me, /api/tenant, /api/members, /api/connections, /api/runtime
          operator/            # /api/operator/* (platform operator role)
          webhooks/            # /webhooks/slack/events, /webhooks/pipedream
          internal/            # /internal/runtimes/:id/usage, /heartbeat
          mcp/                 # /mcp/v1/connections/:connectionId (broker)
        http/
          session-auth.ts      # cookie -> session -> TenantContext + role
          runtime-auth.ts      # bearer -> runtime service token hash -> RuntimeContext
          errors.ts            # safe error mapping (no provider payloads)
      ui/                      # Vite + React dashboard (served as static assets by the api role)
        src/
          pages/               # Onboarding, Runtime, Connections, Members, Operator
      test/                    # route-level tests with fakes (see docs/test-plan.md)
  packages/
    contracts/                 # zod schemas + types: http, webhooks, runtime contract v1, projection, jobs
    config/                    # env schema, ATLAS_ENV rules, fail-closed startup
    database/
      migrations/              # numbered SQL files (node-pg-migrate)
      src/
        context.ts             # TenantContext, PlatformContext, RuntimeContext
        client.ts              # pool, withTenantTransaction (sets app.tenant_id for RLS)
        repositories/          # one file per aggregate; tenant-owned ones take TenantContext first
    secrets/                   # envelope encryption, key ring, SecretRef, redact()
    auth/                      # slack install oauth, slack oidc login, sessions, roles, oauth_states
    slack-gateway/             # verify, parse-minimal, dedup, resolve, self-echo, enqueue
    runtime-client/            # forwardSlackEvent, pushProjection, health; HMAC signer
    provisioning/              # RuntimeProvisioner, state machine, RailwayProvisioner, FakeProvisioner
    connections/               # pipedream client, connect links, account sync, ownership policy, broker, diagnostics
    usage/                     # usage_events ingestion
    audit/                     # audit_events writer with metadata allow-list
    jobs/                      # pg-boss setup, job names, idempotency keys, handlers registry
    testing/                   # fake slack server, fake pipedream server, fake runtime, fixtures (synthetic only)
  docs/
    architecture.md
    repository-structure.md
    data-model.md
    api-contracts.md
    threat-model.md
    test-plan.md
    implementation-plan.md
    open-decisions.md
    operations.md
    adr/
  scripts/
    check-public-safety.mjs    # scans tracked files for non-synthetic emails, IPs, token shapes
```

## Dependency direction

```text
contracts <- config <- database <- secrets
                                     ^
   auth, slack-gateway, runtime-client, provisioning, connections, usage, audit, jobs
                                     ^
                              apps/control-plane
```

- No package imports from `apps/`.
- No package imports Fastify; HTTP concerns stay in `apps/control-plane`.
  Packages expose plain functions that take contexts and typed inputs.
- `testing` may be imported only from test files.

## Tooling defaults

| Concern | Choice | Reason |
| --- | --- | --- |
| Package manager | npm workspaces | matches the Jarvis repository; no extra tool |
| Language | TypeScript 5, `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` | blueprint requires strict configuration |
| Runtime | Node.js 22 LTS | matches Jarvis image base |
| HTTP | Fastify 5 with `@fastify/rate-limit`, `@fastify/cookie`, `@fastify/helmet`, raw-body capture for webhooks | ADR-0004 |
| Validation | zod schemas in `contracts`, used at every HTTP and job boundary | blueprint requirement |
| Database | PostgreSQL 16, `pg` driver, `node-pg-migrate` SQL migrations | ADR-0005 |
| Jobs | pg-boss | ADR-0008 |
| Tests | Node built-in test runner (`node --test`) with `tsx`; PostgreSQL via docker for integration tests | same style as Jarvis (`tsx`, `node:assert/strict`); no framework lock-in |
| Lint/format | Biome | single fast tool, no config sprawl |
| UI | Vite + React, no server-side rendering | dashboard is small and session-cookie authenticated |

## Root scripts

```text
npm run build                # tsc -b for all packages and the app, vite build for ui
npm run typecheck
npm run lint
npm run test                 # unit + route tests with fakes (no network)
npm run test:integration     # requires DATABASE_URL to a disposable postgres
npm run migrate              # node-pg-migrate up
npm run check:public-safety  # must pass before every commit
npm run dev                  # control-plane with ATLAS_ROLES=api,gateway,worker against local postgres
```
