# Provisioning Jobs

What a "job" is — the two flavors and how they hand off to each other.

**Job** = one class, one discrete step.
**Workflow** = ordered list of jobs.
**Coordinator** = the loop that runs them.

---

## Two Job Interfaces

### InstantJob

```typescript
// app/_lib/tenant/provisioning/InstantJob.ts
interface InstantJob {
  start(previousJobRes?: JobStatus): Promise<JobStatus>
}
```

One method. Does work synchronously (or awaits a fast async call). Returns `SUCCESS` or `FAILURE` immediately. Never returns `RUNNING`.

### DelayedJob

```typescript
// app/_lib/tenant/provisioning/DelayedJob.ts
interface DelayedJob extends InstantJob {
  status(jobId: string): Promise<JobStatus>
}
```

`start()` kicks something off externally and returns `RUNNING` + a `jobId` to track it.
Coordinator calls `status(jobId)` each tick until terminal (`SUCCESS` or `FAILURE`).

---

## JobStatus Shape

```typescript
{
  jobId:     string          // commit SHA, pipeline ID, or "tenantId/appName" for Argo
  state:     'PENDING' | 'RUNNING' | 'SUCCESS' | 'FAILURE' | 'SKIPPED'
  message?:  string
  link?:     string          // e.g. GitLab pipeline URL
  startedAt?: Date
  endedAt?:   Date
}
```

---

## Concrete Examples

### InstantJob — `CreateCICDEntryJob`

**Location:** `app/_lib/tenant/provisioning/create/CreateCICDEntryJob.ts`

1. Edits `.tenants.yml` in `cicd-templates` Git repo
2. Commits → gets commit SHA
3. Pushes → triggers GitLab CI pipeline automatically
4. Returns `{ jobId: commitSHA, state: SUCCESS }`

`jobId` = commit SHA — the next job uses this to find the GitLab pipeline.

### DelayedJob — `WaitForClusterPipelineJob`

**Location:** subclass of `app/_lib/tenant/provisioning/create/WaitForGitlabPipelineJob.ts`

The subclass just passes `Repositories.cluster` to the parent — **subclass = configuration, not logic.**

- `start(previousJobRes)`: takes SHA from prior job → finds GitLab pipeline by SHA → returns `{ jobId: pipelineId, state: RUNNING }`
- `status(jobId)`: polls GitLab pipeline status each tick until `SUCCESS` or `FAILURE`

---

## SHA Handoff Pattern

```
CreateCICDEntryJob.start()
  → returns { jobId: "abc123", state: SUCCESS }   ← SHA

WaitForClusterPipelineJob.start(previousJobRes)
  → uses previousJobRes.jobId (SHA) to find pipeline
  → returns { jobId: "456789", state: RUNNING }   ← pipeline ID

Coordinator polls WaitForClusterPipelineJob.status("456789")
  → each tick: GET /projects/:id/pipelines/456789
  → eventually returns SUCCESS
  → next job starts
```

---

## TL;DR

`InstantJob` = do it now, return `SUCCESS`/`FAILURE`. `DelayedJob` = start it, return `RUNNING`, get polled until done. The `jobId` field is the handoff mechanism — a commit SHA flows out of a create-job and into the next wait-job's `start()`.
