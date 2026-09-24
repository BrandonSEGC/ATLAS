# ADR-0011: Jarvis images published to GHCR and pinned by digest

Status: Accepted

## Context

ATLAS must provision a versioned Jarvis release rather than build or vendor
Jarvis source. The Jarvis repository has a `Dockerfile` but no image
publishing workflow today.

## Decision

- The Jarvis repository publishes `ghcr.io/<org>/jarvis` from a GitHub
  Actions workflow on version tags: tags `<semver>` and `sha-<commit>`, plus
  `latest` for convenience only. The image build sets `JARVIS_RELEASE=<semver>`
  and adds a `HEALTHCHECK` on `/health` (Jarvis change J6).
- ATLAS configuration holds `JARVIS_RELEASE_IMAGE` as a full reference with
  digest (`ghcr.io/<org>/jarvis@sha256:...`). New tenants are provisioned
  with the configured reference; `runtime_instances.release` records the
  exact reference used.
- Changing the default release for new tenants is a configuration change.
  Upgrading existing runtimes uses `provisioner.updateRelease` per runtime
  from an operator action; fleet-wide rolling upgrade is a later milestone.
- The registry is read by the provider using a read-only pull credential
  held by the provider configuration, never by tenants.

## Consequences

- ATLAS and Jarvis release lifecycles stay separate.
- Reproducibility: a runtime's exact image is known from the database.

## Alternatives rejected

- Building Jarvis inside ATLAS's pipeline: couples the repositories.
- Tag-only references (`:1.4.2`): mutable; digest pinning is required for
  provenance.
