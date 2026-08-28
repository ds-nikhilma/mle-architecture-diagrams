# Fleet Findings

Observations from reading all 26 MLE-owned services in `lightning-core` at revision
`3318998a9a4d6618f053d776a1f2971402fd5d3a` while authoring the diagrams in this repo.

These are observations, not tickets. Some are real drift worth fixing, some are just
worth knowing, and a couple are things that turned out to be correct once understood.

## Resource declarations

**1. Three worker Deployments declare no `resources` block at all.**
`parl-detection-worker`, `tooth-recognition-worker` and `parl-worker` ship Go
containers with neither requests nor limits, so they are unbounded on the node and
land in the BestEffort QoS class — first to be evicted under node pressure, and
invisible to the scheduler's capacity maths.

**2. Two sidecars set a CPU limit but no memory limit while requesting 25Gi.**
`parl-detection-worker` and `cbct-segmentation-worker` both cap CPU at 5 and request
25Gi of memory with no corresponding memory limit. A leak in the Python process is
bounded only by the node.

## Image pinning

**3. Four different sidecar pinning conventions coexist.**

| Convention | Example | Used by |
| --- | --- | --- |
| Build-time placeholder | `compute-img` | 5 families |
| Date tag | `:20250717`, `:20250401`, `:20260513` | several |
| Tag + digest | `:20250204@sha256:af1a512a…` | `cbct-segmentation-worker` |
| Build SHA | `v-3400-23fa9abce` | `tooth-modeling-worker` |

**4. `cbct-segmentation-worker` is the only digest-pinned sidecar** — and it is the
one the others should probably copy. A date tag is mutable; a digest is not.

**5. Crossed naming between container and image.** The container named
`cbct-segmentation-compute` in `cbct-segmentation-worker` actually runs the
`cbctai-anatomy-segmentation-compute` image. The same image appears in
`cbct-anatomy-segmentation-worker` at a different tag (`:20260513` there,
`:20250204@sha256:…` here), so the two families run different builds of the same
compute code under names that suggest otherwise.

## Scheduling

**6. MLE runs two node pools.** The default pool takes
`nodeSelector: {node_pool: mle-node-pool}` with a `nodeType=mle:NoSchedule`
toleration. The GPU pool (`nodeType=mle_gpu` plus `nvidia.com/gpu=present`) is used
by exactly one workload, `cbct-segmentation-worker`, which is why that Deployment
alone uses `strategy: Recreate` — it cannot briefly double up on a scarce GPU.

## Monitoring

**7. Two entire families have no Datadog monitors.** All four `tooth-modeling*`
services and `mle-pentest-worker` appear in none of the maps in
`infrastructure/platform/common/mle/constants.tf`. Notably
`tooth-modeling-storage` is unmonitored while the equivalent
`cbct-segmentation-storage` carries opdoc DIA-011 — so the gap is not a deliberate
policy about storage tiers.

**8. Three monitor keys have no matching service directory.** `constants.tf` still
carries ene-era entries (`cbctai-panoramic-curve-proposal*`,
`margin-line-detection-job`) that point at nothing in `lightning-core`.

**9. Three `CODEOWNERS` rows are stale.** 29 rows match `@DS-DP-S/mle`; 26 resolve
to a directory that exists.

Monitoring is entirely `for_each`-driven from `constants.tf` — adding a monitor
means editing that one file, not `monitors.tf`.

## `mle-pentest-worker`

**10. It pulls a compute image from the `tap` artifactory namespace.** Every other
MLE sidecar comes from the MLE namespace; this one carries
`di-comparison-compute:v1-1c23fcbe34` from `dpns-tap-docker-local-dev`. Worth
confirming as intentional rather than a copy-paste.

**11. Its s2 overlay can publish the Python sidecar straight to the public Istio
gateway — and it is correctly guarded.** Two independent opt-in catches stand
between the sidecar and the internet, each with an explicit operator comment saying
to revert it:

- the Deployment sits at `replicas: 0`, and
- the `VirtualService` is commented out of the s2 `kustomization.yaml`.

Both must be reverted by hand, so nothing is exposed unless someone deliberately
exposed it. This is the design working as intended. The only follow-up worth having
is a periodic audit that both catches are still engaged after a test — not a change
to the mechanism.

**12. It uses a seventh startup shape.** Across the fleet, `core-packages-go/setup`
gets used through two different constructors — `setup.GrpcServiceConfig` /
`NewGrpcService` here, `setup.Config` in the storage tiers — with the LRO families
going through `lro.SetupHandler` instead. Seven distinct startup shapes appear
across 26 services.

## The seven startup shapes

For reference, since it took reading everything to see the pattern:

1. **Plain LRO** — `lro.SetupHandler` front + worker + Python sidecar (6 families)
2. **Hand-rolled front** — own config, auth and MQ wiring, with an AliCloud branch
   (`cbct-segmentation-lro`, `tooth-modeling-broker`)
3. **Two-tier LRO chain** — a full LRO family stacked on another
   (`cbct-anatomy-segmentation`)
4. **Cross-service orchestrator** — composes downstream LRO families, no sidecar
   (`parl-worker`, `cbct-anatomy-segmentation-worker`)
5. **Storage tier** — `setup.Config`, dual auth, no LRO at all
   (`cbct-segmentation-storage`, `tooth-modeling-storage`)
6. **External adapter** — no LRO, dials services outside the repo (`tooth-modeling`)
7. **No-op skeleton** — parked at `replicas: 0`, exists to host a sidecar
   (`mle-pentest-worker`)

Two LRO spines also coexist: nine workers use the external
`github.com/DS-DP-S/postgres-lro` module, while `cbct-segmentation-worker` and
`tooth-modeling-worker` use the in-repo `pkg/worker-lro`.
