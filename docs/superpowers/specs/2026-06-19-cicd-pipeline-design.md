# CI/CD Pipeline Design

## Overview

A fully in-cluster, event-driven CI/CD pipeline using Argo Events and Argo Workflows. GitHub push webhooks trigger per-service build pipelines that produce Docker images and automatically update the cluster-config repo so Argo CD rolls out the new version.

**Services:** `knodes-backend`, `knodes-backoffice`, `knodes-app`  
**Config repo:** `syzore/WorldKnowledgeGraph-Cluster-Config` (at `/Users/avivprofesorsky/dev/graph-knowledge-game-config/`)  
**Registry:** Self-hosted Docker registry backed by OCI Object Storage at `registry.knodes.app`

---

## Architecture

### Components

| Component | Namespace | Purpose |
|---|---|---|
| Argo Events | `argo-events` | Receives GitHub push webhooks, fires WorkflowTemplate triggers |
| Argo Workflows | `argo-workflows` | Executes build/push/deploy jobs as DAG pods |
| Kaniko | (per-workflow pod) | Builds Docker images without Docker daemon |
| crane | (per-workflow pod) | Pushes pre-built image tar to registry |
| git + yq | (per-workflow pod) | Patches image tag in cluster-config and pushes |

### Pipeline Flow

```
GitHub push
  → webhook POST to https://webhooks.knodes.app/push/<repo-name>
    → Argo Events EventSource (one per repo)
      → Sensor matches repo, extracts commit SHA
        → triggers WorkflowTemplate with parameters
          → Pod 1: clone repo → shared PVC
          → Pod 2: Kaniko build → image tar on PVC
          → Pod 3: crane push → image at registry.knodes.app/<service>:<sha>
          → Pod 4: clone cluster-config → patch image tag → git push
            → Argo CD auto-sync → new pods roll out
```

---

## Pipeline Steps

Each service runs the same 4-pod DAG. Steps are sequential (each waits on the previous).

### Pod 1 — Clone
- Image: `alpine/git`
- Clones the service repo at the triggering commit SHA into a shared PVC at `/workspace`
- Output: source code at `/workspace/src`

### Pod 2 — Build
- Image: `gcr.io/kaniko-project/executor:latest`
- Reads `Dockerfile` from `/workspace/src`
- Builds image, writes OCI image tar to `/workspace/image.tar` (flags: `--no-push --tar-path /workspace/image.tar`)
- Does not push — isolated from registry credentials at this step

### Pod 3 — Push
- Image: `gcr.io/go-containerregistry/crane:latest`
- Reads `/workspace/image.tar`
- Pushes to `registry.knodes.app/<service-name>:<commit-sha>` and also tags as `latest`
- Uses registry credentials from a Kubernetes Secret in `argo-workflows` namespace

### Pod 4 — Deploy
- Image: `alpine/git` + `mikefarah/yq`
- Clones `syzore/WorldKnowledgeGraph-Cluster-Config`
- Patches `infra/<service-name>/values.yaml` key `image.tag` to `<commit-sha>`
- Commits `chore: deploy <service-name>@<commit-sha>` and pushes
- Argo CD detects the commit and auto-syncs within seconds
- Uses a GitHub PAT (repo scope) stored as a SealedSecret

---

## Parameterization

A single `WorkflowTemplate` is shared by all 3 services, parameterized with:

| Parameter | Example |
|---|---|
| `repo-url` | `https://github.com/syzore/knodes-backend.git` |
| `service-name` | `knodes-backend` |
| `commit-sha` | `a3f8c12` |
| `dockerfile-path` | `Dockerfile` (relative to repo root) |

Each service's Sensor passes its own values when triggering the WorkflowTemplate.

---

## Argo Events Setup (per repo)

### EventSource
One `EventSource` per repo, listening for GitHub push events on a dedicated path:
- `knodes-backend` → `/push/backend`
- `knodes-backoffice` → `/push/backoffice`
- `knodes-app` → `/push/app`

All three `EventSource` resources share a single `Service` and `IngressRoute` at `webhooks.knodes.app`.

GitHub webhook secret validated via `X-Hub-Signature-256`. Secret stored as a SealedSecret per repo in `argo-events` namespace.

### Sensor
One `Sensor` per repo. Subscribes to its `EventSource`, extracts `commit.id` from the webhook payload, and triggers the shared `WorkflowTemplate` with the service-specific parameters.

---

## Shared Workspace (PVC)

Each workflow run creates an ephemeral `PersistentVolumeClaim` (`ReadWriteOnce`, 2Gi) that is mounted by all 4 pods. The PVC is deleted on workflow completion (success or failure).

On a single-node k3s cluster, `ReadWriteOnce` works because all pods are scheduled to the same node.

---

## Secrets

| Secret | Namespace | Contains | How stored |
|---|---|---|---|
| `github-webhook-secret-backend` | `argo-events` | GitHub webhook secret for knodes-backend | SealedSecret |
| `github-webhook-secret-backoffice` | `argo-events` | GitHub webhook secret for knodes-backoffice | SealedSecret |
| `github-webhook-secret-app` | `argo-events` | GitHub webhook secret for knodes-app | SealedSecret |
| `registry-push-credentials` | `argo-workflows` | Docker registry username + password | SealedSecret |
| `cluster-config-deploy-key` | `argo-workflows` | GitHub PAT (repo scope on cluster-config repo) | SealedSecret |

---

## Cluster-Config Integration

Each service gets a directory under `infra/` in the cluster-config repo:
- `infra/knodes-backend/` — Kubernetes manifests / Helm values with `image.tag`
- `infra/knodes-backoffice/` — same
- `infra/knodes-app/` — same

An Argo CD `Application` per service watches its `infra/<service>/` path and auto-syncs on change.

---

## Traefik Ingress

A new `IngressRoute` exposes `webhooks.knodes.app` pointing to the Argo Events webhook service in `argo-events` namespace. No TLS termination difference from existing pattern — Let's Encrypt via Traefik ACME.

---

## Argo CD Apps

Two new entries in `apps/`:

| File | Points to |
|---|---|
| `apps/argo-events.yaml` | `infra/argo-events/` |
| `apps/argo-workflows.yaml` | `infra/argo-workflows/` |

Each service deployment app is added when the service's `infra/<service>/` directory is created.

---

## Extensibility

The 4-pod sequential DAG is designed for easy extension:

- **Tests:** Add a `test` pod after `clone`, running in parallel with (future) `lint`. Gate `build` on both passing.
- **Lint:** Add a `lint` pod parallel to `test`, same gating.
- **Security scan:** Add a `scan` pod after `build` (reads image tar), gate `push` on scan passing.
- **Staging deploy:** Add a `deploy-staging` pod before `deploy-prod`, requiring manual approval gate.

None of these require restructuring — they are additive DAG nodes.

---

## Out of Scope

- Multi-environment promotion (staging → prod) — not needed yet
- Image signing / provenance — not needed yet
- Test/lint implementation — barebone pipeline only for now; structure accommodates them later
- Rollback automation — Argo CD supports manual rollback via UI/CLI; auto-rollback not in scope
