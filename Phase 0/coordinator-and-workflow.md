# Workflow Definition

## Where the list lives

The list lives in a single constant: `PROVISION_TENANT_WORKFLOW` in `app/_lib/tenant/provisioning/TenantProvisionCoordinator.ts:29`.

It's just a plain object that conforms to the `Workflow` type:

```ts
export const PROVISION_TENANT_WORKFLOW: Workflow = {
  name: provisionTenantWorkflowName,
  displayName: 'Provision Tenant',
  sendNotifications: true,
  emoji: ':new:',
  jobs: [
    createWorkflowEntry(CreateCICDEntryJob),                //  1
    createWorkflowEntry(CreateClusterEntryJob),             //  2
    createWorkflowEntry(WaitForClusterPipelineJob),         //  3
    createWorkflowEntry(DownloadKubeConfigJob),             //  4
    createWorkflowEntry(CreateClusterServicesEntryJob),     //  5
    createWorkflowEntry(WaitForClusterServicesPipelineJob), //  6
    createWorkflowEntry(CreateTenantEntryJob),              //  7
    createWorkflowEntry(WaitForTenantPipelineJob),          //  8
    createWorkflowEntry(TriggerDexPipelineJob),             //  9
    createWorkflowEntry(CreateTenantApplicationEntryJob),   // 10
    createWorkflowEntry(SyncArgoParentApplicationJob),      // 11
    createWorkflowEntry(CreateDatadogDowntimeJob),          // 12
    createWorkflowEntry(CreateGrafanaSilenceTrafficJob),    // 13
    createWorkflowEntry(SyncArgoGrafanaJob),                // 14
    createWorkflowEntry(SyncArgoCequenceAspJob),            // 15
    createWorkflowEntry(CreateDefenderMappingFilesJob),     // 16
    createWorkflowEntry(CreateDefenderPoolEntryJob),        // 17
    createWorkflowEntry(WaitForDefenderPoolPipelineJob),    // 18
    createWorkflowEntry(SyncArgoDefenderPoolJob),           // 19
    createWorkflowEntry(SyncTenantDagsPipelineJob),         // 20
    createWorkflowEntry(WaitForUapApplicationHealthJob),    // 21
    createWorkflowEntry(TriggerPauseTenantWorkflow),        // 22
    createWorkflowEntry(SyncTenantFromSourcesJob),          // 23
  ],
  audit: {
    action: AuditActions.PROVISION,
    target: AuditTargets.TENANT,
  },
}
```

That `jobs: [...]` array **is** the workflow. Add a job → drop a new line in here. Reorder → change the order in this array. That's it.

---

## How it gets wired to the coordinator

Right below it in the same file:

```ts
export class TenantProvisionCoordinator extends Coordinator {
  private static instance: TenantProvisionCoordinator | null = null

  private constructor() {
    super(PROVISION_TENANT_WORKFLOW, 'ProvisionTenantCoordinator')
    //    ^^^^^^^^^^^^^^^^^^^^^^^^
    //    The job list is handed to the base Coordinator here
  }
  ...
}
```

The base `Coordinator` constructor (`Coordinator.ts:338`) stores it:

```ts
protected constructor(workflow: Workflow, coordinatorName: string) {
  this.workflow = workflow
  ...
}
```

Now `this.workflow.jobs` is the array the coordinator walks on every tick.

---

## What `createWorkflowEntry` does

It's a tiny helper that wraps a job class into a `WorkflowJob` object — basically just `{ class, name }` — so the coordinator can do `new workflowJob.class(tenant)` later. Defined in `app/_lib/utils.ts`. Nothing magical.

---

## Where the coordinator uses the list

Three key spots in `Coordinator.ts`, all referencing `this.workflow.jobs`:

| Location | What it does |
|---|---|
| `checkPendingJobs` (line 366) | `for (const workflowJob of this.workflow.jobs)` — walks the list in order on each tick |
| `getPreviousJob` (line 420) | `this.workflow.jobs.findIndex(...)` — finds the job before the current one |
| `isPreviousJobComplete` (line 453) | `this.workflow.jobs.findIndex(...)` — checks no earlier job failed |

So the entire engine has one array to look at: `this.workflow.jobs`. **The order in that array literally is the order of execution.**

---

## Does the coordinator track every tenant?

Yes — but indirectly. The coordinator doesn't hold per-tenant state in memory. Every tick it calls:

```ts
getPendingJobs() → Tenant.getTenantsWithPendingWorkflowJobs(this.workflow.name)
```

This is a MongoDB query that finds all tenants whose `provisionTenantStatus` field still has at least one non-terminal job. The coordinator then loops over that result set and advances each one.

So the answer to "does the coordinator track all of them?" is:

- It tracks all of them, yes — anything pending shows up in `getPendingJobs()`.
- But it **doesn't hold them in memory** — state lives in MongoDB on the tenant document. Each tick re-queries.

This is what makes it durable: restart the server and the next tick picks up exactly where it left off.

---

## Mental model — putting it together

```
┌─────────────────────────────────────────────────────────────┐
│ TenantProvisionCoordinator.ts                               │
│                                                             │
│  PROVISION_TENANT_WORKFLOW = {                              │
│    jobs: [ Job1, Job2, ..., Job23 ]   ← the recipe          │
│  }                                                          │
│                                                             │
│  class TenantProvisionCoordinator extends Coordinator {     │
│    constructor() { super(PROVISION_TENANT_WORKFLOW, ...) }  │
│  }                                                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ Coordinator.ts (base class)                                 │
│                                                             │
│  this.workflow.jobs ← stored reference to the recipe        │
│                                                             │
│  every tick:                                                │
│    tenants = getPendingJobs()    ← MongoDB query            │
│    for each tenant:                                         │
│      for job of this.workflow.jobs:                         │
│        apply state machine                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Other workflows, same pattern

Each workflow is defined the same way — a `WORKFLOW` constant + a singleton coordinator class:

| Workflow | File | Constant |
|---|---|---|
| Provision | `TenantProvisionCoordinator.ts` | `PROVISION_TENANT_WORKFLOW` |
| Destroy | `TenantDestroyCoordinator.ts` (likely) | `DESTROY_TENANT_WORKFLOW` |
| Pause | `TenantPauseCoordinator.ts` | `PAUSE_TENANT_WORKFLOW` |
| Resume | `TenantResumeCoordinator.ts` | `RESUME_TENANT_WORKFLOW` |
| UAP upgrade | `UapMinorUpgradeCoordinator.ts` | `UAP_MINOR_UPGRADE_WORKFLOW` |
| EKS cluster upgrade | `TenantEksClusterUpdateCoordinator.ts` | `TENANT_EKS_CLUSTER_UPDATE_WORKFLOW` |

Same machinery, different `jobs: [...]` arrays.

---

## TL;DR

> The job list lives in one place: the `PROVISION_TENANT_WORKFLOW.jobs` array in `app/_lib/tenant/provisioning/TenantProvisionCoordinator.ts:29`. The coordinator constructor passes that workflow into the base `Coordinator`, which stores it as `this.workflow` and walks `this.workflow.jobs` in order on every tick. Each tick also re-queries MongoDB to find every tenant that still has pending work, so the coordinator effectively tracks all tenants without holding any of them in memory.
