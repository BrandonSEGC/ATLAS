# API and event contracts

All request and response bodies are defined as zod schemas in
`packages/contracts` and validated at the boundary. Responses use
`application/json`. Errors use one shape everywhere:

```json
{ "error": { "code": "connection_not_found", "message": "That connection does not exist.", "requestId": "req_00000001" } }
```

`code` is stable and machine-readable; `message` is plain language and never
contains provider payloads, stack traces, or identifiers from another tenant.
Not-found and forbidden are both returned as `404 not_found` for tenant-owned
resources so existence is not leaked across tenants; `403 forbidden` is used
only when the resource is in the caller's tenant but the role is insufficient.

Authentication classes:

| Class | Mechanism | Used by |
| --- | --- | --- |
| `public` | none | OAuth starts, static assets |
| `provider-signed` | Slack signing secret or Pipedream webhook signature over the raw body | `/webhooks/*` |
| `session` | `atlas_session` cookie -> `sessions` row -> member + role | `/api/*`, `/auth/logout` |
| `operator` | `session` with `is_platform_operator` | `/api/operator/*` |
| `runtime` | `Authorization: Bearer <runtime service token>` -> hash -> runtime + tenant | `/internal/*`, `/mcp/*` |

Rate limits (per IP unless stated): OAuth start 20/min, OAuth callback 20/min,
login 10/min, connection link 10/min per member, Slack events 1000/min per
team, MCP broker 600/min per runtime.

## 1. Public and authentication routes

### `GET /auth/slack/install`

Query: `return_to` (optional relative path). Creates an install state and
redirects (302) to Slack's `oauth/v2/authorize` with `client_id`, `scope`
(ADR-0012), `state`, `redirect_uri`. Sets cookie `atlas_oauth` (HttpOnly,
Secure, SameSite=Lax, 10 min) with the browser nonce.

### `GET /auth/slack/install/callback`

Query: `code`, `state`, or `error`. Behaviour:

| Condition | Result |
| --- | --- |
| `error` present | 302 to `/install?status=denied` |
| state missing, unknown, expired, consumed, or cookie nonce mismatch | 400 `invalid_oauth_state` page |
| Slack exchange fails or `ok: false` | 502 `slack_exchange_failed` page (no Slack payload shown) |
| success | 302 to `/onboarding?tenant=<slug>`; no session is created (installing and signing in are separate flows) |

Side effects on success (one transaction): tenant upsert, installation
upsert, owner member upsert, secrets write, audit `slack.installed`, job
`runtime.provision` with singleton key `tenant:<tenant_id>:provision`.

### `GET /auth/slack/login`

Query: `return_to`, `tenant` (optional slug to bind the login to a tenant).
Redirects to Slack's OpenID Connect authorize endpoint with scopes
`openid profile email`, `state`, `nonce`, PKCE `code_challenge` (S256).

### `GET /auth/slack/login/callback`

Validates state and cookie; exchanges code with `code_verifier`; validates ID
token (`iss = https://slack.com`, `aud = client_id`, signature via JWKS,
`exp`, `iat`, `nonce`). Then:

| Condition | Result |
| --- | --- |
| no active installation for `team_id`/`enterprise_id` | 302 `/install?reason=not_installed` |
| tenant `suspended` | 302 `/suspended` |
| `expected_tenant_id` set and does not match | 400 `tenant_mismatch` |
| member disabled | 403 page `member_disabled` |
| success | create member if new (role `member`), create session, set `atlas_session` cookie, 302 `return_to` or `/` |

### `POST /auth/logout`

Session. Revokes the session row and clears the cookie. CSRF: requires
`Origin` header matching `ATLAS_PUBLIC_URL` (applied to every state-changing
session route).

### `POST /webhooks/slack/events`

Provider-signed. Raw body retained. Request handling per
`docs/architecture.md` section 5.3. Responses:

| Case | Status | Body |
| --- | --- | --- |
| `url_verification` | 200 | `{ "challenge": "<from body>" }` |
| signature invalid or missing | 401 | empty |
| timestamp skew > 300 s | 401 | empty |
| body not JSON or missing `event_id` for `event_callback` | 400 | empty |
| duplicate `event_id` | 200 | empty |
| dropped (unknown/revoked/suspended/not ready/self-echo) | 200 | empty |
| queued | 200 | empty |
| database unavailable | 503 | empty (Slack retries) |

Handled event types in v1: `message` (all channel types, including subtypes
`file_share` and `bot_message`), `app_mention`, `reaction_added`,
`reaction_removed`, `app_uninstalled`, `tokens_revoked`, `member_joined_channel`.
Unknown types are acknowledged and recorded as `dropped/unsupported_type`.

### `POST /webhooks/slack/interactions`

Provider-signed with the same verifier. Interactivity is enabled on the app
manifest (ADR-0012) but not consumed in the first milestone: payloads are
acknowledged with 200 and counted as `dropped/unsupported_type`.

### `POST /webhooks/pipedream`

Provider-signed with the webhook secret configured on the ATLAS Connect
project (`X-Pipedream-Signature` HMAC over raw body; if Pipedream does not
provide one for the event type, the URL carries a path secret and the request
is treated as a hint that triggers a server-side account sync rather than as
trusted data). Payload fields used: `external_user_id`, `account.id`,
`account.app.name_slug`, `event` (`CONNECTION_SUCCESS`, `CONNECTION_ERROR`).
The handler only enqueues `connection.sync` for the matching connection; it
never writes provider data directly.

## 2. Authenticated dashboard API (`session`)

All routes resolve `TenantContext` from the session. Path IDs are looked up
with `tenant_id` in the query.

### `GET /api/me`

```json
{
  "member": { "id": "...", "displayName": "Casey", "role": "member", "slackUserId": "U0000000003" },
  "tenant": { "id": "...", "slug": "example-contractor-4f2a", "displayName": "Example Contractor", "status": "active" },
  "isPlatformOperator": false
}
```

### `GET /api/tenant`

Tenant summary plus installation summary (`teamName`, `botUserId`,
`installedAt`, `scopes`, `revoked`) and subscription entitlements. No tokens.

### `GET /api/members`

List of members (`id`, `displayName`, `role`, `status`, `lastLoginAt`).
Email is included only for `owner`/`admin` callers.

### `PATCH /api/members/:id/role`

Owner only for granting or removing `owner`; owner/admin for
`admin <-> member`. Body `{ "role": "admin" }`. Rejects demoting the last
owner (`409 last_owner`). Audit `member.role_changed`.

### `PATCH /api/members/:id/status`

Owner/admin. Body `{ "status": "disabled" }`. Disabling revokes the member's
sessions and their personal connections' projection entries.

### `GET /api/runtime`

```json
{
  "id": "...",
  "status": "ready",
  "statusText": "Ready",
  "release": "jarvis 1.4.2",
  "lastHealthAt": "2026-01-01T00:00:00Z",
  "lastHealthStatus": "ok",
  "provisionAttempts": 1,
  "canRetry": false
}
```

`statusText` is the plain-language label: Preparing, Ready, Needs attention,
Suspended, Failed. Provider references are never included.

### `POST /api/runtime/retry`

Owner/admin. Allowed only when `status in (failed, degraded)`. Enqueues
`runtime.provision` with the same singleton key (no duplicate). 202
`{ "status": "provisioning" }`. Audit `runtime.retry_requested`.

### `GET /api/connections`

Returns connections visible to the caller:

- all `tenant`-owned connections (`id`, `appSlug`, `displayName`, `status`,
  `alias`, `ownerType`, `createdAt`, `lastCheckedAt`);
- the caller's own `member`-owned connections in full;
- other members' personal connections as `{ appSlug, ownerDisplayName,
  status }` only, for owner/admin callers, without IDs.

### `GET /api/connections/apps?q=`

Proxy of the Pipedream app catalog restricted to `has_actions=true`, cached
for 10 minutes, returning `nameSlug`, `name`, `description`, `imgSrc`,
`authType`, and ATLAS policy `allowedOwnership: ["tenant","member"]`. The
policy table lives in `packages/connections/app-policy.ts` (for example
`gmail`, `google_calendar`, `microsoft_outlook`, `dropbox` are member-only by
default; `hubspot`, `quickbooks`, `procore` are tenant-only; unknown apps
allow both).

### `POST /api/connections/pipedream/link`

Body: `{ "appSlug": "gmail", "ownership": "member" }`.

| Check | Failure |
| --- | --- |
| app slug format and policy | 400 `unsupported_app` / 400 `ownership_not_allowed` |
| `ownership = tenant` and role is `member` | 403 `forbidden` |
| tenant suspended or runtime not `ready`/`degraded` | 409 `tenant_not_active` |
| live connection with same alias exists | 409 `already_connected` |

Response 201:

```json
{ "connectionId": "...", "connectLinkUrl": "https://pipedream.com/_static/connect.html?token=...&app=gmail", "expiresAt": "..." }
```

The `external_user_id` is computed server-side from the session and
`ownership`. The Connect token is not persisted. Audit
`connection.link_created` with `{ appSlug, ownerType }`.

### `GET /api/connections/:id/status`

Triggers a bounded sync (at most once per 10 seconds per connection) and
returns `{ "status": "connected", "accountLabel": "casey@example.com",
"authorizedScopes": [...], "lastCheckedAt": "..." }`. Personal connections are
visible only to their owner; others get 404.

### `POST /api/connections/:id/diagnose`

Owner/admin for tenant connections; owner of a personal connection. Runs the
staged diagnostic (ADR-0003) and returns:

```json
{
  "stages": [
    { "stage": "developer_token", "ok": true },
    { "stage": "account_lookup", "ok": true, "accounts": 1 },
    { "stage": "mcp_initialize", "ok": true },
    { "stage": "tool_list", "ok": false, "code": "upstream_error", "message": "Pipedream did not return tools for this app." },
    { "stage": "health_tool", "ok": null, "skipped": true }
  ]
}
```

### `DELETE /api/connections/:id`

Owner/admin for tenant connections; owner (or owner/admin) for personal.
Marks `revoked`, deletes the Pipedream account for that external user and
app, enqueues projection push. Audit `connection.revoked`.

## 3. Operator API (`operator`)

```text
GET  /api/operator/tenants                      # id, slug, displayName, status, runtimeStatus, memberCount, connectionCount
GET  /api/operator/tenants/:tenantId            # tenant, installation summary, runtime, subscription, recent drops
POST /api/operator/tenants/:tenantId/suspend    # body { reason }
POST /api/operator/tenants/:tenantId/resume
POST /api/operator/tenants/:tenantId/runtime/retry
POST /api/operator/tenants/:tenantId/runtime/rotate-credentials   # forwarding secret, admin token, runtime token
POST /api/operator/tenants/:tenantId/connections/:id/diagnose
GET  /api/operator/tenants/:tenantId/audit?cursor=
GET  /api/operator/health                        # counters: events, drops by reason, deliveries, provisioning
POST /api/operator/platform/rotate-master-key    # re-wrap tenant DEKs under the newest master key
```

Every operator mutation writes an audit event with `actor_type = operator`.

## 4. Runtime-facing API (`runtime`)

The bearer token identifies `(tenant_id, runtime_id)`. `:id` must equal
`runtime_id` or the request is rejected with 404 and audited as
`runtime.identity_mismatch`.

### `POST /internal/runtimes/:id/usage`

```json
{
  "events": [
    { "clientEventId": "run-00000001-1", "occurredAt": "...", "kind": "model_usage",
      "quantity": 1234, "unit": "tokens",
      "dimensions": { "provider": "anthropic", "model": "example-model", "direction": "input" },
      "estimatedCostUsd": 0.0037 }
  ]
}
```

At-least-once; duplicates by `(tenant_id, source, client_event_id)` are
ignored. Maximum 500 events per request. 202 `{ "accepted": 1, "duplicates": 0 }`.

### `POST /internal/runtimes/:id/heartbeat`

Optional in v1. Body `{ "release": "...", "uptimeSeconds": 1200 }`. Updates
`last_health_*`.

### `GET /internal/runtimes/:id/projection`

Returns the current projection (section 6.4 of the architecture document) so
a runtime can pull on boot once J1 lands. ETag = `last_projection_hash`.

### MCP broker: `/mcp/v1/connections/:connectionId`

Implements the MCP Streamable HTTP transport as a reverse proxy:

- `POST` JSON-RPC request bodies are forwarded unchanged; `Accept`,
  `Content-Type`, `Mcp-Session-Id`, and `Mcp-Protocol-Version` are passed
  through in both directions.
- `GET` opens the server-to-client SSE stream when the upstream supports it.
- `DELETE` closes the session.
- Added upstream headers: `Authorization: Bearer <platform developer access
  token>`, `x-pd-project-id`, `x-pd-environment`, `x-pd-external-user-id`,
  `x-pd-app-slug`, `x-pd-registry: all`, and `x-pd-tool-mode` from
  `connections.policy.toolMode` when set.
- Required inbound header for personal connections:
  `X-Atlas-Actor-Slack-User-Id`. Missing or mismatched -> JSON-RPC error
  `-32001` with message "This tool uses a personal connection. Ask the owner
  to run it, or connect your own account in ATLAS." and no upstream call.
- Upstream authorization-required responses are rewritten to JSON-RPC error
  `-32002` with a message telling the requesting member to reconnect the app
  in ATLAS; the connection is marked `unhealthy` and an audit event is
  written.
- Timeouts: 30 s request, 5 min SSE idle.

### Model gateway: `/model/v1/:provider/*` (`runtime`)

Reverse proxy to the model providers (ADR-0014). The runtime is configured
with `ANTHROPIC_BASE_URL=https://<atlas>/model/v1/anthropic` and its runtime
service token as the API key value, so authentication arrives as
`x-api-key` (Anthropic) or `Authorization: Bearer` (OpenAI-compatible); both
are accepted and mapped to the runtime.

| Step | Behaviour |
| --- | --- |
| auth | token hash -> `(tenant_id, runtime_id)`; runtime `ready`/`degraded`; tenant `active`; otherwise 401 in the provider's error envelope |
| credit check | `balance >= estimate` else 402 with provider-shaped body (Anthropic `{"type":"error","error":{"type":"billing_error","message":"..."}}`; OpenAI `{"error":{"code":"insufficient_quota","message":"..."}}`) |
| limits | per-tenant concurrent requests and requests per minute from entitlements; 429 in provider format |
| forward | path suffix, method, body, and headers passed through except auth headers, which are replaced with the tenant's provider key; `stream_options.include_usage=true` injected for OpenAI streams |
| meter | usage parsed from the response; `usage_events` + `credit_ledger` debit written on completion; provider request ID recorded |
| errors | provider errors passed through unchanged (status and body); no debit |

Allowed upstream paths per provider are an allow-list (for example Anthropic
`/v1/messages`, `/v1/messages/count_tokens`; OpenAI `/v1/chat/completions`,
`/v1/responses`, `/v1/embeddings`, `/v1/audio/transcriptions`,
`/v1/images/generations`). Anything else returns 404.

### `GET /api/credits`

Session. `{ "balanceMicros": 12500000, "balanceDisplay": "1,250 credits",
"burn7dMicros": ..., "burn30dMicros": ..., "lowThresholdReached": null }`.
Members receive balance fields only; owner/admin also receive
`usageByPeriod` (current month by SKU with quantity and priced amount).

### `GET /api/credits/ledger?cursor=`

Owner/admin. Paginated ledger entries without `actor_id` for system entries.

### `GET /api/credits/statements/:yyyy-mm`

Owner/admin. Statement per `docs/credits-and-billing.md`.

### Operator credit routes

```text
POST /api/operator/tenants/:tenantId/credits/grant        # body { amountMicros, memo, referenceId }  (idempotent on referenceId)
POST /api/operator/tenants/:tenantId/credits/adjust       # body { amountMicros, memo, referenceId }
GET  /api/operator/tenants/:tenantId/credits/reconciliation
GET  /api/operator/price-book                             # versions and items
POST /api/operator/price-book/versions                    # body { effectiveFrom, notes, items: [...] }
POST /api/operator/tenants/:tenantId/model-scopes/rotate  # rotate the tenant's provider key
```

## 5. ATLAS -> runtime calls (`packages/runtime-client`)

### Forward Slack event

```http
POST {ingress_url}/slack/events
Content-Type: application/json
X-Slack-Request-Timestamp: <unix seconds at send time>
X-Slack-Signature: v0=<hmac_sha256(forwarding_secret, "v0:<ts>:<raw body>")>
X-Atlas-Event-Id: Ev0000000001
X-Atlas-Tenant-Id: 00000000-0000-4000-8000-000000000001
X-Atlas-Runtime-Id: 00000000-0000-4000-8000-000000000002
X-Atlas-Attempt: 1

<byte-exact Slack event_callback body>
```

Success is any 2xx. 401 means the forwarding secret is out of sync
(runtime -> `degraded`, credential re-push). 5xx and network errors retry with
exponential backoff (1 s, 5 s, 30 s, 2 min, 10 min, then hourly to 24 h), then
`expired`.

### Push projection

Idempotent sequence against the runtime admin API with
`Authorization: Bearer <admin token>`:

1. `GET /api/admin/connectors` and diff by `jarvisConnectorId`/alias.
2. Upsert one `mcp-http` connector per projected server with `alias`, `url`,
   and the broker token as its credential; delete connectors that are no
   longer projected.
3. Until J1: `POST /api/admin/restart` when anything changed, then wait for
   `/health`.
4. Record `last_projection_hash`.

### Health

`GET {ingress_url}/health`, 3 s timeout, `ok` body expected. Two consecutive
failures -> `degraded`; one success -> `ready`.

## 6. Job contracts (`packages/jobs`)

| Job | Singleton key | Payload | Retry policy |
| --- | --- | --- | --- |
| `runtime.provision` | `tenant:<tenantId>:provision` | `{ tenantId, runtimeId }` | 8 attempts, exponential backoff from 30 s; on exhaustion `failed` |
| `runtime.push_projection` | `tenant:<tenantId>:projection` (debounced 5 s) | `{ tenantId, runtimeId }` | 5 attempts |
| `runtime.health_check` | scheduled every 60 s | `{}` fan-out per ready runtime | none |
| `runtime.deliver_slack_event` | `event:<eventId>` | `{ deliveryId, tenantId, runtimeId }` | schedule in section 5 |
| `runtime.suspend` / `runtime.resume` | `tenant:<tenantId>:lifecycle` | `{ tenantId, runtimeId, reason }` | 5 attempts |
| `runtime.wake` | `tenant:<tenantId>:lifecycle` | `{ tenantId, runtimeId }` | 5 attempts; `stopped -> starting -> ready`, then re-enqueues pending deliveries |
| `runtime.idle_sweep` | scheduled every 5 min | `{}` | stops runtimes with `last_activity_at` older than the idle window, no active deliveries, and `always_on = false` |
| `usage.meter_runtime_hours` | scheduled hourly | `{}` | writes `runtime_hour.<class>` usage and debits |
| `credits.notify_thresholds` | `tenant:<tenantId>:credits-notify` | `{ tenantId, level }` | 3 attempts |
| `billing.reconcile` | scheduled daily | `{}` fan-out per tenant and provider | 3 attempts |
| `runtime.destroy` | `tenant:<tenantId>:lifecycle` | `{ tenantId, runtimeId }` | 10 attempts; requires `tenants.status = deleting` and `purge_after < now()` |
| `connection.sync` | `connection:<connectionId>:sync` | `{ tenantId, connectionId }` | 3 attempts |
| `maintenance.purge` | scheduled hourly | `{}` | none |

Job payloads never contain secrets or message bodies. Every handler re-loads
state by ID inside a tenant transaction and re-checks preconditions, because
the job may run after the state that enqueued it has changed.

## 7. Versioning

- `packages/contracts` exports `RUNTIME_CONTRACT_VERSION = 1`. The projection
  document carries `version`. Breaking changes bump the version and the
  runtime-client keeps the previous serializer until every runtime is on a
  release that accepts the new one.
- Dashboard API changes are additive within `/api`. A breaking change gets
  `/api/v2`.
