# ADR-0009: Railway GraphQL API adapter behind `RuntimeProvisioner`

Status: Superseded as the first provider by ADR-0015 (Fly Machines). The
`RuntimeProvisioner` interface defined here remains the contract; the Railway
adapter described below is optional and not part of the first milestone.

## Context

The first provider is Railway. Application logic must not depend on Railway
identifiers, and provisioning must be idempotent, retry-safe, and give each
tenant a dedicated volume and a private ingress.

## Decision

- `packages/provisioning` defines:

```ts
interface RuntimeProvisioner {
  provision(input: ProvisionRuntimeInput): Promise<RuntimeRecord>;
  status(runtimeId: string): Promise<RuntimeStatus>;
  updateRelease(runtimeId: string, release: string): Promise<void>;
  updateVariables(runtimeId: string, variables: Record<string, string>): Promise<void>;
  suspend(runtimeId: string): Promise<void>;
  resume(runtimeId: string): Promise<void>;
  destroy(runtimeId: string): Promise<void>;
}
```

  `ProvisionRuntimeInput` carries `runtimeId`, `tenantId`, `release` (image
  reference by digest), `resourceClass`, `region`, `variables` (already
  resolved plaintext for this call only), and `idempotencyKey`.
  `RuntimeRecord` returns opaque `providerServiceRef`, `providerVolumeRef`,
  and `ingressUrl`.
- `RailwayProvisioner` uses Railway's public GraphQL API with a platform
  token held only by ATLAS. Resources are named deterministically
  (`atlas-rt-<runtimeId>`) and looked up by name before creation, so retries
  resume. Steps: ensure service -> ensure volume mounted at `/data` -> set
  variables -> set image and start command (`--adapter=slack:webhook
  --port=3002 --ui=/opt/jarvis/ui/dist --skills=/opt/jarvis/skills /data`)
  -> deploy -> wait for deployment success -> return the private hostname
  (`<service>.railway.internal:3002`).
- Runtimes get no public domain. ATLAS and its tenant runtimes run in the
  same Railway project and environment so the private network is shared;
  each ATLAS environment (staging, production) maps to one Railway
  environment. If per-project limits are reached, a second "pool" project
  gets its own ATLAS gateway replica; `runtime_instances.region` records the
  pool.
- Suspend = stop the service's deployment (volume retained); resume =
  redeploy. Destroy = delete service and volume, only from the deletion state
  machine (ADR-0013).
- `FakeProvisioner` in the same package implements the interface in memory
  with injectable failures for tests.
- Resource limits and the resource class mapping are configuration, not
  code: `resourceClass -> { cpu, memoryMb, volumeGb }`.

## Consequences

- No Railway type leaks past `packages/provisioning`. A Fly.io or Kubernetes
  adapter is a new file.
- Railway API changes are contained; the adapter is covered by contract tests
  against recorded fixtures with synthetic IDs.
- The private-network requirement constrains topology: ATLAS must be
  deployed on Railway in the same project as the runtimes for v1.

## Alternatives rejected

- Railway templates or CLI-driven provisioning: not idempotent or scriptable
  enough for per-signup automation.
- Public runtime domains with token auth: larger attack surface; private
  network plus per-runtime credentials is simpler to reason about.
- Kubernetes from the start: premature orchestration for a pilot.
