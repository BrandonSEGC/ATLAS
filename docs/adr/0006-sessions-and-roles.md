# ADR-0006: Server-side sessions in PostgreSQL; roles; operator flag

Status: Accepted

## Context

Dashboard users authenticate with Sign in with Slack. Sessions must be
revocable, must not carry secrets, and must reflect role changes immediately.
Platform operators need a separate, auditable privilege.

## Decision

- Sessions are rows in `sessions`. The cookie `atlas_session` holds a random
  256-bit token; the row stores its SHA-256. Cookie flags: HttpOnly, Secure
  (except `ATLAS_ENV=local`), SameSite=Lax, Path=/, no `Domain`.
- Expiry: idle 12 hours (extended on activity, at most once per 5 minutes),
  absolute 7 days. Logout, member disable, tenant suspension for security
  reasons, and operator "revoke sessions" delete rows.
- Role is loaded from `members` on each request. Roles: `owner`, `admin`,
  `member`, as defined in `docs/architecture.md` section 4. The installer is
  the first owner. Slack workspace admin status is not mapped to any ATLAS
  role.
- State-changing session routes require `Origin == ATLAS_PUBLIC_URL`.
- Platform operators are identified at login by matching `(team_id,
  slack_user_id)` against `platform_operators`, which is managed by a CLI
  script (`npm run operator:add -- --team T... --user U...`). The session gets
  `is_platform_operator = true`; operator routes require it. Operators still
  act inside a tenant context per request; there is no cross-tenant query
  without an explicit tenant ID in the operator call.

## Consequences

- No JWTs and no client-side claims. Sessions are queryable and revocable.
- One extra query per request for the member row; acceptable.

## Alternatives rejected

- Stateless JWT sessions: cannot revoke, and roles would go stale.
- Redis session store: extra infrastructure.
- Deriving operators from an environment variable: not auditable.
