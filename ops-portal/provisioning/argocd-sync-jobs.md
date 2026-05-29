# ArgoCD Sync Jobs

How ArgoCD sync jobs work in the provisioning system and why they exist.

---

## Background: What ArgoCD Is

ArgoCD is a GitOps CD tool for Kubernetes. It watches a Git repo and reconciles what's running in K8s with what's in Git. The portal uses it to deploy applications into a provisioned cluster.

---

## Why Explicit Sync (Not Auto-Sync)

ArgoCD has an auto-sync mode that fires every ~3 minutes. The portal does not rely on it:

| Reason | Detail |
|---|---|
| **Sequencing** | The portal must know exactly when each K8s deployment is live before starting the next job. Auto-sync gives no signal back. |
| **Speed** | 5 Argo sync steps × 3 min auto-poll delay = 15+ min of unnecessary idle time. Explicit sync fires immediately. |
| **Reliability** | Explicit sync + poll surfaces failures to Slack. Auto-sync failures are invisible to the portal. |
| **Retry control** | Explicit sync passes `retryStrategy` directly (3 retries, 20s backoff, maxDuration 4m). |

---

## Two Layers

| Layer | Location | Role |
|---|---|---|
| `ArgoCd` | `app/_lib/argocd/argoCd.ts` | Raw API wrapper |
| `SyncArgoCdApplicationJob` | `app/_lib/tenant/provisioning/SyncArgoCdApplicationJob.ts` | `DelayedJob` base class |

---

## ArgoCd Client

**Four SDK clients:** `read/write × dev/prod`

`getArgoCdClientForTenant(tenantId, operation)` picks the right one.

**Application naming:**
```
getTenantApplicationName(tenantId, applicationName)
  → "${tenantId}-${applicationName}"

e.g. tenantId="tenant1-prod", app="tenant"
  → Argo app name = "tenant1-prod-tenant"
```

### Key Methods

| Method | What it does |
|---|---|
| `syncApplication()` | Hard refresh first, then sync with `retryStrategy` + `prune: true` |
| `getApplicationOperationStatus()` | Poll current operation phase |
| `isApplicationMissing()` | Treats 403/404/PermissionDenied as missing _(ArgoCD v2.14+ returns 403 for non-existent apps — security fix GHSA-2q5c-qw9c-fmvq)_ |
| `syncApplicationResources()` | Scoped sync on specific resources — avoids full-app restart |
| `terminateOperation()` | Cancel an in-progress sync |
| `deleteApplication()` | Remove an Argo app |

---

## SyncArgoCdApplicationJob (`DelayedJob`)

### `start()`

```
jobId = "${tenantId}/${applicationName}"   ← string, not numeric like GitLab
calls argo.syncApplication()
returns { jobId, state: RUNNING }
```

### `status(jobId)`

```
parse tenantId + applicationName from jobId
calls argo.getApplicationOperationStatus()
maps OperationStatus → State:
  RUNNING    → RUNNING
  SUCCEEDED  → SUCCESS
  FAILED     → FAILURE
  UNKNOWN    → FAILURE
  undefined  → FAILURE
```

---

## Subclass Pattern

Same "subclass = configuration" pattern as `WaitForGitlabPipelineJob`:

| Subclass | Application |
|---|---|
| `SyncArgoParentApplicationJob` | `TENANT_APPLICATION.TENANT` |
| `SyncArgoGrafanaJob` | `TENANT_APPLICATION.GRAFANA` |
| `SyncArgoCequenceAspJob` | `TENANT_APPLICATION.CEQUENCE_ASP` |
| `SyncArgoDefenderPoolJob` | `TENANT_APPLICATION.DEFENDER` |

Each subclass is just a different application name — same engine.

---

## How It Fits Into Provisioning

```
CreateTenantApplicationEntryJob
  → writes ArgoCD Application manifest to Git

SyncArgoParentApplicationJob.start()
  → POST /api/v1/applications/{name}/sync
  → returns { jobId: "tenant1-prod/tenant", state: RUNNING }

Coordinator polls status() each tick
  → Argo applies K8s resources
  → operationState.phase = Succeeded
  → returns SUCCESS → next job
```

---

## Compared to GitLab Pipeline Jobs

| | GitLab | ArgoCD |
|---|---|---|
| `start()` | Finds existing pipeline by SHA (push triggered it) | Actively calls sync API |
| `jobId` | Numeric pipeline ID | String `"tenantId/appName"` |
| Auth | 1 token | 4 tokens (read/write × dev/prod) |
| Retry | Portal-side (`RETRY_MAX=5`) | Argo-side `retryStrategy` |

---

## Resilience

- `withRetry` (3 attempts, exponential backoff + jitter) on read paths
- Argo-side `retryStrategy` on sync operations
- Hard refresh before every sync (prevents stale manifest issues)
- Dry-run mode returns fake `RUNNING → SUCCESS`

---

## TL;DR

Portal explicitly calls Argo sync API instead of waiting for auto-sync — for sequencing, speed, and observability. `SyncArgoCdApplicationJob` is a `DelayedJob`: `start()` triggers sync and returns `RUNNING`; coordinator polls `status()` until `SUCCEEDED`/`FAILED`. Four subclasses, all same engine — just different application names.
