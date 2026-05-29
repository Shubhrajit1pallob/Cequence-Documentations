# ArgoCD Sync Jobs

_Notes in progress — captures how the ArgoCD sync steps fit into the tenant provisioning workflow._

## Scope

The provisioning workflow contains four Argo sync jobs (see [[Tenant provisioning workflow]] for the full ordered list):

- `SyncArgoParentApplicationJob` — Phase 5 (application wiring)
- `SyncArgoGrafanaJob` — Phase 6 (observability)
- `SyncArgoCequenceAspJob` — Phase 6 (observability)
- `SyncArgoDefenderPoolJob` — Phase 7 (defender)

## Open questions

- What does each `Sync*Job` actually call into Argo? (REST API, CLI, k8s CRD update?)
- Which Argo Application(s) does each one target?
- How does an Argo sync interact with the GitLab pipelines that ran earlier — what's the contract between pipeline output and Argo state?
- What happens on partial-sync failure? Does the workflow retry, or fail the whole tenant?
- How does this run in reverse during destroy?

## Notes
