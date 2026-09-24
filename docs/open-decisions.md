# Open product decisions

Each item names the default that implementation will follow if no answer
arrives, so work is never blocked. Items marked "blocks" change tenant
ownership, security, deletion, or pricing and should be confirmed before the
slice that depends on them.

| ID | Decision | Default adopted | Blocks | Why it matters |
| --- | --- | --- | --- | --- |
| OD-1 | Deletion grace window and backup retention (ADR-0013) | 30 day grace; database PITR 7 days plus daily snapshots 30 days; volume backups capped at 30 days | Slice 7 | Customer terms, storage cost, compliance answers |
| OD-2 | Who may request tenant deletion | Any `owner` with typed confirmation; operators; automatic after 14 days uninstalled | Slice 7 | Accidental deletion risk versus self-service |
| OD-3 | Model provider key ownership per tenant | ATLAS issues one provider key per tenant from the platform's provider account (via the provider's admin/key API where available) so spend is attributable and revocable per tenant; pilot fallback is one platform key per environment with per-tenant attribution from runtime usage reports | Slice 4 | Cost attribution, blast radius of a compromised runtime, pricing |
| OD-4 | Should the installing user be the only automatic `owner`? | Yes; every other user who signs in is `member` until promoted. Slack workspace admin status is ignored | Slice 2 | Tenant ownership |
| OD-5 | Can a `member` see that other members have personal connections? | Only `owner`/`admin` see `{ appSlug, ownerDisplayName, status }`; members see only their own | Slice 5 | Privacy expectations inside a company |
| OD-6 | App ownership policy list (which apps are personal-only, shared-only, or both) | Initial list in `packages/connections/app-policy.ts`: mail, calendar, personal drive apps are member-only; CRM, accounting, project management are tenant-only; unknown apps allow both | Slice 5 | Confused-deputy risk and support burden |
| OD-7 | Which channel messages reach Jarvis | Everything the installation is authorized for (DMs, group DMs, channels the bot was invited to); Jarvis decides whether to respond | Slice 3 | Noise, cost, and privacy inside customer workspaces |
| OD-8 | Runtime resource class and region for the pilot | One `standard` class, one region, configurable per environment | Slice 4 | Cost and latency |
| OD-9 | Pilot entitlements | `maxMembers: 10`, `maxConnections: 10`, enforced with plain messages; no billing | Slice 5 | Sets the shape of future plans |
| OD-10 | Automatic suspension triggers | Manual only in the pilot (operator action); no automatic suspension for usage | Slice 4 | Pricing and abuse handling |
| OD-11 | Customer-visible name for the runtime and the bot | "Jarvis" as the bot name in Slack, "your private Jarvis runtime" in the dashboard | Slice 1 | Branding of the distributed Slack app cannot change without reinstalls |
| OD-12 | Should ATLAS run in the same Railway project as tenant runtimes (ADR-0009)? | Yes for v1, one project per ATLAS environment, to use the private network | Slice 4 | Topology, provider limits, and blast radius |
| OD-13 | Reinstall semantics when a different user reinstalls the app | The new installer is added as `owner` alongside existing owners; nobody is demoted automatically | Slice 1 | Tenant ownership |

## Technical questions to verify during implementation (not product decisions)

- Confirm Pipedream's MCP server behaviour behind a reverse proxy
  (Streamable HTTP session handling, SSE) during slice 6; if it proves
  fragile, the fallback is a broker that speaks MCP itself and calls
  Pipedream's Connect tool APIs directly.
- Confirm Pipedream webhook signature details for Connect events during
  slice 5; the handler treats webhooks as sync hints regardless.
- Confirm Railway private networking behaviour across services created via
  the API and the exact internal hostname format during slice 4.
- Confirm Jarvis admin API payloads for creating an `mcp-http` connector with
  a credential during slice 6 (the survey found the routes; the exact body
  shape must be read from `src/admin/service.ts`).
