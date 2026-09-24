# Implementation plan for the first milestone

Ordered vertical slices. Each slice ends with its tests from
`docs/test-plan.md` passing in CI and a working, demonstrable behaviour. Do
not start a slice before the previous one's tests pass; do not build
abstractions a later slice needs unless the current slice exercises them.

Jarvis-side changes (`docs/architecture.md` 6.6) are listed where a slice
depends on them; they are delivered as separate PRs in the Jarvis repository.

## Slice 0: Foundation

Deliverable: `npm run dev` starts the control plane against local PostgreSQL,
serves a placeholder dashboard, and refuses to start with invalid config.

- Monorepo scaffold per `docs/repository-structure.md`; strict TypeScript;
  Biome; CI running typecheck, lint, unit tests, integration tests against a
  PostgreSQL service container, and `check:public-safety`.
- `packages/config`: environment schema, `ATLAS_ENV`, `ATLAS_ROLES`,
  production guards (HTTPS URLs, Pipedream `production`, key ring present).
- `packages/database`: pool, `withTenantTransaction`/`withPlatformTransaction`,
  migration runner, first migration containing every table in
  `docs/data-model.md` plus RLS policies and the two roles.
- `packages/secrets`: key ring, envelope encryption, `SecretRef`, `redact`.
- `packages/audit`, `packages/jobs` (pg-boss wiring, no handlers yet).
- Structured logger with redaction. Request IDs. Safe error mapper.
- Tests: SC-01..SC-06, TI-03, TI-04.

## Slice 1: Tenant + Slack install

Deliverable: Add to Slack against a dev Slack app creates a tenant, an
installation, an owner, and a queued provisioning job; the browser lands on
an onboarding page saying the runtime is being prepared.

- `packages/auth`: `oauth_states` repository, install start and callback,
  Slack OAuth v2 exchange client with fake, installation upsert transaction.
- Tenant, member, installation repositories.
- `runtime_instances` row created in `requested` with a `runtime.provision`
  job enqueued (handler is a stub that logs until slice 4).
- Onboarding page (static) and `/install` landing page.
- Handling of `app_uninstalled` and `tokens_revoked` is deferred to slice 3
  where the gateway exists.
- Tests: SO-01..SO-05, TI-01, TI-02 (for members and installations).

## Slice 2: Sign in with Slack + dashboard session

Deliverable: the installer and a second user can sign in; the dashboard shows
`/api/me`, tenant, members, and runtime status (`Preparing`); owner can change
roles.

- OIDC login start/callback with PKCE, nonce, JWKS validation (fake Slack
  JWKS in tests).
- Sessions repository and cookie handling; `Origin` check; rate limits.
- Role authorization helper (`requireRole`).
- Routes: `/api/me`, `/api/tenant`, `/api/members`, `PATCH
  /api/members/:id/role`, `PATCH /api/members/:id/status`, `/api/runtime`
  (read only), `/auth/logout`.
- Dashboard pages: Overview (runtime status), Members.
- Platform operator flag and `operator:add` CLI; `GET /api/operator/tenants`
  and tenant detail (read only).
- Tests: SO-06..SO-14, TI-05, TI-11.

## Slice 3: Event verify + tenant route

Deliverable: Slack events for an installed workspace are verified,
deduplicated, resolved to a tenant, and delivered to a `FakeRuntime` with a
valid forwarding signature; drops are visible to operators.

- `packages/slack-gateway`: raw-body route, verifier, minimal parser,
  receipts, resolution, self-echo rule, delivery row, enqueue.
- `packages/runtime-client`: `v0` signer, `forwardSlackEvent` with retry
  policy, `health`.
- `runtime.deliver_slack_event` handler, purge job for delivered bodies.
- `app_uninstalled` / `tokens_revoked` handling.
- Operator health counters endpoint.
- Because no real runtime exists yet, staging validation uses a `FakeRuntime`
  deployed as a stub service; that keeps this slice independent of slice 4.
- Tests: SE-01..SE-15, TI-10, SO-10.

## Slice 4: Provisioning job + runtime status

Deliverable: a new installation results in a real Jarvis runtime on the
Railway staging pool, `Ready` in the dashboard, and Slack DMs answered by
that runtime.

Depends on Jarvis J6 (published image) and J7 (explicit
`--adapter=slack:webhook` verified). Everything else uses existing Jarvis
behaviour.

- `packages/provisioning`: interface, state machine, `FakeProvisioner`,
  `RailwayProvisioner` (service, volume, variables, image, deploy, private
  hostname), resource class mapping.
- Per-runtime credential generation and storage; `runtime_service_tokens`.
- `runtime.provision` handler with step markers; health polling; `ready`.
- `runtime.health_check` scheduled job; `degraded` transitions.
- `runtime.suspend` / `runtime.resume`; operator suspend/resume/retry;
  `POST /api/runtime/retry`.
- Per-tenant model provider key handling per OD-3 default.
- Dashboard: runtime status with plain-language states and retry button.
- Tests: PV-01..PV-04, PV-06, PV-08, PV-09; staging checklist items 1, 2, 5.

## Slice 5: Pipedream link + connection status

Deliverable: a member connects a personal app and an owner connects a shared
app through Connect Links; status shows `Connected`; diagnostics work; no
Pipedream developer credential leaves ATLAS.

- `packages/connections`: Pipedream client (token provider, apps, accounts,
  connect tokens) with fake; app policy table; ownership rules; external
  user bookkeeping; connection repository; sync job; webhook handler.
- Routes: `/api/connections`, `/apps`, `/pipedream/link`, `/:id/status`,
  `/:id/diagnose` (stages 1 and 2 only until slice 6 adds MCP stages),
  `DELETE /:id`; `POST /webhooks/pipedream`.
- Dashboard: Connections page with personal/shared choice and status.
- Tests: PD-01..PD-05, PD-08 (stages 1-2), PD-10..PD-13, TI-06.

## Slice 6: Projection + MCP broker (tools reach Jarvis)

Deliverable: after connecting an app, Jarvis in Slack can list and call that
app's tools through ATLAS; a second member cannot use the first member's
personal connection.

Depends on Jarvis J2 for personal connections (shared connections work
without it). J1 removes the restart on projection change; the slice ships
with restart first.

- Projection builder (aliases, ownership blocks) and `runtime.push_projection`
  handler against the runtime admin API, idempotent by hash.
- MCP broker route: runtime auth, connection resolution, actor rule, header
  injection, Streamable HTTP proxy (POST/GET/DELETE, session ID
  passthrough), error rewriting, tool-call usage events.
- Diagnostics stages 3 to 5.
- `GET /internal/runtimes/:id/projection` for future pull.
- Tests: PD-06, PD-07, PD-08 (all stages), PD-09, TI-07, TI-08, PV-07;
  staging checklist items 4, 6, 7.

## Slice 7: Usage, credential rotation, deletion, hardening

Deliverable: operator capabilities from the acceptance criteria are complete;
threat model M1 checklist is green.

- `POST /internal/runtimes/:id/usage` and platform-side usage recorders
  (external users, connected accounts, runtime uptime). Jarvis J5 enables
  runtime reports; platform-side metrics work without it.
- Operator credential rotation (per-runtime and master key); rotation
  runbooks in `docs/operations.md` exercised in staging.
- Deletion state machine per ADR-0013; `runtime.destroy`; purge job.
- Rate limits reviewed; response schema secret scan in CI; log review.
- Tests: TI-09, PV-05, PV-10, SC-02; staging checklist items 3, 8, 9.

## Exit criteria for the milestone

- All test IDs in `docs/test-plan.md` sections 1 to 6 implemented and
  passing in CI.
- Staging checklist section 7 completed by someone other than the
  implementer, using a fresh Slack workspace.
- Threat model M1 checklist complete.
- Open decisions in `docs/open-decisions.md` either answered or explicitly
  deferred by the product owner with the default recorded.

## Explicitly not in this plan

Billing collection, Slack Marketplace listing, native MCP OAuth, fleet-wide
automatic upgrades, full support dashboard, Enterprise Grid org-wide
installs, usage-based pricing, mobile, migration of existing Jarvis
deployments. Extension points already present: `subscriptions` table and
suspension state, `connections.provider` check constraint, `updateRelease` on
the provisioner, `RUNTIME_CONTRACT_VERSION`, usage ledger.
