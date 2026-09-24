# ADR-0007: Envelope encryption with a master key ring and per-tenant DEKs

Status: Accepted

## Context

ATLAS stores tenant Slack bot tokens, per-runtime credentials, per-tenant
model keys, and platform secrets. A database backup must not be enough to
read them, and keys must be rotatable without downtime.

## Decision

- `ATLAS_MASTER_KEYS` is a JSON key ring: `[{"id":"k2","key":"<base64 32
  bytes>"},{"id":"k1",...}]`. The first entry is the active key. It is
  supplied by the deployment secret manager, validated at startup, never
  written to disk or logs.
- Each tenant has a data-encryption key (DEK) generated on tenant creation,
  wrapped with the active master key (AES-256-GCM) and stored in
  `tenant_keys`. Platform secrets (no tenant) are encrypted directly with the
  active master key.
- Secrets are encrypted with AES-256-GCM using a random 96-bit nonce and AAD
  `"<tenant_id>|<kind>|<secret_id>"`, so a ciphertext cannot be moved to
  another row or tenant.
- `packages/secrets` exposes `store(ctx, kind, plaintext) -> SecretRef`,
  `reveal(ctx, ref) -> Buffer` (only callable from `runtime-client`,
  `provisioning`, `connections`, and `auth` modules), `retire(ref)`, and
  `redact(value)`. The `SecretRef` type is a branded UUID that cannot be
  confused with plaintext.
- Bearer tokens ATLAS only needs to verify (runtime service tokens, session
  tokens, OAuth state) are stored as SHA-256 hashes, not encrypted.
- Rotation:
  - Master key: add a new key at the front of the ring, deploy, run
    `POST /api/operator/platform/rotate-master-key` which re-wraps every DEK
    and re-encrypts platform secrets under the new key; remove the old key
    after the job reports zero rows on it.
  - Tenant DEK: `rotateTenantKey(tenant)` re-encrypts that tenant's secrets;
    used on suspected compromise.
  - Individual secrets: write a new row, update the reference, mark the old
    row `retired_at`; callers that need overlap (runtime service tokens) keep
    both valid for a grace window.
- Provider-side rotation runbooks (Slack signing secret, Pipedream client
  secret, Railway token) are in `docs/operations.md`.

## Consequences

- No external KMS dependency for the pilot. The interface allows wrapping
  DEKs with a cloud KMS later by adding a key-ring provider.
- Every secret read is explicit and auditable by module.

## Alternatives rejected

- Encrypting each secret directly with the master key: rotation would touch
  every row and cross-tenant AAD binding would be weaker.
- Storing plaintext in provider variables only: ATLAS needs the values to
  re-provision and to sign forwarded events.
