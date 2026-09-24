# ATLAS architecture

Status: planning baseline for the first milestone. Changes to tenancy,
identity, credential ownership, or deletion require a new ADR in `docs/adr/`.

## 1. Purpose and boundary

ATLAS is the control plane. Jarvis is the data plane.

| Concern | Owner |
| --- | --- |
| Signup, tenant and member identity, roles | ATLAS |
| Slack app installation, Sign in with Slack, installation tokens | ATLAS |
| Slack Events API ingestion, verification, dedup, routing | ATLAS |
| Pipedream developer credentials, Connect Links, connection ownership | ATLAS |
| Runtime provisioning, lifecycle, health, release version | ATLAS |
| Usage metering, audit, subscription/entitlement state | ATLAS |
| Agent process, workspace, memory, skills, tool execution | Jarvis (one per tenant) |
| Customer files, agent configuration inside the workspace | Jarvis (one per tenant) |

ATLAS never runs a shared multi-tenant agent, never reads a tenant workspace,
and never persists customer conversation content or model reasoning. Jarvis
never holds platform-wide credentials (Pipedream developer client, Railway
token, Slack signing secret of the distributed app).

The Jarvis repository stays independent. ATLAS provisions a versioned Jarvis
container image (ADR-0011) and talks to it only through the runtime contract
in section 6.

## 2. Deployables for the first milestone

One deployable TypeScript service, `apps/control-plane`, runs four logical
roles behind explicit module boundaries so they can be split later without a
rewrite:

```text
apps/control-plane (single Node.js process per replica)
  |-- http: web (dashboard SPA assets + server routes)
  |-- http: api (/auth, /api, /webhooks, /internal, /mcp)
  |-- http: slack gateway (/webhooks/slack/events) -- same process, isolated module
  +-- worker: pg-boss job handlers (provisioning, delivery, health, projection)
```

Role selection is by environment (`ATLAS_ROLES=api,gateway,worker`). Local,
staging, and production each run all roles in one service. The Slack gateway
and worker are the first candidates for separate deployables once event volume
or provisioning latency justifies it; nothing in the code may assume they share
memory (no in-process caches used for correctness, no in-memory dedup).

Supporting infrastructure:

- PostgreSQL: all control-plane state, sessions, job queue (pg-boss), dedup
  records, audit and usage events.
- Encrypted secret storage: a PostgreSQL table written only through
  `packages/secrets` using envelope encryption (ADR-0007). No external KMS is
  required for the pilot; the interface allows one later.
- Provisioning provider: Railway via its public GraphQL API (ADR-0009), behind
  the provider-neutral `RuntimeProvisioner` interface.

## 3. Module map

`docs/repository-structure.md` has the directory layout. The modules and their
allowed dependencies:

| Module | Responsibility | May depend on |
| --- | --- | --- |
| `contracts` | Zod schemas and TS types for HTTP, webhook, runtime, job payloads. Versioned. | nothing |
| `config` | Environment parsing and validation at startup; fails closed. | contracts |
| `database` | Migrations, `TenantContext`, tenant-scoped repositories, RLS session helpers. | config |
| `secrets` | Envelope encryption, key ring, redaction helpers. | config, database |
| `auth` | Slack install OAuth, Sign in with Slack (OIDC), sessions, role checks. | database, secrets, contracts |
| `slack-gateway` | Raw-body signature verification, URL verification, dedup, tenant resolution, self-echo rejection, enqueue delivery. | database, contracts |
| `runtime-client` | Authenticated calls to a runtime: forward event, push projection, health, and the runtime-facing MCP broker auth. | database, secrets, contracts |
| `provisioning` | `RuntimeProvisioner` interface, state machine, idempotency, `RailwayProvisioner`, `FakeProvisioner`. | database, secrets, contracts |
| `connections` | Pipedream Connect Links, account sync, connection ownership rules, MCP broker, staged diagnostics. | database, secrets, contracts |
| `usage` | Ingest runtime usage reports and platform-side usage (external users, accounts, uptime). | database |
| `audit` | Append-only audit events with metadata allow-list. | database |
| `jobs` | pg-boss wiring, job names, retry policies, idempotency keys. | database, contracts |
| `apps/control-plane` | Fastify server composition, routes, worker bootstrap, dashboard UI. | all of the above |

Rules:

- Only `database` issues SQL. Repositories require a `TenantContext` for any
  tenant-owned table; platform-scoped tables (`oauth_states`, `job` internals,
  `slack_event_receipts` before resolution) use a separate `PlatformContext`
  so a missing tenant cannot be defaulted.
- Only `secrets` touches ciphertext. Everything else deals in opaque
  `SecretRef` values.
- Provider SDKs (Slack Web API, Railway GraphQL, Pipedream REST/MCP) are
  wrapped by a small interface with a fake implementation in the same package,
  used by tests.

## 4. Tenancy model

Full schema: `docs/data-model.md`. Invariants:

1. Every tenant-owned row carries `tenant_id`. Composite foreign keys
   (`(tenant_id, member_id)`) keep child rows inside the parent's tenant.
2. Identity is scoped: a member is `(tenant_id, slack_user_id)`, an
   installation is `(team_id, enterprise_id)`, and a runtime is
   `(tenant_id, runtime_id)`. A Slack user ID or an email is never a global
   identity.
3. Tenant identity for a dashboard request is derived from the server-side
   session row, never from a client-supplied value.
4. Tenant identity for a runtime request is derived from the runtime service
   token hash, never from a path or header value. Path `:id` values are
   compared to the authenticated identity and mismatches are rejected.
5. PostgreSQL row-level security is enabled on tenant-owned tables as defense
   in depth; the application sets `app.tenant_id` per transaction
   (ADR-0001). Repository-level scoping remains the primary control.
6. Roles: `owner` (installer, full control, cannot be removed while sole
   owner), `admin` (members, shared connections, runtime retry), `member`
   (own personal connections, read tenant and runtime status). Role changes
   are explicit; Slack workspace admin status is never mapped automatically.

## 5. Primary flows

### 5.1 Add to Slack (installation)

1. `GET /auth/slack/install` creates an `oauth_states` row (`purpose =
   install`, 10 minute TTL, single use), sets a short-lived HttpOnly cookie
   holding the state's browser binding nonce, and redirects to
   `https://slack.com/oauth/v2/authorize` with the bot scopes in ADR-0012.
2. `GET /auth/slack/install/callback` validates state (exists, unexpired,
   unused, cookie nonce matches), marks it consumed in the same transaction,
   exchanges `code` server-side, and validates `ok`, `team.id`, `bot_user_id`,
   `authed_user.id`, and `scope`.
3. In one transaction, upsert the tenant by `(slack_team_id,
   slack_enterprise_id)`, upsert the installation by `team_id`, store the bot
   token as a `SecretRef`, upsert the installer as `owner`, write an audit
   event, and enqueue `runtime.provision` with idempotency key
   `tenant:<id>:provision`.
4. Redirect to `/onboarding?tenant=<slug>` without any token material.

Replaying the callback is rejected by state consumption. Reinstalling the app
in the same workspace updates the existing installation (new token, scopes,
`revoked_at = null`) and does not create a second tenant or provisioning job.

### 5.2 Sign in with Slack (dashboard login)

1. `GET /auth/slack/login` creates an `oauth_states` row (`purpose = login`)
   with a `nonce` and PKCE verifier, binds it to a browser cookie, and
   redirects to `https://slack.com/openid/connect/authorize` with scopes
   `openid profile email`.
2. The callback validates state and cookie binding, exchanges the code, and
   verifies the ID token signature against Slack's JWKS, plus `iss`, `aud`,
   `exp`, `iat`, and `nonce`.
3. Resolve `https://slack.com/team_id` and `https://slack.com/user_id`
   claims. Look up an active installation for that team. If none: show
   "Install Jarvis first" and stop. Look up or create the member by
   `(tenant_id, slack_user_id)` with role `member` (the installer already
   exists as `owner`).
4. Create a `sessions` row and set an HttpOnly, Secure, SameSite=Lax cookie
   containing only the opaque session ID (ADR-0006).

Email and display name from the ID token are stored as display data only and
refreshed on each login; they are never used for lookup.

### 5.3 Slack event ingestion and routing

```text
Slack --POST /webhooks/slack/events--> gateway
  1. read raw body bytes (no JSON parsing yet)
  2. verify X-Slack-Signature over v0:<ts>:<raw>, reject |now - ts| > 300s
  3. url_verification -> respond challenge
  4. parse minimal fields: type, event_id, team_id, enterprise_id, api_app_id,
     event.type, event.channel, event.user, event.bot_id, event.ts, event.thread_ts
  5. INSERT slack_event_receipts(event_id) ON CONFLICT DO NOTHING
       -> duplicate: respond 200, done
  6. resolve installation by (team_id, enterprise_id); require tenant.status =
     active and installation.revoked_at IS NULL and runtime.status in
     (ready, degraded) -> otherwise respond 200, record drop reason, done
  7. exact self-echo: event.user == installation.bot_user_id -> 200, drop
  8. store raw body in slack_event_deliveries(tenant_id, runtime_id, status =
     queued) and enqueue runtime.deliver_slack_event(delivery_id) with
     singletonKey = event_id
  9. respond 200 within the 3 second budget
worker: load delivery, sign raw body with the runtime's forwarding secret
        using Slack's v0 HMAC scheme (ADR-0002, ADR-0010), POST to the
        runtime's /slack/events, retry with backoff, mark delivered/failed
```

Policy for messages the gateway cannot route (unknown team, revoked, suspended,
runtime not ready): acknowledge to Slack, persist a receipt with a drop reason,
increment a counter for the operator view, never forward. Bot-authored
messages from other apps are forwarded normally; Jarvis decides whether to
respond. The forwarded body is the byte-exact Slack payload; nothing is
truncated (ADR-0002).

Outbound messages are sent by the runtime with the tenant's bot token, which
ATLAS injects into that runtime only (section 6.3).

### 5.4 Runtime provisioning

`runtime.provision` job (idempotent on `tenant:<id>:provision`):

```text
requested -> provisioning -> ready
                 |-> failed (retryable via POST /api/runtime/retry)
ready <-> degraded (health checks)
ready|degraded -> suspended -> ready (resume)
suspended -> destroying -> destroyed
```

Steps, each retry-safe and each recorded in `runtime_instances.provider_state`:

1. Generate per-runtime secrets if absent: forwarding secret (Slack v0 HMAC
   key), admin token, runtime service token (ATLAS stores only its hash),
   `JARVIS_MASTER_KEY`, and the model provider key for this tenant
   (ADR-0007, open decision OD-3).
2. `provisioner.provision()` with the idempotency key: create or find the
   service, attach or find the volume mounted at `/data`, set variables,
   pin the image reference to the current Jarvis release, deploy.
3. Poll `GET /health` on the private ingress until it returns `ok` or the
   deadline passes.
4. Push the configuration projection (section 6.4) and mark `ready`.
5. Audit `runtime.provisioned`.

`runtime.health_check` runs on a schedule per ready runtime and moves state
between `ready` and `degraded`. `runtime.suspend` stops the service; the
gateway checks tenant and runtime status before enqueuing delivery, so
suspended tenants receive nothing. `runtime.destroy` runs only from
`tenants.status = deleting` and after the retention window (ADR-0013).

### 5.5 Pipedream connection

1. Member opens Connections, picks an app, and picks `personal` or `shared`.
   `shared` requires `owner` or `admin`. The server decides the external user
   ID from the session and the choice, never from the request body:
   `tenant:<tenant_id>` or `tenant:<tenant_id>:member:<member_id>` (ADR-0003).
2. `POST /api/connections/pipedream/link` creates a `connections` row in
   `pending`, then a Connect token/link scoped to that external user and the
   production environment (development only when `ATLAS_ENV=local|test`),
   appends `app=<slug>`, and returns the link URL and expiry. The link is not
   persisted.
3. After redirect back to `/connections?connection=<id>`, the dashboard calls
   `GET /api/connections/:id/status`; the server lists accounts for the
   external user and app from Pipedream, records the account ID, health, and
   authorized scopes, and moves the row to `connected`. `POST
   /webhooks/pipedream` performs the same sync when a signed webhook arrives.
4. The connection change enqueues `runtime.push_projection` for the tenant.
   The projection contains one MCP server entry per connection pointing at
   the ATLAS MCP broker (section 6.5). Jarvis reconnects without a process
   restart once the Jarvis-side change J1 lands; until then, the projection
   push triggers a supervised restart.

## 6. Runtime contract (ATLAS <-> Jarvis)

This section is grounded in the current Jarvis codebase. Paths cited are in
the Jarvis repository.

### 6.1 What Jarvis provides today

| Capability | Jarvis surface | Notes |
| --- | --- | --- |
| Inbound Slack over HTTP | `POST /slack/events` (`src/adapters/slack-webhook.ts`) | Selected automatically when only `MOM_SLACK_BOT_TOKEN` is set (no app token). Verifies Slack `v0` HMAC using `MOM_SLACK_SIGNING_SECRET` with a 300 s skew window; trusts `x-crawdad-dev-verified: true` (ATLAS will not use that header). Handles `url_verification`. Exact self-echo rejection on `event.user === botUserId`; bot messages otherwise treated as participants. In-memory dedup only. |
| Outbound Slack | `chat.postMessage` with `MOM_SLACK_BOT_TOKEN` (`src/adapters/slack-base.ts`) | Thread routing: channel messages reply in `thread_ts ?? ts`; DMs reply at channel level. |
| Health | `GET /health` -> `200 ok` (`src/gateway.ts`) | No readiness or version endpoint. |
| Admin/config | `/api/admin/*` with `Authorization: Bearer <JARVIS_ADMIN_TOKEN>` (`src/admin/service.ts`) | Connectors, credentials, settings, `POST /api/admin/restart` (supervised exit 75). |
| MCP servers | `settings.json` `mcpServers[]` with `alias`, `transport: http`, `url`, `tokenEnv` or `headers` (`src/mcp-client/config.ts`, `src/admin/managed-config.ts`) | Tools are exposed as `<alias>__<tool>`. Managed `mcp-http` connectors get token env `JARVIS_MCP_<ID>_TOKEN`. |
| Pipedream today | `src/pipedream.ts`, `managed-config.ts` ~1871 | One connector -> one MCP entry with alias `pipedream` and `x-pd-app-slug` = comma-joined slugs; obtains its own Pipedream access token with `PIPEDREAM_CLIENT_ID/SECRET` in the runtime. Both conflict with the blueprint; ATLAS does not use this path. |
| Secrets at rest | `.jarvis/secrets.json` encrypted with `JARVIS_MASTER_KEY` | Per-runtime key generated by ATLAS. |
| Persistence | positional workspace dir (`/data` in the image) | `IDENTITY.md`, `MEMORY.md`, `settings.json`, `.jarvis/`, `log.jsonl`, `attachments/`, `skills/`. One volume per runtime. |
| Usage | JSON log lines `{ event: "model_usage", provider, model, inputTokens, outputTokens, estimatedUsd }` (`src/agent.ts`) | No HTTP export yet. |
| Image | `Dockerfile`: `node:22-bookworm`, `EXPOSE 3002`, `ENTRYPOINT crawdad`, CMD `--sandbox=host --port=3002 --ui=... --skills=... /data` | No published registry image or `HEALTHCHECK` yet. |

Everything is process-global: one workspace, one Slack token set, one
connector set per process. That is exactly why isolation is per deployment
(ADR-0001).

### 6.2 Contract version 1 (what ATLAS relies on)

ATLAS -> runtime (over the private network, ADR-0010):

| Blueprint name | Concrete call in v1 | Auth |
| --- | --- | --- |
| `POST /internal/runtimes/:id/events/slack` | `POST <runtime>/slack/events` with byte-exact Slack body, `X-Slack-Request-Timestamp`, `X-Slack-Signature` computed with the per-runtime forwarding secret, plus informational `X-Atlas-Event-Id`, `X-Atlas-Tenant-Id`, `X-Atlas-Runtime-Id`, `X-Atlas-Attempt` | HMAC (forwarding secret); network privacy |
| `PUT /internal/runtimes/:id/config-projection` | Admin API calls: upsert `mcp-http` connectors (one per connection), settings, then `POST /api/admin/restart` until J1 provides hot reload | Bearer per-runtime admin token |
| `POST /internal/runtimes/:id/connections/changed` | Same as projection push (full, idempotent projection every time) | Bearer per-runtime admin token |
| `GET /internal/runtimes/:id/health` | `GET <runtime>/health` | network privacy (no secret) |
| Runtime version | Image reference recorded by ATLAS at deploy; `GET /version` after J4 | n/a |
| Suspend/resume | Provider-level stop/start of the service (volume retained) | provider |

Runtime -> ATLAS (public HTTPS or private network):

| Purpose | Endpoint | Auth |
| --- | --- | --- |
| MCP tool traffic for Pipedream connections | `POST/GET/DELETE /mcp/v1/connections/:connectionId` (MCP Streamable HTTP, proxied) | Bearer runtime service token; `X-Atlas-Actor-Slack-User-Id` for personal connections (J2) |
| Usage reports | `POST /internal/runtimes/:id/usage` | Bearer runtime service token (after J5) |
| Heartbeat / registration | `POST /internal/runtimes/:id/heartbeat` | Bearer runtime service token (optional; ATLAS polls `/health` in v1) |

The `/internal/runtimes/:id/...` paths from the blueprint are therefore
implemented in ATLAS as the runtime-facing endpoints, and the ATLAS -> runtime
direction is implemented in `packages/runtime-client` against the Jarvis
surface above. The `:id` in every path must equal the identity proven by the
credential, otherwise the request is rejected and audited.

### 6.3 Runtime environment injected by ATLAS

Set as provider service variables at provision time; all values are unique per
tenant and generated by ATLAS unless noted.

| Variable | Purpose |
| --- | --- |
| `MOM_SLACK_BOT_TOKEN` | This tenant's installation bot token (rotated by updating the variable and redeploying). |
| `MOM_SLACK_SIGNING_SECRET` | Per-runtime forwarding secret used by ATLAS to sign forwarded events. Not the distributed app's signing secret. |
| `JARVIS_ADMIN_TOKEN` | Per-runtime admin token used by ATLAS for projection pushes. |
| `JARVIS_MASTER_KEY` | Per-runtime key for `.jarvis/secrets.json`. |
| `ATLAS_BASE_URL`, `ATLAS_RUNTIME_ID`, `ATLAS_RUNTIME_TOKEN`, `ATLAS_TENANT_ID` | Runtime -> ATLAS identity and the MCP broker/usage endpoints. |
| Model provider key | Per-tenant key (OD-3). |

The broker bearer token for each projected `mcp-http` connector is delivered
through the projection push (admin API), which stores it in the runtime's own
encrypted `.jarvis/secrets.json` and hydrates `JARVIS_MCP_<ID>_TOKEN` at boot.
It is the same runtime service token; no per-connection secret exists.

Never injected: Pipedream developer credentials, Railway token, distributed
Slack app client secret or signing secret, platform database credentials.

### 6.4 Configuration projection

ATLAS computes a full projection for a tenant and pushes it idempotently:

```json
{
  "version": 1,
  "tenantId": "00000000-0000-4000-8000-000000000001",
  "runtimeId": "00000000-0000-4000-8000-000000000002",
  "mcpServers": [
    {
      "alias": "pipedream__hubspot",
      "transport": "http",
      "url": "https://atlas.example.com/mcp/v1/connections/00000000-0000-4000-8000-000000000010",
      "auth": { "type": "runtime-token" },
      "ownership": { "type": "tenant" },
      "appSlug": "hubspot",
      "displayName": "HubSpot (company)"
    },
    {
      "alias": "pipedream__gmail__m00000003",
      "transport": "http",
      "url": "https://atlas.example.com/mcp/v1/connections/00000000-0000-4000-8000-000000000011",
      "auth": { "type": "runtime-token" },
      "ownership": {
        "type": "member",
        "memberId": "00000000-0000-4000-8000-000000000003",
        "slackUserId": "U0000000003"
      },
      "appSlug": "gmail",
      "displayName": "Gmail (Casey, personal)"
    }
  ]
}
```

Alias rules (ADR-0003): shared connections use `pipedream__<app_slug>`;
personal connections use `pipedream__<app_slug>__m<first 8 hex of member
id>`. Aliases are stable for the life of a connection and unique per tenant.
The ownership block lets Jarvis restrict personal tools to turns initiated by
the owning Slack user and lets the broker enforce the same rule.

### 6.5 MCP broker for Pipedream

The runtime never holds Pipedream developer credentials. Each projected MCP
server points at ATLAS, which proxies MCP Streamable HTTP to
`https://remote.mcp.pipedream.net/v3`:

```text
runtime --Bearer <runtime token>--> ATLAS /mcp/v1/connections/:connectionId
  1. resolve runtime by token hash; require runtime.status in (ready, degraded)
  2. load connection; require connection.tenant_id == runtime.tenant_id and
     connection.status == connected, otherwise 404 (fail closed)
  3. personal connection: require X-Atlas-Actor-Slack-User-Id to match the
     owning member's slack_user_id, otherwise 403 with an MCP error whose
     message is a safe "connect this app in ATLAS" prompt
  4. obtain the platform developer access token (client credentials, cached)
  5. forward with x-pd-project-id, x-pd-environment, x-pd-external-user-id
     (derived from the connection owner), x-pd-app-slug (exactly one slug),
     x-pd-registry: all, and pass Mcp-Session-Id through both ways
  6. record usage_events(kind = tool_call) with tool name and outcome only
```

This puts ATLAS in the tool-call path. The alternative, vending a Pipedream
access token to the runtime, is rejected because client-credentials tokens are
scoped to the whole Connect project and would let a compromised runtime name
any external user (ADR-0003). Revisit if Pipedream offers per-external-user
tokens accepted by its MCP server.

Diagnostics (`POST /api/connections/:id/diagnose`, owner/admin) run the same
path in stages and return only the failing stage name and a sanitized message:
`developer_token`, `account_lookup`, `mcp_initialize`, `tool_list`,
`health_tool`.

### 6.6 Required Jarvis-side changes

These are tracked in the Jarvis repository and land as separate PRs there.
ATLAS milestone slices note which ones they depend on.

| ID | Change | Needed by |
| --- | --- | --- |
| J1 | Hot reload of managed `mcp-http` connectors (call `mcpBridge.reconnectServer` on connector upsert instead of `restartRequired`), or a dedicated `atlas` connector type that reads a projection file and reconnects. | Slice 6 polish; slice 6 works with restart. |
| J2 | Include `X-Atlas-Actor-Slack-User-Id` (and channel ID) on MCP requests made during a turn, sourced from the authenticated inbound event, and honour the `ownership` block so personal tools are offered only to their owner. | Personal connections (slice 6). |
| J3 | Optional: reject `/slack/events` requests whose `X-Atlas-Runtime-Id` does not match `ATLAS_RUNTIME_ID`. Defense in depth only. | Post-pilot. |
| J4 | `GET /version` returning the release identifier (`JARVIS_RELEASE` baked at image build). | Operator view; optional for pilot. |
| J5 | Post `model_usage` records to `ATLAS_BASE_URL/internal/runtimes/:id/usage` with the runtime token, batched, at-least-once with a client event ID. | Usage metering before billing; optional for pilot. |
| J6 | CI workflow publishing `ghcr.io/<org>/jarvis:<semver>` and `:sha-<commit>` on tagged releases, with `HEALTHCHECK` and `JARVIS_RELEASE` in the image. | Slice 4 (provisioning) needs a pullable image. |
| J7 | Startup warning when `MOM_SLACK_BOT_TOKEN` is set without an app token already selects webhook mode; confirm `--adapter=slack:webhook` is passed explicitly by ATLAS so auto-detection changes cannot alter behaviour. | Slice 4. |

## 7. Environments

| Environment | Purpose | Slack app | Pipedream env | Provider |
| --- | --- | --- | --- | --- |
| local | developer machine, `FakeProvisioner`, fake Slack/Pipedream servers | dev app or none | development | none / docker |
| staging | end-to-end with real Slack dev app and Railway | dev app | development | Railway staging pool |
| production | customers | distributed app | production | Railway production pool |

`ATLAS_ENV` selects defaults; `config` refuses to start production with
development Pipedream environment, HTTP callbacks, or missing key material.

## 8. Observability and operations

- Structured JSON logs with a redaction layer keyed on known secret field
  names and token-shaped prefixes (`xoxb-`, `xapp-`, `xoxe`, `Bearer `).
  Slack event bodies are not logged by default.
- Metrics counters: events received/dropped by reason, deliveries
  succeeded/failed, provisioning transitions, broker calls by stage.
- Operator endpoints (`/api/operator/*`, ADR-0006 operator role) expose
  tenant and runtime status, provisioning retry, suspend/resume, safe audit
  events, and staged diagnostics. See `docs/operations.md`.
