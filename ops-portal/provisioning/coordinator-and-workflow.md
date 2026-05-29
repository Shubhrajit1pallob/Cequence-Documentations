# Coordinator and Workflow

The singleton coordinator, where the job list lives, and how job transitions work.

---

## Where the Job List Lives

```
app/_lib/tenant/provisioning/TenantProvisionCoordinator.ts:29
  PROVISION_TENANT_WORKFLOW.jobs = [ job1, job2, ... ]
```

Array order = execution order. To add or reorder jobs, change this array.

---

## Singleton Design

`TenantProvisionCoordinator.getInstance()` returns **one instance for the whole server** — not one per tenant.

That single instance handles every tenant currently provisioning simultaneously.

**Why singleton?** Without it, multiple `setInterval` instances would race, causing duplicate GitLab pipelines and duplicate Slack messages for the same tenant.

**Coordinator is stateless about tenants.** No tenant fields, no tenant list in memory. All tenant state lives in MongoDB on each tenant's document (`provisionTenantStatus` field). This makes the coordinator restart-safe: the next tick re-queries Mongo and picks up where it left off.

---

## Polling Loop

```
Coordinator.ts:663  startIntervalWatcher()
  setInterval → checkPendingJobs()
  interval = PROVISIONER_WATCHER_UPDATE_INTERVAL env var
```

---

## Per-Tick Logic (`checkPendingJobs`, `Coordinator.ts:350`)

```
1. if isRunning → skip tick (re-entrancy guard)
2. getPendingJobs() → Mongo query: tenants with any non-terminal job
3. for each tenant:
     walk workflow.jobs in order → find first non-terminal job
     apply state machine (see below)
4. advance one job per tenant per tick → move to next tenant
```

### State Machine

| Current state | Condition | Action |
|---|---|---|
| `PENDING` | prev job is `SUCCESS` or `SKIPPED` | call `job.start(previousJobRes)` → persist result |
| `RUNNING` | — | check 2h timeout (PLAT-1441) → call `job.status(jobId)` → persist |
| `SUCCESS` | last job in workflow | mark workflow `SUCCESS` |
| `SUCCESS` | not last job | advance to next job |
| `SKIPPED` | — | same as `SUCCESS` |
| `FAILURE` | — | mark workflow `FAILURE`; stop processing this tenant |

---

## `previousJobRes` Handoff (`Coordinator.ts:593`)

Coordinator calls `getPreviousJobResp()` before starting each job and passes the prior job's `JobStatus` into `job.start(previousJobRes)`.

This is how the commit SHA flows from `CreateCICDEntryJob` into `WaitForClusterPipelineJob.start()`.

---

## `isRunning` Guard

Prevents two overlapping ticks from racing on slow external API calls (GitLab, ArgoCD). Set to `true` at the start of each tick, cleared when done.

---

## Workflow Catalog

Each workflow type has its own singleton coordinator:

| Workflow | Coordinator class |
|---|---|
| Provision | `TenantProvisionCoordinator` |
| Destroy | `TenantDestroyCoordinator` |
| Pause | `TenantPauseCoordinator` |
| Resume | `TenantResumeCoordinator` |
| UAP minor upgrade | `UapMinorUpgradeCoordinator` |
| EKS cluster upgrade | `TenantEksClusterUpdateCoordinator` |

All share the same base `Coordinator.ts` engine — same polling loop, same state machine, different job arrays.

---

## TL;DR

One coordinator instance per workflow type handles all tenants. All state lives in Mongo — coordinator is purely a runner. Each tick: re-query Mongo → walk each tenant's job list → apply state machine → advance one job → repeat. The `previousJobRes` field is the only inter-job data channel.
