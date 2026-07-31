# Discovery Job — Implementation Notes (S7)

**Status: IN PROGRESS** — dev cluster manual run complete (2026-06-11); prod cluster run pending; manifest split designed (shared CronWorkflow + per-cluster WorkflowTemplate).

Covers S7 (implement scheduled workflow) and S2 (credential strategy — static IAM user). Also serves as supporting evidence for S1 (Argo CronWorkflow confirmed working).

Related: [[Workflow platform decision]] (S1) · [[Credential strategy]] (S2) · [[Phase 2/Index|Phase 2 Docs]]

---

## What the job does

Headless Node.js job (TypeScript/ESM, Docker image) that runs as an Argo CronWorkflow. Discovers orphaned AWS resources for deleted tenants using AWS Resource Explorer + supplementary `Describe*` calls. Currently logs results to stdout (dry-run). Never deletes anything.

---

## What was built

Location: `ops-portal/scripts/orphaned-resource-discovery-job/`

| File | What it does |
|---|---|
| `src/index.ts` | Job entrypoint — `loadTargetsFromEnv()` parses `TENANT_TARGETS` as a `string[]`, derives `awsAccountId` from the single entry in `ACCOUNT_VIEW_MAP` (hard-errors if map has >1 account) → discover → write inventory (stdout / apiWriter.ts) → exit 0/1 |
| `src/aws/credentials.ts` | `chainProvider()` picks up `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` from env; AssumeRole option remains in code but unused |
| `src/discovery/resourceExplorer.ts` | Resource Explorer search (primary layer) |
| `src/discovery/supplementaryDiscover.ts` | Supplementary (untagged) resource discovery (fallback layer) |
| `src/tenants/deletedTenants.ts` | Exports `DiscoveryTarget` interface only; all mongoose/schema/DB logic removed |
| `src/output/apiWriter.ts` | POSTs to ops-portal API when `DRY_RUN=false` + `API_URL` set; dry-run by default |
| `Dockerfile` | Single-stage `pulumi/pulumi-nodejs` base image; compiles TypeScript |
| `.gitlab-ci.yml` | typecheck → build → push to ECR (Gate 3 pending) |
| `manifests/cronworkflow.yaml` | Schedule + `workflowTemplateRef: orphan-discovery-job`; byte-identical on both clusters |
| `manifests/workflowtemplate-dev.yaml` | Dev cluster values — secret name, account, tenant list, view ARN (verified working 2026-06-11) |
| `manifests/workflowtemplate-prod.yaml` | Prod cluster scaffold — placeholders for secret key names, prod tenant list, prod view ARN |

---

## Two-cluster topology

Each cluster scans exactly one AWS account. One credential provider per run is sufficient — no cross-account `AssumeRole` ever needed.

| Cluster | AWS account | K8s secret |
|---|---|---|
| Dev (central-ops dev) | 545681961293 | `saas-workloads-sdlc-read` |
| Prod (central-ops prod) | 552447114887 | `saas-workloads-prod-read` |

---

## Manifest structure

| File | Purpose | Applied to |
|---|---|---|
| `manifests/cronworkflow.yaml` | Schedule + reference only; byte-identical everywhere | Both clusters |
| `manifests/workflowtemplate-dev.yaml` | Dev env vars, secret name, tenant list, view ARN | Dev cluster only |
| `manifests/workflowtemplate-prod.yaml` | Prod env vars, secret name, tenant list (placeholders) | Prod cluster only |

**Why the split:** Argo cannot substitute `{{workflow.parameters}}` into `secretKeyRef.name` — confirmed from official Argo docs. The secret name must be a literal. WorkflowTemplate is the correct per-cluster boundary; the CronWorkflow stays identical on both clusters and resolves the WorkflowTemplate by name on whichever cluster it runs.

Deploy per cluster:
```bash
kubectl apply -f manifests/cronworkflow.yaml manifests/workflowtemplate-<dev|prod>.yaml -n argo-workflows
```

---

## Verified run

**Dev cluster manual run — 2026-06-11 ✅**
- `shub-test` → 16 tagged resources, 0 accounts failed
- Matches CLI parity
- `DRY_RUN=true` — nothing persisted

---

## Extending discovery coverage

The job has two discovery layers with distinct miss conditions:

| Layer | How it works | Miss condition |
|---|---|---|
| **Tagged** (Resource Explorer) | Finds anything tagged `tag.value:<tenant>` across all resource types | Resource was provisioned without the tenant tag |
| **Supplementary** (anchor-walking) | Walks from known anchors (VPCs → children, EKS clusters → node groups, CFN stacks → children, tenant-prefix → ASGs) | Resource is not a child of a known anchor AND not named after the tenant |

Any resource that is (a) not tagged, AND (b) not reachable from a known anchor, AND (c) not an ASG named after the tenant — is completely invisible. Current examples: untagged S3 buckets, standalone IAM roles not inside a CFN stack, untagged RDS instances.

### Adding a new resource type — two steps, always together

1. **Add the IAM action** to the static IAM user's inline policy.
2. **Update the CLI** (`ops-portal/scripts/orphaned-resources-AWS`) first, then re-copy the relevant section into `src/discovery/supplementaryDiscover.ts`.

**Parity oracle rule:** the CLI is the reference implementation. `supplementaryDiscover.ts` must stay at parity with it. Always update the CLI first, then port.

**The sync hazard:** an IAM action with no corresponding discovery code does nothing. Discovery code without the IAM permission throws `AccessDeniedException` — but the `try/catch` in the supplementary loop swallows this as a warning and **silently skips** those resources. No error is surfaced; the gap is invisible. Both the policy and the code must be updated together.

---

## Production deployment gates

| Gate | What's needed | Status |
|---|---|---|
| **Gate 1** | Argo controller installed on central-ops dev cluster | ✅ Done |

> Prod cluster deployment to be done by a senior engineer. Gates for prod IAM keys, image registry, and regcred tracked under Open Questions — Production below.

---

## Open items

| Item | Status |
|---|---|
| Build ops-portal `POST /api/external/orphaned-resources` | Needed for write path |
| Build ops-portal `GET /api/external/orphaned-resources/targets` | Replaces `TENANT_TARGETS` env var |
| Slack exit handler (`message` → `text` param) | [[PLAT-1699]] — In Progress |
| Switch schedule from `*/1` → `0 6 * * *` | Deferred — do last, after all phases and testing complete |

---

## Open Questions — Production

These items are blocked on senior engineer input and prod cluster access. Prod deployment is not scheduled for this phase.

| Question | Notes |
|---|---|
| What are the secret key names inside `saas-workloads-prod-read`? | Needed to fill `workflowtemplate-prod.yaml` `secretKeyRef.key` fields |
| What is the prod tenant list? | Need actual deleted prod tenants to scan — currently placeholders in `workflowtemplate-prod.yaml` |
| Does `regcred` exist on the prod cluster? Who creates it? | Required image-pull secret for the discovery job Docker image |
| Who owns prod cluster deployment? | Prod apply of manifests to be done by a senior engineer |
