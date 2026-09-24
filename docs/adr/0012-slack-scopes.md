# ADR-0012: Initial Slack bot and sign-in scopes

Status: Accepted

## Context

The distributed app needs the minimum bot scopes for Jarvis's Slack adapter
(`slack-app-manifest.yaml` in the Jarvis repository lists the scopes it uses
today) and Sign in with Slack needs OpenID Connect scopes. Fewer scopes ease
installation review and Marketplace listing later.

## Decision

Bot token scopes requested on install:

| Scope | Why |
| --- | --- |
| `app_mentions:read` | respond to mentions in channels |
| `chat:write` | send replies |
| `im:history`, `im:read`, `im:write` | direct messages with Jarvis |
| `mpim:history` | group DMs |
| `channels:history`, `groups:history` | messages in channels the bot is invited to (public and private) |
| `users:read` | resolve display names for attribution |
| `reactions:read`, `reactions:write` | reaction-based acknowledgements used by the adapter |
| `files:read`, `files:write` | attachments in conversations |

Dropped relative to the current Jarvis manifest: `channels:read`,
`groups:read` (not needed for event-driven operation; the adapter's channel
download command is not used in ATLAS). Re-add with a new ADR if a feature
needs channel listing.

Event subscriptions: `app_mention`, `message.channels`, `message.groups`,
`message.im`, `message.mpim`, `reaction_added`, `reaction_removed`,
`app_uninstalled`, `tokens_revoked`, `member_joined_channel`.

Sign in with Slack (user, OpenID Connect): `openid`, `profile`, `email`.
No user token scopes beyond OIDC are requested; ATLAS never acts as a user.

Interactivity: enabled with the request URL pointing at ATLAS
(`/webhooks/slack/interactions`) but not consumed in the first milestone.

Socket Mode: disabled on the distributed app.

## Consequences

- Jarvis's Socket Mode local setup keeps its own dev app manifest; the
  distributed app manifest lives in this repository under `docs/` once the
  app is created (without IDs or secrets).

## Alternatives rejected

- Requesting `channels:join` or `chat:write.public`: not required for the
  pilot; users invite the bot explicitly.
