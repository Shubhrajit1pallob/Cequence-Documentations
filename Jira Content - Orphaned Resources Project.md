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
- ### [[Story - Phase 1 — Get Oriented]]
- ### [[Story - Phase 2 — Discovery Workflow]]
- ### [[Story - Phase 3 — Portal "Orphans" UI]]
- ### [[Story - Phase 4 — Approval & Destruction]]
- ### [[Story - Phase 5 — Monitoring & Rollout]]
- ### [[Story - Phase 6 — Root-cause Fixes (STRETCH)]]
- 