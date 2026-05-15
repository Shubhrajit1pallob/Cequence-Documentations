# GitLab Pipeline Trigger + Tracking

## The two-layer setup

GitLab integration is split into two layers:

1. **`GitlabApi`** — `app/_lib/GitlabApi.ts`
   A thin wrapper around the `@gitbeaker/rest` GitLab SDK. Talks raw HTTP to `gitlab.com`. Knows nothing about workflows or tenants.
2. **`GitlabPipelineJob`** — `app/_lib/tenant/provisioning/destroy/GitlabPipelineJob.ts`
   The provisioning-aware base class. Wraps `GitlabApi` and translates GitLab's responses into the portal's `JobStatus`/`State` enum. Every "wait for pipeline" job extends this class.

> **Note on the path:** `GitlabPipelineJob.ts` happens to live in the `destroy/` folder, but it's used by both create AND destroy jobs. It's a base class, not destroy-specific.

---

## How a pipeline gets triggered — two paths

This is the part that often confuses people. There are two completely different ways a pipeline gets started in this codebase.

### Path 1 — Implicit trigger via git push (the common path)

This is what the provisioning workflow normally does:

```
InstantJob (e.g., CreateCICDEntryJob)
  ├─ Edits a file in a Git repo
  ├─ Commits it → gets a commit SHA
  └─ Pushes the branch         ← GitLab CI auto-triggers a pipeline on this push
              ↓
              ↓  (the pipeline is now running on GitLab's side, but the portal
              ↓   doesn't know its pipeline ID yet — it only knows the SHA)
              ↓
DelayedJob (e.g., WaitForClusterPipelineJob)
  ├─ start(previousJobRes):
  │    sha = previousJobRes.jobId       ← the SHA from the InstantJob
  │    pipeline = getPipelineByCommit(sha)
  │    → records pipeline.id, returns RUNNING
  └─ status(pipelineId): polled by coordinator until SUCCESS/FAILURE
```

The portal never calls "create pipeline" here — **pushing the commit is the trigger**. The `DelayedJob`'s only job is to *discover* which pipeline that push produced.

`getPipelineByCommit` (`GitlabPipelineJob.ts:124`) does this by calling `GitlabApi.findPipelineByCommit` (`GitlabApi.ts:89`), which uses GitLab's "show commit" endpoint and grabs `last_pipeline.id`.

### Path 2 — Explicit trigger via API (`startPipeline`)

`startPipeline` (`GitlabPipelineJob.ts:91`) calls `GitlabApi.startPipeline` (`GitlabApi.ts:42`), which hits GitLab's `POST /projects/:id/pipeline` endpoint, passing:

- `repositoryDetails.id` — which GitLab project
- `defaultBranch` (e.g., `master`)
- any pipeline variables you want to set

---

## How status is tracked

Once a pipeline ID is known, the coordinator polls `status()` every tick. That flows through:

```
WaitForGitlabPipelineJob.status(pipelineId)
        ↓
GitlabPipelineJob.getStatus(pipelineId)
        ↓
GitlabApi.getPipelineById(projectId, pipelineId)
        ↓
GitLab API: GET /projects/:id/pipelines/:pipelineId
        ↓
response.status (a GitLab string like "running", "success", "failed", ...)
        ↓
getStateFromGitlabStatus()  ← maps string → our State enum
        ↓
returns { state, url, id, sha, artifact } as PipelineStatus
        ↓
becomes the new JobStatus persisted to MongoDB
```

The `getStateFromGitlabStatus` function (`GitlabPipelineJob.ts:18`) is the small but important translation layer. It uses string lists from `GITLAB_API_STATUS` in `constants.ts` to bucket GitLab's many status strings into the four portal states: `FAILURE`, `SUCCESS`, `RUNNING`, `SKIPPED`.

---

## Artifact retrieval

When a pipeline succeeds, some jobs need to read an artifact it produced (e.g., a generated kubeconfig). The repo config can declare `artifactJobName` + `artifactName`, and on `SUCCESS` the base class automatically fetches the artifact via `GitlabApi.getArtifactFromJob` and attaches it to the `JobStatus`. That's how `DownloadKubeConfigJob` gets the new cluster's kubeconfig downstream.

---

## Resilience features worth noting

- **Retry with backoff:** `GitlabApi` methods (`startPipeline`, `getPipelineById`, `findPipelineByCommit`) all recursively retry up to `RETRY_MAX = 5` on error.
- **Dry-run mode:** `isDryRun()` short-circuits everything to return fake pipeline IDs/URLs. `DRY_RUN_HOLD_PIPELINES=true` keeps them in `RUNNING` forever for testing the watcher behavior.
- **Instrumentation:** Every GitLab call is wrapped in `instrumentGitlabRequest(...)` (from `instrumentation.node.ts`) — adds tracing/metrics for observability.

---

## TL;DR

> The portal triggers GitLab pipelines two ways: (1) **implicitly by git push** (the common case — `InstantJob` commits + pushes, then a `DelayedJob` finds the resulting pipeline by SHA), and (2) **explicitly via `POST /pipelines`** (for retries or jobs not tied to a commit). Status tracking is a straight pass-through: poll `GET /pipelines/:id`, translate GitLab's status string to our `State` enum, return it as a `JobStatus`. `GitlabApi` is the raw client; `GitlabPipelineJob` is the workflow-aware base class that every `WaitFor*PipelineJob` extends.
