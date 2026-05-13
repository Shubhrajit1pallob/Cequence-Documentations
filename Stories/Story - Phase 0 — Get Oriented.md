
**Title:** Phase 0 — Get Oriented

**Description:**

Understand the tenant lifecycle end-to-end (Portal → GitLab pipeline → Pulumi → AWS), study historical deletion failures, and produce a categorized failure-mode list.

**Gate criteria:**

- Can walk through the full lifecycle (provision and destroy)
- Can name 5+ specific resource types likely to leak
- Have a categorized failure-mode list from historical pipeline review

**Documentation:**

- Problem statement: `service-portal/ui/features/orphaned-resources/problem-statement.md`
- Access requests: `service-portal/ui/features/orphaned-resources/access-requests.md`
- Decision log: `service-portal/ui/features/orphaned-resources/decisions.md`
- Failure mode analysis: `service-portal/ui/features/orphaned-resources/phase-0/failure-mode-analysis.md`

**Labels:** `coop-2026`

### Phase 0 Subtasks

All subtasks belong under the "Phase 0 — Get Oriented" story.

---

### Subtask 1

**Title:** Read project plan end-to-end and clone reference repos

**Description:**

Read the full project plan. Clone all reference repos:

- Layers 0–5 (Account Services, Cluster, Cluster Services, Tenant, Applications, Defender Pool)
- Ops Portal (service-portal)
- pulumi-k8s (shared library)
- cequence-asp (Helm chart)
- Applications (ArgoCD + Helmfile)

Confirm read access to each repo.

**Acceptance criteria:**

- [ ] All repos cloned locally
- [ ] Read access confirmed for each (checklist in access-requests.md)

---

### Subtask 2

**Title:** Request and obtain required access

**Description:**

Submit access requests with justification for Phase 0 investigation.

See: `service-portal/ui/features/orphaned-resources/access-requests.md`

**Requests:**

- Descope ops-portal-admin role (for cq-pulumi)
- AWS read access (Cost Explorer, EC2, EKS, ELB, ECR, VPC, etc.)
- DocumentDB read access
- Ops Portal repo write access

**Acceptance criteria:**

- All Phase 0 access granted and documented in access-requests.md

---

### Subtask 3

**Title:** Set up feature directory and commit initial documentation

**Description:**

Create the feature directory in the Ops Portal repo and commit initial markdown files:

- `problem-statement.md`
- `access-requests.md`
- `decisions.md`

**Acceptance criteria:**

- Feature directory exists at `service-portal/ui/features/orphaned-resources/`
- Initial docs committed via MR
- Decision log has first entries

---

### Subtask 4

**Title:** Pair with mentor — walk through one provision and one destroy in dev

**Description:**

Pair session to observe the happy path:

- One tenant provision (Portal → GitLab pipeline → Pulumi up → AWS resources created)
- One tenant destroy (Portal → GitLab pipeline → Pulumi destroy → AWS resources deleted)

Document the lifecycle steps observed.

**Acceptance criteria:**

- Can describe the full lifecycle from Portal trigger to AWS resource creation/deletion
- Understand what each layer (0–5) provisions and in what order
- Understand the destroy sequence (reverse order: 5→0)

---

### Subtask 5

**Title:** Study historical failures — categorize destroy pipeline failure modes

**Description:**

Pull GitLab pipeline history for destroy jobs across layers 0–5. Read the most recent N failed runs and categorize what failed.

Expected failure categories (starting list):

- S3 bucket not empty
- RDS snapshot retention
- IAM dependency blocking deletion
- ENI still attached
- Timeout
- Finalizer stuck
- Manual gate (Blocked pipeline) never approved

**AI lens:** Consider using Claude Code to ingest a batch of failed pipeline logs and produce a categorized summary. Verify a sample manually.

**Acceptance criteria:**

- Reviewed N failed destroy pipelines across layers
- Categorized failure modes documented in `phase-0/failure-mode-analysis.md`

---

### Subtask 6

**Title:** Trace 2–3 orphaned tenants back to pipeline runs

**Description:**

Pick 2–3 tenants from the orphan list and trace each back to its pipeline run to identify which step left resources behind.

**Starting tenants:**

- `carsales` — $2,576/month (highest cost, prod/ap-southeast-2)
- `dmitry-test-1` — internal dev/test (suggested in project plan)

For each tenant:

1. Find the deletion trigger in Ops Portal
2. Locate the GitLab pipeline(s) that ran
3. Check pipeline status — passed, failed, or blocked?
4. If failed: which layer, which step, what error?
5. Compare Pulumi state vs. actual AWS resources
6. Document which resources were left behind and why

**Acceptance criteria:**

- [ ] 2–3 tenants fully traced with findings documented
- [ ] Root cause identified for each

---

### Subtask 7

**Title:** Re-run cost baseline on fresh tenants.csv export

**Description:**

Export a fresh `tenants.csv` from the Ops Portal and re-run the cost baseline analysis. Confirm the orphan list manually in AWS Resource Explorer.

Compare against the 2026-05-09 snapshot:

- 14 orphan tenants / $5,318 monthly
- 17 no-date tenants / $1,758 monthly

**Acceptance criteria:**

- Updated cost baseline documented
- Orphan list confirmed against AWS Resource Explorer
- Any discrepancies noted

---

### Subtask 8

**Title:** Study TenantDestroyCoordinator.ts — understand destroy orchestration

**Description:**

Read through `TenantDestroyCoordinator.ts` to understand how the destroy workflow is orchestrated. 

Document:

- What triggers a destroy?
- What order are layers destroyed in?
- What error handling exists?
- Where could failures leave orphaned resources?

**AI lens:** Consider using Claude Code to walk through the file and produce

a sequence diagram of the destroy flow.

**Acceptance criteria:**

- Destroy orchestration logic documented
- Sequence diagram or flow description produced
- Potential leak points identified

---

### Subtask 9

**Title:** Phase 0 gate review — prepare write-up

**Description:**

Prepare the Phase 0 gate review write-up for mentor and reviewer sign-off. 

Must demonstrate:

- Can walk through the lifecycle (Portal → GitLab → Pulumi → AWS)
- Can name 5+ specific resource types likely to leak
- Categorized failure-mode list from historical pipeline review

Bring 2–3 written options at any decision points rather than committing early.

**Acceptance criteria:**

- Write-up reviewed and signed off by mentor + one reviewer
- Phase 0 gate passed