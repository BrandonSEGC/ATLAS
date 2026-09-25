# Operations

Runbook outline for the first milestone. Procedures reference operator API
routes in `docs/api-contracts.md` section 3. Real hostnames, project IDs,
and credential locations are kept in the private operations record, never in
this repository.

## Environments and configuration

Every variable is validated by `packages/config` at startup. Values come from
the deployment's secret manager; `.env.example` lists names with synthetic
values.

| Variable | Notes |
| --- | --- |
| `ATLAS_ENV` | `local`, `staging`, `production` |
| `ATLAS_ROLES` | comma list of `api`, `gateway`, `worker` |
| `ATLAS_PUBLIC_URL` | HTTPS outside local; used for OAuth redirects and `Origin` checks |
| `DATABASE_URL` | `atlas_app` role; `MIGRATION_DATABASE_URL` for `atlas_migrator` |
| `ATLAS_MASTER_KEYS` | JSON key ring (ADR-0007) |
| `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_SIGNING_SECRET`, `SLACK_APP_ID` | distributed app |
| `PIPEDREAM_CLIENT_ID`, `PIPEDREAM_CLIENT_SECRET`, `PIPEDREAM_PROJECT_ID`, `PIPEDREAM_ENVIRONMENT`, `PIPEDREAM_WEBHOOK_SECRET` | platform Connect project |
| `PROVISIONER` | `fly`, `kubernetes`, `railway`, or `fake` |
| `FLY_API_TOKEN`, `FLY_ORG_SLUG`, `FLY_REGION`, `FLY_RUNTIME_APP` | Fly organization and app that holds tenant machines for this ATLAS environment |
| `RUNTIME_IDLE_MINUTES` | idle window before stop (default 30) |
| `BACKUP_BUCKET_URL`, `BACKUP_CREDENTIALS_REF` | per-tenant workspace backups |
| `JARVIS_RELEASE_IMAGE` | digest-pinned image reference (ADR-0011) |
| `ANTHROPIC_ADMIN_API_KEY`, `OPENAI_ADMIN_API_KEY` | platform administration keys used only to create per-tenant scopes and keys (ADR-0014); never used for inference |
| `MODEL_PROVIDERS_ENABLED` | comma list, default `anthropic,openai` |
| `COMPUTER_PROVIDER` | `hosted`, `fly` (later), or `fake` |
| `COMPUTER_PROVIDER_API_KEY`, `COMPUTER_PROVIDER_PROJECT_ID` | hosted browser provider project for this ATLAS environment (ADR-0016) |
| `COMPUTER_IDLE_MINUTES`, `COMPUTER_MAX_SESSION_MINUTES` | defaults 5 and 60 (OD-19) |
| `COMPUTER_DOWNLOAD_MAX_MB`, `COMPUTER_DOWNLOAD_ALLOWED_TYPES` | download policy (OD-21) |

## Routine operations

### See tenant and runtime health

`GET /api/operator/tenants` for the fleet; `GET /api/operator/tenants/:id`
for one tenant (runtime status, last health, recent drop reasons,
provisioning attempts, connection statuses). `GET /api/operator/health` for
counters.

### Retry failed provisioning

`POST /api/operator/tenants/:id/runtime/retry`. Inspect `provider_state` in
the tenant detail first; if the provider shows an orphaned resource with the
deterministic name, the retry adopts it.

### Suspend and resume a tenant

`POST /api/operator/tenants/:id/suspend` with a reason. Effects: runtime
stopped, runtime tokens revoked, event delivery dropped with reason
`suspended`, dashboard shows Suspended, logins allowed only to read the
notice. `POST .../resume` reverses it and re-pushes credentials and
projection.

### Inspect audit events

`GET /api/operator/tenants/:id/audit`. Metadata is safe by construction; if
an investigation needs message content, it must be obtained from the
customer's Slack workspace, not from ATLAS.

### Diagnose a Pipedream connection

`POST /api/operator/tenants/:id/connections/:cid/diagnose`. The failing stage
tells you where to look:

| Stage | Likely cause |
| --- | --- |
| `developer_token` | platform Pipedream credentials wrong or rotated |
| `account_lookup` | external user has no account for the app, or the account is dead; ask the member to reconnect |
| `mcp_initialize` | Pipedream MCP outage or broker misconfiguration |
| `tool_list` | app has no MCP tools or scope profile mismatch |
| `health_tool` | third-party API rejected the call; account needs re-authorization |

## Credential rotation

All rotations are operator actions and never display new values.

| Credential | Procedure |
| --- | --- |
| Per-runtime (forwarding secret, admin token, service token) | `POST /api/operator/tenants/:id/runtime/rotate-credentials`; verify runtime returns to `ready` and a test event is delivered |
| Master key ring | add a new key at the front of `ATLAS_MASTER_KEYS`, deploy, `POST /api/operator/platform/rotate-master-key`, confirm zero rows on the old key, remove it, deploy |
| Slack signing secret | regenerate in the Slack app settings, update `SLACK_SIGNING_SECRET`, deploy (Slack accepts both during a short overlap only if the app supports it; otherwise expect a brief 401 window that Slack retries) |
| Slack client secret | regenerate, update `SLACK_CLIENT_SECRET`, deploy; in-flight OAuth states will fail and users restart the flow |
| Pipedream client secret | create a new OAuth client secret in the Connect project, update, deploy, then revoke the old one |
| Hosting provider token | create a new token, update, deploy, revoke the old one |
| Per-tenant model provider key | `POST /api/operator/tenants/:id/model-scopes/rotate`: creates a new key in the tenant's provider scope, swaps it in the gateway, revokes the old one at the provider; no runtime change needed |
| Provider administration keys | rotate at the provider, update, deploy; they are used only during provisioning and rotation |
| Tenant bot token | happens through reinstall by the customer; ATLAS updates the runtime variable and redeploys |

## Deletion

`docs/adr/0013-tenant-deletion-and-backups.md`. Operators can cancel a
deletion during the grace window from the tenant detail page. The purge job
logs each destroyed resource by opaque reference and writes an audit event.

## Backups and restore

- Control-plane database: managed PITR and daily snapshots per ADR-0013.
  Restore into a fresh database, point a staging ATLAS at it with the same
  key ring, verify, then cut over. Sessions and OAuth states are discarded.
- Runtime volumes: provider snapshots; restore is per tenant and requires
  the runtime to be suspended first.

## Credits and pricing

- Grant pilot credits: `POST /api/operator/tenants/:id/credits/grant` with a
  memo and a unique `referenceId` (for example the internal ticket number)
  so retries cannot double-grant.
- Publish new prices: create a price book version with `effectiveFrom` in
  the future; history is never re-priced. Announce to tenants before the
  effective time.
- Investigate a drift alert: open the reconciliation row, compare the
  provider's report for the tenant scope with the ledger for the same day;
  common causes are provider price changes not yet in the price book and
  usage the provider reports in a different timezone bucket.
- Incident bypass: if the gateway is degraded, an operator can set a tenant's
  `modelGatewayBypass` entitlement, which injects the tenant's provider key
  directly into the runtime and redeploys. Metering falls back to
  reconciliation only; clear the flag and redeploy once resolved.

## The agent's computer

- A tenant reports "Jarvis is stuck in the browser": open the tenant's
  computer view, check whether a human holds control (a forgotten takeover
  blocks the agent until the grant expires), end the session if needed.
- Reset a tenant's browser logins on request from an owner: use the
  dashboard action as that owner, or the operator endpoint with a memo;
  both are audited.
- Provider outage: sessions fail to create; the agent receives a plain
  "browser unavailable" tool error and continues without it. Watch the
  `computer.session_create_failed` counter.
- Rotate `COMPUTER_PROVIDER_API_KEY`: create a new key in the provider
  project, update, deploy, revoke the old one; running sessions continue
  because the provider authenticates sessions, not keys, at the CDP layer
  (verify with the provider).

## Sleeping runtimes and wake

Runtimes show `Sleeping` after the idle window. A Slack message wakes them;
if wake fails twice the runtime moves to `Needs attention` and the delivery
stays queued for the retry schedule. Tenants with scheduled tasks are marked
`always_on` in the tenant detail and are never stopped.

## Incident notes

- Gateway 401 spike: check whether the platform signing secret was rotated
  without a deploy.
- Deliveries expiring for one tenant: runtime unreachable; check health,
  provider deployment state, and forwarding secret sync (runtime 401s mark
  `degraded`).
- Broker errors for one app across tenants: Pipedream-side; run diagnostics
  on one connection to find the stage.
- Broker errors for one tenant only: check runtime token revocation and
  connection status.
