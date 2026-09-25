# Open product decisions

Each item names the default that implementation will follow if no answer
arrives, so work is never blocked. Items marked "blocks" change tenant
ownership, security, deletion, or pricing and should be confirmed before the
slice that depends on them.

| ID | Decision | Default adopted | Blocks | Why it matters |
| --- | --- | --- | --- | --- |
| OD-1 | Deletion grace window and backup retention (ADR-0013) | 30 day grace; database PITR 7 days plus daily snapshots 30 days; volume backups capped at 30 days | Slice 7 | Customer terms, storage cost, compliance answers |
| OD-2 | Who may request tenant deletion | Any `owner` with typed confirmation; operators; automatic after 14 days uninstalled | Slice 7 | Accidental deletion risk versus self-service |
| OD-3 | Model provider key ownership per tenant | **Decided**: per-tenant provider scope and key, held only by ATLAS; all model traffic through the ATLAS model gateway; real-time metering into a credit ledger with pass-through pricing (ADR-0014) | Slice 4a | Cost attribution, blast radius of a compromised runtime, pricing |
| OD-4 | Should the installing user be the only automatic `owner`? | Yes; every other user who signs in is `member` until promoted. Slack workspace admin status is ignored | Slice 2 | Tenant ownership |
| OD-5 | Can a `member` see that other members have personal connections? | Only `owner`/`admin` see `{ appSlug, ownerDisplayName, status }`; members see only their own | Slice 5 | Privacy expectations inside a company |
| OD-6 | App ownership policy list (which apps are personal-only, shared-only, or both) | Initial list in `packages/connections/app-policy.ts`: mail, calendar, personal drive apps are member-only; CRM, accounting, project management are tenant-only; unknown apps allow both | Slice 5 | Confused-deputy risk and support burden |
| OD-7 | Which channel messages reach Jarvis | Everything the installation is authorized for (DMs, group DMs, channels the bot was invited to); Jarvis decides whether to respond | Slice 3 | Noise, cost, and privacy inside customer workspaces |
| OD-8 | Runtime resource class and region for the pilot | One `standard` class, one region, configurable per environment | Slice 4 | Cost and latency |
| OD-9 | Pilot entitlements | `maxMembers: 10`, `maxConnections: 10`, enforced with plain messages; no billing | Slice 5 | Sets the shape of future plans |
| OD-10 | Automatic suspension triggers | **Decided**: exhausted credits stop model calls only (gateway 402 with a plain message and owner DM); Slack, tools, and the runtime stay up. Whole-tenant suspension remains a manual operator action | Slice 4a | Pricing and abuse handling |
| OD-11 | Customer-visible name for the runtime and the bot | "Jarvis" as the bot name in Slack, "your private Jarvis runtime" in the dashboard | Slice 1 | Branding of the distributed Slack app cannot change without reinstalls |
| OD-12 | Hosting provider for tenant runtimes | **Recommendation for confirmation**: Fly.io Machines first (per-tenant microVM and volume, API stop/start, private network), Kubernetes as the second-tier adapter, Railway optional; plus provider-independent wake-on-event and object-storage workspace backups (ADR-0015) | Slice 4 | Isolation, idle cost, wake latency, durability, ops burden |
| OD-13 | Reinstall semantics when a different user reinstalls the app | The new installer is added as `owner` alongside existing owners; nobody is demoted automatically | Slice 1 | Tenant ownership |
| OD-14 | Pass-through pricing policy | `unit_price = provider cost x 1.25` for model and tool SKUs; `runtime_hour.standard` priced to cover hosting at roughly 1.5x; credits displayed as 1 credit = USD 0.01; pilot tenants funded by operator grants | Slice 4a | Revenue, competitiveness, statement clarity |
| OD-15 | Low-credit behaviour details | Thresholds at 20 / 5 / 0 percent of last top-up; DM to every owner; no grace below zero (`allowNegativeUntilMicros = 0`) | Slice 4a | Customer experience when credits run out |
| OD-16 | Idle stop window | 30 minutes without inbound events, gateway calls, or active runs; tenants with scheduled tasks are `always_on`; the plan may sell `always_on` as an entitlement | Slice 4 | Idle cost versus first-reply latency |
| OD-17 | Which model providers to enable at launch | Anthropic and OpenAI through the gateway; Fireworks behind a flag | Slice 4a | Provider scope creation and price book scope |
| OD-18 | Browser backend for the agent's computer | Hosted browser provider (persistent contexts, interactive live view, downloads API) behind `BrowserProvider`; self-hosted Fly machine backend later (ADR-0016) | Slice 6a | Time to market, privacy posture, per-minute cost |
| OD-19 | Session limits | 1 concurrent session per tenant profile; 5 minute idle timeout; 60 minute max, auto-extended while the agent is active; `computer_minute` priced at provider hourly cost / 60 x markup | Slice 6a | Cost, fairness, and abuse |
| OD-20 | Tool mode | Accessibility-snapshot tools (Playwright MCP) by default; vision/screenshot mode enabled as a fallback capability | Slice 6a | Reliability and token cost |
| OD-21 | Downloads and recordings | Allow documents, spreadsheets, images, archives up to 50 MB; refuse executables and scripts; session recordings disabled by default (enable per tenant on request) | Slice 6a | Malware risk, privacy, storage cost |
| OD-22 | Whose browser logins are shared | v1 has one shared company profile per tenant (owner/admin can reset); personal member profiles follow later with the connection ownership rules | Slice 6a | Privacy inside a company; confused-deputy risk |

## Technical questions to verify during implementation (not product decisions)

- Confirm Pipedream's MCP server behaviour behind a reverse proxy
  (Streamable HTTP session handling, SSE) during slice 6; if it proves
  fragile, the fallback is a broker that speaks MCP itself and calls
  Pipedream's Connect tool APIs directly.
- Confirm Pipedream webhook signature details for Connect events during
  slice 5; the handler treats webhooks as sync hints regardless.
- Confirm Fly Machines API behaviour for idempotent create by name, volume
  attach on create, stop/start latency with the Jarvis image, snapshot
  retention settings, and private address format during slice 4.
- Confirm which provider administration APIs allow programmatic creation of
  per-tenant scopes and keys (Anthropic Console workspaces and keys; OpenAI
  projects and project keys) and what cost-report granularity they expose
  for reconciliation, during slice 4a. Where an API is missing, the
  operator-assisted path creates the key by hand and pastes it once into an
  operator form that stores it encrypted.
- Confirm, for the chosen browser provider, during slice 6a: persistent
  context API, whether connect URLs can be session-scoped (lets ATLAS avoid
  proxying CDP), interactive live view embedding rules (iframe allowed?),
  downloads API, and whether Playwright MCP's current release accepts
  custom headers for `--cdp-endpoint` (otherwise a short-lived capability
  token in the URL, rotated with the runtime token).
- Confirm exact usage field names for streaming responses per provider
  (Anthropic `message_start`/`message_delta`, OpenAI final usage chunk with
  `stream_options`) against current API docs during slice 4a.
- Confirm Jarvis admin API payloads for creating an `mcp-http` connector with
  a credential during slice 6 (the survey found the routes; the exact body
  shape must be read from `src/admin/service.ts`).
