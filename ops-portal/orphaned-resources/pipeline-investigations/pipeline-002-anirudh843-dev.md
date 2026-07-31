# Pipeline Investigation #002 — anirudh843-dev (cluster layer — silent orphan)

**Status: ACTIVE INVESTIGATION** — abnormalities found, investigation ongoing

## Overview

| Field | Value |
|---|---|
| Tenant | anirudh843-dev |
| Environment | dev |
| AWS Account | 545681961293 |
| AWS Region | us-east-1 (N. Virginia) |
| Destroy triggered | 2026-03-25 |
| Destroy status (portal) | SUCCESS — misleading, see findings below |
| Pulumi state last updated | 2026-03-11 (never updated during destroy) |

---

## What the portal destroy workflow shows

All jobs show SUCCESS or SKIPPED. Portal considers tenant fully destroyed. Key anomalies in the job statuses:

| Job | Result | Notes |
|---|---|---|
| DestroyClusterPipelineJob | SKIPPED (jobId: -1) | Should have run — bug, see below |
| DestroyClusterFilesJob | SUCCESS "Removed files" | File existed — contradicts the skip above |
| DestroyTenantPipelineJob | SKIPPED (jobId: -1) | Skipped — no pipeline ran |
| DestroyTenantFilesJob | SKIPPED (jobId: -1) | Anomaly — code cannot produce this, likely prior partial run |
| DestroyClusterServicesPipelineJob | SKIPPED (jobId: -1) | Skipped — no pipeline ran |
| DestroyClusterServicesFilesJob | SKIPPED (jobId: -1) | Anomaly — same as above |
| CompleteTenantDestroyJob | SUCCESS | Ran last — marked tenant deleted |

---

## Root cause — portal code bug (confirmed in ops-portal source)

Bug introduced: 2026-02-04, commit `17fdd6a7` — "Ops Portal destroy should skip phases that don't need to run during destroy"

### How the bug works

Before triggering any destroy pipeline, `GitlabPipelineJobBase.start()` checks:

```ts
const fileExists = await gitlabApi.fileExists(repository.id, stackFilePath, branch)
if (!fileExists) {
  return { jobId: -1, state: State.SKIPPED }
}
```

`GitlabApi.fileExists()` returns `false` in two situations:

1. File genuinely does not exist (real 404) — correct skip
2. GitLab API call failed after 5 retries (timeout, rate limit, hiccup) — WRONG skip

Both look identical to the caller. There is no way to distinguish them.

### The smoking gun

`DestroyClusterPipelineJob` returned SKIPPED (fileExists = false). `DestroyClusterFilesJob` ran immediately after on the same file, same repo, same branch and reported "Removed files for tenant anirudh843-dev" — meaning the file WAS there.

Same file. Same repo. Same branch. One said missing, the next found and deleted it. The file existed. The skip was a false negative caused by a transient GitLab API failure.

### What happened as a result

```
User clicked Destroy in ops-portal
        ↓
DestroyClusterPipelineJob: fileExists() returned false (API hiccup)
        ↓
Job returned SKIPPED — no GitLab pipeline triggered, no Pulumi destroy ran
        ↓
DestroyClusterFilesJob: found and deleted Pulumi.anirudh843-dev.yaml from Git
        ↓
CompleteTenantDestroyJob: marked tenant deleted in DB
        ↓
Workflow = SUCCESS (SKIPPED counts as SUCCESS in coordinator)
        ↓
AWS infra never destroyed. Stack config file deleted. Portal cannot retry.
```

### Why the user had no warning

- `SKIPPED` looks identical to a legitimate skip (e.g. defender disabled) in the UI
- No Slack alert was sent for the skip
- No FAILURE was recorded — workflow completed as SUCCESS
- The user believed the tenant was fully destroyed

---

## Pulumi state file analysis

State file exported: `anirudh843-dev.txt` (last modified 2026-03-11 — creation date). 57 resources recorded. Nothing deleted. Pulumi never ran a destroy against this stack.

The state file being unchanged from creation confirms the destroy pipeline never executed.

---

## Orphaned resources confirmed in AWS (Resource Explorer — tag: anirudh843)

### Primary cost drivers

| Resource | ID | Estimated cost |
|---|---|---|
| NAT Gateway | `nat-0da83aa2f11a5b8de` | ~$32-45/month |
| Elastic IP #1 | `eipalloc-08a0c31545dc7d92c` | ~$3.60/month |
| Elastic IP #2 | `eipalloc-00fa19b9b34fa6c56` | ~$3.60/month |

### Full orphaned resource list

| Resource | Type | ID/ARN |
|---|---|---|
| VPC | ec2:vpc | `vpc-05eee428182cfe7eb` |
| Internet Gateway | ec2:internet-gateway | `igw-0761c22f9f7e54bc0` |
| NAT Gateway | ec2:natgateway | `nat-0da83aa2f11a5b8de` |
| Elastic IP #1 | ec2:elastic-ip | `eipalloc-08a0c31545dc7d92c` |
| Elastic IP #2 | ec2:elastic-ip | `eipalloc-00fa19b9b34fa6c56` |
| Subnet (private-0) | ec2:subnet | `subnet-05ab1e1fcdc755f45` |
| Subnet (private-1) | ec2:subnet | `subnet-067b13e3a4f2b5f6a` |
| Subnet (public-0) | ec2:subnet | `subnet-0416a232e540c7f3e` |
| Subnet (public-1) | ec2:subnet | `subnet-0b1ccd18ee37f5a22` |
| Route Tables (x4) | ec2:route-table | `rtb-02f8...`, `rtb-0725...`, `rtb-0f2f...`, `rtb-0f73...` |
| Security Group | ec2:security-group | `sg-0f8e270ef5dc16ec6` |
| Launch Template #1 | ec2:launch-template | `lt-05a3c1ecde436b5ce` |
| Launch Template #2 | ec2:launch-template | `lt-09e47df1f4e456b1a` |
| CFN Stack (karpenterInfra) | cloudformation:stack | `anirudh843-dev-cluster-karpenterInfra/9d1b35a0...` |
| CFN Stack (karpenterInfra-v1) | cloudformation:stack | `anirudh843-dev-cluster-karpenterInfra-v1/9d55cda0...` |
| IAM Role (worker) | iam:role | `worker-role-b82d19c` |
| IAM Role (EKS) | iam:role | `anirudh843-dev-cluster-eksRole-role-b6fa97e` |
| IAM Policy (ECR) | iam:policy | `worker-role-policy-ecr-73d4e32` |
| IAM Instance Profile | iam:instance-profile | `instance-profile-ec88563` |
| SQS Queue | sqs:queue | `anirudh843-dev-cluster` |
| CloudWatch Log Group | logs:log-group | `/aws/eks/anirudh843-dev-cluster/cluster` |
| EventBridge Rules (x4) | events:rule | Various karpenter event rules |

### Already cleaned up (not in Resource Explorer)

- EKS cluster — gone (likely deleted separately or by a prior partial run)
- EC2 worker nodes — gone

---

## Unresolved anomaly

`DestroyTenantFilesJob` and `DestroyClusterServicesFilesJob` both show `SKIPPED` with `jobId: -1`. The current ops-portal code for these jobs (`DestroyGitFilesJob.ts`) can only return SUCCESS or FAILURE with a positive jobId — it has no path to produce SKIPPED:-1. These entries likely come from a prior partial destroy attempt before this run. Flagged for further investigation.

---

## Bugs to fix in ops-portal (do not action without team discussion)

| # | File | Fix needed |
|---|---|---|
| 1 | `app/_lib/GitlabApi.ts:120-142` | Distinguish real 404 from API error — only return false on true 404, throw on other errors |
| 2 | `app/_lib/tenant/provisioning/GitlabPipelineJobBase.ts:54-68` | For destroy: treat inability to confirm file existence as FAILURE, not SKIPPED |
| 3 | `app/_lib/tenant/provisioning/destroy/DestroyGitFilesJob.ts` | Do not delete stack file unless destroy pipeline actually ran and succeeded |
| 4 | `app/_lib/tenant/provisioning/Coordinator.ts:374-376` | SKIPPED infra teardown should not roll up to SUCCESS silently |

---

## Failure category

| # | Tenant | Layer | Category | Estimated ongoing cost |
|---|---|---|---|---|
| 2 | anirudh843-dev | Layer 1 (cluster) | Silent orphan — false-negative fileExists check caused destroy pipeline skip, portal reported SUCCESS | ~$40-52/month |
