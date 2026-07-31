# Pipeline Investigation #004 — life360-saas-prod

**Status: ACTIVE** — orphan confirmed live; 17 replica secrets stuck in an undeletable state

## Overview

| Field | Value |
|---|---|
| Tenant | life360-saas-prod |
| Environment | prod |
| Customer | Life360 (POV) |
| AWS account | 552447114887 |
| Primary region | us-west-1 (defender secret replicated to 30 regions) |
| Created → Deleted | 2026-05-06 00:03 → 2026-05-06 22:52 UTC (~23h) |
| Destroy status | SUCCESS — genuine, clean across all stacks |
| Failure category | #3 — Multi-region secret replica orphan (partial teardown; undeletable end state) |

---

## How found

| Source | Finding |
|---|---|
| Tenant JSON | `destroyStatus.status = SUCCESS`; all pipelines have real jobIds + links; `dailyCosts` steady $0.22/day for 27 days post-deletion |
| AWS Resource Explorer CSV (tag `life360-saas`) | 17 rows, all `secretsmanager:secret`, one identifier `saas/life360-saas/defender-DRhi6w`; no other resource types |
| Pulumi destroy artifacts (5 stacks) | all 0 errors; tenant stack shows the secret deleted with 30 replicas in state |
| Deletion-script audit logs (deletion-log-dmitry, May 21–22) | 17 delete attempts on this secret, 0 succeeded |

---

## Workflow result

| Stack | Pipeline | Result |
|---|---|---|
| dex | https://gitlab.com/cequence/saas/infrastructure/dex/-/pipelines/2505716882 | SUCCESS (apply, 160 same) |
| defender-pool | https://gitlab.com/cequence/saas/infrastructure/tenants/defender-pool/-/pipelines/2505734004 | SUCCESS — 47 deleted, 0 errors |
| tenant | https://gitlab.com/cequence/saas/infrastructure/tenants/tenant/-/pipelines/2505761968 | SUCCESS — 57 deleted, 0 errors |
| cluster-services | https://gitlab.com/cequence/saas/infrastructure/tenants/cluster-services/-/pipelines/2505768765 | SUCCESS — 103 deleted, 0 errors |
| cluster | https://gitlab.com/cequence/saas/infrastructure/tenants/cluster/-/pipelines/2505783193 | SUCCESS — 121 deleted, 0 errors |

All artifacts: `maybeCorrupt: false`, zero error diagnostics. Infra teardown genuinely complete.

---

## Root cause

A clean destroy that cannot fully remove a multi-region secret. The defender secret `saas/life360-saas/defender` was created with 30 regional replicas. On destroy, the primary (us-west-1) was deleted and 13 replicas were removed, but **17 replicas were left behind — with 0 errors reported**. Because the primary is gone, those 17 replicas are now stuck: AWS rejects direct deletion of a replica secret, so they cannot be removed by normal means.

---

## Why / code

| Location | What it does |
|---|---|
| `tenant/src/defender.ts:111-159` | Creates secret `saas/<tenant>/defender`, `recoveryWindowInDays: 0`, `replicas: aws.getRegionsOutput()` (all enabled regions), built inside a `pulumi.all().apply()` callback (resources-in-apply anti-pattern) |
| `tenant/src/defender.ts:114-125` | Replica list = every account-enabled region |
| `tenant.ts:260-273` | `new Defender(...)` instantiated unconditionally → every tenant gets the multi-region secret |

---

## Pulumi state

| Stack | Deleted | Errors | maybeCorrupt |
|---|---|---|---|
| saas-tenant | 57 | 0 | false |
| saas-tenant-cluster-services | 103 | 0 | false |
| saas-tenant-cluster | 121 | 0 | false |
| saas-defender-pool | 47 | 0 | false |

Tenant-stack log: defender secret `op=delete`, ARN `…secret:saas/life360-saas/defender-DRhi6w`, 30 replicas in state, succeeded with 0 errors. Empty post-destroy state is the expected healthy end-state; the orphan lives outside Pulumi state.

---

## Orphaned resources

| Resource | Type | AWS name / ARN | Status |
|---|---|---|---|
| Defender routing secret (×17) | secretsmanager:secret | `saas/life360-saas/defender` (ARN `…defender-DRhi6w`) | Orphaned in 17 regions (listed below) |

Orphaned regions (17):

```
af-south-1, ap-east-1, ap-east-2, ap-northeast-1, ap-northeast-2, ap-northeast-3,
ap-south-2, ap-southeast-4, ap-southeast-5, ca-west-1, eu-central-1, eu-west-1,
eu-west-2, eu-west-3, il-central-1, us-east-1, us-east-2
```

**Count reconciliation:** 30 created → 13 removed by pipeline → 17 orphaned.

**Cost:** 17 × $0.40/mo = $6.80/mo = $0.22/day, matching the JSON `dailyCosts` exactly (confirms 17 live and no orphaned compute/NAT/EIP).

---

## Anomaly

| Item | Detail |
|---|---|
| Partial teardown | 13 of 30 replicas removed, 17 left — inconsistent across regions, with 0 errors surfaced |
| Cleanup script failed | 17 delete attempts all failed: "Operation not permitted on a replica secret. Call must be made in primary secret's region" (primary already deleted → undeletable); plus 48 earlier rows "No credentials loaded for owner account 552447114887" |
| Files-job sentinel | `DestroyDefenderPoolFilesJob = SKIPPED:-1` while `DestroyDefenderPoolPipelineJob = SUCCESS` (recurring, unexplained by current code) |
| Replica-teardown mechanism | Pulumi logs record resource ops, not AWS SDK calls; exact replica-removal path only visible in CloudTrail (us-west-1, 2026-05-06 destroy window) |

---

## Remediation (where to look — not how to fix)

> Do not action any cleanup without team approval/discussion.

| Where | What we found there |
|---|---|
| `tenant/src/defender.ts:111-159` | Replica configuration + `pulumi.all().apply()` wrapper; origin of replica-teardown behavior |
| `tenant.ts:260-273` | Unconditional `Defender` instantiation — fleet-wide blast radius |
| AWS Secrets Manager (17 regions) | Orphans are replica secrets whose primary is gone; `delete-secret` (incl. `--force-delete-without-recovery`) is rejected. API of record for this state: `StopReplicationToReplica` (operates on a replica in its own region); `RemoveRegionsFromReplication` (operates from a primary) |
| Cleanup script (deletion-log-dmitry) | Cannot delete these (replica-without-primary error); also had credential-loading failures for account 552447114887 |
| CloudTrail (us-west-1, destroy window) | Where the actual replica-handling API calls would be confirmed |

---

## ops-portal bugs

None at the coordinator level for this tenant — the destroy workflow ran correctly and all pipelines genuinely succeeded. The defect is in the tenant IaC (`tenant/src/defender.ts`), not the ops-portal orchestration. Minor: the recurring `DestroyDefenderPoolFilesJob SKIPPED:-1` sentinel anomaly.

---

## Failure category

| # | Tenant | Layer | Repo | Category | Detection signal |
|---|---|---|---|---|---|
| 3 | life360-saas-prod | Layer 3 | tenant (`tenant/src/defender.ts`) | Multi-region secret replica orphan (clean destroy; partial replica removal; survivors undeletable — primary gone) | `destroyStatus=SUCCESS`, 0 pipeline errors, `saas/<tenant>/defender` present in non-primary regions, `delete-secret` rejected as "replica" |
