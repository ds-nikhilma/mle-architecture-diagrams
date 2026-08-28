# MLE Architecture Diagrams

Interactive architecture diagrams for every service family owned by the **MLE team**
in [`DS-DP-S/lightning-core`](https://github.com/DS-DP-S/lightning-core), plus a
reference set for the `postgres-lro` work-queue framework that most of them ride on.

> **Internal.** These diagrams cite real service names, container images, node pools,
> Datadog opdoc IDs and Kubernetes overlay paths. Keep this repository private.

## Opening the diagrams

Every `architecture.html` is a **self-contained, single-file** interactive diagram —
no build step, no server, no network access. Double-click it, or:

```sh
open */architecture.html            # all of them
open cbct-segmentation/architecture.html   # just one
```

Each diagram has guided views (top-right), hover states on every component, and
source citations that point at the exact file and line in `lightning-core`.

## What is here

### The shared spine

| Folder | Diagrams | What it covers |
| --- | --- | --- |
| [`postgres-lro/`](postgres-lro/) | architecture, sequence, lifecycle | The claim-based work queue on PostgreSQL that the MLE fleet is built on — `FOR UPDATE SKIP LOCKED`, the active/inactive table split, claim renewal, and the work-item state machine |

### The eleven MLE families

| Family | System | Shape |
| --- | --- | --- |
| [`ai-scan-orientation`](ai-scan-orientation/) | moa | LRO front + worker + Python sidecar — the canonical shape |
| [`cbct-panoramic-curve-proposal`](cbct-panoramic-curve-proposal/) | dia | LRO front + worker + Python sidecar |
| [`di-segmentation`](di-segmentation/) | dia | LRO front + worker + Python sidecar |
| [`margin-line-detection`](margin-line-detection/) | rst | LRO front + worker + Python sidecar |
| [`parl-detection`](parl-detection/) | dia | LRO front + worker + Python sidecar |
| [`tooth-recognition`](tooth-recognition/) | dia | LRO front + worker + Python sidecar |
| [`parl`](parl/) | dia | Cross-service orchestrator — composes two downstream LRO families, no sidecar |
| [`cbct-anatomy-segmentation`](cbct-anatomy-segmentation/) | dia | Two-tier LRO chain — a full LRO family stacked on another |
| [`cbct-segmentation`](cbct-segmentation/) | dia | Hand-rolled front + GPU worker + dedicated storage tier |
| [`tooth-modeling`](tooth-modeling/) | tmo | Broker + worker + storage + a non-LRO external adapter |
| [`mle-pentest-worker`](mle-pentest-worker/) | pen | Parked no-op pod that hosts a sidecar for security testing |

Ownership was resolved from two independent sources that agree exactly: the
`@DS-DP-S/mle` rows in `CODEOWNERS`, and `owner: mle` in each service's Backstage
`catalog-info.yaml`. That is 26 services across the 11 families above.

See [`FINDINGS.md`](FINDINGS.md) for what reading all 26 services turned up.

## Regenerating

Diagrams are generated with [archify](https://github.com/ds-nikhilma/archify) from the
`architecture.json` spec next to each HTML file. The JSON is the source of truth; the
HTML is a build artifact, committed so the repo is useful without the toolchain.

```sh
A=/path/to/archify/bin/archify.mjs
R=/path/to/lightning-core

cd cbct-segmentation
node $A validate architecture architecture.json --repo-root $R
node $A deliver  architecture architecture.json architecture.html \
     --quality showcase --repo-root $R
```

Notes that will save you time:

- `--repo-root` is **required** on both verbs whenever a spec declares component
  `sources`. It verifies every cited path actually exists.
- The output path is **positional**, not a `--out` flag.
- `--quality showcase` enforces all nine artifact checks. Every diagram here passes
  with zero errors and zero warnings.
- A dirty `lightning-core` working tree is fine — the evidence check verifies paths
  exist, not that the tree is clean.

All specs are pinned to `lightning-core` revision
`3318998a9a4d6618f053d776a1f2971402fd5d3a`. Re-verify the citations before trusting a
diagram against a much later HEAD.
