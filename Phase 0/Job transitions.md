# Job Transitions

## What handles the transitions

Yes, there's a singleton. It's `TenantProvisionCoordinator`, defined in `app/_lib/tenant/provisioning/TenantProvisionCoordinator.ts:65`:

```ts
export class TenantProvisionCoordinator extends Coordinator {
  private static instance: TenantProvisionCoordinator | null = null

  private constructor() {
    super(PROVISION_TENANT_WORKFLOW, 'ProvisionTenantCoordinator')
  }

  public static getInstance(): TenantProvisionCoordinator {
    if (!TenantProvisionCoordinator.instance) {
      TenantProvisionCoordinator.instance = new TenantProvisionCoordinator()
    }
    return TenantProvisionCoordinator.instance
  }
}
```

Classic singleton pattern: private constructor + static `getInstance()`. The default export at the bottom of the file (`export default TenantProvisionCoordinator.getInstance()`) ensures only one instance exists per Node.js process.

Each workflow has its own singleton coordinator (provisioning, destroy, pause, resume, UAP upgrade, etc.) — same base class `Coordinator`, different workflow definitions.

---

## How a transition actually happens

The transitions are driven by a **polling loop**, not by jobs calling each other. Here's the mechanism step by step.

### 1. Boot — start the watcher

When the Next.js server starts, each singleton coordinator calls `startIntervalWatcher()` (`Coordinator.ts:663`):

```ts
public async startIntervalWatcher() {
  setInterval(() => this.checkPendingJobs(), parseInt(GetEnvironment().provisionerWatcherUpdateInterval))
}
```

This sets up a `setInterval` that calls `checkPendingJobs` every N milliseconds (configurable via env var). That's the heartbeat of the whole system.

### 2. Every tick — `checkPendingJobs`

`Coordinator.ts:350`. On each tick:

- `getPendingJobs()` queries MongoDB for tenants that have at least one non-terminal job in this coordinator's workflow.
- For each such tenant, the coordinator walks the workflow's job list in order, looking at the persisted state of each job.
- It only acts on the **first** job that needs attention (then breaks out — does not advance two jobs in one tick).

### 3. The state machine for each job

This is the actual transition logic — a switch on the job's current state (`Coordinator.ts:372`):

```
PENDING   → if previous job is SUCCESS/SKIPPED:
              call job.start() (or job.start(previousJobRes))
              new state becomes whatever start() returned (RUNNING / SUCCESS / FAILURE)
              persist; stop processing this tenant this tick

RUNNING   → (only DelayedJobs reach this)
              check 2h timeout (PLAT-1441) — force-fail if expired
              call job.status(jobId)
              if RUNNING → leave it; stop processing this tenant this tick
              if SUCCESS/FAILURE → persist; stop this tick (next tick advances)

SUCCESS   → if last job in workflow, mark workflow SUCCESS
              else continue to look at the next job

SKIPPED   → same as SUCCESS — continue

FAILURE   → mark workflow FAILURE; break out of this tenant's processing
```

### 4. The `previousJobRes` handoff

When `start()` is called on a `PENDING` job, the coordinator looks up the previous job's persisted `JobStatus` and passes it in (`Coordinator.ts:593`):

```ts
const previousJobRes = await this.getPreviousJobResp(workflowJob.name, tenant, this.workflow.name)
jobResponse = await job.start(previousJobRes, isRetry)
```

This is how the SHA flows from `CreateCICDEntryJob` → `WaitForClusterPipelineJob`. The coordinator is the wiring; **jobs never call each other directly**.

### 5. Where the transition gets recorded

After `start()` or `status()` returns, `transitionStatus()` (`Coordinator.ts:667`) writes the new `JobStatus` back to MongoDB on the tenant document, updates the Slack summary message, and broadcasts on failure/success.

### 6. The `isRunning` guard

Inside `checkPendingJobs`:

```ts
if (this.isRunning) return
try {
  this.isRunning = true
  ...
} finally {
  this.isRunning = false
}
```

This prevents two overlapping ticks from racing if the previous one is still in flight (e.g., a slow GitLab API call).

---

## Mental model

```
                ┌──────────────────────────────────────────┐
                │  TenantProvisionCoordinator (singleton)  │
                │   setInterval → checkPendingJobs() ◀─────┼──── heartbeat
                └──────────────┬───────────────────────────┘
                               │ every tick
                               ▼
            ┌─────────────────────────────────────┐
            │  For each tenant with pending work: │
            │                                     │
            │  Walk workflow.jobs in order        │
            │  Find first non-terminal job        │
            │  Apply state machine:               │
            │    PENDING → start()                │
            │    RUNNING → status()               │
            │    SUCCESS/SKIPPED → next job       │
            │    FAILURE → stop, mark workflow    │
            │                                     │
            │  Persist JobStatus → MongoDB        │
            └─────────────────────────────────────┘
```

So the transition from "CI/CD record" → "Cluster row" → "Wait for cluster pipeline" → "Download kubeconfig" → ... is **not** the jobs handing off to each other. It's the coordinator looking at the persisted state on each tick and deciding "previous job is `SUCCESS`, this one is `PENDING`, time to start it."

---

## Why this design?

Because most jobs are long-running external pipelines (minutes to hours). A synchronous chain wouldn't survive a server restart. By persisting state to MongoDB and driving it from a polling loop, the workflow is **durable and resumable** — the server can crash mid-provision, come back up, and the next tick picks up exactly where it left off.
