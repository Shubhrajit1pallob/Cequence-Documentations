# Phase 0 — Gate Review Write-up
Orphaned Resources Detection & Remediation · 2026-06-02

> Vault write-up prepared for the Phase 0 gate review (mentor + reviewer sign-off). This is the
> personal-vault assembly; the official deliverable lives in the ops-portal repo
> (`service-portal/ui/features/orphaned-resources/phase-0/`) and is pushed separately.

**Gate criteria (from [[Story - Phase 0 — Get Oriented|Phase 0 story]], Subtask 9):**

1. Walk through the full tenant lifecycle (Portal → GitLab → Pulumi → AWS).
2. Name 5+ specific resource types likely to leak.
3. Present a categorized failure-mode list from historical pipeline review.
4. Bring 2–3 written options at any decision point rather than committing early.

---

## 1. Tenant lifecycle walkthrough (Portal → GitLab → Pulumi → AWS)

### Provision

Source: [[tenant-provisioning-workflow]], [[coordinator-and-workflow]], [[gitlab-pipeline-integration]], [[argocd-sync-jobs]].

1. **Portal trigger.** UI "Confirm" (`ProvisionWorkflow.tsx`) → server action (`actions.ts`) → `Coordinator.ts` seeds a `WorkflowStatus` doc in MongoDB with all **23 jobs** as `PENDING`/`SKIPPED`, posts the initial Slack message, writes the audit log, and **returns to the UI immediately — no jobs run synchronously**.
2. **Background driver.** `Coordinator.checkPendingJobs` runs on a `setInterval` loop and advances one job at a time. The ordered job list is `PROVISION_TENANT_WORKFLOW` in `TenantProvisionCoordinator.ts`.
3. **Layered build (GitLab + Pulumi).** Each job follows the repeating pattern **create DB/Git entry → trigger GitLab pipeline → wait/poll → sync ArgoCD app**:

```
CICD record (.tenants.yml)
  → Cluster (EKS) + kubeconfig
    → Cluster services
      → Tenant infra + Dex (SSO)
        → ArgoCD wiring (manifests → K8s)
          → Observability silencing (Datadog + Grafana)
            → Defender pool (optional — skipped if defender disabled)
              → DAGs
                → Health gate
                  → Done
```

4. **AWS materialization.** The GitLab pipelines run `pulumi up` per layer, which creates the actual AWS resources (VPC/subnets/NAT/EKS in the cluster layer, tenant infra incl. the defender secret in the tenant layer, etc.).

### Destroy

Source: [[failure-and-retry-semantics]] + the stack evidence across investigations #001–#005.

- Destroy runs the layers in **reverse order (5 → 0)**: `dex` → `tenant` → `cluster-services` → `cluster` (and `defender-pool` when present). Each stack runs `pulumi destroy` via its GitLab pipeline; the coordinator polls each to a terminal state.
- **Operator actions** on a failed job: **retry / skip / stop** (see [[failure-and-retry-semantics]]); jobs carry a **2h timeout** (PLAT-1441).
- A genuinely clean destroy shows every stack `SUCCESS` with `0 errors` and `maybeCorrupt: false` in the Pulumi artifacts — yet orphans can still survive (see §3, categories #2 and #3), so a green workflow is **not** proof of full teardown.

---

## 2. Resource types likely to leak (5+)

Source: [[orphaned-resource-types]] (collector script, dev account 545681961293, 19 tenants — 379 resources) and [[failure-mode-analysis]].

| # | Resource type | Why it leaks | Cost signal |
|---|---|---|---|
| 1 | `secretsmanager:secret` | Defender secret replicated to ~30 regions; primary deleted, replicas stranded & undeletable. **Widest spread (17+ regions/tenant), most common.** | ~$0.40/replica/mo (e.g. $6.40–6.80/mo) + stale credentials |
| 2 | `ec2:natgateway` | Survives when the cluster destroy pipeline never runs (silent skip) | **Top cost driver, ~$32–45/mo each** |
| 3 | `ec2:elastic-ip` | Unattached EIP left after VPC teardown skipped | ~$3.60/mo each |
| 4 | `ec2:vpc` / `ec2:subnet` | Entire VPC + subnets left running on a skipped cluster destroy | Indirect (blocks dependents) |
| 5 | `iam:role` + `cloudformation:stack` | Karpenter CFN stack stuck `DELETE_FAILED`; role never retried | $0 (but stuck stack) |
| 6 | `ec2:volume` (EBS) | Detached volumes survive | Per-GB-month |
| 7 | `logs:log-group` | EKS cluster log groups outlive the cluster | Per-GB ingest/storage |

(Others observed: `ec2:launch-template`, `elasticloadbalancing:*`, `acm:certificate`, `route53:healthcheck`, `events:rule`, `sqs:queue`, `iam:instance-profile`, `iam:policy` — full list in [[orphaned-resource-types]].)

---

## 3. Categorized failure-mode list (historical pipeline review)

Five destroy pipelines investigated manually (#001–#005); three distinct failure modes confirmed. Each can occur **even when the portal/pipeline reports SUCCESS**, so none raises an operational alert.

| Cat | Failure mode | Layer / repo | Cost | Investigations |
|---|---|---|---|---|
| **#1** | CFN no-retry → orphaned Karpenter IAM role. CFN doesn't retry the role delete after the instance-profile blocker clears; stack stuck `DELETE_FAILED`. | Layer 1 / cluster | **$0** | [[pipeline-001-testreportfeature-dev]] (+ `urvashi-sentiel-dev` suspected) |
| **#2** | Silent false-negative `fileExists` skip → cluster destroy pipeline never ran, portal rolled up to SUCCESS. Full VPC + NAT + EIPs left billing. | ops-portal coordinator | **~$40–52/mo** | [[pipeline-002-anirudh843-dev]] |
| **#3** | Multi-region defender-secret replica orphan. Clean SUCCESS destroy, but replicas survive in non-primary regions and become undeletable once the primary is gone. | Layer 3 / tenant (`tenant/src/defender.ts`) | **$6.40–12/mo + security** | [[pipeline-003-api-security-ng1-dev]], [[pipeline-004-life360-saas-prod]], [[pipeline-005-shub-test-dev]] |

**Deliverable scoping:** the official [[failure-mode-analysis]] documents **Cat #1** and **Cat #3** (its "Category 1" and "Category 2"). The silent-skip (**Cat #2**) is documented separately in [[pipeline-002-anirudh843-dev]] and is intentionally **not** in that deliverable.

**Key takeaway:** #1 fails *loudly* (DELETE_FAILED) but costs nothing; #2 and #3 cost real money *and* report green — so detection cannot rely on workflow status alone.

---

## 4. Decision points — options (not yet committed)

### (a) Remediation ownership for the replica-secret leak (Cat #3)
- **Option A — fix tenant IaC** (`tenant/src/defender.ts`): make the secret single-region or gate it on `defenderConfig`. Root-cause fix; touches Layer 3, needs infra-team review.
- **Option B — ops-portal guardrail**: add a post-destroy reconcile check that detaches replicas before deleting the primary. Doesn't fix root cause but contains blast radius.
- **Option C — post-hoc discovery sweep**: scheduled scan + approval-gated cleanup of stranded replicas. Catches existing orphans regardless of source.

### (b) Detection approach
- **Option A — reconcile-on-destroy**: verify AWS state matches expected-empty at end of each destroy workflow.
- **Option B — scheduled discovery scan**: the Phase 1/2 discovery job, independent of the destroy path.
- **Option C — both**: inline check for new destroys + periodic sweep for the backlog. *(Leaning here, pending discussion.)*

### (c) Replica-orphan cleanup mechanism (Cat #3)
- **Option A — `StopReplicationToReplica`** (run per-replica, in each replica's own region).
- **Option B — `RemoveRegionsFromReplication`** (run from the primary — only works if the primary still exists).
- **Option C — re-create a primary, then `RemoveRegionsFromReplication`** for already-stranded replicas (primary already deleted).
- > No cleanup to be actioned without team approval — the exact teardown path is only confirmable in CloudTrail.

---

## 5. Gate-criteria checklist

| Criterion | Met by |
|---|---|
| Walk the full lifecycle (Portal → GitLab → Pulumi → AWS) | §1 (provision 23-job flow + destroy reverse order) |
| Name 5+ resource types likely to leak | §2 (7 ranked + extended list) |
| Categorized failure-mode list from historical review | §3 (3 categories, 5 investigations) |
| Bring 2–3 options at decision points | §4 (3 decisions, options each) |

**Status:** ready for mentor + reviewer sign-off → Phase 0 gate.
