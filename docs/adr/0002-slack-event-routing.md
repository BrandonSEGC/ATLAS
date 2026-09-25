# ADR-0002: Central Slack Events API gateway with durable dedup and re-signed forwarding

Status: Accepted

## Context

One distributed Slack app serves every tenant, so Slack delivers all events to
one request URL. Jarvis's existing webhook adapter (`src/adapters/slack-webhook.ts`)
verifies Slack's `v0` HMAC scheme with a configurable signing secret and
handles `url_verification`; its Socket Mode adapter cannot be used because
many runtimes sharing one app-level token would compete for deliveries.

Slack requires an acknowledgement within 3 seconds and retries on failure with
the same `event_id`. Forwarded messages must arrive complete; Jarvis's
collaboration contract forbids silent truncation and forbids dropping messages
because the sender is a bot.

## Decision

1. ATLAS hosts `POST /webhooks/slack/events`. It keeps the raw body bytes,
   verifies the platform signing secret with constant-time comparison and a
   300 second skew window, and answers `url_verification` itself.
2. Deduplication is durable: `INSERT INTO slack_event_receipts (event_id) ...
   ON CONFLICT DO NOTHING` decides whether an event is new. There is no
   in-memory dedup used for correctness.
3. Routing key is `(team_id, enterprise_id)` from the verified body ->
   `slack_installations` -> tenant -> runtime. Events for unknown, revoked,
   suspended, or not-ready targets are acknowledged with 200, recorded with a
   drop reason, and not forwarded.
4. Loop control: the gateway rejects only the exact authenticated self-echo
   (`event.user == installation.bot_user_id`). Messages authored by other bots
   or agents are forwarded. Deciding whether to respond belongs to Jarvis
   (`yield_no_action`).
5. Delivery is asynchronous through a `runtime.deliver_slack_event` job with
   singleton key `event:<event_id>`. The raw body is stored in
   `slack_event_deliveries` only until delivery succeeds (then cleared) and
   at most 24 hours on failure; job payloads carry IDs only.
6. The forwarded request is the byte-exact Slack body, re-signed with a
   per-runtime forwarding secret using Slack's own `v0=<hmac>` scheme and a
   fresh timestamp, sent to the runtime's `/slack/events` over the private
   network. Jarvis therefore authenticates forwarded events with code it
   already has; `MOM_SLACK_SIGNING_SECRET` in each runtime is that runtime's
   forwarding secret, not the platform secret. Informational headers
   `X-Atlas-Event-Id`, `X-Atlas-Tenant-Id`, `X-Atlas-Runtime-Id`,
   `X-Atlas-Attempt` accompany the request.
7. Outbound Slack traffic is sent by the runtime with the tenant's bot token;
   ATLAS does not proxy replies in this milestone.

## Consequences

- Message content transits and briefly rests in the control plane database.
  Retention is minimized and documented (threat model T6, T13). A future
  option is forwarding synchronously with a short timeout and falling back to
  the queue only on failure.
- Jarvis's in-memory dedup remains as a second layer but is not relied upon.
- Because the forwarded body is unchanged, ATLAS cannot add fields inside the
  Slack payload; tenant and runtime identity travel in headers and in the
  per-runtime secret itself. A runtime holds exactly one tenant's secret, so
  a valid signature is proof of tenant.
- Socket Mode stays available in Jarvis for local development only.

## Alternatives rejected

- Socket Mode per runtime with one shared app token: competing consumers.
- Separate Slack app per tenant: not distributable, per-tenant Slack setup.
- Custom ATLAS envelope with a new Jarvis endpoint: requires a Jarvis change
  before any event can flow; kept as a v2 option if headers prove
  insufficient.
- Synchronous forwarding inside the 3 second window: risks Slack retries and
  duplicate agent turns when a runtime is slow.
