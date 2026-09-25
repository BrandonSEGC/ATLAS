# ADR-0003: Platform-owned Pipedream Connect, opaque external user IDs, ATLAS MCP broker

Status: Accepted

## Context

Pipedream Connect identifies end users by an `external_user_id` chosen by the
developer, scopes connected accounts to `(project, environment,
external_user_id)`, and exposes tools through a remote MCP server that expects
`x-pd-project-id`, `x-pd-environment`, `x-pd-external-user-id`, and a single
`x-pd-app-slug` per connection, authenticated with a developer OAuth
client-credentials access token.

Jarvis today (`src/pipedream.ts`, `src/admin/managed-config.ts`) expects the
developer client ID and secret inside the runtime, and joins several app slugs
into one header under one alias `pipedream`. Both conflict with the blueprint:
customers must never handle Pipedream developer credentials, and a
client-credentials token is scoped to the whole Connect project, so any
runtime holding it could name any tenant's external user.

## Decision

### Identity

1. ATLAS owns exactly one Pipedream Connect project per ATLAS environment.
   Its OAuth client credentials are platform secrets held only by ATLAS.
2. External user IDs are opaque and stable:
   - shared (company) connection: `tenant:<tenant_id>`
   - personal connection: `tenant:<tenant_id>:member:<member_id>`
   Names, emails, and Slack IDs never appear in external user IDs.
3. External users are created lazily: the tenant external user on the first
   shared connection, a member external user on that member's first personal
   connection. `provider_external_users` records them for cost metering.
4. Ownership policy: shared connections require `owner` or `admin`; personal
   connections are created, listed in full, and used only by their owner.
   `packages/connections/app-policy.ts` restricts obviously personal apps
   (mail, calendar, personal storage) to `member` ownership and obviously
   shared systems (CRM, accounting, project management) to `tenant`.
5. Environment: `production` everywhere except `ATLAS_ENV = local | test`,
   enforced by `packages/config`.

### Runtime integration

6. One connection = one MCP server entry in the runtime projection with alias
   `pipedream__<app_slug>` (shared) or `pipedream__<app_slug>__m<first 8 hex
   of member_id>` (personal). Never a comma-joined slug list.
7. The runtime does not hold Pipedream credentials. Each projected MCP server
   URL points at the ATLAS MCP broker
   (`/mcp/v1/connections/:connectionId`), authenticated with the runtime's
   service token. The broker resolves the connection within the runtime's
   tenant, derives every `x-pd-*` header from the database row, injects the
   platform developer token, and proxies MCP Streamable HTTP to Pipedream.
8. For personal connections the broker requires
   `X-Atlas-Actor-Slack-User-Id` equal to the owning member's Slack user ID
   (Jarvis change J2 supplies it). Otherwise it returns a safe JSON-RPC error
   prompting the requester to connect their own account.
9. Authorization-required or unhealthy-account responses from Pipedream are
   converted into a safe prompt for the requesting member, the connection is
   marked `unhealthy`, and an audit event is written. Raw provider errors are
   never shown to end users.
10. Staged diagnostics run `developer_token -> account_lookup ->
    mcp_initialize -> tool_list -> health_tool` and report only the failing
    stage and a sanitized message.

## Consequences

- ATLAS is in the tool-call path (latency, availability). Mitigated by a
  stateless proxy, cached developer tokens, and per-runtime limits. Tool-call
  usage metering comes for free.
- Jarvis's existing `pipedream` connector type is not used by ATLAS; the
  generic `mcp-http` connector is. Jarvis change J1 (hot reload) removes the
  restart on projection changes; J2 enables personal connections.
- A member who leaves loses their personal connections' projection entries
  when disabled; the Pipedream account is deleted on revocation.

## Alternatives rejected

- Vending the developer access token to runtimes: project-wide scope enables
  cross-tenant confused-deputy attacks (threat model T7).
- One Pipedream project per tenant: requires per-tenant OAuth clients and
  Pipedream administration for every signup, contradicting the blueprint.
- Per-external-user Connect tokens on the MCP path: not confirmed to be
  accepted by Pipedream's MCP server for tool execution; revisit if it becomes
  available, since it would allow removing the broker from the hot path.
