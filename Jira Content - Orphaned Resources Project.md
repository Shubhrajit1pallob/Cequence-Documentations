> **Purpose:** Copy-paste content for creating Jira tickets.

> **Project:** PLAT

> **Tag:** `coop-2026`

> **Note:** In each story/subtask, link to the relevant markdown doc in `service-portal/ui/features/orphaned-resources/` rather than duplicating content.

---

## Epic

**Title:** Orphaned AWS Resource Detection & Cleanup (Co-op project)

**Description:**

Deleted tenants continue to incur AWS infrastructure costs due to orphaned

resources that persist after tenant deletion.

This project will build automated discovery, per-tenant visibility in the

Ops Portal, and approval-gated cleanup workflows to eliminate ongoing waste

and prevent future resource leakage.

**Cost baseline:**

- 14 orphaned tenants (deleted >20 days): $5,318/month
- 17 tenants with no Deleted At, still costing: $1,758/month
- Estimated annual savings: ~$85K

**Deliverables:**

1. Discovery job — scheduled scan across AWS accounts
2. Ops Portal "Orphans" tab — per-tenant visibility with cost estimates
3. Approval-gated cleanup — human-in-the-loop batch approvals
4. Monitoring dashboards — orphan count, $ saved, workflow health
5. Root cause analysis — Pulumi leak patterns + upstream PRs
6. Cost reduction — measurable $/month reduction
7. Decision log — tracked in git

**Documentation:** `ops-portal/service-portal/ui/features/orphaned-resources/`

**Labels:** `coop-2026`

---

## Stories

- ### [[Story - Phase 0 — Get Oriented]]
- ### [[Story - Phase 1 — Discovery Design]]
- ### [[Story - Phase 2 — Discovery Workflow]]
- ### [[Story - Phase 3 — Portal "Orphans" UI]]
- ### [[Story - Phase 4 — Approval & Destruction]]
- ### [[Story - Phase 5 — Monitoring & Rollout]]
- ### [[Story - Phase 6 — Root-cause Fixes (STRETCH)]]

---

## Failure-Category Tickets (from pipeline investigations)

> Draft ticket descriptions, one per distinct failure category found across the destroy-pipeline investigations ([[pipeline-001-testreportfeature-dev]], [[pipeline-002-anirudh843-dev]], [[pipeline-003-api-security-ng1-dev]], [[pipeline-004-life360-saas-prod]]). Focus is on **what** the problem is and its impact — not how to fix it. Ordered by priority.

---

### Ticket — Defender secret replicas orphaned across regions after tenant destroy

**Priority:** High
- Widespread (fleet-wide, affects every destroyed tenant), silent, with recurring cost and a security exposure.

**What is happening:**
Every tenant's "defender" secret in AWS Secrets Manager is created replicated to all enabled AWS regions (~30). When the tenant is destroyed, the destroy runs cleanly and reports SUCCESS with zero errors, but only the primary secret (and sometimes a subset of replicas) is removed — the remaining replicas survive as standalone secrets in their regions. Because the primary is gone, AWS then refuses to delete those replicas ("operation not permitted on a replica secret"), so they are stranded in an undeletable state. The teardown can be partial and inconsistent across regions, and nothing surfaces the leftovers.

**Evidence:**
Confirmed on two tenants. `api-security-ng1-dev` (dev — [[pipeline-003-api-security-ng1-dev]]): 30 replicas created, survivors visible in ~17 regions, exact count not yet cost-verified. `life360-saas-prod` (prod — [[pipeline-004-life360-saas-prod]]): 30 created → 13 removed → **17 orphaned**, cost-verified at $6.80/month matching the tenant's post-deletion daily-cost data; a cleanup script failed all 17 delete attempts. Both destroys reported clean SUCCESS with zero pipeline errors. Secret naming pattern: `saas/<tenant>/defender-<suffix>`.

**Scope:**
Two tenants confirmed (one dev, one prod). Systematic and fleet-wide: the secret is created for every tenant unconditionally, so every destroyed tenant strands replicas. Affects both dev and prod. Reproducible by design — not a transient event.

**Impact:**
Recurring cost up to ~$12/month per tenant (30 replicas × $0.40) where all survive; $6.80/month confirmed on the prod case (17 survivors). **Security:** each orphan is a stale traffic-ingestion credential left live across many regions worldwide — up to 30 per destroyed tenant. Operationally **silent**: the destroy reports success with no errors, so nothing is alerted; orphans accumulate fleet-wide and become undeletable by normal means once stranded.

**Resource types orphaned:**
`secretsmanager:secret` (regional replica secrets).

---

### Ticket — Karpenter CloudFormation stack stuck in DELETE_FAILED, IAM role orphaned

**Priority:** Low
- Single confirmed case (plus one suspected), zero ongoing cost, and it surfaces as a failed job rather than silently — a manual fix exists.

**What is happening:**
During the cluster destroy, the Karpenter CloudFormation stack attempts to delete its IAM node role before detaching it from the instance profile, hits a 409 ("must remove roles from instance profile first"), then deletes the instance profile but never goes back to retry the role within the same operation. The stack is left in DELETE_FAILED with the IAM role orphaned. This is a known CloudFormation limitation (it does not retry a failed resource deletion within one delete operation), not a deletion-ordering bug.

**Evidence:**
Confirmed on `testreportfeature-dev` ([[pipeline-001-testreportfeature-dev]]), verified in the AWS Console: 8 of 9 stack resources DELETE_COMPLETE, only `KarpenterNodeRole` DELETE_FAILED. `urvashi-sentiel-dev` shows the same DELETE_FAILED state (2026-04-29) and is flagged as likely the same pattern, not yet investigated.

**Scope:**
One confirmed, one suspected — dev. Likely recurs wherever the role-vs-instance-profile delete ordering races, but bounded to the Karpenter CFN stack. **Not silent** — the destroy job actually fails, so it is visible in the pipeline.

**Impact:**
**Zero ongoing cost** — the orphan is an IAM role plus a stuck CFN stack; neither bills. No credential exposure beyond a dangling role. It does surface operationally (the destroy job fails), and a manual CloudFormation "Retry delete" clears it once the instance-profile blocker is gone.

**Resource types orphaned:**
`iam:role`, `cloudformation:stack` (stuck in DELETE_FAILED).
