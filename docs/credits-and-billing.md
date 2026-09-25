# Credits, metering, and pass-through billing

Design for ADR-0014. Usage is metered in real time at the ATLAS model
gateway and MCP broker, priced from a versioned price book, and debited from
a per-tenant credit balance. Payment collection is a thin layer on top of the
ledger and is the first slice after the milestone.

## Metering points

| Source | Where metered | Unit | SKU |
| --- | --- | --- | --- |
| Model calls | ATLAS model gateway, from the provider's usage block | tokens by class | `model.<provider>.<model>.{input,output,cache_read,cache_write}` |
| Tool calls through Pipedream | MCP broker | calls | `tool_call.pipedream` |
| Runtime uptime | worker, on health checks and stop/start transitions | hours by resource class | `runtime_hour.<class>` |
| Workspace storage | worker, from provider volume metrics or backup size | GB-hours | `storage_gb_hour` |
| Pipedream external users | `provider_external_users` rows per month | user-months | `connect_external_user_month` |

Every metering point writes a `usage_events` row and, if the SKU is priced, a
`credit_ledger` debit in the same transaction. Usage without a price (for
example during a free pilot) is still recorded so pricing can be introduced
later from real data.

## Price book

```text
price_book_versions (id, effective_from, created_by, notes)
price_book_items    (version_id, sku, unit, unit_cost_micros, unit_price_micros, markup_basis)
```

- A new version is created for any price change; items are copied forward
  and edited. Usage is priced with the version whose `effective_from` is the
  latest one not after `occurred_at`.
- `unit_cost_micros` is the provider's list price per unit at the time of
  publishing. `unit_price_micros` is what the tenant pays. `markup_basis`
  records how the price was derived (`factor:1.25`, `flat`, `manual`).
- Provider price changes are handled by publishing a new version; there is
  no automatic scraping.

## Ledger

```text
credit_ledger (id, tenant_id, occurred_at, entry_type, amount_micros, balance_after_micros,
               usage_event_id, price_version_id, reference_type, reference_id, actor_type, actor_id, memo)
credit_balances (tenant_id, balance_micros, low_threshold_micros, last_notified_level, updated_at)
```

- `entry_type`: `purchase`, `grant`, `debit`, `adjustment`, `refund`,
  `expiry`. Debits are negative amounts.
- Writes happen inside `withTenantTransaction`, taking
  `SELECT ... FOR UPDATE` on `credit_balances` first, so `balance_after` is
  exact and monotonic per tenant.
- The ledger is append-only. Corrections are `adjustment` entries with a
  memo and an audit event.

## Gateway enforcement

1. Estimate: `estimate = price(input) x approxInputTokens(body) + price(output) x max_tokens`.
   `approxInputTokens` is bytes / 4 rounded up; it errs high.
2. Allow if `balance_micros >= estimate` or if the tenant's entitlement has
   `allowNegativeUntilMicros` (small grace, default 0).
3. Refuse otherwise with a provider-shaped 402 and a plain message. Jarvis
   shows the model error to the user; the message reads: "Your ATLAS credits
   are exhausted. An owner can add credits in the ATLAS dashboard."
4. After the response completes, debit the actual metered cost. If the
   stream is aborted by the client, debit what the provider reported so far
   (Anthropic sends cumulative output tokens in `message_delta`).
5. If the provider returns an error, no debit is written; the usage event is
   recorded with `outcome = error` and zero quantity for observability.

Concurrency: overrun is bounded by the number of in-flight requests per
tenant times one request's `max_tokens`. Jarvis limits model calls per run;
the gateway additionally caps concurrent requests per tenant from
entitlements (`maxConcurrentModelRequests`, default 4).

## Notifications

`credit_balances.last_notified_level` tracks the last threshold crossed (20
percent of the last purchase or grant, 5 percent, 0). Crossing a threshold
downward sends one Slack DM to each tenant owner using the tenant's bot token
(the same token Jarvis uses to reply) and writes an audit event. Topping up
resets the level.

## Dashboard and operator surfaces

- Tenant (owner/admin): balance, burn rate over 7 and 30 days, usage by SKU
  for the current period, statements by month, "Add credits" (pilot: shows
  contact instructions; post-milestone: checkout).
- Tenant (member): balance only.
- Operator: per-tenant balance and margin, grant/adjust with memo,
  reconciliation drift report, price book versions and publish.

## Statements

A monthly statement is a deterministic query over `usage_events` and
`credit_ledger` for `[period_start, period_end)`: quantities and amounts by
SKU, opening and closing balance, purchases and grants. Statements are
rendered on demand and cached; they are never the source of truth.

## Payment collection (post-milestone slice 8)

- Payment provider hosted checkout for credit packs; the webhook writes a
  `purchase` ledger entry keyed by the payment intent ID (idempotent).
- Optional auto-top-up: when balance falls below a threshold, charge the
  saved payment method for the configured pack.
- Refunds write `refund` entries; disputes are operator adjustments.
- Invoicing for larger customers can be added as a `grant` on invoice and a
  receivable outside the ledger.

`subscriptions` keeps plan and entitlements; the ledger keeps money. A
monthly platform fee, if introduced, is a scheduled `debit` against the
`runtime_hour` or a `platform_month` SKU rather than a separate billing path.

## Reconciliation

Daily job per provider: fetch the provider's cost or usage report for each
tenant scope (workspace/project) for the previous day; compare against
`SUM(unit_cost_micros x quantity)` for that tenant and day; write a
`reconciliation_runs` row with the drift; alert operators above 2 percent or
USD 1, whichever is larger.

## What is deliberately not stored

Prompts, completions, tool arguments, tool results, or any message content.
`usage_events.dimensions` holds provider, model, token class, request ID,
and outcome only.
