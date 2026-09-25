# ADR-0015: Runtime hosting provider selection

Status: Proposed (recommendation for product owner confirmation; supersedes
ADR-0009 as the first provider once accepted. The `RuntimeProvisioner`
interface from ADR-0009 is unchanged.)

## Context

Railway was the working assumption only because Jarvis was built there. The
product owner is not attached to it and wants the setup that runs most
smoothly. This ADR records the requirements, the candidates considered, and
a recommendation.

### What a tenant runtime actually is

From the Jarvis codebase: a long-running Node.js process with a headless
Chrome (Puppeteer) available, a host sandbox for tool execution, and a
file-based workspace (`IDENTITY.md`, `MEMORY.md`, `settings.json`,
`.jarvis/`, `log.jsonl`, `attachments/`, `skills/`). Practical footprint is
1 to 2 GB RAM and a 1 to 5 GB volume. It is stateful, mostly idle, and bursts
when a Slack message arrives or a scheduled task fires.

### Requirements, in priority order

1. Isolation: one runtime and one volume per tenant with a real security
   boundary between tenants (ADR-0001).
2. Programmatic, idempotent lifecycle: create, update image, stop, start,
   destroy, all from ATLAS in seconds, with stable API semantics.
3. Durability of the workspace volume, with snapshots and restore.
4. Private networking: runtimes have no public ingress; ATLAS reaches them
   privately.
5. Idle economics: hundreds of mostly idle tenants must not each cost an
   always-on VM if the platform can stop and wake them. The ATLAS gateway
   already decouples Slack's acknowledgement from delivery, so wake-on-event
   is possible.
6. Low operational burden: managed control plane, built-in metrics and logs,
   no node management.
7. Provisioning latency: a new customer should see "Ready" within a few
   minutes of installing.
8. Clear path to the next scale tier without rewriting the provisioner.

### Candidates

| | Railway | Fly.io Machines | AWS ECS Fargate + EFS | Kubernetes (GKE Autopilot / EKS) | Single host (`hostd`) |
| --- | --- | --- | --- | --- | --- |
| Isolation | containers | Firecracker microVMs | Firecracker microVMs (Fargate) | containers; gVisor optional | containers on one VM |
| Lifecycle API | GraphQL, aimed at developers not fleets; rate limited | REST Machines API built for "one machine per customer"; create in ~1 s | SDK/CloudFormation; robust; slower (30 to 90 s task start with a large image) | declarative apply via API; robust | custom |
| Volume | 1 per service, backups add-on, host-pinned | 1 per machine, host-pinned, scheduled snapshots with configurable retention | EFS: multi-AZ, durable, AWS Backup, per-tenant access points | PVC with CSI snapshots; regional disks available | local disk |
| Private network | yes | yes (6PN WireGuard mesh) | yes (VPC, Cloud Map) | yes (cluster network, NetworkPolicy) | localhost |
| Stop / wake | app sleeping, not API-driven per tenant | `stop`/`start` API, auto-stop on idle, wake in ~1 to 3 s | stop/run task; slow wake | scale replicas 0/1; 20 to 60 s wake | n/a |
| Idle cost per tenant | always-on container | stopped machine costs volume only (cents) | always-on task or slow wake | pod cost only while running | very low |
| Ops burden | very low | low | medium (VPC, IAM, ECS, EFS, RDS) | medium (Autopilot/Fargate reduce it) | high at scale, single blast radius |
| Compliance posture | limited | moderate | strongest (AWS controls, regions) | strong | weak |
| Fit for hundreds of tenants | cluttered project; API limits | designed for it | yes | yes | no |

### Recommendation

Use **Fly.io Machines** as the first provider, with a **Kubernetes adapter as
the documented second tier** for compliance-driven customers or when the
fleet outgrows Fly's per-organization limits. Keep the Railway adapter out of
the first milestone; it can be added later if ever needed because the
interface is unchanged.

Why Fly first:

- The Machines API is the only candidate designed around exactly this shape:
  one microVM per customer, created and started programmatically, with
  per-machine volumes and a private network, and idempotent creation by
  machine name.
- Stop/start is a first-class, fast API. Combined with the ATLAS gateway,
  idle tenants cost almost nothing (volume only) and wake in seconds when a
  Slack event arrives. This is the single largest cost lever for a
  mostly-idle agent fleet and it makes per-tenant pricing viable.
- MicroVM isolation is stronger than plain containers without operating
  Kubernetes.
- Volume snapshots exist; the host-pinned durability weakness is covered by
  the platform-independent workspace backup below.

Why not the others first:

- Railway: fine for one Jarvis, weak for a programmatically managed fleet
  (developer-oriented API, no per-tenant stop/start, container isolation).
- ECS Fargate + EFS: the most durable option and the right answer if a
  customer requires AWS-only residency, but wake latency and setup cost make
  it a poor pilot platform. It is the natural second provider if the
  Kubernetes route is not chosen.
- Kubernetes: the right long-term fleet substrate, but premature ops for a
  pilot. The provisioner interface maps directly (Deployment + PVC +
  NetworkPolicy + Service per tenant), so it is a later adapter, not a
  redesign.
- Single host: Jarvis's own `hostd/` proves the pattern but a single host is
  a single blast radius and a capacity ceiling.

### Provider-independent decisions that make the fleet run smoothly

These apply regardless of provider and are part of the first milestone:

1. **Wake-on-event.** Runtime status gains `stopped` (idle, volume retained).
   The gateway queues the delivery, the worker calls `provisioner.resume()`,
   waits for `/health`, then delivers. An idle timer (default 30 minutes with
   no inbound events, no active runs, and no due scheduled tasks reported by
   the runtime) stops the machine. Tenants with scheduled tasks or heartbeat
   features enabled are marked `alwaysOn` and never stopped.
2. **Workspace backup independent of the provider volume.** The runtime image
   includes a small backup sidecar (or ATLAS triggers it through the admin
   API) that snapshots the workspace to per-tenant, server-side-encrypted
   object storage every hour and before every image upgrade. Restore is an
   operator action. This removes dependence on any provider's volume
   durability and makes provider migration a copy.
3. **Digest-pinned images and per-tenant release** (ADR-0011) so upgrades are
   controlled and reversible.
4. **Resource classes** (`small`, `standard`, `large`) mapped per provider,
   attached to the subscription entitlements and priced through the credit
   system (ADR-0014).
5. **Control plane on managed infrastructure**: ATLAS runs as at least two
   replicas with a managed PostgreSQL (provider-managed or a dedicated
   PostgreSQL vendor with point-in-time recovery). Because ATLAS is in the
   Slack, tool, and model paths, its availability target is higher than any
   single runtime's.
6. **Regions**: one region for the pilot; `runtime_instances.region` is
   already present so a second region is a configuration addition.

## Consequences

- ADR-0009's Railway specifics become the description of an optional
  adapter. `docs/architecture.md` and `docs/operations.md` reference Fly
  Machines for v1.
- Jarvis change J6 (published image) remains a prerequisite; the image must
  also start quickly for wake-on-event to be pleasant, so image size and
  Chrome start-up are tracked as a Jarvis performance item (J8).
- The gateway and worker gain the wake path and idle timer; the state machine
  gains `stopped`.

## Alternatives rejected

- Serverless containers without volumes (Cloud Run, Cloudflare Containers):
  Jarvis's workspace is a filesystem with append-heavy logs; object-storage
  FUSE mounts are not a good fit.
- Agent sandbox vendors (ephemeral sandbox APIs): built for short-lived
  execution, not a persistent, privately addressable service per customer.
