# Tenant Provisioning Workflow

End-to-end flow of how tenant provisioning is triggered and executed.

---

## Trigger Chain

| Step | Location | What happens |
|---|---|---|
| UI "Confirm" | `app/tenants/[tenant]/ProvisionWorkflow.tsx:66` | Calls `startProvision(tenantId, provisionInPausedState)` |
| Server action | `app/tenants/[tenant]/actions.ts:355` | Permission check → reset archive flag → mark paused flag → calls `startWorkflow()` |
| Workflow seeding | `Coordinator.ts:200` | Writes `WorkflowStatus` to MongoDB with all 23 jobs as `PENDING`/`SKIPPED`. Posts initial Slack message. Writes audit log. **Returns to UI immediately — no jobs run yet.** |
| Background driver | `Coordinator.ts:350` `checkPendingJobs` | Runs on `setInterval`, advances jobs one at a time. |
| Workflow definition | `app/_lib/tenant/provisioning/TenantProvisionCoordinator.ts:29` `PROVISION_TENANT_WORKFLOW` | The ordered list of 23 jobs. |

---

## The 23 Jobs

### Phase 1 — Bootstrap CICD
1. **CreateCICDEntryJob** — adds tenant ID to `.tenants.yml` in `cicd-templates` repo

### Phase 2 — Cluster
2. **CreateClusterEntryJob** — cluster DB row + triggers GitLab cluster pipeline
3. **WaitForClusterPipelineJob** — polls GitLab until EKS cluster is built
4. **DownloadKubeConfigJob** — pulls kubeconfig for the new cluster

### Phase 3 — Cluster Services
5. **CreateClusterServicesEntryJob** — triggers `cluster-services` GitLab pipeline
6. **WaitForClusterServicesPipelineJob** — polls until done

### Phase 4 — Tenant Infra
7. **CreateTenantEntryJob** — tenant record + triggers tenant GitLab pipeline
8. **WaitForTenantPipelineJob** — polls until done
9. **TriggerDexPipelineJob** — triggers Dex (SSO/IDP) pipeline

### Phase 5 — ArgoCD Wiring
10. **CreateTenantApplicationEntryJob** — writes ArgoCD Application manifest to Git
11. **SyncArgoParentApplicationJob** — tells ArgoCD to sync → deploys tenant app to K8s

### Phase 6 — Observability Silencing
12. **CreateDatadogDowntimeJob** — Datadog downtime window
13. **CreateGrafanaSilenceTrafficJob** — Grafana silence
14. **SyncArgoGrafanaJob** — syncs Grafana app _(skipped in dev)_
15. **SyncArgoCequenceAspJob** — syncs Cequence ASP app

### Phase 7 — Defender _(skipped if defender disabled)_
16. **CreateDefenderMappingFilesJob** — defender mapping files
17. **CreateDefenderPoolEntryJob** — defender pool entry
18. **WaitForDefenderPoolPipelineJob** — polls pipeline
19. **SyncArgoDefenderPoolJob** — syncs defender pool

### Phase 8 — DAGs
20. **SyncTenantDagsPipelineJob** — syncs tenant Airflow DAGs

### Phase 9 — Health Gate + Finalize
21. **WaitForUapApplicationHealthJob** — waits for UAP app to be healthy in ArgoCD
22. **TriggerPauseTenantWorkflow** — if "provision paused" was selected
23. **SyncTenantFromSourcesJob** — pulls final data from Vitally etc.

---

## Mental Model

```
CICD record
  → Cluster (EKS) + kubeconfig
    → Cluster services
      → Tenant infra + Dex (SSO)
        → ArgoCD wiring (manifests → K8s)
          → Observability silencing (Datadog + Grafana)
            → Defender pool (optional)
              → DAGs
                → Health gate
                  → Done
```

**Repeating pattern:** create DB/Git entry → trigger GitLab pipeline → wait → sync ArgoCD app.

---

## TL;DR

UI confirm → server seeds 23 jobs in Mongo → returns immediately → background `setInterval` loop advances one job at a time → each job is either instant (returns terminal state) or delayed (returns `RUNNING`, coordinator polls until terminal) → full cluster + services + apps deployed in sequence.
