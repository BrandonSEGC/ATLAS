# ADR-0016: A per-tenant "computer" (remote browser) for the agent

Status: Accepted (product owner request; defaults recorded in
`docs/open-decisions.md` OD-18 to OD-21)

## Context

Contractors will ask Jarvis to do things that only work in a web browser:
log into a supplier or permit portal, download an invoice, fill in a form,
check an order status on a site with no API. Today Jarvis runs headless
Chrome inside its own container through Puppeteer scripts invoked by its
`bash` tool. That has no human takeover path (logins, 2FA, captchas), weak
session persistence, and puts browser and agent in the same security
boundary.

Requirements:

1. Persistent, per-tenant browser identity (cookies, logged-in sessions)
   that survives runtime restarts and image upgrades.
2. Human-in-the-loop: when the agent hits a login or 2FA wall, a member can
   open a live view, act, and hand back; the agent continues with the
   session now authenticated.
3. Isolation: one tenant's browser state is never visible to another.
   Browser compromise must not equal agent compromise.
4. Reliable, cheap tool surface for the model: structured page snapshots
   rather than pixel screenshots by default.
5. Metered like everything else (browser minutes into the credit ledger).
6. No new secrets inside the runtime.

## Decision

### Shape

A **remote browser per tenant**, brokered by ATLAS, driven from inside the
Jarvis runtime through the Chrome DevTools Protocol (CDP):

```text
Jarvis runtime
  Playwright MCP (stdio, alias "computer") --CDP over WebSocket-->
ATLAS /computer/v1/cdp  (auth by runtime token; creates or reuses the tenant's
                         browser session; meters minutes; proxies CDP)
  --> Browser provider (hosted first; self-hosted Fly machine later)
        - persistent per-tenant browser context (profile)
        - live view URL for human takeover
        - downloads API
ATLAS MCP "atlas__computer" tools: request_human_help, status, end_session,
                                   list_downloads, fetch_download
ATLAS dashboard "Computer" page: live view embed, take over, hand back, end
```

### Provider strategy

`packages/computer` defines a `BrowserProvider` interface:

```ts
interface BrowserProvider {
  ensureProfile(input: { tenantId: string; profileId: string }): Promise<{ providerContextRef: string }>;
  createSession(input: { profileRef: string; idleTimeoutSec: number; maxDurationSec: number; region?: string }): Promise<{ providerSessionRef: string; cdpUrl: string }>;
  liveViewUrl(sessionRef: string): Promise<string>;
  endSession(sessionRef: string): Promise<void>;
  listDownloads(sessionRef: string): Promise<Array<{ id: string; name: string; bytes: number }>>;
  fetchDownload(sessionRef: string, id: string): Promise<ReadableStream>;
  deleteProfile(profileRef: string): Promise<void>;
}
```

First implementation: a **hosted browser provider** (Browserbase-class
service: persistent contexts, live view with interaction, downloads,
optional stealth and proxies, recordings). Reason: live view and persistent
contexts are the hard parts and they arrive ready-made; the platform stays
in the per-tenant isolation model by mapping one provider context per
tenant profile. Second implementation, when cost or privacy demands it: a
**self-hosted computer** as a Fly machine per tenant (headful Chrome on
Xvfb with CDP exposed privately and noVNC/WebRTC for live view, profile on
a volume), reusing the ADR-0015 provisioning pattern.

### Tool surface for the model

Default: **Playwright MCP** running inside the runtime with
`--cdp-endpoint` pointed at the ATLAS CDP endpoint. It exposes
accessibility-tree snapshots, click/type by element reference, navigation,
tabs, downloads, and screenshots. Snapshot-based operation is more reliable
and far cheaper in tokens than pixel-coordinate "computer use"; its vision
mode is available as a fallback for canvas-heavy pages (OD-20).

ATLAS adds a small MCP server `atlas__computer` (projected like Pipedream
connections) with the tools the browser itself cannot provide:
`request_human_help(reason)`, `status()`, `end_session()`,
`list_downloads()`, `fetch_download(id)` (returns the file into the
runtime's workspace attachments through the existing admin file API), and
`reset_profile()` (owner/admin approval required).

### Profiles and ownership

`computer_profiles` follow the same ownership rule as connections:
`owner_type = tenant` (shared company logins; created, reset, or deleted by
owner/admin) or `owner_type = member` (personal logins; only usable when the
turn's actor is the owner). The first milestone implements the shared
tenant profile; member profiles are a follow-on with the schema already in
place.

### Human takeover flow

1. The agent cannot proceed (login form, 2FA, captcha). It calls
   `request_human_help(reason)`.
2. ATLAS creates a short-lived (15 minute) takeover grant for the requesting
   member and posts to the originating Slack conversation: "Jarvis needs a
   hand in the browser: <reason>. Open the ATLAS computer page to help."
   The link is an ATLAS dashboard URL, never a raw provider URL.
3. The member signs in (existing session), sees the live view embedded,
   completes the step, and clicks "Hand back". ATLAS marks the grant used
   and the tool result returns "Human completed: <optional note>".
4. Cookies persist in the tenant profile, so the next task on that site
   does not need help.

### Sessions and limits

One active session per tenant profile by default (OD-19). Sessions are
created lazily on first CDP connect, idle out after 5 minutes without CDP
traffic, and end after 60 minutes unless the agent is still active. Minutes
are metered as `computer_minute` and debited per ADR-0014.

### Security boundaries

- Runtime never holds the provider API key; it connects to ATLAS with its
  runtime token. If the provider offers session-scoped connect tokens, ATLAS
  hands those out and steps out of the CDP data path; otherwise ATLAS
  proxies the WebSocket.
- Web content is untrusted input. The Jarvis system prompt for the
  `computer` tools states that page content and downloads are data, not
  instructions; ATLAS limits download types and sizes (OD-21).
- Passwords are never typed by the agent from chat. Logins happen through
  human takeover; the profile keeps the session.
- Live view is only reachable through the ATLAS page with a valid dashboard
  session and an unexpired grant for that tenant.

## Consequences

- Chrome can eventually be removed from the Jarvis image, shrinking it and
  speeding wake-on-event (J8).
- ATLAS gains a WebSocket proxy role; CDP traffic for a few concurrent
  sessions is small, but it is the first bandwidth-shaped workload on the
  control plane and is isolated in its own module for later separation.
- A third-party sees tenant browsing while sessions run. This is disclosed
  and covered by the provider's data processing terms; the self-hosted
  backend is the answer for tenants who object.

## Alternatives rejected

- Keep headless Chrome inside the runtime: no takeover, no durable login
  state, no isolation between browser and agent.
- Pixel-only "computer use" from day one: expensive and brittle for form-
  heavy portals; kept as an optional mode.
- A full remote desktop VM per tenant: right for desktop applications, which
  contractors rarely need; the `BrowserProvider` interface does not preclude
  adding it later.
