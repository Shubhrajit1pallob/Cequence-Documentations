# Pipeline Investigation #006 — ache-prod

**Status: ACTIVE** — destroy FAILED; defender-pool stack `maybeCorrupt`; SG + a surviving running instance live in AWS

## Overview

| Field | Value |
|---|---|
| Tenant | ache-prod |
| Environment | prod |
| AWS account | 552447114887 (from resource ARNs) |
| Region | us-west-1 |
| Deleted at | PENDING (tenant JSON not yet pulled) |
| Destroy status | FAILED — defender-pool stack destroy aborted (other stacks: PENDING tenant JSON) |
| Failure category | #4 (new) — ASG surviving-instance DependencyViolation |

---

## How found

| Source | Finding |
|---|---|
| Artifact `pulumi-destroy-ache-prod.log` (saas-defender-pool) | destroy FAILED: 39 deleted, 3 error diagnostics, `maybeCorrupt: true` |
| Error message | `DeleteSecurityGroup … DependencyViolation: resource sg-05bb690f3deeafe80 has a dependent object` |
| Live AWS check (user) | a running EC2 instance survived the ASG deletion — its ENI is the SG's dependent |

---

## Workflow result

| Stack | Pipeline | Result |
|---|---|---|
| defender-pool | PENDING (link of failed run) | FAILED — 39 deleted, then DependencyViolation on the defender SG |
| dex / tenant / cluster-services / cluster | PENDING tenant JSON | unknown |

---

## Root cause

The defender-pool destroy proceeded in correct dependency order — listener → ALB → ASG (`ache-us-west-1-asg.ache`, deletion completed, verified via resOutputsEvent, 0 errors) → SG. But a running EC2 instance survived the ASG deletion; its ENI still referenced the defender SG `ache-us-west-1-asg-defender-sg` (`sg-05bb690f3deeafe80`), so AWS rejected the SG delete with DependencyViolation, aborting the destroy and leaving the stack state `maybeCorrupt`. The ASG's settings rule out the obvious explanations — `forceDelete: false` and `protectFromScaleIn: false` mean the provider drains instances before deleting, and the ASG reported empty — so the instance was no longer ASG-managed at delete time. Candidate mechanisms (unresolved): a detached instance, or a spot/capacity-rebalance race (`capacityRebalance: true`).

---

## Why / code

| Location | What it does |
|---|---|
| `defender-pool/src/defenderPool.ts:109` | `cequence:saas:DefenderPoolEC2` component |
| `defender-pool/src/defenderPool.ts:600` | creates `${name}-defender-sg` (the SG that failed) |
| `defender-pool/src/defenderPool.ts:943` | creates the `aws.autoscaling.Group` (`forceDelete:false`, `protectFromScaleIn:false`, `capacityRebalance:true`) |
| Teardown gap | nothing waits for instance/ENI release between ASG deletion completing and the SG delete |

---

## Pulumi state

| Stack | Deleted | Errors | maybeCorrupt |
|---|---|---|---|
| saas-defender-pool | 39 | 3 (all on the defender SG) | true — state out of sync; needs refresh before re-destroy |

Delete timeline (epoch s):

```
listener …641→…642 · ALB …699→…728 ✓ · defender-SG start …6094 ✗ (no completion)
· target group …6101→…6105 ✓ (independent of the SG; Pulumi continues past failures —
also why it appears "after")
```

---

## Orphaned resources

| Resource | Type | AWS name / ID | Status |
|---|---|---|---|
| Defender security group | ec2:security-group | `ache-us-west-1-asg-defender-sg` / `sg-05bb690f3deeafe80` | Orphaned — delete blocked |
| Surviving EC2 instance (≥1) | ec2:instance | PENDING (ID from describe) — fingerprint: m6g.medium, AMI `ami-070ffa249b31e7ba6`, LT `lt-005a38de582762356`, subnets `subnet-02c567ef9cc9681b9`/`subnet-033b42a961df3894e`, profile `ache-us-west-1-asg-instance-profile` | RUNNING — live billing (user-confirmed) |
| Its ENI(s) | ec2:network-interface | PENDING | attached → the SG's "dependent object" |
| Defender secret (tenant stack) | secretsmanager:secret | `saas/ache/defender` | Unverified — separate stack, outside this artifact |

---

## Anomaly

| Item | Detail |
|---|---|
| Instance survived a non-force, non-protected ASG delete | ASG saw 0 instances and deleted cleanly while an instance kept running → detached vs spot/rebalance race, classification PENDING instance details |
| No instance/ENI IDs in the artifact | ASG-launched instances are not Pulumi resources — the log can only fingerprint them via the launch template |
| Failure visibility | Unlike categories #1/#3, this failure was surfaced — the pipeline genuinely reported FAILURE |

---

## Remediation (where to look — not how to fix)

> Do not action any cleanup without team approval/discussion.

| Where | What we found there |
|---|---|
| The surviving instance (us-west-1) | The blocker; `describe-instances`/`describe-network-interfaces` filtered by `group-id sg-05bb690f3deeafe80` names it; its tags (`aws:autoscaling:groupName`), `InstanceLifecycle` (spot) and `LaunchTime` classify the mechanism |
| `defender-pool/src/defenderPool.ts:600,943` | SG + ASG definitions; the teardown window between ASG deletion and SG deletion |
| Pulumi state (saas-defender-pool) | `maybeCorrupt: true` — needs refresh + re-destroy once the blocker is cleared |
| ache tenant stack | Separate check for the `saas/ache/defender` secret (category #3) |

---

## ops-portal bugs

None — notably, this failure was correctly reported by the workflow (FAILURE, not silent SUCCESS). The defect is in the defender-pool teardown's tolerance for non-ASG-managed instances, not the orchestration.

---

## Failure category

| # | Tenant | Layer | Repo | Category | Detection signal |
|---|---|---|---|---|---|
| 4 | ache-prod | defender-pool | defender-pool (`src/defenderPool.ts`) | ASG surviving-instance DependencyViolation (hard destroy failure, maybeCorrupt state) | `DestroyDefenderPoolPipelineJob` FAILURE; DependencyViolation on `<tenant>-defender-sg`; a running instance with an ENI on that SG |
