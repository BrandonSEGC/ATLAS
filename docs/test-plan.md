# Test plan

Every test in this plan must exist and pass before the slice that owns it is
considered done (`docs/implementation-plan.md`). Test IDs are stable and
appear in test names so coverage can be audited against this document.

## Layers and tooling

| Layer | Runner | Dependencies | Location |
| --- | --- | --- | --- |
| Unit | `node --test` via `tsx` | none | `packages/*/test` |
| Repository / RLS | `node --test` | disposable PostgreSQL with real migrations | `packages/database/test` |
| Route | `node --test` with Fastify `inject` | PostgreSQL, fakes for Slack, Pipedream, provisioner, runtime | `apps/control-plane/test` |
| End to end (staging) | manual checklist plus a scripted smoke run | real Slack dev app, Pipedream development environment, Railway staging pool | `docs/operations.md` |

Fakes live in `packages/testing`:

- `FakeSlack`: OAuth v2 `access`, OpenID `token`/`userinfo`/JWKS with a
  test-only signing key, `chat.postMessage` recorder, and a signer that
  produces valid `X-Slack-Signature` headers for a given secret.
- `FakePipedream`: `oauth/token`, `connect/apps`, `connect/:project/users/:user/accounts`,
  `connect/:project/tokens`, and a minimal MCP Streamable HTTP server that
  records the `x-pd-*` headers it received and can return
  authorization-required errors on demand.
- `FakeProvisioner`: in-memory implementation with programmable failures per
  step and an inventory that tests inspect for duplicates.
- `FakeRuntime`: HTTP server exposing `/health`, `/slack/events` (verifies the
  `v0` signature against a given forwarding secret and records bodies), and
  `/api/admin/connectors` (records projections, optionally requires bearer).

All fixtures are synthetic: tenant IDs `00000000-0000-4000-8000-0000000000NN`,
Slack IDs `T0000000001`/`U0000000001`/`B0000000001`, emails at `example.com`,
hostnames `atlas.example.com` and `runtime-a.internal.example`.

## 1. Tenant isolation (`TI-*`)

| ID | Test | Layer |
| --- | --- | --- |
| TI-01 | Repository: `members.list(ctxA)` never returns tenant B rows even when B has more members; same for `connections`, `runtime_instances`, `slack_installations`, `audit_events`, `usage_events`. | Repository |
| TI-02 | Repository: `members.updateRole(ctxA, memberB.id)` affects zero rows and returns not found. Same for `connections.revoke`, `runtime.retry`. | Repository |
| TI-03 | RLS: with `app.tenant_id = A`, a raw `SELECT * FROM connections` returns only A's rows; with no setting it returns zero rows. Verified against `atlas_app` role. | Repository |
| TI-04 | Composite FK: inserting a connection with `tenant_id = A` and `owner_member_id` from B fails. | Repository |
| TI-05 | Route: session for A calling `GET /api/connections/:idB/status` gets 404, not 403. | Route |
| TI-06 | Route: member A1 cannot see member A2's personal connection ID or status; owner A0 sees only `{ appSlug, ownerDisplayName, status }`. | Route |
| TI-07 | Broker: runtime A calling `/mcp/v1/connections/:idB` gets 404 and an `runtime.identity_mismatch`-class audit entry; no upstream call recorded by `FakePipedream`. | Route |
| TI-08 | Broker: personal connection call without `X-Atlas-Actor-Slack-User-Id` or with another member's ID returns JSON-RPC `-32001`; no upstream call. | Route |
| TI-09 | Runtime usage: runtime A posting to `/internal/runtimes/<idB>/usage` is rejected with 404 and audited. | Route |
| TI-10 | Gateway: a valid event for team B is never enqueued for runtime A (delivery rows carry B's runtime only). | Route |
| TI-11 | Session tenant binding: a request body containing `tenantId` of B is ignored; the handler uses the session tenant. | Route |

## 2. Slack OAuth and login (`SO-*`)

| ID | Test | Layer |
| --- | --- | --- |
| SO-01 | Install callback with valid state creates exactly one tenant, one installation, one owner member, one bot token secret, and one `runtime.provision` job. | Route |
| SO-02 | Replaying the same callback (same `state` and `code`) returns 400 and creates nothing new; job count unchanged. | Route |
| SO-03 | Reinstall into the same team (new state) updates the existing installation, rotates the secret (old row retired), keeps one tenant and does not enqueue a second provision job while one exists. | Route |
| SO-04 | Missing, unknown, expired, consumed, and cookie-mismatched state are each rejected with `invalid_oauth_state`; Slack exchange is not called. | Route |
| SO-05 | Slack exchange `ok: false` yields `slack_exchange_failed`; response body contains no Slack payload fields. | Route |
| SO-06 | Login callback for a team with no installation redirects to `/install?reason=not_installed` and creates no member or session. | Route |
| SO-07 | Login resolves membership by `(team_id, user_id)`: two users with the same email in different teams map to different members; the same user ID in two teams maps to two members. | Route |
| SO-08 | ID token with wrong `iss`, wrong `aud`, expired, bad signature, or nonce mismatch is rejected. | Unit + Route |
| SO-09 | Login with `expected_tenant_id` bound to A but token for team B is rejected with `tenant_mismatch`. | Route |
| SO-10 | Revoked installation (`revoked_at` set): login redirects to install page; gateway drops events with reason `revoked`. | Route |
| SO-11 | Session cookie attributes: HttpOnly, Secure (non-local), SameSite=Lax; token is not the row ID; logout revokes; disabled member's requests fail after disable. | Route |
| SO-12 | State-changing session route without matching `Origin` is rejected. | Route |
| SO-13 | Rate limits on `/auth/slack/login` and callbacks return 429 after the configured burst. | Route |
| SO-14 | Installer becomes `owner`; subsequent login users become `member`; a Slack admin flag in `userinfo` does not change role. | Route |

## 3. Slack events (`SE-*`)

| ID | Test | Layer |
| --- | --- | --- |
| SE-01 | Valid signature and fresh timestamp: 200, receipt `queued`, one delivery row, one job. | Route |
| SE-02 | Invalid signature: 401; no receipt. Modified body with original signature: 401. | Route |
| SE-03 | Timestamp older than 300 s or in the future by more than 300 s: 401. | Route |
| SE-04 | `url_verification` returns the challenge and does not require an installation. | Route |
| SE-05 | Duplicate `event_id` (Slack retry with `X-Slack-Retry-Num`): 200, still one delivery, `retry_num` updated. | Route |
| SE-06 | Exact self-echo (`event.user == bot_user_id`): 200, receipt `dropped/self_echo`, no delivery. | Route |
| SE-07 | Message from another bot (`bot_id` set, `subtype = bot_message`, user not the installation bot): forwarded. | Route |
| SE-08 | Unknown team, revoked installation, suspended tenant, runtime `provisioning`/`failed`/`suspended`: 200 with the corresponding drop reason, no delivery. | Route |
| SE-09 | Wrong `api_app_id`: dropped. | Route |
| SE-10 | Provider-sized message: a 40 000 character text with blocks and 5 files is delivered to `FakeRuntime` byte-for-byte (SHA-256 equal) with a valid forwarding signature. | Route |
| SE-11 | Delivery retry: `FakeRuntime` fails twice then succeeds; exactly one final delivery; `raw_body` is NULL after success. | Route |
| SE-12 | Delivery 401 from runtime marks runtime `degraded` and stops retrying that delivery until credentials are re-pushed. | Route |
| SE-13 | Gateway path performs no work requiring the runtime before responding (response under budget with `FakeRuntime` stalled). | Route |
| SE-14 | `app_uninstalled` and `tokens_revoked` set `revoked_at`, revoke sessions, and audit. | Route |
| SE-15 | Forwarded headers include `X-Atlas-Event-Id`, `X-Atlas-Tenant-Id`, `X-Atlas-Runtime-Id`; body unchanged. | Route |

## 4. Pipedream (`PD-*`)

| ID | Test | Layer |
| --- | --- | --- |
| PD-01 | `POST /api/connections/pipedream/link` as member with `ownership: member` creates a `pending` connection with `external_user_id = tenant:<A>:member:<m>`; `FakePipedream` recorded that exact ID and `x-pd-environment: production`. | Route |
| PD-02 | Same as PD-01 with `ownership: tenant` as `member` role: 403; no connection, no Pipedream call. As `admin`: 201 with `external_user_id = tenant:<A>`. | Route |
| PD-03 | Request body with a foreign `externalUserId` or `tenantId` field is ignored (schema strips unknown keys) and the server-derived ID is used. | Route |
| PD-04 | `ATLAS_ENV=production` with Pipedream environment `development` fails config validation at startup. | Unit |
| PD-05 | Every dashboard and operator response schema is scanned: no property named `token`, `secret`, `clientSecret`, `authorization`, and no value matching token prefixes. Connect Link responses contain the link URL only. | Route |
| PD-06 | Projection for a tenant with three connections (two shared apps, one personal) produces three MCP server entries with distinct aliases `pipedream__hubspot`, `pipedream__quickbooks`, `pipedream__gmail__m<8hex>`, each with exactly one app slug and the broker URL. | Unit |
| PD-07 | Broker forwards exactly one `x-pd-app-slug`, the connection's `external_user_id`, project, and environment; client-supplied `x-pd-*` headers are stripped. | Route |
| PD-08 | Diagnostics: `FakePipedream` fails at each stage in turn; the response names that stage and the message contains no token or upstream body. | Route |
| PD-09 | Authorization-required upstream error becomes JSON-RPC `-32002` with the safe prompt, connection status `unhealthy`, audit written. | Route |
| PD-10 | Revoking a connection deletes the Pipedream account (recorded by fake) and enqueues a projection push; the alias becomes reusable. | Route |
| PD-11 | External user created lazily: no `provider_external_users` row until the first connection for that owner. | Route |
| PD-12 | Connection mutations while the tenant is suspended: 409. | Route |
| PD-13 | Rate limit on link creation per member. | Route |

## 5. Provisioning (`PV-*`)

| ID | Test | Layer |
| --- | --- | --- |
| PV-01 | Running `runtime.provision` three times for the same tenant yields one service, one volume, one runtime row, one set of credentials in `FakeProvisioner` inventory. | Route/worker |
| PV-02 | `FakeProvisioner` fails after service creation but before volume attach; retry completes without a second service; `provider_state` shows resumed steps. | Worker |
| PV-03 | Two tenants provisioned concurrently get distinct volumes, ingress URLs, forwarding secrets, admin tokens, service tokens, and master keys. | Worker |
| PV-04 | Suspension: `runtime.suspend` stops the service; gateway drops events with `suspended`; connection link/delete return 409; broker returns 404 for that runtime's token. Resume restores all three. | Route/worker |
| PV-05 | `runtime.destroy` refuses unless `tenants.status = deleting` and `purge_after < now()`; running it twice is a no-op; volume and service absent afterwards; Pipedream external users deleted. | Worker |
| PV-06 | Health check: two failed `/health` polls move `ready -> degraded`; one success moves back. | Worker |
| PV-07 | Projection push is idempotent: identical projection twice results in one restart request to `FakeRuntime`; a changed projection triggers a restart and updates `last_projection_hash`. | Worker |
| PV-08 | Variables passed to the provisioner never include Pipedream credentials, the platform Slack secrets, or the Railway token (assert against the fake's captured input). | Worker |
| PV-09 | `POST /api/runtime/retry` is allowed only from `failed`/`degraded` and by owner/admin; enqueues under the same singleton key. | Route |
| PV-10 | Credential rotation: rotate service token; old token valid during grace, invalid after; new token valid immediately; audit written. | Route/worker |

## 6. Secrets, audit, config (`SC-*`)

| ID | Test | Layer |
| --- | --- | --- |
| SC-01 | Encrypt/decrypt round trip; ciphertext with swapped `aad` fails; wrong key fails. | Unit |
| SC-02 | Master key rotation re-wraps every DEK; old key removal after rotation still decrypts all secrets. | Repository |
| SC-03 | Logger redacts `xoxb-`, `xapp-`, `atlr_`, `Bearer ...`, and any configured secret value injected into a log line. | Unit |
| SC-04 | Audit writer rejects metadata containing disallowed keys (`token`, `text`, `body`) at type level and at runtime. | Unit |
| SC-05 | Config: missing master key ring, HTTP callback URL in production, or unknown `ATLAS_ROLES` fails startup with a clear message and no partial listen. | Unit |
| SC-06 | `check:public-safety` fails on a fixture containing a non-`example.com` email or a public IP outside RFC 5737. | Script |

## 7. Staging end-to-end checklist (manual, before pilot)

1. Fresh Slack dev workspace: Add to Slack -> onboarding page shows
   "Preparing your private Jarvis".
2. Dashboard shows `Ready` within the provisioning deadline without operator
   action.
3. Sign in with Slack as the installer (owner) and as a second user (member).
4. Connect one shared app as owner and one personal app as member; member
   cannot create a shared connection.
5. DM Jarvis; receive a reply in the DM. Mention Jarvis in a channel thread;
   reply lands in that thread.
6. Ask Jarvis to use a read-only tool from the shared connection; verify the
   broker recorded the call with the tenant external user.
7. Ask, as the member, for a personal-connection tool; verify success for the
   owner and the safe prompt for another user.
8. Operator: view tenant, retry provisioning on a deliberately failed tenant,
   suspend and resume, rotate runtime credentials, run diagnostics with a
   revoked Pipedream account and confirm the stage reported.
9. Delete the synthetic tenant and confirm purge after the grace window
   (shortened by configuration in staging).
