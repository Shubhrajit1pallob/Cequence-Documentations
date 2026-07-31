# Failure & Retry Semantics

What happens when a job fails and how operators recover from it.

---

## What Happens on Job Failure

When `start()` or `status()` returns `FAILURE`, the coordinator's `transitionStatus()` does three things immediately:

1. Persists `{ state: FAILURE, message: "..." }` to MongoDB on the tenant document
2. Updates the Slack summary message in place (thread turns red)
3. Broadcasts two Slack messages into the thread:
   - **Alert:** "Job X failed for tenant Y" with investigation link
   - **Detail:** full error message

Workflow is marked `FAILURE` and the coordinator stops processing that tenant. No further jobs run.

---

## Three Operator Actions

All handled by server actions in `app/tenants/[tenant]/actions.ts`.

### 1. RETRY — `handleJobAction(..., State.PENDING)` — `actions.ts:523`

- Validates current job state is `FAILURE`, `RUNNING`, or `PENDING`
- Writes `{ state: PENDING, jobId: -1 }` to MongoDB for that job
- Audit log: "Job X retried"
- Next coordinator tick picks it up:
  - job is `PENDING` + `jobId === -1` → `isRetry = true`
  - → `job.start(previousJobRes, isRetry=true)`
  - → instead of looking up old pipeline by stale SHA, explicitly calls `startPipeline([])` → brand new GitLab pipeline created

> `isRetry=true` is the key distinction — normal start looks up a pipeline by commit SHA; retry creates a fresh pipeline because the old SHA/pipeline may be stale or gone.

### 2. SKIP — `handleJobAction(..., State.SKIPPED)` — `actions.ts:523`

- Same function as RETRY, different target state
- Writes `{ state: SKIPPED, jobId: -1 }` to MongoDB
- Next coordinator tick: `SKIPPED` is treated the same as `SUCCESS` → advances to next job
- Used when a job failed for a non-critical reason and the operator wants to continue anyway

> Example: Grafana silence job failed but the tenant can still provision — operator skips it.

### 3. STOP — `stopJob()` — `actions.ts:577`

- Validates current state is `RUNNING`
- Writes `{ state: FAILURE, jobId: -1, message: 'Stopped by user' }`
- Coordinator sees `FAILURE` next tick → marks workflow `FAILURE` → stops
- Used for stuck `RUNNING` jobs — operator force-fails, then can retry or skip

---

## State Guard

`handleJobAction` only allows an action if the current state is `FAILURE`, `RUNNING`, or `PENDING`. Attempts on `SUCCESS` or `SKIPPED` jobs are rejected.

Not a correctness issue — coordinator treats `SUCCESS` and `SKIPPED` identically (both advance to next job) — but a UX safeguard: the action would be pointless and likely indicates a stale UI.

---

## 2-Hour Hard Timeout — PLAT-1441

`Coordinator.ts:316` — `checkJobTimedOut()`

For any `DelayedJob` that records `startedAt`, the coordinator checks on every tick:

```
elapsedMs = now - currentJobStatus.startedAt
if elapsedMs > JOB_TIMEOUT_MS (default 2h, set via env var) → force FAILURE
```

- Skips calling `status()` entirely — just force-sets `FAILURE`
- Message: `"Job timed out after Xh Ym (limit 2h). Investigate via the linked job and retry or skip."`
- Preserves the original job link so the operator can click through to GitLab/Argo to investigate
- Prevents stuck pipelines from holding a tenant in `RUNNING` forever

---

## All Failure Paths — Summary

| Path | How it triggers | What coordinator does |
|---|---|---|
| Job fails naturally | `start()` or `status()` returns `FAILURE` | Persists FAILURE → Slack alert + detail → workflow stops |
| Job times out (PLAT-1441) | `>2h` in `RUNNING` | Force-fails without calling `status()` → same Slack broadcast |
| Operator retries | Sets job to `PENDING` (`jobId=-1`) | Picks up next tick → `isRetry=true` → fresh GitLab pipeline |
| Operator skips | Sets job to `SKIPPED` | Treats as `SUCCESS` → next job starts |
| Operator stops stuck job | Sets job to `FAILURE` ("Stopped by user") | Workflow halts → operator can then retry or skip |

---

## Who Is an Operator?

An internal Cequence platform / DevOps / SRE engineer with access to the ops-portal. Not the customer. When provisioning fails, they get alerted via Slack, open the portal, inspect the failed job, and decide to retry, skip, or investigate.
