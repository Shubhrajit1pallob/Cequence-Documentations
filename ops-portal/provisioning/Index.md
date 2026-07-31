# ops-portal / Provisioning — Reference Docs

Detailed reference documentation for the tenant provisioning subsystem of the ops-portal.
Code lives in `app/_lib/tenant/provisioning/` in the ops-portal repo.

## Docs

- [[tenant-provisioning-workflow]] — End-to-end flow: trigger chain, 23 jobs in order, mental model
- [[provisioning-jobs]] — What a job is, InstantJob vs DelayedJob, SHA handoff pattern
- [[coordinator-and-workflow]] — Singleton coordinator, polling loop, state machine, workflow catalog
- [[gitlab-pipeline-integration]] — Two trigger paths, status tracking, artifact retrieval, resilience
- [[cicd-templates-repo]] — The `.tenants.yml` registry: structure, operations, role in CI
- [[argocd-sync-jobs]] — Why explicit sync, ArgoCd client, SyncArgoCdApplicationJob base class

- [[failure-and-retry-semantics]] — Job failure handling, operator actions (retry/skip/stop), PLAT-1441 2h timeout

## Remaining topics to document

- **D** — Other workflows catalog (Destroy, Pause/Resume, UAP upgrade, EKS upgrade, Synthetic traffic)
- **E** — Data model (`app/_lib/tenant/tenant.ts` — WorkflowStatus, JobStatus schema)
