---
tags:
  - ops-portal
  - plat-981
  - uap-upgrade
  - cluster-services
  - decision-log
created: 2026-05-26
status: shipped
jira: PLAT-981
branch: shub/plat-981
---

# PLAT-981 — UAP Upgrade: Auto-update ECK/Strimzi Operator Versions

## Problem

When a tenant is upgraded to a new UAP version, the ECK and Strimzi operator versions in the `cluster-services` Pulumi stack (`Pulumi.{tenantName}-{environment}.yaml`) were never updated automatically. If the new UAP release bundled different operator versions, someone had to manually edit the file and re-run the cluster-services GitLab pipeline — easy to forget, not enforced anywhere.

---

## What Already Existed (no new patterns invented)

- Every UAP release has a `ResourceVersion` record (e.g. `cequence-asp-8.5.0`) in the portal DB. This record already had `eckOperatorVersion` and `strimziOperatorVersion` fields. The data existed — nothing was using it during upgrades.
- `CreateClusterServicesEntryJob` (provisioning) already writes the Pulumi YAML keys: `saas-tenant-cluster-services:eckVersion` and `saas-tenant-cluster-services:strimziVersion`.
- `WaitForClusterServicesPipelineJob` (EKS upgrade path) already triggers and polls the cluster-services GitLab pipeline. Reused unchanged.
- `UpdateApplicationsUapVersionJob` is the structural template: extend `GitJob`, implement `InstantJob`, look up `ResourceVersion`, edit YAML, commit, push.

---

## What Was Built

### New: `UpdateClusterServicesOperatorVersionsJob.ts`

**Location:** `app/_lib/tenant/provisioning/upgrade/uap/`

**Step-by-step logic:**

1. Read `tenant.targetUapUpgradeVersion` — FAIL if missing.
2. Look up `ResourceVersion` for `cequence-asp-{version}` — FAIL if not found.
3. Extract `eckOperatorVersion` and `strimziOperatorVersion` — FAIL if either is unset. _(The RV admin form already gates this; missing = data error, not an intentional omission.)_
4. Clone the `cluster-services` repo to a temp directory.
5. Open `Pulumi.{name}-{environment}.yaml`.
6. Read current `eckVersion` and `strimziVersion` values.
7. If both already match the target → return `SUCCESS` with a no-op message. No commit. The cluster-services pipeline still runs (Pulumi no-op is ~1–2 min).
8. If either differs → update the two keys, write the file back with `sortKeys: false` (preserves key order), commit with message: `"Update ECK/Strimzi operator versions for {name}-{environment} (UAP {version})"`
9. Push — `pushWithOptions()` automatically deletes the local clone after the push.

### Reused Unchanged: `WaitForClusterServicesPipelineJob`

**Location:** `app/_lib/tenant/provisioning/upgrade/EKS/WaitForClusterServicesPipelineJob.ts`

No code changes. Imported into the UAP coordinator at a new position. Pipeline always runs regardless of whether a commit was made.

### Modified: `TenantUapMinorUpgradeCoordinator.ts`

2 new imports, 2 new `createWorkflowEntry` lines. No existing lines reordered.

---

## Final Workflow Order

| # | Job | |
|---|-----|-|
| 1 | `VerifyUapApplicationSyncJob` | |
| 2 | `CreateDatadogDowntimeJob` | |
| 3 | `CreateGrafanaSilenceTrafficJob` | |
| 4 | `UpdateApplicationsUapVersionJob` | push to tenant-apps |
| 5 | `UpdateClusterServicesOperatorVersionsJob` | **NEW** — push to cluster-services |
| 6 | `UpdateKeycloakThemeJob` | |
| 7 | `WaitForTenantPipelineJob` | tenant pipeline (critical path) |
| 8 | `WaitForClusterServicesPipelineJob` | **NEW position** — cluster-services pipeline |
| 9 | `SyncArgoCequenceAspJob` | |
| 10 | `UpdateTenantUapVersionJob` | |
| 11 | `WaitForUapApplicationHealthJob` | |
| 12 | `RemoveAllDatadogDowntimeJob` | |
| 13 | `RemoveAllAlertSilencesJob` | |
| 14 | `TriggerGitlabCallbackJob` | |
| 15 | `SyncTenantFromSourcesJob` | already reconciles operator versions back to tenant record |

**Rationale:** All three git pushes land before any pipeline runs. Cluster-services pipeline runs after the tenant pipeline (critical path), within the silence window, before the health check. `SyncTenantFromSourcesJob` at the end already calls `pulumiProject.getTenantClusterServicesConfig()` so the tenant's `eckOperatorVersion` / `strimziOperatorVersion` are reconciled automatically.

---

## Key Decisions

| Decision | Choice | Reason |
|---|---|---|
| Missing operator versions on RV | FAIL | Data error — RV admin form gates this |
| Pipeline when no commit made | Always run | Pulumi no-op is cheap; avoids framework complexity |
| Pipeline ordering | Sequential after tenant pipeline | Tenant pipeline is critical path |
| New file for UAP pipeline job? | No — reuse EKS class | Identical behaviour, no duplication |

---

## Edge Cases Handled

| Case | Behaviour |
|---|---|
| No `targetUapUpgradeVersion` on tenant | FAILURE |
| UAP `ResourceVersion` not in DB | FAILURE |
| RV missing `eckOperatorVersion` or `strimziOperatorVersion` | FAILURE |
| Pulumi file missing for tenant | FAILURE |
| Versions already match | SUCCESS + no-op message; pipeline still runs |
| Cluster-services pipeline fails | Workflow halts; operator retries |
| Two UAP upgrades back-to-back | Each reads its own `targetUapUpgradeVersion` |
| Manually-pinned operator version | Overwritten with RV value — UAP RV is source of truth |

---

## Tests

**File:** `__tests__/_lib/tenant/provisioning/upgrade/uap/UpdateClusterServicesOperatorVersionsJob.test.ts`

13 tests. All passing.

Coverage: all FAILURE cases, no-op SUCCESS, commit+push SUCCESS, correct commit message, YAML key order preservation.

---

## Future Improvements (noted, not implemented)

- **Parallel pipelines:** run tenant and cluster-services pipelines simultaneously. Needs fan-out/fan-in support in the workflow framework.
- **Conditional pipeline skip:** skip the cluster-services pipeline when no commit was made. Also needs framework support (jobs passing richer signals to the next step).
- **Operator version drift detection:** a read-only audit job that compares the tenant record against the live cluster and alerts if they diverge.

---

## Decision Log

Full decision log entries (4 decisions) are recorded at:
`service-portal/ui/features/orphaned-resources/decisions_logs.md`
under the `[PLAT-981]` label at the top of the file.
