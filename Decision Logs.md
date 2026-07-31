Sample File structure- 

```
## 2026-05-11 — Project documentation strategy

**Context:** Starting the orphaned resources project. Need to decide 
where and how to track progress.

**Options considered:**
- A: Jira-only tracking (familiar, visible)
- B: Git markdown only (detailed, reviewable, LLM-readable)
- C: Both — git as source of truth, Jira as public mirror

**AI / agentic angle:** Used Claude to draft initial problem statement 
and structure deliverables. Verified against supervisor's "what done 
looks like" spec. AI was useful for organizing the scope but the 
cost baseline and tech stack details came from internal docs/screenshots.

**Decision:** Option C. Markdown in 
`service-portal/ui/features/orphaned-resources/` is the source of truth. 
Jira PLAT epic mirrors status for stakeholder visibility.

**Open questions / next review:** Need to confirm Ops Portal repo 
write access to start committing docs.
```


The file will in be in the following folder: 
```
ops-portal/service-portal/ui/features/orphaned-resources/decision-logs/
```

---

## 2026-07-10 — S13 decided: extend existing tenant endpoint (Option A); S12 shipped as TypeScript, not agentic

**Context:** S13 (PLAT-1730) and S12 (PLAT-1748) both shipped since the last log entry. This entry closes out the open decisions from the 2026-06-22 (S13) and 2026-06-30 (S12 comparison) entries. Full details: [[Targets API endpoint]], [[Tenant data source design decision]], [[Jira ticket creation]].

**S13 — tenant data source:** Option A shipped — the discovery job now calls the existing `GET /api/external/tenants?environment=<env>&deleted=true` (not a new purpose-built endpoint as Option B proposed). `TENANT_TARGETS` remains as fallback. A live, unpatched bug was found in this work: `apiTargets.ts` reads a `costSinceDeletion` field that doesn't exist on the real response — the actual field is `recentCostSinceDeletion` — so the "skip zero-cost deleted tenants" filter is currently a silent no-op.

**S12 — Jira ticket creation:** Neither `claude-tool` nor `claude-mcp` from the agentic comparison ([[jira-ticket-creation-approach-comparison]]) was used. The team shipped a plain TypeScript implementation (`createTicket.ts`) once the ticket description was redesigned as a summary (fixing a 32,767-character Jira description limit bug) rather than a full per-resource table — removing the need for Claude to do any formatting judgment. Auth moved to OAuth 2.0 client credentials after Basic Auth and Bearer-token attempts both failed. A CSV inventory attachment (with retry + Argo-artifact fallback) and a fail-loud exit-code redesign were added.

**AI / agentic angle:** The agentic comparison work (2026-06-30) turned out to be moot for S12 — once the root cause of the original ADF-formatting problem was fixed structurally (summarize instead of listing every resource), the need for an LLM to do ticket formatting disappeared. This is a useful pattern to remember: before reaching for an agentic solution to a formatting/generation problem, check whether the underlying data shape can be redesigned to avoid needing generation at all.

**Decision:** S13 = Option A (extend existing endpoint). S12 = plain TypeScript implementation, no agentic layer. Both verified against the current codebase on 2026-07-10 — all claims from the implementation report confirmed except the `costSinceDeletion` field bug, which was confirmed as a real, still-open defect.

**Open questions / next review:** Patch the `costSinceDeletion` → `recentCostSinceDeletion` field name in `apiTargets.ts`. Confirm with ops-portal team whether the separately-found "tenants incorrectly flagged deleted" bug (DocumentDB read-replica lag suspected) has been resolved and on which side.

---

## 2026-06-30 — Jira ticket creation approach: agentic comparison open for senior decision

**Context:** Phase 2 S12 (PLAT-1748) requires a post-discovery Argo step to create a Jira ticket in CLOPS after every run that finds orphaned resources. A pure shell/curl approach was rejected (ADF formatting is unreliable from a shell script). Two agentic approaches using the Claude API were evaluated. Full comparison: [[Phase 2/jira-ticket-creation-approach-comparison]].

**Options considered:**
- A (claude-tool): Stable Anthropic `tool_use` API — Claude formats the ADF ticket body and fills in parameters; the Argo pod executes the Jira REST call directly. Two HTTP calls; team owns the schema and the Jira call. No OAuth lifecycle. Unblocked today.
- B (claude-mcp): Beta Anthropic MCP connector (`mcp-client-2025-11-20`) — Claude calls Atlassian's remote MCP server directly; the Anthropic backend executes the Jira call. One HTTP call from the pod; no tool schema; OAuth token lifecycle required. Blocked — Atlassian MCP URL and OAuth owner not yet confirmed for this environment.

**AI / agentic angle:** Both approaches use the Anthropic Messages API as the AI layer. In Option A, Claude is used for ADF formatting and judgment via `tool_use` — a stable production API. In Option B, Claude acts as an orchestrator calling Atlassian's MCP server via a beta MCP connector, with Anthropic's backend managing the connection. Option B is the more agentic pattern but depends on external infrastructure not yet confirmed for this environment. The comparison document itself was drafted using an agent with access to the Anthropic API docs and the current S12 implementation context. Remote MCP support in the Anthropic API was confirmed via live doc research during this session.

**Decision:** PENDING — awaiting senior input on four open questions: (1) Atlassian MCP server URL, (2) OAuth lifecycle ownership, (3) beta risk appetite for a production CronWorkflow, (4) data retention concerns (Approach B is not ZDR eligible).

**Open questions / next review:** See [[Phase 2/jira-ticket-creation-approach-comparison]] open questions section. Confluence: https://xangent.atlassian.net/wiki/spaces/Dotstar/pages/4816896013

---

## 2026-06-17 — Agentic attribution evaluation: defer, expand deterministic layer first

**Context:** Phase 2 S9 asked whether an LLM agent could attribute resources in the discovery job's blind spot — resources that are neither tagged with a tenant name nor reachable from any structural anchor (VPC → children, EKS → node groups, CFN → children, ASG prefix). Full evaluation: [[Agentic attribution evaluation]].

**Options considered:**
- A: Build the agentic layer now — batch blind-spot resources per run, prompt an LLM to attribute each to a deleted tenant by name pattern + creation timestamp + related resource metadata. Handle low-confidence answers via `needsReview` flag, Phase 4 human gate as backstop.
- B: Expand the deterministic layer first, then reassess — the two highest-cost blind spots (ALB/NLB and ElastiCache) are VPC-scoped and solvable deterministically by adding `DescribeLoadBalancers` and `DescribeCacheClusters` to the existing VPC anchor-walking loop in `supplementaryDiscover.ts`. ~30-line addition, zero false-positive risk.
- C: Do nothing — accept the current blind spot and close S9 without further action.

**AI / agentic angle:** The evaluation was agent-executed: read `supplementaryDiscover.ts` to map current coverage, enumerated the full blind-spot resource set with naming patterns and cost tiers, designed the agentic prompt shape and confidence-handling strategy, estimated LLM cost per run (negligible at any model price given volumes), and proposed an accuracy measurement approach using tagged resources as ground truth. The key finding was agent-produced: the most expensive blind-spot items (ALB/NLB, ElastiCache) are deterministically solvable without any LLM. The agentic approach is coherent but its value depends on how large the residual blind spot is after the deterministic fix — which can only be measured from real production runs.

**Decision:** Option B — expand the deterministic layer first. Add `DescribeLoadBalancers` (ELBv2) and `DescribeCacheClusters` (ElastiCache) to the VPC anchor-walking loop in `supplementaryDiscover.ts`. After that expansion and 2–4 weeks of production data, reassess whether the remaining blind spot (largely free/near-free resource types) justifies an agentic layer. S9 transitioned to In Review; agentic approach formally deferred to Phase 3 or later.

**What would change this decision:**
- Production data shows the remaining blind spot (post-deterministic expansion) is generating significant orphan spend
- A resource type emerges with no naming pattern and no structural anchor — the only remaining case where an agent adds value over deterministic code

**Open questions / next review:** Track the ALB/NLB + ElastiCache deterministic expansion as a separate task (update CLI first per parity oracle rule, then port to the job).

---

## 2026-06-17 — S6 Declined: racing destroy safety not implemented

**Context:** Phase 2 S6 asked how to handle the race condition where the discovery job fires while a tenant destroy workflow is mid-flight, potentially flagging mid-destroy resources as orphans. Four mechanisms were evaluated. The subtask was ultimately Declined rather than implemented.

**Options considered:**
- A: Grace period — don't flag resources as orphaned until N days after `deletedAt`; simple but arbitrary and delays legitimate detection
- B: Tenant-keyed Argo mutex — block the discovery CronWorkflow if a destroy workflow holds the mutex; requires destroy workflows to also participate in the mutex, which they currently don't
- C: Destruction pipeline status check — query GitLab / portal for the tenant's last destroy pipeline status before scanning; adds a cross-system dependency and the pipeline status is not always reliable (failure categories #1/#4 from Phase 0 confirm pipelines can succeed while resources survive)
- D: Do nothing (rely on existing safeguards) — `TENANT_TARGETS` contains tenants confirmed deleted by portal operators; the daily `0 6 * * *` schedule provides natural time spacing; the Phase 4 human approval gate is the last line of defence before any cleanup is approved

**AI / agentic angle:** The decision was reached by reasoning through the Phase 0 failure categories — specifically that pipeline SUCCESS is not a reliable signal of clean teardown (category #1: CFN no-retry IAM role; category #4: ASG DependencyViolation with `maybeCorrupt`). An agent surfacing this pattern from the investigation archive was the input that made option C obviously weak. Option D is defensible precisely because Phase 4 is a human gate: false positives from a rare race survive into the inventory and are caught by a human before any delete action is taken.

**Decision:** Option D — no mechanism implemented. Rationale: (1) `TENANT_TARGETS` are operator-confirmed deleted tenants, not candidates; (2) the daily schedule makes a same-day race statistically unlikely; (3) Phase 4 human approval means any false positive is survivable and doesn't cause data loss or accidental deletion. The complexity of options A–C is not justified for a read-only discovery job. Subtask transitioned to Declined in Jira (PLAT-1647).

**Open questions / next review:** If Phase 4 automation (no human gate) is ever considered, revisit option A (grace period) at minimum.

---

## 2026-06-17 — Write path pivot: MongoDB removed → stdout + ops-portal API

**Context:** The 2026-06-09 decision log entry selected DocumentDB (`orphanedresourcerepos`, `saas-portal` DB) as the inventory persistence layer, with the job writing directly via an inline Mongoose schema. During S7 implementation this approach was revisited and replaced. This entry supersedes the 2026-06-09 entry.

**Options considered:**
- A: Direct MongoDB write (original decision) — job connects to DocDB, writes rows via inline Mongoose schema mirroring `orphanedResourceRepo.ts`. Requires `MONGODB_HOST/USER/PASSWORD` secrets in the cluster, a network path from `argo-workflows` ns to DocDB, and keeping two schema copies in sync.
- B: ops-portal external API write (`apiWriter.ts`) — job POSTs discovered rows to `POST /api/external/orphaned-resources`; the ops-portal service owns the write into `orphanedresourcerepos`. Job needs no DB credentials, no Mongoose, no network path to DocDB — only `API_URL`. Decouples the discovery job from the portal's data layer.
- C: Stdout-only dry run — default mode (`DRY_RUN=true`); rows logged to stdout, nothing persisted. Used while the ops-portal API endpoint is not yet built.

**AI / agentic angle:** The pivot was identified during implementation when mapping the operational dependencies of option A: DocDB network access from `argo-workflows` ns, a secret with DB credentials, and a schema copy to maintain. Recognising that the ops-portal already owns the canonical schema and already has an external-API pattern (`app/api/external/`) made option B the obvious separation of concerns. The agent reviewing the ops-portal codebase confirmed the external API pattern exists and is the correct boundary.

**Decision:** Option C now (dry run, stdout), transitioning to Option B (apiWriter → ops-portal API) once `POST /api/external/orphaned-resources` is built in Phase 3. MongoDB is removed from the job entirely — no Mongoose, no DB secrets, no DocDB dependency. `DRY_RUN=false` + `API_URL` env vars activate the write path. The ops-portal service owns all writes into `orphanedresourcerepos`; the discovery job is stateless with respect to persistence.

**Open questions / next review:** `POST /api/external/orphaned-resources` must be built in Phase 3 before the write path can be enabled in production. Until then all runs are effectively dry runs regardless of the `DRY_RUN` flag's intent.

---

## 2026-06-11 — Credential strategy pivot: cq-pulumi → IRSA

**Context:** Phase 2 S2 selected cq-pulumi with a Descope access key as the credential path for the discovery job (`orphan-discovery-creds` k8s secret). When attempting to implement this in the running cluster, it was determined that cq-pulumi grants permissions that are too broad for a pod whose sole function is read-only AWS resource discovery. The principle of least privilege requires a credential path scoped specifically to the job's required read operations. Full investigation: [[IRSA credential strategy]].

**Options considered:**
- A: cq-pulumi (Descope access key) — the original choice; blocked because it grants scope well beyond what a read-only discovery job requires. Not appropriate for a single pod/container.
- B: Static keys (long-lived AWS credentials stored as a k8s secret) — avoids the cq-pulumi scope problem, but credentials are long-lived and require manual rotation; lives in etcd.
- C: IRSA (IAM Roles for Service Accounts) — Kubernetes-native; the pod assumes a scoped IAM role via the cluster's OIDC identity provider. Credentials are temporary and auto-rotated. Trust policy can be locked to exactly one service account in one namespace. No secrets stored in the cluster. AWS SDK picks up credentials automatically through the standard provider chain.

**AI / agentic angle:** Research into IRSA implementation was agent-executed: 100 agent calls across 18 sources, 77 claims extracted, 25 adversarially verified, 6 killed, 8 confirmed findings. Every implementation step carries a source citation and a verification count. The research surfaced a critical detail: the SDK credential chain checks env vars before IRSA (6th in precedence) — meaning any static key present in the pod will shadow the IRSA credentials. This is why `index.ts` currently deletes `AWS_WEB_IDENTITY_TOKEN_FILE` and `AWS_ROLE_ARN` for the cq-pulumi path; removing those deletions is all that's needed to switch to IRSA.

**Decision:** Option B — static IAM user with minimum read-only permissions. Access keys stored as a k8s secret in the `argo-workflows` namespace and mounted as env vars (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) in the CronWorkflow. IRSA was evaluated (see [[IRSA credential strategy]]) but the additional platform overhead (OIDC provider registration, IAM role trust policy, service account annotation) was not justified for the current scope. Static keys are simpler to set up and adequate for a read-only job; they require a rotation policy.

**Open questions / next review:** Define a key rotation policy. IRSA remains the right long-term path if the job scope expands or security posture tightens.

---

## 2026-06-09 — Inventory persistence: DocumentDB (`orphanedresourcerepos`)

**Context:** Phase 2 needs to persist the orphan inventory so the Phase 3 Portal UI and Phase 4 cleanup workflow can read from it. The story listed four options: S3 bucket, DocumentDB, Postgres, and the portal's existing data store. The discovery job implementation made this decision in practice before the formal design review. Full working doc: [[Inventory persistence decision]].

**Options considered:**
- A: S3 bucket — simple and cheap but not queryable; Phase 3 UI needs `find({ tenantName: X })`, not file downloads
- B: DocumentDB (`saas-portal` DB) — the portal's existing data store; tenants, clusters, defender pools, and audit logs all live there; already the team's established pattern (`app/_lib/tenant/tenantRepo.ts`, etc.)
- C: Postgres — relational and queryable but doesn't exist in the current stack; new infrastructure cost
- D: Portal's existing store — DocumentDB *is* the portal's existing store; B and D are the same answer

**AI / agentic angle:** The evaluation was completed by reading the existing portal codebase patterns and the implemented job code, with agent-assisted cross-referencing of `orphanedResourceRepo.ts` indexes against the Phase 2 story's questions. No novel reasoning required — the implementation had already answered every story question; this entry documents the rationale.

**Decision:** Option B — DocumentDB, `orphanedresourcerepos` collection, `saas-portal` DB. Read pattern: by tenant (`awsAccountId` + `tenantName`), by recency (`discoveredAt` desc), by run (`runId`) — three indexes already defined. Data volume: ~400 docs/run (379 resources confirmed in the Phase 1 live dev run). Zero new infrastructure; native in-cluster network access; familiar to the team.

**Open questions / next review:** Retention policy — the schema has no TTL index. Options: keep last N runs, keep all (data is small), or TTL by `discoveredAt`. Decide with mentor before S3 can be fully closed.

---

## 2026-06-04 — Discovery workflow runtime: Argo CronWorkflow vs Lambda vs ECS vs portal-side cron

**Context:** Phase 2 needs a scheduled job that scans the two AWS accounts (dev 545681961293, prod 552447114887) for orphaned resources, attributes them to deleted tenants, and writes an inventory. It must run potentially long-running multi-region scans (full prod duration unverified — the Phase 1 dev run was 1m1s for 19 tenants), read the deleted-tenant list from DocumentDB, reach both accounts read-only, and not race in-flight destroy workflows. A repo-wide investigation (ops-portal + all sibling infra repos, agent-executed) evaluated four runtimes on 8 dimensions: the story's 5 (operational complexity, team familiarity, credential management, logging/observability, infrastructure fit) plus 3 job-specific (runtime limits, DocumentDB network access, deployment/iteration speed). Full working doc: [[Workflow platform decision]].

**Options considered:**
- A: Argo CronWorkflow — `argo-workflows/` is a migration PROPOSAL (README:3, Phase-1 stub templates; zero CronWorkflows; manual kubectl deploy; no IaC; no controller install found in any repo). Reusable assets exist: Slack/GitLab/ArgoCD task templates, Go/TS job-image examples with ECR CI, WORKFLOW_CONCURRENCY.md mutex design. History: an Argo submit client for esPvcResize was built and REMOVED (approvalHandler.integration.test.ts comment). No runtime limits; in-cluster so DocumentDB reachable natively.
- B: Lambda + EventBridge — NO precedent anywhere in the IaC. 15-minute hard runtime cap conflicts with potentially long scans. DocumentDB requires VPC attachment + SG change (DocDB SG admits only the cluster SG, `saas-service-portal/src/servicePortalDB.ts:63-70`).
- C: ECS scheduled task — NO precedent; no ECS cluster exists; all-EKS shop; highest bootstrap cost with no offsetting advantage.
- D: Portal-side cron — STRONG precedent (`instrumentation.ts:40-115` setInterval schedulers: CustomerSync 12h, EksVersionSync 7d; InstantJob/DelayedJob coordinator with 2h JOB_TIMEOUT_MS). Native DocumentDB + destroy-state awareness. Weakness: a long scan inside the 1-replica Next.js pod dies on every deploy. (Investigation also surfaced plain K8s CronJob as a fifth option with Pulumi precedent — `pulumi-k8s/src/namespaced-user/namespaceScalerCronJob.ts:32-118`.)

Credential context (applies to all): collector CLI creds are SSO/.env (`scripts/orphaned-resources-AWS/src/shared/credentials.ts`) and do NOT transfer to headless; platform pattern is IRSA (6 components in pulumi-k8s, e.g. `karpenter.ts:218`); cross-account AssumeRole shape exists (`tenant/src/dataExportIam.ts:53-62`).

**AI / agentic angle:** The evaluation itself was agent-executed: parallel repo agents swept ops-portal + sibling infra repos; every matrix cell carries a file:line citation or an UNVERIFIED tag; negative findings state searched scope. Note for the future: a CronWorkflow emitting structured inventory is a clean substrate for a later agentic attribution pass over the unattributable bucket (per the 2026-06-03 decision).

**Decision:** Option A — Argo CronWorkflow. Rationale: strategic alignment with the team's planned argo-workflows migration — building the discovery job on Argo produces a working reference implementation and team familiarity ahead of that migration; runtime isolation for long scans (survives portal deploys); no runtime ceiling; native retry/fan-out/concurrency primitives (withItems, mutexes). Chosen WITH eyes open about the one-time controller-bootstrap cost the alternatives avoid and the esPvcResize removal history. **Mentor confirmation: CONFIRMED** — Argo controller verified installed on central-ops dev cluster (2026-06-11 manual run succeeded); S1 and S7 both In Review.

**Open questions / next review:**
1. Is Argo Workflows already installed on the central-ops dev/prod clusters? UNVERIFIED — kubectl access blocked (neither SSO role mapped in aws-auth); no trace in IaC but hand-install can't be ruled out.
2. IaC home for the controller install — proposal: standalone Pulumi project copying the saas-service-portal pattern (StackReference → kubeconfig → Helm release); mentor to confirm.
3. esPvcResize: why was the prior Argo client removed, and does that history change anything?
4. Credential direction: IRSA + sts:AssumeRole composite (recommended) vs reusing portal static keys as spike shortcut (debt).
5. Race handling vs in-flight destroys: DocDB freshness check vs maintenance window (Argo mutexes don't help — destroys don't run in Argo).

---

## 2026-06-03 — Discovery approach: deterministic hybrid vs agentic

**Context:** Phase 1 design work. Need to pick the discovery strategy for the orphaned-resources detection job. Four approaches evaluated: RE-only, AWS Config, Cost Explorer + SDK, and the existing hybrid (RE + per-service fallbacks). A fifth dimension — agentic discovery — was also evaluated per the supervisor's AI lens requirement.

**Options considered:**
- A: AWS Resource Explorer only — tag-indexed, fast, free, but blind to 4.2% of orphaned resources that are systematically untagged by design
- B: AWS Config aggregator — authoritative full coverage, but $10s–$100s/mo cost, 3–5 days setup, disproportionate to a bounded gap
- C: Cost Explorer + per-service SDK — cost-first, but misses $0-cost orphans (IAM roles, CFN stacks, route tables) and has 24–48h attribution lag
- D: Hybrid (RE + per-service fallbacks) — already built, tested against 19 dev tenants (379 resources, 1m1s runtime), covers all 3 confirmed systematically untagged classes
- E: Agentic — LLM agent plans which APIs to call per tenant rather than deterministic enumeration

**AI / agentic angle:** Evaluated agentic discovery seriously. An agent reasoning about resource lineage (VPC → subnet → ENI → instance → volume) could handle novel tag schemes and unknown resource types without pre-coded fallback logic. However, three signals pointed against it as the primary approach: (1) auditability — orphan remediation involves approval-gated deletion of real resources; a non-deterministic inventory that can't be replayed is a liability; (2) the untagged scope is already bounded and well-understood (3 confirmed resource types), so an agent's flexibility adds cost without proportional benefit; (3) the deterministic hybrid already exists with live evidence. The right place for an agent is the **unattributable bucket** — once the deterministic scan runs, any resource with no tag and no relationship anchor can be handed to an agent to reason about likely ownership using naming patterns, creation timestamps, and resource relationships.

**Decision:** Option D — hybrid approach. Lowest implementation cost (already built), evidence-backed (16 fallback-found resources confirmed in live run), free to run, minimal IAM scope. Config and Cost Explorer don't justify their overhead. Agentic deferred to a layered addition on the unattributable bucket, not the core scan.

**What would change this decision:**
- A new systematically untagged resource class emerges with no relationship anchor (nothing to walk from) — Config or an agent would be needed
- The unattributable bucket grows to significant cost impact — agent layer becomes justified
- Prod account scan reveals resource types not covered by the current fallbacks

**Open questions / next review:** Formalize in ADR after design review with mentor + senior engineer. Prod account coverage deferred (no access in Phase 1). Phase 2 additions recommended: live-state verification for RE echoes, ENI fallback, agentic unattributable classifier.


