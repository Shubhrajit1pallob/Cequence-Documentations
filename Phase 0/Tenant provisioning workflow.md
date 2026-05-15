## Trigger

**UI action** — `app/tenants/[tenant]/ProvisionWorkflow.tsx:66`
User clicks "Confirm" in the Provision dialog → calls server action `startProvision(tenantId, provisionInPausedState)`.

**Server action** — `app/tenants/[tenant]/actions.ts:355` (`startProvision`)
- Permission check → reset archive flag → mark paused flag → call `startWorkflow(tenantId, PROVISION_TENANT_WORKFLOW)`

**Workflow seeding** — `app/_lib/tenant/provisioning/Coordinator.ts:200` (`startWorkflow`)
- Writes a `WorkflowStatus` document on the tenant in MongoDB with every job in `PENDING` (or `SKIPPED` based on config). No jobs run yet.

**Driver** — `app/_lib/tenant/provisioning/Coordinator.ts:350` (`checkPendingJobs`, run on an interval by `startIntervalWatcher`)
- This background loop is what actually advances jobs one-by-one.

**Workflow definition** — `app/_lib/tenant/provisioning/TenantProvisionCoordinator.ts:29` (`PROVISION_TENANT_WORKFLOW`)

---

## What gets created, in order

> Note: cluster does not come first — a CI/CD entry comes first. Then cluster. Here's the full ordered list with the file controlling each step.

### Phase 1 — Bootstrap CI/CD

1. **CI/CD entry** — `app/_lib/tenant/provisioning/create/CreateCICDEntryJob.ts`
   Creates the CI/CD record in DB that downstream GitLab pipeline jobs reference.

### Phase 2 — Cluster

2. **Cluster DB entry** — `create/CreateClusterEntryJob.ts`
   Creates the Cluster document in MongoDB and triggers the GitLab cluster-provisioning pipeline.
3. **Wait for cluster pipeline** — `create/WaitForClusterPipelineJob.ts`
   Polls GitLab until the cluster pipeline completes (EKS cluster is built).
4. **Kubeconfig download** — `create/DownloadKubeConfigJob.ts`
   Pulls the kubeconfig for the new cluster so subsequent jobs can talk to it.

### Phase 3 — Cluster services (in-cluster platform components)

5. **Cluster Services entry** — `create/CreateClusterServicesEntryJob.ts`
   Triggers the GitLab pipeline that installs cluster-level services (ingress, monitoring agents, etc.).
6. **Wait for cluster services pipeline** — `create/WaitForClusterServicesPipelineJob.ts`

### Phase 4 — Tenant infra in the cluster

7. **Tenant entry** — `create/CreateTenantEntryJob.ts`
   Creates the tenant-specific record and kicks off the tenant GitLab pipeline.
8. **Wait for tenant pipeline** — `create/WaitForTenantPipelineJob.ts`
9. **Dex pipeline** — `create/TriggerDexPipelineJob.ts`
   Triggers the Dex (SSO/IDP) configuration pipeline.

### Phase 5 — ArgoCD application wiring

10. **Tenant Application entry** — `create/CreateTenantApplicationEntryJob.ts`
    Creates the ArgoCD Application manifest entry for the tenant.
11. **Sync Argo parent application** — `create/SyncArgoParentApplicationJob.ts`
    Tells ArgoCD to sync, which deploys the tenant app into the cluster.

### Phase 6 — Observability silencing (so noisy alerts don't fire mid-bringup)

12. **Datadog downtime window** — `CreateDatadogDowntimeJob.ts`
13. **Grafana traffic silence** — `CreateGrafanaSilenceTrafficJob.ts`
14. **Sync Argo Grafana** — `create/SyncArgoGrafanaJob.ts` (skipped in dev)
15. **Sync Argo Cequence ASP** — `create/SyncArgoCequenceAspJob.ts`

### Phase 7 — Defender (skipped entirely if defender disabled)

16. **Defender mapping files** — `create/CreateDefenderMappingFilesJob.ts`
17. **Defender pool entry** — `create/CreateDefenderPoolEntryJob.ts`
18. **Wait for defender pool pipeline** — `create/WaitForDefenderPoolPipelineJob.ts`
19. **Sync Argo defender pool** — `create/SyncArgoDefenderPoolJob.ts`

### Phase 8 — DAGs / Airflow content

20. **Sync tenant DAGs pipeline** — `create/SyncTenantDagsPipelineJob.ts`

### Phase 9 — Health gate + optional pause + finalize

21. **Wait for UAP application health** — `upgrade/uap/WaitForUapApplicationHealthJob.ts`
    Blocks until the tenant's UAP app is reporting healthy in ArgoCD.
22. **Trigger pause workflow** (conditional) — `create/TriggerPauseTenantWorkflow.ts`
    If the user chose "provision in paused state", this kicks off the pause workflow at the end.
23. **Sync tenant from sources** — `SyncTenantFromSourcesJob.ts`
    Pulls in final data from external sources (Vitally, etc.) to mark the tenant fully bootstrapped.

---

## Mental model of "what comes after what"

```
CI/CD record
   ↓
Cluster row → GitLab cluster pipeline → kubeconfig
   ↓
Cluster-services row → GitLab cluster-services pipeline
   ↓
Tenant row → GitLab tenant pipeline → Dex pipeline
   ↓
ArgoCD Application row → Argo sync (deploys tenant app)
   ↓
Datadog/Grafana silences → Argo Grafana sync → Argo ASP sync
   ↓
(if defender enabled) Defender mapping files → Defender pool → pipeline wait → Argo defender sync
   ↓
DAGs sync
   ↓
Wait UAP healthy → (optional) pause → sync from sources → DONE
```

Each "wait" job uses the **DelayedJob pattern** — `start()` kicks off the pipeline and stores a `jobId`, then the background watcher polls `status(jobId)` until terminal (with a 2h hard timeout per PLAT-1441).
