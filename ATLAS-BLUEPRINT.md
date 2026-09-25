# ATLAS

Automated Task, Logistics & Activity System

## Agent handoff

This document is the starting blueprint for a new SaaS product built around the
existing Jarvis AI-agent runtime.

ATLAS is the commercial SaaS platform. Jarvis remains the isolated AI-agent
runtime that performs work for one contractor company.

The initial customer experience should be:

1. A contractor creates an ATLAS account.
2. They click Add to Slack and install one publicly distributed Slack app.
3. ATLAS creates their company tenant.
4. ATLAS provisions an isolated Jarvis runtime and persistent workspace.
5. The contractor signs into the ATLAS dashboard with Slack.
6. They connect company or personal applications through Pipedream Connect.
7. Jarvis receives the corresponding tools and can use them from Slack.

The first sellable milestone is complete when this flow works without manually
creating customer infrastructure, editing environment variables, or asking the
customer to create a Pipedream developer account.

## Product boundary

### ATLAS control plane

ATLAS owns:

- Customer signup and onboarding
- Tenant and member identity
- Slack app installation OAuth
- Sign in with Slack
- Role-based dashboard access
- Pipedream Connect integration
- Runtime provisioning and lifecycle
- Central Slack event ingestion and tenant routing
- Subscription and entitlement state
- Usage metering and audit records
- Connection authorization state
- Future native MCP authorization

### Jarvis data plane

Each contractor receives an isolated Jarvis runtime that owns:

- Its agent process
- Customer workspace and memory
- Customer configuration
- Customer connector projection
- Skills and agent behavior
- Tool execution
- Customer-specific persistent files

The existing Jarvis repository should remain deployable independently. ATLAS
must provision a versioned Jarvis image or release rather than copy Jarvis
source code into the ATLAS repository.

## Isolation decision

Do not convert the current Jarvis process into one shared multi-tenant agent.

Provision one isolated runtime and persistent volume per contractor company.
The current runtime assumes one trusted workspace, one memory store, one
connector set, and one security boundary. Deployment-level tenant isolation is
safer and less invasive than attempting to retrofit tenant checks throughout
the agent.

## Proposed system architecture

```text
Customer browser
      |
      v
ATLAS web application / API
  |-- Tenant and member management
  |-- Slack OAuth and dashboard login
  |-- Connection management
  |-- Pipedream Connect links
  |-- Runtime provisioning
  |-- Subscription and usage state
  |
  +--> PostgreSQL
  +--> Encrypted secret storage
  +--> Provisioning provider (Railway first)
  |
  +--> Slack Events Gateway
           |
           | verify, deduplicate, route by team_id
           v
      Dedicated Jarvis runtime
        |-- private authenticated ingress
        |-- dedicated persistent volume
        |-- tenant workspace and memory
        +-- tenant-scoped tool connections
```

## Repository strategy

Create a new private repository for ATLAS. Keep Jarvis in its existing
repository.

Recommended initial monorepo layout:

```text
apps/
  web/                  # Customer dashboard and onboarding
  api/                  # Control-plane HTTP API
  slack-gateway/        # Slack Events API verification and tenant routing
packages/
  database/             # Schema, migrations, and repositories
  auth/                 # Slack installation and login flows
  provisioning/         # Provider-neutral runtime lifecycle interface
  connections/          # Pipedream now, native MCP later
  contracts/            # Shared API/event types
  config/               # Environment validation and shared configuration
docs/
  architecture.md
  threat-model.md
  operations.md
```

It is acceptable to begin with one deployable TypeScript service if that keeps
the first milestone simple. Keep module boundaries explicit so the Slack
gateway and workers can be separated later.

## Core tenancy model

Every control-plane record must carry an explicit tenant identity. Never infer
tenant ownership from a connection name, hostname, or current session.

Initial entities:

### tenants

- id
- slug
- display_name
- status: provisioning | active | suspended | deleting | deleted
- slack_team_id
- slack_enterprise_id, nullable
- created_at
- updated_at

### members

- id
- tenant_id
- slack_user_id
- display_name
- email, nullable
- role: owner | admin | member
- status
- created_at
- updated_at

Unique identity should include tenant/workspace scope. A Slack user ID alone is
not a global identity across every workspace.

### slack_installations

- id
- tenant_id
- team_id
- enterprise_id, nullable
- bot_user_id
- Encrypted bot access token reference
- Installer Slack user ID
- Granted scopes
- Installation metadata
- installed_at
- revoked_at, nullable

Persist the complete installation response needed to operate the distributed
Slack app. Never store Slack tokens in browser storage or return them to the
frontend.

### runtime_instances

- id
- tenant_id
- provider
- Provider resource reference
- Jarvis release/image version
- status: requested | provisioning | ready | degraded | suspended | failed
- Private ingress identity
- Volume/resource reference
- Last health-check state
- created_at
- updated_at

Do not store live provider credentials in this table.

### connections

- id
- tenant_id
- owner_type: tenant | member
- owner_member_id, nullable
- provider: initially pipedream; later mcp
- app_slug
- Human-readable alias
- status: pending | connected | unhealthy | revoked
- Provider-side non-secret account identifier
- Capability or policy metadata
- created_at
- updated_at

### subscriptions

- id
- tenant_id
- Billing provider customer/subscription references
- Plan
- Status
- Entitlements
- Current period

Billing is not required in the first implementation milestone, but tenant and
runtime lifecycle APIs should already support suspension.

### audit_events

- id
- tenant_id
- Actor type and actor ID
- Action
- Resource type and resource ID
- Safe structured metadata
- Timestamp

Never put access tokens, provider payloads, message bodies, or model reasoning
into audit metadata.

### usage_events

Track at least:

- Model/provider usage
- Pipedream external users
- Connected accounts
- Runtime uptime/resource class
- Storage
- Voice or messaging usage when added

Capture usage before implementing billing so pricing can be based on actual
cost.

## Slack design

ATLAS uses one Slack app distributed to many workspaces.

### Installation OAuth

Add to Slack is an organization installation flow. It must:

- Generate and persist a short-lived, one-time OAuth state.
- Redirect to Slack's OAuth v2 authorization endpoint with the minimum required
  bot scopes.
- Validate state on callback.
- Exchange the authorization code server-side.
- Validate the returned workspace identity.
- Create or update the tenant and installation atomically.
- Encrypt installation credentials.
- Queue idempotent runtime provisioning.
- Redirect to onboarding without exposing tokens.

Installation and runtime provisioning must be retryable. Repeating a callback
or provisioning job must not create duplicate tenants, services, or volumes.

### Sign in with Slack

Sign in with Slack identifies a person using the dashboard. It is separate from
installing the bot.

The login flow must:

- Use Slack OpenID Connect
- Validate state, nonce, issuer, audience, signature, and expiry
- Require the signed-in user to belong to a Slack workspace with an active
  ATLAS installation
- Resolve the member by (team_id, slack_user_id)
- Issue an HttpOnly, Secure, SameSite session cookie
- Never treat possession of a Slack display name or email as identity

The installing user becomes the initial tenant owner. Additional role
assignment should be explicit. Do not automatically make every Slack workspace
administrator an ATLAS owner without a deliberate product decision.

### Slack event gateway

Use Slack's HTTPS Events API for SaaS routing.

The central gateway must:

- Retain the exact raw request body for signature verification.
- Verify Slack's signing secret and reject stale timestamps.
- Handle Slack URL verification.
- Persist provider event IDs for durable deduplication.
- Resolve the installation by team_id and enterprise_id.
- Reject revoked, suspended, or unknown tenants.
- Reject authenticated self-echoes.
- Return Slack's acknowledgement quickly.
- Deliver the event asynchronously to the correct Jarvis runtime.

Forward events through a private authenticated runtime endpoint. The forwarded
envelope should include a stable event ID, tenant ID, Slack conversation
identity, sender identity, and the complete provider-sized message. Do not
silently truncate messages.

Outbound messages must use the tenant's Slack installation token and preserve
conversation/thread routing.

Do not run every tenant runtime against the same Socket Mode app token.
Multiple runtimes would compete for or duplicate deliveries. Socket Mode can
remain a local-development option.

## Pipedream Connect design

Pipedream is the primary managed connection layer for common applications in
the first commercial version.

### Platform ownership

ATLAS should own one production Pipedream Connect project and its developer
credentials.

Customers must not be asked to:

- Create a Pipedream project
- Create an OAuth client
- Paste a Pipedream client secret
- Know an app slug
- Configure MCP headers

ATLAS stores its own Pipedream developer credentials centrally and creates
short-lived Connect Links for authenticated users.

### External user identity

Use stable opaque ATLAS IDs rather than mutable names or email addresses:

```text
Shared company connection:
tenant:<tenant-id>

Personal connection:
tenant:<tenant-id>:member:<member-id>
```

Use company ownership for shared systems such as a CRM, project-management
platform, or shared accounting integration. Use member ownership for personal
email, calendars, and storage.

Do not create a Pipedream external user for every Slack member automatically.
Create one only when that member connects a personal account. External-user
count affects Pipedream cost.

### Connection flow

1. Authenticated member chooses an app in ATLAS.
2. ATLAS checks whether the requested connection may be personal or shared.
3. Shared connections require owner or admin.
4. ATLAS creates a short-lived Connect Link scoped to the correct external user
   ID and the production environment.
5. Browser opens Pipedream's authorization flow.
6. ATLAS receives a webhook or polls provider state after redirect.
7. ATLAS records only provider account metadata and health state.
8. The tenant runtime receives a connection-state update.
9. Tools become available without rebuilding or restarting the runtime.

Pipedream remains responsible for third-party credentials and token refresh.
ATLAS must never request or expose those third-party tokens.

### MCP runtime integration

Pipedream's current MCP interface expects one app slug per MCP connection.
Create one logical MCP connection per enabled app rather than placing a
comma-separated list of app slugs in one header.

Runtime aliases should be deterministic and collision-safe, for example:

```text
pipedream__gmail
pipedream__google_calendar
pipedream__quickbooks
```

Tool execution must carry:

- ATLAS/Pipedream external user ID
- Pipedream project and environment
- App slug
- Optional explicit account ID when multiple accounts are connected

An authorization-required tool result should become a safe connection prompt
for the requesting member, not a raw provider error.

### Diagnostics

Build a staged diagnostic endpoint:

1. Developer token exchange
2. External-user/account lookup
3. MCP initialize
4. Tool listing for the selected app
5. Optional read-only health tool

Return a sanitized stage-specific error. Never return OAuth tokens, Connect
tokens, full upstream payloads, or provider credentials.

## Runtime provisioning

Define a provider-neutral interface before implementing Railway:

```ts
interface RuntimeProvisioner {
  provision(input: ProvisionRuntimeInput): Promise<RuntimeRecord>;
  status(runtimeId: string): Promise<RuntimeStatus>;
  updateRelease(runtimeId: string, release: string): Promise<void>;
  suspend(runtimeId: string): Promise<void>;
  resume(runtimeId: string): Promise<void>;
  destroy(runtimeId: string): Promise<void>;
}
```

### Provisioning properties

- Idempotency key based on tenant and operation
- Dedicated persistent volume
- Versioned Jarvis release/image
- Per-tenant encryption material
- Private control-plane-to-runtime authentication
- Health and readiness checks
- Resource limits
- Explicit status transitions
- Retry-safe provider operations
- No customer credentials in build arguments or source control

The first provider may be Railway, but application logic must not depend on
Railway-specific identifiers.

## Runtime contract

ATLAS and Jarvis need a versioned private API for:

- Health and readiness
- Installation/config projection
- Connection-state changes
- Authenticated inbound Slack events
- Runtime version
- Usage reports
- Graceful suspend/resume

ATLAS remains the source of truth for tenant, identity, installation,
subscription, and connection ownership. Jarvis remains the source of truth for
its workspace and agent memory.

Avoid giving the control plane arbitrary access to tenant workspace contents.

## Native MCP roadmap

Native OAuth-enabled MCP is important, but it follows the first Slack and
Pipedream milestone.

Future capabilities:

- Add a remote HTTP MCP server from the dashboard
- Support unauthenticated, bearer-token, and authorization-code OAuth servers
- Discover protected-resource and authorization-server metadata
- Use PKCE and dynamic client registration where supported
- Support pre-registered clients where required
- Store authorization per tenant or per member
- Encrypt refresh/access tokens
- Return authorization links to the correct requesting member
- Reconnect and refresh tools without a runtime restart
- Show tool count, connection health, and safe errors
- Approval-gate connection changes made from agent conversation

Do not share one member's MCP authorization with another member unless it is
explicitly a company-owned connection.

## Security requirements

### Required from the first commit

- Validate all environment configuration at startup.
- Encrypt provider and Slack credentials at rest.
- Keep secrets out of logs, errors, URLs, analytics, browser storage, and
  database query output.
- Use HTTPS-only callbacks in deployed environments.
- Use one-time, expiring OAuth state and PKCE where supported.
- Bind login callbacks to a browser cookie and expected tenant context.
- Verify Slack request signatures before parsing trusted identity fields.
- Use durable event deduplication.
- Authorize every database query and mutation by tenant.
- Separate admin, company connection, and personal connection permissions.
- Rate-limit login, OAuth-start, callback, and connection-link endpoints.
- Maintain audit events for security-sensitive mutations.
- Use synthetic fixtures in tests.
- Support credential rotation and installation revocation.
- Fail closed when tenant, installation, or connection ownership is ambiguous.

### Avoid

- One global customer workspace
- One global mutable currentTenant
- Tenant identity supplied only by browser request bodies
- Shared connector tokens in runtime environment variables
- Long-lived authorization links
- Raw provider errors shown to end users
- Logging Slack event bodies by default
- Using email as the sole stable identity
- Directly exposing Railway or Pipedream developer credentials to customers
- Persisting model reasoning or private customer conversations in the control
  plane

Create `docs/threat-model.md` before the private pilot. It should cover tenant
isolation, Slack spoofing/replay, OAuth state theft, SSRF through connector
URLs, secret leakage, confused-deputy tool execution, cross-user personal
connections, compromised runtimes, deletion, and backups.

## API outline

Exact paths may change, but keep the responsibilities distinct.

### Public/authentication

```text
GET  /auth/slack/install
GET  /auth/slack/install/callback
GET  /auth/slack/login
GET  /auth/slack/login/callback
POST /auth/logout
POST /webhooks/slack/events
POST /webhooks/pipedream
```

Provider webhooks are public only at the network layer. They must
authenticate using provider signatures or a scoped secret.

### Authenticated dashboard

```text
GET    /api/me
GET    /api/tenant
GET    /api/members
PATCH  /api/members/:id/role

GET    /api/connections
POST   /api/connections/pipedream/link
GET    /api/connections/:id/status
DELETE /api/connections/:id

GET    /api/runtime
POST   /api/runtime/retry
```

### Private runtime

```text
POST /internal/runtimes/:id/events/slack
PUT  /internal/runtimes/:id/config-projection
POST /internal/runtimes/:id/connections/changed
POST /internal/runtimes/:id/usage
GET  /internal/runtimes/:id/health
```

Use service authentication and short timeouts. Private endpoints must not rely
on dashboard cookies.

## First implementation milestone

### In scope

- TypeScript project and strict configuration
- PostgreSQL schema and migrations
- Tenant-aware repository layer
- Distributed Slack installation OAuth
- Sign in with Slack
- Owner/admin/member sessions and route authorization
- Slack Events API signature verification, deduplication, and routing
- Provider-neutral runtime provisioning interface
- Railway provisioning adapter
- Runtime registration and health state
- Basic onboarding dashboard
- Platform-owned Pipedream Connect Link generation
- Company and personal connection records
- Staged Pipedream diagnostics
- Structured audit events
- Automated tests for tenant isolation and OAuth security

### Explicitly out of scope

- Billing collection
- Slack Marketplace listing
- Native MCP OAuth
- Automatic upgrades across every runtime
- Full support dashboard
- Enterprise Grid organization-wide installation
- Complex usage-based pricing
- Mobile application
- Migrating existing Jarvis deployments

Design extension points for these items, but do not delay the first working
onboarding flow to implement them.

## Required automated tests

At minimum:

### Tenant isolation

- Tenant A cannot read or mutate Tenant B members, runtime, connections, or
  installation.
- A personal connection can only be used by its owner.
- A company connection is visible only to members of its tenant.
- Runtime event routing rejects mismatched tenant/runtime identities.

### Slack OAuth and login

- Valid installation creates exactly one installation and provisioning job.
- Replayed callback does not duplicate resources.
- Missing, expired, or mismatched state is rejected.
- Login rejects a user from a workspace without an active installation.
- Login resolves membership using workspace and Slack user ID.
- Revoked installation cannot deliver events.

### Slack events

- Valid signature accepted.
- Invalid signature and stale timestamp rejected.
- Duplicate event acknowledged but delivered once.
- Exact authenticated self-echo rejected.
- Unestablished conversation handled according to policy.
- Complete provider-sized messages are preserved.

### Pipedream

- Connect Link is scoped to the authenticated tenant/member.
- Member cannot create a shared connection without permission.
- Production environment is used outside local/test environments.
- Provider credentials and tokens never appear in API responses.
- One MCP connection is generated per app slug.
- Diagnostic failures identify the stage without leaking secrets.

### Provisioning

- Repeated provisioning requests are idempotent.
- Partial provider failure can be retried.
- Tenant gets a dedicated volume and runtime identity.
- Suspension blocks event delivery and connection mutations.
- Destroy requires explicit lifecycle state and is retry-safe.

## Acceptance criteria for the first sellable pilot

A new contractor who has never used ATLAS can:

1. Open the ATLAS onboarding page.
2. Install Jarvis into their Slack workspace.
3. Receive confirmation that their private Jarvis runtime is being prepared.
4. Sign into the dashboard with the same Slack workspace.
5. See runtime readiness without contacting an operator.
6. Connect one personal application and one company application through
   Pipedream without seeing Pipedream developer credentials.
7. Message Jarvis in Slack.
8. Have Jarvis use a connected read-only tool and respond in the correct
   conversation.
9. Add another member who can connect a personal account but cannot modify
   company connections.
10. Remain isolated from every other pilot customer's data, tools, events, and
    runtime.

An operator can:

- See tenant and runtime health
- Retry failed provisioning
- Suspend a tenant
- Rotate platform credentials
- Inspect safe audit events
- Determine whether a Pipedream failure occurred during token exchange,
  account lookup, MCP initialization, or tool listing

## Engineering principles

- Prefer boring, explicit infrastructure over premature orchestration.
- Start on Railway; keep provisioning provider-neutral.
- Keep ATLAS and Jarvis release lifecycles separate.
- Make all external operations idempotent.
- Treat tenant boundaries as security boundaries.
- Keep user-facing errors plain and actionable.
- Preserve provider-sized messages without silent clipping.
- Do not implement loop prevention by dropping bot-authored messages; use
  authenticated self-echo rejection and durable deduplication.
- Keep sensitive operational details and customer data out of the repository.
- Use synthetic example.com, RFC 5737, +1 555, and obviously fake IDs in tests
  and documentation.
- Write architecture decisions down as ADRs when they affect tenancy,
  identity, credential ownership, or data deletion.

## Decisions to record before implementation

Use reasonable defaults for scaffolding, but document these as architecture
decisions:

- Web/API framework
- Database and migration tool
- Session storage and expiration
- Secret-encryption and key-rotation strategy
- Job queue for provisioning and event delivery
- Railway API/IaC integration method
- Private runtime authentication method
- Jarvis release/image registry
- Initial Slack bot scopes
- Tenant deletion and backup-retention policy

Recommended defaults:

- TypeScript on Node.js
- PostgreSQL
- Server-side sessions persisted in PostgreSQL or Redis
- Background jobs with durable retries
- A shared versioned contracts package
- Schema validation at every HTTP and queue boundary
- One deployment environment each for local, staging, and production

## Instructions for the implementing agent

1. Treat this document as product and architecture context, not as permission
   to invent production credentials or infrastructure identifiers.
2. Begin by inspecting the empty/new repository and choosing the smallest
   coherent TypeScript structure for the first milestone.
3. Create `docs/architecture.md`, `docs/threat-model.md`, and short ADRs for
   tenant isolation, Slack event routing, and Pipedream identity.
4. Form a concrete test plan before writing implementation code.
5. Implement vertical slices rather than disconnected abstractions:
   - Tenant + Slack install
   - Slack login + dashboard session
   - Event verify + tenant route
   - Provisioning job + runtime status
   - Pipedream link + connection status
6. Use mock Slack, Pipedream, and provisioning providers in automated tests.
7. Never require real customer or production data in tests.
8. Keep billing and native MCP out of the first milestone unless the product
   owner explicitly expands scope.
9. Stop for a product decision only when it materially changes tenant
   ownership, security, deletion, or pricing. Use safe defaults for ordinary
   implementation details.
10. End the first planning pass with:
    - Proposed repository structure
    - Data model
    - API and event contracts
    - Threat model
    - Ordered implementation plan
    - Open product decisions
