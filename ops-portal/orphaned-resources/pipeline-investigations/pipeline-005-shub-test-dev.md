# Pipeline Investigation #005 — shub-test-dev

**Status: ACTIVE** — 16-region secret orphan live; compute entries are stale Resource-Explorer echoes (pending `describe-instances` confirmation)

## Overview

| Field | Value |
|---|---|
| Tenant | shub-test-dev (internal test tenant; owner shubhrajit.pallob) |
| Environment | dev |
| AWS account | 545681961293 |
| Region | us-east-2 (defender secret replicated across regions) |
| Created → Deleted | 2026-05-07 → 2026-06-02 20:29 UTC (deleted same day as investigation) |
| Destroy status | SUCCESS — all pipelines ran with real IDs + links |
| Defender ASG | None (no `defenderConfig`; `defenderConfiguration` feature off) |
| Note | `internalNotes`: "Tenant created by TenantPulumiDetailUpdater missing from tenant stacks" |
| Failure category | #3 — Multi-region secret replica orphan |

---

## How found

| Source | Finding |
|---|---|
| Tenant JSON | `destroyStatus` SUCCESS; all pipelines real IDs + links; ~$40/day while alive ($920 May), deleted June 2 |
| Resource Explorer + collect script (`run.sh collect --tenant shub-test`) | 47 tagged: 16 `secretsmanager:secret` + 31 Karpenter/EKS compute objects |
| Pulumi destroy artifacts (4 stacks) | all 0 errors, `maybeCorrupt: false` |

---

## Workflow result

| Stack | Pipeline | Result |
|---|---|---|
| dex | https://gitlab.com/cequence/saas/infrastructure/dex/-/pipelines/2571420284 | SUCCESS (apply) |
| tenant | https://gitlab.com/cequence/saas/infrastructure/tenants/tenant/-/pipelines/2571425817 | SUCCESS — 53 deleted, 0 errors |
| cluster-services | https://gitlab.com/cequence/saas/infrastructure/tenants/cluster-services/-/pipelines/2571433686 | SUCCESS — 109 deleted, 0 errors |
| cluster | https://gitlab.com/cequence/saas/infrastructure/tenants/cluster/-/pipelines/2571454530 | SUCCESS — 121 deleted, 0 errors |
| defender-pool | — | SKIPPED (jobId −142) — correctly skipped, no defender |

---

## Root cause

A clean, complete destroy whose only genuine survivor is an out-of-scope resource. The cluster stack deleted the full substrate — VPC, all 4 subnets, 2 NAT gateways, 2 EIPs, internet gateway, EKS cluster, both Karpenter CFN stacks, node group, launch templates (121 resources, 0 errors). The lone genuine orphan is the multi-region defender secret. The 31 Karpenter/EKS compute objects in Resource Explorer are terminal-state index echoes, not live resources.

---

## Why / code

| Location | What it does |
|---|---|
| `tenant/src/defender.ts:111-159` | Creates secret `saas/<tenant>/defender`, `recoveryWindowInDays: 0`, `replicas: aws.getRegionsOutput()` (all regions), inside a `pulumi.all().apply()` callback |
| `tenant.ts:260-273` | `new Defender(...)` instantiated unconditionally → the multi-region secret is created even though this tenant has no defender ASG |

---

## Pulumi state

| Stack | Deleted | Errors | maybeCorrupt |
|---|---|---|---|
| saas-tenant | 53 | 0 | false |
| saas-tenant-cluster-services | 109 | 0 | false |
| saas-tenant-cluster | 121 | 0 | false |

Cluster-stack deletes include `aws:ec2/vpc:Vpc`, 4× `aws:ec2/subnet:Subnet`, 2× `aws:ec2/natGateway:NatGateway`, 2× `aws:ec2/eip:Eip`, `aws:eks/cluster:Cluster`, and 2× `aws:cloudformation/stack:Stack` (…-karpenterInfra, …-karpenterInfra-v1). Tenant-stack secret recorded `op=delete` with 30 replicas in state, primary `us-east-2`.

---

## Orphaned resources

| Resource | Type | AWS name / ARN | Status |
|---|---|---|---|
| Defender routing secret | secretsmanager:secret | `saas/shub-test/defender` (ARN `…defender-RLG5qK`) | Orphaned — 30 created, 16 surviving per Resource Explorer → partial teardown (14 removed, 16 left) |
| Karpenter/EKS compute (6 instance, 8 fleet, 14 ENI, 4 spot-request) | ec2:* | tagged `shub-test-dev-cluster` in us-east-2 | Stale RE index echoes — terminal-state (cluster incl. VPC/subnets deleted with 0 errors ⇒ subnets empty); not billable; live-state not yet confirmed via `describe-instances` |

Surviving secret-replica regions (16):

```
af-south-1, ap-east-1, ap-east-2, ap-northeast-1, ap-northeast-2, ap-northeast-3,
ap-south-2, ap-southeast-4, ap-southeast-5, ca-west-1, eu-central-1, eu-west-1,
eu-west-2, eu-west-3, il-central-1, us-east-1
```

**Count reconciliation:** 30 created → 14 removed by pipeline → 16 orphaned.

**Cost (estimate):** 16 surviving × ~$0.40/mo ≈ **~$6.40/mo** — an estimate at the per-replica rate used in [[pipeline-004-life360-saas-prod]], **not** independently cost-verified against `dailyCosts` for this tenant. The `~$40/day` / `$920 May` figure is the tenant's **live** cost, not the orphan cost.

---

## Anomaly

| Item | Detail |
|---|---|
| Resource Explorer overcount | RE returns terminated/deleted Karpenter compute as if present (same behavior as api-security-ng1's `ec2:fleet` rows, which `describe-fleets` showed empty) |
| Partial secret teardown | 30 replicas created, 16 surviving — same inconsistent removal seen on life360 (there 30→17) |
| Tenant provenance | `internalNotes` says the record was reconstructed by `TenantPulumiDetailUpdater` because it was missing from the tenant stacks |

---

## Remediation (where to look — not how to fix)

> Do not action any cleanup without team approval/discussion.

| Where | What we found there |
|---|---|
| `tenant/src/defender.ts:111-159` | Replica config + `pulumi.all().apply()` wrapper — origin of replica-teardown behavior |
| `tenant.ts:260-273` | Unconditional `Defender` instantiation — secret created even with no defender ASG |
| AWS Secrets Manager (16 regions) | Surviving replicas; if primary (us-east-2) is gone they hit the replica-without-primary undeletable state. API of record: `StopReplicationToReplica` (in-region) / `RemoveRegionsFromReplication` (from primary) |
| `describe-instances` + June 3–4 cost | Where to confirm the 31 compute entries are terminal (state = terminated; cost → ~$0) |

---

## ops-portal bugs

None at the coordinator level — the destroy workflow ran correctly, all pipelines genuinely succeeded, and defender-pool was correctly skipped. The defect is the tenant IaC (`tenant/src/defender.ts`).

---

## Failure category

| # | Tenant | Layer | Repo | Category | Detection signal |
|---|---|---|---|---|---|
| 3 | shub-test-dev | Layer 3 | tenant (`tenant/src/defender.ts`) | Multi-region secret replica orphan (clean destroy; partial replica survival) | `destroyStatus=SUCCESS`, 0 pipeline errors, `saas/<tenant>/defender` present in non-primary regions; compute in RE is terminal-state noise |
