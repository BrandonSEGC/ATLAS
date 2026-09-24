# Agent Guidelines for This Repository

ATLAS is the control plane for isolated Jarvis runtimes. Read
`ATLAS-BLUEPRINT.md` and `docs/architecture.md` before changing anything.
`docs/implementation-plan.md` defines the order of work; do not start a later
slice before the earlier slice's tests pass.

## Repository safety

Treat every tracked file, commit message, branch name, and review comment as
permanently public even though the repository is private.

Never commit:

- Customer names, Slack workspace or user IDs, emails, messages, or
  conversation content from real installations.
- Live deployment topology: hostnames, service or volume IDs, project IDs,
  ingress URLs, private network names, or provider account identifiers.
- Credentials, signing secrets, OAuth client secrets, session cookies,
  Connect Links, or realistic token-shaped strings.
- Session logs, transcripts, memory files, or model reasoning.

Tests, fixtures, and documentation must use unmistakably synthetic values:
`example.com` hosts and emails, RFC 5737 addresses (`192.0.2.0/24`,
`198.51.100.0/24`, `203.0.113.0/24`), `+1 555` phone numbers, sequential fake
UUIDs (`00000000-0000-4000-8000-000000000001`), fake Slack IDs (`T0000000001`,
`U0000000001`, `B0000000001`), and generic names (Example Contractor, Casey,
Operator).

Runtime data directories, `.env*` files (other than `.env.example`), and any
`scratch/` or `memory/` directory stay untracked.

## Tenancy and security invariants

These are enforced by tests and must never be weakened:

- Every repository function that reads or mutates tenant-owned data takes a
  `TenantContext` as its first argument and includes `tenant_id` in the query.
  There is no global or mutable "current tenant".
- Tenant identity for dashboard requests comes from the server-side session,
  never from a request body, query string, or header supplied by the browser.
- Slack request signatures are verified against the exact raw body before any
  field of the payload is trusted.
- Private runtime endpoints authenticate with service credentials, never with
  dashboard cookies.
- Secrets are only ever stored through `packages/secrets` (encrypted at rest)
  and are redacted by the logger. API responses never include token material.
- Personal connections are usable only by their owning member. Shared
  connections require `owner` or `admin` to create, modify, or delete.
- Bot-authored Slack messages are not dropped as loop prevention. The gateway
  enforces authenticated self-echo rejection (exact bot user match) and
  durable provider event deduplication only. Loop control belongs to the
  agent decision boundary inside Jarvis.
- Forwarded Slack events carry the complete provider-sized message. Never
  truncate a message silently.

## Communication

Use plain language in user-facing text: `provisioning`, `ready`, `needs
attention`, `suspended`. Do not surface raw provider errors, internal job
names, or stage codes to customers. Stage codes are for operator diagnostics
only.

Commit messages describe code behaviour only, never a live deployment or a
customer interaction.
