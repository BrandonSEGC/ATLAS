# ADR-0014: Per-tenant model provider keys, ATLAS model gateway, and credit metering

Status: Accepted (product owner decision: per-tenant keys are required and
API fees must be tracked and passed on to the tenant through a credit
system). Resolves OD-3 and OD-10.

## Context

Every model call a runtime makes costs money at the provider. The product
owner requires that each tenant's provider usage be attributable to that
tenant, that fees are passed on, and that the platform tracks this as
credits.

Two facts from the Jarvis codebase shape the design:

- Jarvis honours `ANTHROPIC_BASE_URL`, `OPENAI_BASE_URL`,
  `FIREWORKS_BASE_URL`, and `JARVIS_MODEL_ROUTER_BASE_URL` in every model
  path (agent loop, router classifier, Claude CLI passthrough, media
  analyzers). A proxy in front of the providers therefore requires no Jarvis
  change.
- Jarvis already computes `inputTokens`, `outputTokens`, `cacheReadTokens`,
  `cacheWriteTokens`, and `estimatedUsd` per call in its logs, but only
  locally and only as JSON lines; nothing enforces a budget.

## Decision

### 1. Per-tenant provider identity

For each tenant and each enabled provider, ATLAS creates a dedicated
provider-side scope (an Anthropic Console workspace, an OpenAI project, or
the provider's equivalent) and one API key inside it, using the provider's
administration API where it exists and an operator-assisted step where it
does not. The key is stored per ADR-0007 (`secrets.kind =
model_provider_key`) and is **held only by ATLAS**; it is never injected into
the runtime.

Benefits: provider-side cost reports per tenant for reconciliation,
independent revocation per tenant, provider-side spend caps as a backstop
where the provider offers them.

### 2. ATLAS model gateway

Runtimes are configured with
`ANTHROPIC_BASE_URL=https://<atlas>/model/v1/anthropic`,
`OPENAI_BASE_URL=https://<atlas>/model/v1/openai` (and equivalents), and the
runtime service token as the provider "API key" value. The gateway:

1. Authenticates the runtime by token hash -> `(tenant_id, runtime_id)`;
   requires runtime status in `ready | degraded` and tenant `active`.
2. Checks credit availability: `balance >= estimatedCost(request)` where the
   estimate is `(input tokens estimated from body size + max_tokens) x price`.
   If not, it returns a provider-shaped error (HTTP 402 with the provider's
   error envelope, `type: "billing_error"` for Anthropic, `code:
   "insufficient_quota"` for OpenAI) so Jarvis surfaces a plain "out of
   credits" message, and it notifies the tenant owner once per threshold.
3. Forwards the request unchanged to the provider with the **tenant's
   provider key**, streaming responses through. For OpenAI-compatible
   streams it sets `stream_options.include_usage = true` so the final chunk
   carries usage.
4. Parses the usage block from the response (non-streaming body, or
   `message_start` plus `message_delta` events for Anthropic, final usage
   chunk for OpenAI), including cache read/write tokens.
5. Writes one `usage_events` row (`kind = model_usage`) and one
   `credit_ledger` debit in the same transaction, priced from the active
   `price_book` version. Provider request IDs are recorded for
   reconciliation; prompts and completions are **not** stored.
6. Enforces per-tenant rate and concurrency limits from entitlements.

The gateway is provider-agnostic at the transport level (path prefix selects
the provider) and only needs a usage parser per provider format.

### 3. Credit system

- Denomination: integer micro-USD (`bigint`) in the ledger; the dashboard
  displays credits where 1 credit = USD 0.01 (OD-14 may change the display
  unit without touching the ledger).
- `credit_ledger` is append-only: `purchase`, `grant`, `debit`, `adjustment`,
  `refund`, `expiry`. `credit_balances` is a per-tenant materialized balance
  updated in the same transaction under row lock.
- `price_book` is versioned with `effective_from`. Each SKU has
  `unit_cost_micros` (what the provider charges ATLAS) and
  `unit_price_micros` (what the tenant pays). Model SKUs are per provider,
  model, and token class (`input`, `output`, `cache_read`, `cache_write`).
  Non-model SKUs: `tool_call`, `runtime_hour` by resource class,
  `storage_gb_hour`, `connect_external_user_month`.
- Pass-through policy: `unit_price = unit_cost x markup` with a default
  markup factor (OD-14) plus flat platform SKUs. Because cost and price are
  both stored, margin per tenant is a query.
- Hard stop: model calls are refused when the balance cannot cover the
  estimate. Nothing else is suspended: Slack still works and Jarvis replies
  that credits are exhausted; tools and the runtime stay up so the owner can
  top up. Overrun is bounded by one in-flight request per concurrent run.
- Thresholds: owner notification via the tenant's Slack bot token (DM to the
  owner) at 20 percent, 5 percent, and 0.
- Statements: monthly statement per tenant from the ledger with usage by
  SKU, visible in the dashboard and to operators.
- Pilot funding: operators add `grant` entries; self-service `purchase`
  through a payment provider is the first post-milestone slice and lands as
  a webhook that writes `purchase` entries. The ledger does not change.

### 4. Reconciliation

A daily job pulls provider cost reports per tenant scope (where the provider
exposes them) and compares against the sum of `unit_cost_micros` in the
ledger for the same day, flagging drift above a tolerance to operators.

## Consequences

- ATLAS is on the model hot path (as it already is for tools). Requirements:
  streaming proxy with back-pressure, at least two replicas, provider
  timeouts passed through, and a bypass switch per tenant for incidents
  (inject the tenant key directly, losing real-time metering but not
  attribution).
- Blast radius improves: a compromised runtime holds no provider key at all.
- The `estimatedUsd` Jarvis computes locally becomes advisory; ATLAS's ledger
  is authoritative. Jarvis change J5 (usage reports) is no longer needed for
  metering and is downgraded to optional telemetry.
- Prompt caching: cache read/write tokens are priced separately so tenants
  benefit from cache discounts.

## Alternatives rejected

- Injecting the per-tenant provider key into the runtime and metering from
  runtime reports: metering is after the fact and at-least-once, hard stops
  are impossible until the provider's cost report lags catch up, and a
  compromised runtime can spend freely until the key is revoked.
- One platform key with attribution from runtime reports: no provider-side
  attribution, no per-tenant revocation.
- Prepaid holds per request: more correct under high concurrency, but Jarvis
  bounds concurrency per run and the added ledger complexity is not
  justified for the pilot. Revisit if overrun becomes material.
