# UAP Update Workflow

"UAP" (Unified Application Platform = `cequence-asp`) has a minor upgrade workflow that is well-implemented across the whole stack.

---

## Coordinator & Jobs

The core is `TenantUapMinorUpgradeCoordinator.ts`, which runs 13 sequential jobs:

1. **VerifyUapApplicationSyncJob** — checks ArgoCD sync before starting
2. **CreateDatadogDowntimeJob** — 14-day monitoring downtime
3. **CreateGrafanaSilenceTrafficJob** — silences Grafana alerts
4. **UpdateApplicationsUapVersionJob** — commits new version to the applications Git repo
5. **UpdateKeycloakThemeJob** — updates Keycloak theme version in Pulumi config (skips if already correct)
6. **WaitForTenantPipelineJob** — waits for the GitLab pipeline
7. **SyncArgoCequenceAspJob** — syncs ArgoCD
8. **UpdateTenantUapVersionJob** — persists new version to MongoDB
9. **WaitForUapApplicationHealthJob** — waits for the ArgoCD app to go healthy
10. **RemoveAllDatadogDowntimeJob** — cleans up downtime
11. **RemoveAllAlertSilencesJob** — cleans up Grafana silences
12. **TriggerGitlabCallbackJob** — fires an optional webhook callback (for external GitLab pipeline integration)
13. **SyncTenantFromSourcesJob** — final sync

---

## Entry Points

| Surface | Location |
|---|---|
| UI (single tenant) | `app/tenants/[tenant]/UapMinorUpgradeWorkflow.tsx` — version picker dialog with workflow progress display |
| UI (bulk) | `app/tenants/BulkUapVersionUpdateWorkflow.tsx` — multi-tenant table with ArgoCD sync status and renewal date warnings |
| Internal API | `POST /api/tenants/bulk-uap-upgrade` |
| External API | `POST /api/external/tenants/{tenant}/upgrade` — Descope-authenticated, supports a `callbackUrl` for GitLab webhooks |

---

## Key Tenant Fields

| Field | Purpose |
|---|---|
| `uapVersion` | Current deployed version |
| `targetUapUpgradeVersion` | Set at start of upgrade, cleared when done |
| `uapUpgradeCallbackUrl` | Optional GitLab webhook URL |

---

## Guardrails

- Blocked by active freeze windows and maintenance windows
- Tenant must be fully provisioned, not paused/deleted/archived, and have no other running workflow
- Requires upgrade permission

---

## Docs & Tests

- Feature spec: `features/minor-upgrades.md`
- GitLab callback doc: `docs/GITLAB_CALLBACK_INTEGRATION.md`
- Good test coverage: coordinator, individual jobs, UI component, API route, and a dedicated regression test file
