# Workflow Platform Decision (Subtask 1)

**Status: DECISION DRAFTED** — Argo CronWorkflow chosen; **mentor confirmation PENDING** (S1 not closeable until confirmed). Official decision-log entry in the ops-portal repo (`service-portal/ui/features/orphaned-resources/decisions_logs.md`) is written separately by the repo agent — this is the vault working doc.

Evaluation input: the repo-agent investigation run from [[Phase 2/agent-prompts/subtask-1-workflow-platform-prompt|the S1 agent prompt]] (2026-06-04), run against the local `/saas` checkout (ops-portal + all sibling infra repos).

---

## The question

Where should the Phase 2 discovery workflow run, and how should it be scheduled?

**Constraints:**
- Potentially long-running multi-region scans — duration **unverified** for a full prod multi-region pass (the Phase 1 dev run was 1m1s for 19 tenants; prod tenants have defender pools, more regions, larger footprints)
- Must read the deleted-tenant list from DocumentDB (DocDB SG admits only the cluster SG — `saas-service-portal/src/servicePortalDB.ts:63-70`)
- Must reach **two** AWS accounts read-only (dev 545681961293, prod 552447114887)
- Must not race in-flight destroy workflows

**Evaluated on 8 dimensions:** the story's 5 (operational complexity, team familiarity, credential management, logging/observability, infrastructure fit) + 3 job-specific (runtime limits, DocumentDB network access, deployment/iteration speed).

---

## Key investigation findings

### Argo Workflows is a migration proposal, not a running system
- `ops-portal/argo-workflows/README.md:3` — the directory exists for "migrating the existing custom workflow engine to Argo Workflows 3.7+"; current state = Phase-1 stub templates
- **Zero CronWorkflows exist**; only a provision-tenant WorkflowTemplate stub + reusable task templates (`templates/common/{gitlab-pipeline,argocd-sync,notification}-task.yaml`)
- **No controller install found in any IaC**; whether one was hand-installed on central-ops is UNVERIFIED (kubectl access blocked)
- History: a test file comment suggests there may have been a prior Argo Workflows integration — `approvalHandler.integration.test.ts:7` reads "Mock the resize executor (replaces former Argo Workflows submit client)". The full history is unclear from source alone; worth confirming with the team.
- Reusable assets: Slack/GitLab/ArgoCD task templates, Go/TS job-image examples with ECR CI, `WORKFLOW_CONCURRENCY.md` mutex design (514 lines — feeds S6)

### Credential patterns (feeds S2)
| Pattern | Status | Evidence |
|---|---|---|
| IRSA (OIDC → AssumeRoleWithWebIdentity) | Extensive platform precedent | 6 components in `pulumi-k8s` (e.g. `karpenter.ts:218`, `certManager.ts:127`) |
| Cross-account `sts:AssumeRole` | Precedent exists (per-tenant shape) | `tenant/src/dataExportIam.ts:53-62` |
| Static keys via Pulumi config → k8s secret | The portal's actual pattern | `saas-service-portal/src/config.ts:93-112` |
| Collector CLI creds (SSO/.env) | **Does NOT transfer to headless** | `scripts/orphaned-resources-AWS/src/shared/credentials.ts` |

Any scheduler choice needs a new headless credential path. Cleanest composite with precedent on both halves: **IRSA + sts:AssumeRole into each account**.

### Precedent check (full /saas tree)
| Option | Precedent |
|---|---|
| Argo CronWorkflow | None active; prior client removed |
| Lambda + EventBridge | **None** (only hit is the orphan-cleanup handler *for* Lambdas) |
| ECS / Fargate scheduled task | **None** — no ECS cluster anywhere; all-EKS shop |
| Portal-side cron | **Strong** — `instrumentation.ts:40-115` (CustomerSync 12h, EksVersionSync 7d, UpgradeSchedulePoller) + InstantJob/DelayedJob coordinator (2h `JOB_TIMEOUT_MS`) |
| Plain K8s CronJob *(5th option, discovered during search)* | Yes, with IaC — `pulumi-k8s/src/namespaced-user/namespaceScalerCronJob.ts:32-118` |

---

## Comparison summary (condensed)

| Dimension | Argo CronWorkflow | Lambda + EventBridge | ECS | Portal-side cron | K8s CronJob |
|---|---|---|---|---|---|
| Credentials / cross-account | IRSA claimed but no SA/role in IaC — must build | Native IAM, all net-new | Native, all net-new | Reuses existing static keys | New IRSA role (6 precedents) |
| Observability | `argo logs` only + Slack exit handlers | CloudWatch (no team dashboards) | CloudWatch | **Prom metrics + pino + Slack already wired** | Pod logs → cluster pipeline; alerting needs adding |
| Race vs destroys | Native mutexes — but useless until destroys also move to Argo | Poll portal API/DB | Same | **Best — coordinator runs the destroys** | DocDB freshness check |
| Precedent / familiarity | None active | None | None | **Strong** | Yes (with IaC) |
| Runtime limits | None | **15-min hard cap** | None | 1-replica pod; deploys kill scans | None (own pod) |
| DocumentDB access | Native (in-cluster) | VPC + SG change needed | VPC + SG change | **Native** | Native |
| Iteration speed | Controller bootstrap first, then kubectl-apply | New Pulumi project + CI | New Pulumi + ECS bootstrap | **Fastest** (branch push) | Existing Pulumi pattern + image CI |

Lambda is effectively disqualified (15-min cap vs unverified scan duration + no precedent + DocDB hurdle); ECS has no offsetting advantage in an all-EKS shop.

---

## Decision

**Argo CronWorkflow** — chosen with eyes open about the controller-bootstrap cost and the esPvcResize removal history.

**Rationale:**
1. **Strategic alignment** — the team is planning to adopt Argo Workflows as the official workflow engine (the `argo-workflows/` migration direction). Building the discovery job on Argo produces a working reference implementation and team familiarity ahead of that migration. *Premise = team's stated direction; mentor confirmation PENDING.*
2. **Runtime isolation** — the scan runs in its own pod, surviving portal deploys (the portal-side option's fatal weakness for long scans)
3. **No runtime ceiling** (`activeDeadlineSeconds` configurable) — robust against the unverified prod scan duration
4. **Native primitives** — retry, fan-out (`withItems`), and the concurrency/mutex design already written in `WORKFLOW_CONCURRENCY.md`

**Accepted costs:** one-time controller bootstrap (no controller installed today, no IaC); the mutex benefits don't apply until destroys also run in Argo (race handling needs a DocDB freshness check regardless); the prior Argo client removal (esPvcResize) is unexplained history to ask about.

**Fallback documented below** — plain K8s CronJob; the swap is days, not weeks.

---

## The designated fallback: plain K8s CronJob

### What it is

A CronJob is a **built-in Kubernetes resource** — no controller to install, no platform to bootstrap. Three nested concepts:

```
CronJob ──(on schedule)──▶ Job ──(runs)──▶ Pod (your container)
```

The CronJob holds a cron expression (`0 2 * * *`) and a pod template; at each tick Kubernetes creates a Job; the Job runs the container to completion, retrying per `restartPolicy`/`backoffLimit`. It is "run this container on a schedule" — the Kubernetes-native crontab.

### Team precedent

Two production confirmations:

1. **`pulumi-k8s/src/namespaced-user/namespaceScalerCronJob.ts`** — the team's real, Pulumi-deployed example (pauses/resumes tenant namespaces on a schedule to save dev-cluster costs). Its shape maps 1:1 onto the discovery job:

| Their scaler | The discovery job would be |
|---|---|
| `schedule: stopCron` (line 40) | `schedule: '0 2 * * *'` (nightly scan) |
| `image: registry.gitlab.com/cequence/orcx-team/cqctl/main` (line 10) | the collector image from the ECR CI pipeline |
| `serviceAccount: NAMESPACE_SCALER_SERVICE_ACCOUNT` (line 47) | an IRSA-annotated ServiceAccount for AWS access |
| `env: [{MODE: 'pause'}, {NAMESPACE: …}]` (lines 54–63) | `env: [{ACCOUNT: …}, {REGIONS: …}]` |
| `restartPolicy: 'OnFailure'` (line 48) | same — free retry on crash |
| Pulumi ComponentResource, deployed by `pulumi up` (lines 17–19) | same — **the IaC story Argo lacks** |

2. **cequence-asp Helm templates** — `metrics-cleanup-cronjob.yml` and `metrics-aggregation-cronjob.yml`, deployed to tenant clusters via Helm. Second confirmation the team operates CronJobs in production.

### vs Argo CronWorkflow

Same spot in the matrix on most dimensions (in-cluster → native DocumentDB access, IRSA-able, no runtime ceiling, ~free). The differences:

**What plain CronJob gives that Argo doesn't:**
- **Zero bootstrap** — works today on any cluster; no controller install, no IaC-home debate, no mentor dependency
- **Pulumi deployment precedent** — the scaler proves the team deploys these via `pulumi up`; Argo templates are kubectl-apply with no IaC
- **One less moving part** — nothing between the kubelet and the container

**What it gives up vs Argo:**
- **No DAG / multi-step orchestration** — a CronJob runs one container; Argo's "fan out per region with `withItems`, aggregate, notify" as separate steps with per-step retry becomes a loop over regions inside the program
- **No exit handlers** — Argo's `onExit` → Slack is declarative; a CronJob's code does its own try/finally Slack post (or alert on failed Jobs via Prometheus)
- **No mutexes/semaphores** — only `concurrencyPolicy: Forbid` (don't start tick N+1 while tick N runs) — which, frankly, is the one actually needed
- **No UI** — Argo server's workflow browser vs `kubectl get jobs` + pod logs
- **Weaker run history** — note the scaler sets `successfulJobsHistoryLimit: 0` (line 41), keeping no record of past runs; the discovery job would want nonzero limits

### Why it's the fallback

A plain K8s CronJob is the **degenerate case of the chosen option**: an Argo CronWorkflow with one step ≈ a K8s CronJob with extra machinery. If any of these triggers fire —

1. seniors say the Argo migration is dead,
2. the controller install gets no IaC home,
3. the esPvcResize removal turns out to be a verdict on Argo itself,

— everything else built survives: **the same job image, the same IRSA ServiceAccount, the same DocDB freshness check, the same schedule**. Only the wrapper YAML changes (`kind: CronWorkflow` → `kind: CronJob`) and multi-region fan-out moves from `withItems` into a loop in the code. That's **days, not weeks** — which is what makes it a credible fallback to put in front of a mentor.

### Honest framing for the mentor conversation

If the spike's scope stays "one account, one region, single container, Slack on exit," a plain CronJob would deliver it faster. **The case for Argo is the trajectory** — multi-region DAGs, the team's migration direction, per-step retries — not the first milestone.

---

## Open questions (for the mentor conversation)

1. ~~Is Argo Workflows already installed on the central-ops dev/prod clusters? UNVERIFIED~~ — **CONFIRMED: Argo controller is live on the central-ops dev cluster (Gate 1, verified 2026-06-09).**
2. IaC home for the controller install — proposal: standalone Pulumi project copying the saas-service-portal pattern (StackReference → kubeconfig → Helm release); mentor to confirm.
3. esPvcResize: a test comment (`approvalHandler.integration.test.ts:7`) suggests there was formerly an Argo Workflows integration here — what's the actual history, and does it change anything about this decision?
4. Credential direction: IRSA + sts:AssumeRole composite (recommended) vs reusing portal static keys as spike shortcut (debt).
5. Race handling vs in-flight destroys: DocDB freshness check vs maintenance window (Argo mutexes don't help — destroys don't run in Argo).

---

## Related

- Decision log entry (vault): [[Decision Logs]] — 2026-06-04 entry
- Official decision log (ops-portal repo): `service-portal/ui/features/orphaned-resources/decisions_logs.md` — written by the repo agent
- Investigation prompt: [[Phase 2/agent-prompts/subtask-1-workflow-platform-prompt|S1 agent prompt]]
- Feeds: S2 (credentials), S6 (racing destroys)
