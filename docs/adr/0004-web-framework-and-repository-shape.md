# ADR-0004: Fastify, npm workspaces, single deployable

Status: Accepted

## Context

The first milestone needs an HTTP API, a webhook receiver that must see raw
bytes, a background worker, and a small dashboard. The blueprint allows one
deployable service if module boundaries stay explicit.

## Decision

- TypeScript on Node.js 22, `strict` plus `noUncheckedIndexedAccess` and
  `exactOptionalPropertyTypes`.
- Fastify 5 as the HTTP framework: first-class schema validation hooks (zod
  via type provider), raw-body access per route for webhook signature
  verification, mature cookie, rate-limit, and helmet plugins, and low
  overhead for the gateway path.
- npm workspaces monorepo (`docs/repository-structure.md`) with one
  deployable `apps/control-plane` that enables roles by `ATLAS_ROLES`.
  Packages are framework-free; only the app imports Fastify.
- Vite + React dashboard served as static assets by the api role; session
  cookie authentication, no SSR.
- Biome for lint and format; Node's built-in test runner with `tsx`.

## Consequences

- Splitting the gateway or worker into separate services later means a new
  entry that enables one role; no package changes.
- No in-process state may be relied on for correctness (dedup, locks,
  caches), since replicas and roles may run separately.

## Alternatives rejected

- Next.js full-stack: raw-body handling and long-running workers are awkward;
  the dashboard is small.
- Express: weaker typing and validation story.
- Separate repos per app: premature for one team and one milestone.
