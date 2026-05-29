# What We Are Building

We need to add two new jobs to the upgrade workflow, using patterns that already exist in the codebase.

---

## New Job 1 — `UpdateClusterServicesOperatorVersionsJob`

**What it does:** Updates the `Pulumi.{name}-{env}.yaml` file in `cluster-services` with the correct ECK and Strimzi versions for the target UAP release.

### How it works

1. Reads `tenant.targetUapUpgradeVersion` (e.g. `"8.5.0"`)
2. Looks up the `ResourceVersion` record `cequence-asp-8.5.0` from the DB
3. Reads `eckOperatorVersion` and `strimziOperatorVersion` from that record
4. Clones the `cluster-services` git repo
5. Opens `Pulumi.{name}-{env}.yaml`, reads the current ECK and Strimzi version values
6. If both versions already match → returns `skipped`, does nothing (no commit, no pipeline)
7. If either version is different → updates the YAML, commits, pushes
8. If no versions are set on the `ResourceVersion` → returns `skipped` (not every UAP bumps operators)

**Pattern it follows:** Identical structure to `UpdateApplicationsUapVersionJob` — extends `GitJob`, implements `InstantJob`.

---

## New Job 2 — `WaitForClusterServicesUapPipelineJob`

**What it does:** Triggers the `cluster-services` GitLab pipeline and waits for it to finish (so Pulumi actually applies the new operator versions to the cluster).

### How it works

1. Checks if Job 1 was skipped — if so, skips itself too (no point running a pipeline if nothing changed)
2. Otherwise triggers the `cluster-services` GitLab pipeline with `TENANT`, `ENVIRONMENT`, `TRIGGERED_BY` variables
3. Polls the pipeline status until it succeeds or fails

**Pattern it follows:** Near-identical to `WaitForClusterServicesPipelineJob` that already exists in the EKS upgrade path — just with a UAP-specific display name and the skip-propagation logic.

---

## Updated Workflow Order

| # | Job | Status |
|---|-----|--------|
| 1 | `VerifyUapApplicationSyncJob` | existing |
| 2 | `CreateDatadogDowntimeJob` | existing |
| 3 | `CreateGrafanaSilenceTrafficJob` | existing |
| 4 | `UpdateApplicationsUapVersionJob` | existing — push to tenant-apps |
| 5 | `UpdateKeycloakThemeJob` | existing |
| 6 | `UpdateClusterServicesOperatorVersionsJob` | **NEW** — push to cluster-services |
| 7 | `WaitForTenantPipelineJob` | existing — wait for tenant pipeline |
| 8 | `WaitForClusterServicesUapPipelineJob` | **NEW** — trigger + wait cluster-services pipeline |
| 9 | `SyncArgoCequenceAspJob` | existing |
| 10 | `UpdateTenantUapVersionJob` | existing |
| 11 | `WaitForUapApplicationHealthJob` | existing |
| 12 | `RemoveAllDatadogDowntimeJob` | existing |
| 13 | `RemoveAllAlertSilencesJob` | existing |
| 14 | `TriggerGitlabCallbackJob` | existing |
| 15 | `SyncTenantFromSourcesJob` | existing |

### Why this order?

- Steps 4, 5, 6 all do git pushes upfront — all repo changes are committed before any pipeline runs
- Step 7 (tenant pipeline) is the critical path — runs first
- Step 8 (cluster-services pipeline) runs right after, still within the monitoring silence window, before the health check at step 11

---

## Files We Touch

| Action | File |
|--------|------|
| Create | `app/_lib/tenant/provisioning/upgrade/uap/UpdateClusterServicesOperatorVersionsJob.ts` |
| Create | `app/_lib/tenant/provisioning/upgrade/uap/WaitForClusterServicesUapPipelineJob.ts` |
| Modify | `app/_lib/tenant/provisioning/TenantUapMinorUpgradeCoordinator.ts` — add 2 imports + 2 entries to jobs array |

2 new files, 1 small modification. No new repos, no new workflow, no schema changes.
