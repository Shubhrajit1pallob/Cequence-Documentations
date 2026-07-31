---
status: IN REVIEW — submitted 2026-06-17
updated: 2026-06-17
---

# Orphan Identification Logic (S5)

**Status: IN REVIEW**

Related: [[Discovery job implementation]] (S7) · [[Inventory record schema]] (S4) · [[Phase 2/Index|Phase 2 Docs]]

---

## The core question

When the discovery job finds a resource in AWS tagged with a deleted tenant's name, how confident are we that it's truly orphaned (billing without purpose) vs legitimately still in use?

---

## Decision

**Primary criterion: deleted tenant + tagged/associated resource = orphan candidate.**

A resource is flagged as an orphan candidate when both conditions are met:

1. The tenant is marked `deleted: true` with a `deletedAt` timestamp in the portal (currently supplied via `TENANT_TARGETS` env var; will be replaced by `GET /api/external/orphaned-resources/targets`)
2. The resource is found by either discovery layer (tagged or supplementary)

No additional filtering is applied at discovery time. All candidates are written to the inventory. The discovery job does not make go/no-go cleanup decisions — that belongs to the Phase 4 approval gate. The job is read-only by design.

---

## Confidence tiers

The `discoveryMethod` field in every inventory record captures attribution confidence:

| Value | How found | Confidence |
|---|---|---|
| `tagged` | AWS Resource Explorer: `tag.value:<tenant-name>` search across the account | **High** — resource explicitly tagged itself as belonging to this tenant |
| `supplementary` | Anchor-walking: VPC → subnets/ENIs, EKS cluster → node groups, CFN stack → child resources, ASG prefix match | **Lower** — resource is structurally associated with tenant infrastructure but carries no explicit tag |

The supplementary tier exists to catch resources provisioned without the tenant tag — a known gap in the tagging strategy confirmed across Phase 0 investigations (e.g., ENIs auto-created by EKS, RDS snapshots retained after instance deletion, CFN child resources). These are real orphans; not tagging them was a Pulumi authoring gap, not evidence of legitimate use.

---

## What we evaluated and decided NOT to use

| Signal | Reason excluded |
|---|---|
| **Resource activity** (CloudWatch Metrics: network traffic, IOPS, read counts) | Per-resource CloudWatch calls add significant latency and cost to every run. Marginal accuracy gain — tenant deletion is already authoritative. |
| **Pulumi state cross-reference** | Requires authenticated access to the Pulumi backend per tenant stack. Expensive, fragile, and unnecessary — if a tenant is deleted, surviving resources are orphans regardless of what Pulumi believes. |
| **Grace period after deletion** | S6 (racing destroy safety) was Declined. Tenants in `TENANT_TARGETS` are confirmed deleted by portal operators. Any surviving resources are orphans, not mid-destroy. The daily schedule (`0 6 * * *`) provides natural time spacing. |
| **Re-check before flagging** | The Phase 4 human approval gate serves this purpose. No pre-cleanup re-verification is needed at discovery time. |

---

## Edge cases — acknowledged, deferred to Phase 4 gate

| Edge case | Assessment |
|---|---|
| Shared resource used by multiple tenants | Rare in this architecture — each tenant has its own VPC and account-level scoping. If it occurs, the human reviewer catches it before any cleanup is approved. |
| Tenant deleted then re-provisioned under the same name | New resources get a new `runId` / `discoveredAt`. Old orphan records remain in inventory. The portal UI (Phase 3) can filter by `tenantDeletedAt` to distinguish old vs new. |
| Resource created outside Pulumi (manually provisioned) | If tagged: found by Resource Explorer (high confidence). If untagged and not reachable from a known anchor: not found — accepted coverage gap; tracked in the supplementary discovery miss-condition table in [[Discovery job implementation]]. |
| Supplementary resource is shared infrastructure | Anchor-walking is tenant-scoped (walks from tenant-owned VPCs, clusters, stacks). Cross-tenant false positives are structurally prevented. False positives within the tenant's own infrastructure are possible but rare; Phase 4 gate catches them. |

---

## Summary

The orphan identification logic is intentionally simple:

> **Deleted tenant + resource found by discovery = orphan candidate. Human approves before any cleanup.**

Complexity lives in Phase 4 (approval workflow, ordering constraints), not in Phase 2 discovery. This keeps the discovery job fast, read-only, and easy to reason about.

---

## Open questions

None blocking this subtask. Remaining design questions are Phase 4 concerns (approval ordering, dependency resolution, cleanup audit trail).
