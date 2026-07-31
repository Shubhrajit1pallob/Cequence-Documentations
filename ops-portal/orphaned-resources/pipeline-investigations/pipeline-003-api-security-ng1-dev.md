# Pipeline Investigation #003 — api-security-ng1-dev

**Status: ACTIVE** — orphan confirmed live (30 replica secrets still present as of 2026-06-01)

## Overview

| Field | Value |
|---|---|
| Tenant | api-security-ng1-dev |
| Environment | dev |
| AWS account | 545681961293 |
| Primary region | us-west-2 (defender secret replicated to 30 additional regions per Pulumi state; surviving/orphaned count not cost-verified — see amendment) |
| Created → Deleted | 2026-05-30 → 2026-06-01 12:37 UTC (~1.6 days) |
| Destroy status | SUCCESS — genuine, clean across all stacks |
| Failure category | #3 — Multi-region secret replica orphan |

---

## How found

| Step | Finding |
|---|---|
| Tenant JSON | `destroyStatus.status = SUCCESS`; all destroy pipelines have real jobIds + links |
| AWS Resource Explorer (tag `api-security-ng1`) | secret `saas/api-security-ng1/defender-xVj1TG` in ~17 visible regions + `ec2:fleet` rows |
| `aws ec2 describe-fleets --region us-west-2` | `{"Fleets": []}` — no active fleets (RE entries were stale) |
| `aws secretsmanager describe-secret … us-west-2` | `ResourceNotFoundException` — primary gone |
| `list-secrets` in replica regions | secret still present → orphan confirmed live |

---

## Workflow result

| Stack | Pipeline | Result |
|---|---|---|
| dex | https://gitlab.com/cequence/saas/infrastructure/dex/-/pipelines/2567076790 | SUCCESS (160 same) |
| defender-pool | https://gitlab.com/cequence/saas/infrastructure/tenants/defender-pool/-/pipelines/2567085707 | SUCCESS — 40 deleted, 0 errors |
| tenant | https://gitlab.com/cequence/saas/infrastructure/tenants/tenant/-/pipelines/2567109690 | SUCCESS — 53 deleted, 0 errors |
| cluster-services | https://gitlab.com/cequence/saas/infrastructure/tenants/cluster-services/-/pipelines/2567126342 | SUCCESS — 109 deleted, 0 errors |
| cluster | https://gitlab.com/cequence/saas/infrastructure/tenants/cluster/-/pipelines/2567158468 | SUCCESS — 121 deleted (incl. 4 NAT GW + 2 EIP), 0 errors |

All artifacts: `maybeCorrupt: false`, zero error diagnostics. Compute/networking fully removed.

---

## Root cause

The destroy was genuinely clean, but cannot remove a resource that isn't in Pulumi's effective control: a Secrets Manager secret `saas/<tenant>/defender` replicated to ~30 regions. Pulumi deleted the primary (us-west-2) and reported success; the replicas survived as orphaned standalone secrets (30 created; surviving count not cost-verified — possibly partial, per [[pipeline-004-life360-saas-prod]]).

---

## Why / code

| Location | What it does |
|---|---|
| `tenant/src/defender.ts:111-159` | Creates secret `saas/<tenant>/defender`, `recoveryWindowInDays: 0`, `replicas: aws.getRegionsOutput()` (all regions except primary), built inside a `pulumi.all().apply()` callback (resources-in-apply anti-pattern) |
| `tenant/src/defender.ts:114-125` | Replica region list = every account-enabled region |
| `tenant.ts:260-273` | `new Defender(...)` instantiated unconditionally → every tenant gets the 30-region secret → every destroyed tenant strands replicas |

---

## Pulumi state

| Stack | Deleted | Errors | maybeCorrupt |
|---|---|---|---|
| saas-tenant | 53 | 0 | false |
| saas-tenant-cluster-services | 109 | 0 | false |
| saas-defender-pool | 40 | 0 | false |
| saas-tenant-cluster | 121 | 0 | false |

Tenant-stack log shows the defender secret with 30 replicas: `op=refresh` then `op=delete`, both succeeded, 0 errors. Empty post-destroy state is the expected healthy end-state — the orphan lives entirely outside any Pulumi state.

---

## Orphaned resources

> **Amendment (corrected via [[pipeline-004-life360-saas-prod]]):** 30 replicas were **created** (per Pulumi state); the surviving/orphaned count here was **not cost-verified** and could be partial — `life360-saas-prod` (#004) demonstrated 30 created → 17 orphaned. Resource Explorer showed ~17 visible regions for this tenant. The 30-region list below is the *created* set; the exact survivor count is pending a per-region sweep or cost check.

| Resource | Type | AWS name / ARN | Status |
|---|---|---|---|
| Defender routing secret (≤30) | secretsmanager:secret | `saas/api-security-ng1/defender` (ARN `…defender-xVj1TG`) | Orphaned in non-primary regions (created set listed below; survivor count not cost-verified) |

Replica regions created (30):

```
af-south-1, ap-east-1, ap-east-2, ap-northeast-1, ap-northeast-2, ap-northeast-3,
ap-south-1, ap-south-2, ap-southeast-1, ap-southeast-2, ap-southeast-3, ap-southeast-4,
ap-southeast-5, ca-central-1, ca-west-1, eu-central-1, eu-central-2, eu-north-1,
eu-south-1, eu-south-2, eu-west-1, eu-west-2, eu-west-3, il-central-1, me-central-1,
me-south-1, sa-east-1, us-east-1, us-east-2, us-west-1
```

**Cost:** up to 30 × $0.40/secret/month = ~$12/month per tenant if all 30 survived, recurring (replicas billed as separate secrets). Not cost-verified for this tenant — actual survivor count (and thus cost) pending a per-region sweep; #004 showed the real figure can be lower (17 of 30).

**Security:** up to 30 stale traffic-ingestion credentials persisting worldwide (survivor count not yet confirmed).

---

## Anomaly

| Item | Detail |
|---|---|
| CequenceAsp app job | `DestroyCequenceAspApplicationJob` → SKIPPED:-1 after exactly 2.0h (timeout); in-cluster app only, no infra impact |
| Replica teardown mechanism | Pulumi log records resource ops, not AWS SDK calls. It confirms the outcome (secret reported deleted, replicas survive) but NOT the path (delete vs detach vs promote). Exact sequence is only visible in CloudTrail (us-west-2, 2026-06-01 ~05:18–12:37 UTC: `RemoveRegionsFromReplication` / `StopReplicationToReplica` / `DeleteSecret`). |

---

## Remediation (where to look — not how to fix)

| Where | What we found there |
|---|---|
| `tenant/src/defender.ts:111-159` | The replica configuration + the `pulumi.all().apply()` wrapper; origin of the replica-teardown behavior |
| `tenant.ts:260-273` | Unconditional `Defender` instantiation — determines fleet-wide blast radius |
| tenant destroy pipeline / ops-portal coordinator | Where a destroy-time replica-teardown or post-destroy verification step would belong |
| AWS Secrets Manager, 30 regions | Existing orphans live as per-region standalone secrets `saas/<tenant>/defender`; AWS API of record: `RemoveRegionsFromReplication` (delete-before-primary ordering), `StopReplicationToReplica` |
| CloudTrail (us-west-2, destroy window) | Where the actual replica-handling API calls would be confirmed |

---

## ops-portal bugs

None at the coordinator level for this tenant — the destroy workflow ran correctly and all pipelines genuinely succeeded. The defect is in the tenant IaC (`tenant/src/defender.ts`), not the ops-portal destroy orchestration. (Contrast: category #2 / anirudh843 IS an ops-portal coordinator defect.)

---

## Failure category

| # | Category | Layer | Repo | Tenant | Detection signal |
|---|---|---|---|---|---|
| 3 | Multi-region secret replica orphan (clean destroy; primary deleted, replicas survive — 30 created, survivor count not cost-verified) | Layer 3 | tenant (`tenant/src/defender.ts`) | api-security-ng1-dev | `destroyStatus=SUCCESS`, 0 pipeline errors, but `saas/<tenant>/defender` present in non-primary regions |
