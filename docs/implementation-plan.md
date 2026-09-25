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

Deliverable: a new installation results in a real Jarvis runtime on the Fly
staging organization, `Ready` in the dashboard, Slack DMs answered by that
runtime, idle runtimes stopped and woken on the next message, and hourly
workspace backups in object storage.

Depends on Jarvis J6 (published image) and J7 (explicit
`--adapter=slack:webhook` verified). J8 (fast start, activity signal) and J9
(backup hook) improve this slice but have interim fallbacks: a longer idle
window and a backup taken by stopping the machine and snapshotting the
volume.

- `packages/provisioning`: interface, state machine including
  `stopped`/`starting`, `FakeProvisioner`, `FlyProvisioner` (machine,
  volume, secrets/env, image, start/stop, private address), resource class
  mapping.
- Per-runtime credential generation and storage; `runtime_service_tokens`.
- `runtime.provision` handler with step markers; health polling; `ready`.
- `runtime.health_check` scheduled job; `degraded` transitions.
- `runtime.idle_sweep`, `runtime.wake`, and the delivery worker's wake path.
- Workspace backup job to per-tenant object storage; operator restore.
- `runtime.suspend` / `runtime.resume`; operator suspend/resume/retry;
  `POST /api/runtime/retry`.
- Dashboard: runtime status with plain-language states (Preparing, Ready,
  Sleeping, Waking, Needs attention, Suspended) and retry button.
- Tests: PV-01..PV-04, PV-06, PV-08, PV-09, CR-18; staging checklist items
  1, 2, 5.

## Slice 4a: Model gateway + credits

Deliverable: every model call from a runtime is proxied by ATLAS using a
per-tenant provider key, metered, priced, and debited; owners see balance
and usage; exhausted credits stop model calls with a plain message and DM
the owners.

No Jarvis change required (base URL environment variables already exist).

- `packages/billing`: price book, ledger, balances, statements, threshold
  notifications; operator grant/adjust routes; initial price book version
  seeded from a checked-in synthetic example and published by an operator
  with real provider prices.
- `packages/model-gateway`: provider scope and key management (provider
  admin API where available, operator-assisted otherwise), proxy with
  streaming, usage parsers for Anthropic and OpenAI-compatible formats,
  credit pre-check, per-tenant limits, path allow-list.
- Provisioning sets the runtime's base URLs and token-as-key variables.
- `usage.meter_runtime_hours` and tool-call metering in the MCP broker
  (broker lands in slice 6; the SKU and hook are prepared here).
- Dashboard: Credits page (balance, burn, usage by SKU, ledger, statements);
  operator credit views.
- Tests: CR-01..CR-17, CR-19; staging checklist item 6a.

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

## Slice 6a: The agent's computer

Deliverable: Jarvis can browse the web in a per-tenant remote browser whose
logins persist; when it hits a login or 2FA wall a member finishes the step
from the ATLAS Computer page and Jarvis continues; browser minutes appear in
credits.

Depends on Jarvis J10 (Playwright MCP in the image, `computer` alias from
`ATLAS_COMPUTER_CDP_URL`, prompt rules) and J2 (actor and conversation
headers). Depends on slice 6 (projection and MCP hosting pattern).

- `packages/computer`: `BrowserProvider` interface, `HostedBrowserProvider`
  for the chosen provider, `FakeBrowserProvider`; profiles, sessions,
  grants repositories.
- CDP WebSocket broker route with auth, credit pre-check, session
  attach/create, minute metering, idle and max-duration sweeps.
- `atlas__computer` MCP server (status, request_human_help, end_session,
  list_downloads, fetch_download, reset_profile) and its projection entry.
- Slack help message posting through the tenant bot token into the
  originating thread.
- Dashboard Computer page: live view embed, take over, hand back, end
  session, reset logins, session history; Slack-link landing page.
- Price book SKU `computer_minute`; provisioning sets
  `ATLAS_COMPUTER_CDP_URL`.
- Tests: CU-01..CU-14; staging checklist item 7a.

Follow-on (not in the milestone): member-owned personal profiles;
self-hosted `FlyBrowserProvider` (headful Chrome on a per-tenant machine
with noVNC live view); per-tenant domain allow/deny lists; download
scanning; recordings retention controls.

## Slice 7: Usage, credential rotation, deletion, hardening

Deliverable: operator capabilities from the acceptance criteria are complete;
threat model M1 checklist is green.

- `POST /internal/runtimes/:id/usage` (optional runtime telemetry, J5),
  platform-side usage recorders for external users, connected accounts,
  and storage; `billing.reconcile` daily job against provider cost reports.
- Operator credential rotation (per-runtime, master key, per-tenant provider
  keys); rotation runbooks in `docs/operations.md` exercised in staging.
- Deletion state machine per ADR-0013; `runtime.destroy`; purge job.
- Rate limits reviewed; response schema secret scan in CI; log review.
- Tests: TI-09, PV-05, PV-10, SC-02, CR-14; staging checklist items 3, 8, 9.

## Slice 8 (first post-milestone): Credit purchase

Deliverable: owners buy credit packs through a hosted checkout; purchases
appear in the ledger idempotently; optional auto-top-up.

- Payment provider integration behind a `PaymentProvider` interface with a
  fake; webhook writes `purchase` entries keyed by payment reference.
- Dashboard "Add credits" flow; receipts; refund handling as `refund`
  entries.
- The ledger, statements, and enforcement from slice 4a are unchanged.

## Exit criteria for the milestone

- All test IDs in `docs/test-plan.md` sections 1 to 6 implemented and
  passing in CI.
- Staging checklist section 7 completed by someone other than the
  implementer, using a fresh Slack workspace.
- Threat model M1 checklist complete.
- Open decisions in `docs/open-decisions.md` either answered or explicitly
  deferred by the product owner with the default recorded.

## Explicitly not in this plan

Payment collection (slice 8, immediately after), Slack Marketplace listing,
native MCP OAuth, fleet-wide automatic upgrades, full support dashboard,
Enterprise Grid org-wide installs, mobile, migration of existing Jarvis
deployments, a second hosting provider adapter, self-hosted browser
backend and personal browser profiles. Usage metering, pricing,
credits, and enforcement are **in** the milestone (slice 4a) per the product
owner's decision; only the payment step is deferred. Extension points already
present: `subscriptions` table and suspension state, `connections.provider`
check constraint, `updateRelease` on the provisioner,
`RUNTIME_CONTRACT_VERSION`, ledger `purchase` entry type.
