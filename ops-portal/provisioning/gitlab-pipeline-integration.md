# GitLab Pipeline Integration

How the portal triggers and tracks GitLab CI pipelines.

---

## Two Layers

| Layer | Location | Role |
|---|---|---|
| `GitlabApi` | `app/_lib/GitlabApi.ts` | Raw HTTP wrapper around `@gitbeaker/rest` SDK |
| `GitlabPipelineJob` | `app/_lib/tenant/provisioning/destroy/GitlabPipelineJob.ts` | Workflow-aware `DelayedJob` base class _(lives in `destroy/` but used by both create and destroy jobs)_ |

---

## Two Trigger Paths

### Path 1 — Implicit via Git Push (common path)

```
InstantJob: edit file → commit → push
  → GitLab CI auto-triggers pipeline on push

DelayedJob.start():
  → getPipelineByCommit(commitSHA)
  → returns { jobId: pipelineId, state: RUNNING }

Coordinator: polls status(pipelineId) each tick
```

Used by: `WaitForClusterPipelineJob`, `WaitForClusterServicesPipelineJob`, `WaitForTenantPipelineJob`.

### Path 2 — Explicit via API (retries + non-commit jobs)

```
startPipeline() → POST /projects/:id/pipeline
  → returns { jobId: pipelineId, state: RUNNING }
```

Used for:
- Retries (`isRetry` branch)
- Jobs not tied to a commit (Dex pipeline, defender pool pipeline)
- Pipeline variables: `TENANT`, `ENVIRONMENT`, `TRIGGERED_BY=saas_portal`

---

## Status Tracking

```
status(pipelineId)
  → GitlabApi.getPipelineById()
  → GET /projects/:id/pipelines/:id
  → response.status (GitLab string)
  → getStateFromGitlabStatus()
  → portal State enum
  → returned as JobStatus, persisted to MongoDB
```

GitLab status strings are bucketed into portal states via `GITLAB_API_STATUS` lists in `constants.ts`:

| Portal state | GitLab statuses |
|---|---|
| `RUNNING` | `running`, `pending`, `created`, `waiting_for_resource` |
| `SUCCESS` | `success` |
| `FAILURE` | `failed`, `canceled`, `skipped` |
| `SKIPPED` | _(special cases)_ |

---

## Artifact Retrieval

Repos can declare `artifactJobName` + `artifactName`. On `SUCCESS`:

```
getArtifactFromJob() → attached to PipelineStatus
```

Used by `DownloadKubeConfigJob` to retrieve the cluster `kubeconfig` artifact from the cluster pipeline.

---

## Resilience Features

| Feature | Detail |
|---|---|
| Retry | `GitlabApi` retries up to `RETRY_MAX=5` on error |
| Dry-run | `isDryRun()` short-circuits all calls to fake responses |
| Dry-run hold | `DRY_RUN_HOLD_PIPELINES=true` keeps pipelines in `RUNNING` for watcher testing |
| Instrumentation | `instrumentGitlabRequest()` wraps every call for tracing/metrics |

---

## TL;DR

Two trigger paths: push-implicit (create job commits → GitLab auto-fires → wait job finds pipeline by SHA) and explicit API call (retries, non-commit jobs). Status polling maps GitLab strings to portal `State` enum. Artifacts retrieved on `SUCCESS` for kubeconfig download. All calls go through `GitlabApi` with retries and dry-run support.
