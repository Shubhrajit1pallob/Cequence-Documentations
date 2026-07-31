# Phase 2 — Phase Docs

See [[Phase 2/About|About]] for what this phase covers.

Subtask breakdown lives in [[Story - Phase 2 — Discovery Workflow|the Phase 2 story]]; running task list in [[ToDo Lists/Phase 2|Phase 2 TODO]].

**Prerequisite:** Phase 1 gate passed (design review sign-off + ADR committed to ops-portal repo).

## Documents

- [[Inventory record schema]] — S4 working doc: schema defined (12 fields); `tenantDeletedAt` + `customerName` being added; cost deliberately excluded (Phase 3 joins from Cost Explorer); mentor review pending.
- [[Inventory persistence decision]] — S3 working doc: DocumentDB (`orphanedresourcerepos`, `saas-portal` DB) chosen; zero new infrastructure; no blockers. Decision log entry pending.
- [[IRSA credential strategy]] — S2 IRSA investigation: full setup guide (OIDC, IAM role, service account annotation), code changes needed, open questions for platform team. Pending senior action.
- [[Credential strategy]] — S2 working doc: cq-pulumi + Descope access key chosen (IRSA env vars stripped); one open question (login frequency) pending senior. Diverges from S1's IRSA+AssumeRole recommendation.
- [[S11 Slack notification template]] — S11 working doc: new `manifests/slacknotificationtemplate.yaml` (`slack-notification` WorkflowTemplate), `onExit: notify` re-enabled, ENV_NAME hardcoded per cluster. Blocker: wrong token in `slack-credentials` secret. Follow-up (verified 2026-07-10): two new mechanisms added — onExit "what failed" detail, and a separate attach-failure fallback alert.
- [[S8 error handling retries notifications]] — S8 working doc: retry logic (`RETRY_MAX_ATTEMPTS`, adaptive backoff), error type classification (throttled/permission/timeout/other), separate API write error counter, improved summary block. Slack notifications deferred to S11.
- [[Discovery job implementation]] — S7/S2/S8 implementation notes: directory structure, inventory schema, dev run COMPLETE, **prod run PENDING**, code changes (safe applied / held on cq-pulumi), open gates, deployment steps.
- [[Workflow platform decision]] — S1 working doc: platform evaluation (Argo CronWorkflow vs Lambda vs ECS vs portal-cron vs K8s CronJob), decision drafted (Argo CronWorkflow), mentor confirmation PENDING. Decision log entry in [[Decision Logs]].

- [[Targets API endpoint]] — S13 (PLAT-1730) implementation notes: **DONE**, verified against code 2026-07-10. Shipped Option A — extended existing `GET /api/external/tenants` with a `deleted` filter (not a new endpoint). Live bug found: `costSinceDeletion` field-name mismatch (real field is `recentCostSinceDeletion`) silently no-ops the cost skip filter.
- [[Tenant data source design decision]] — S13 design decision: **DECIDED** — Option A (extend existing endpoint) chosen over Option B (new purpose-built endpoint).
- [[Agentic attribution evaluation]] — S9 working doc: full blind-spot map (ALB/NLB + ElastiCache highest-cost gaps), agentic approach design (prompt shape, confidence tiers, accuracy measurement), recommendation: defer — expand deterministic VPC anchor-walking first, reassess after production data.
- [[Orphan identification logic]] — S5 working doc: primary criterion (deleted tenant + tagged/supplementary resource = orphan candidate), confidence tiers (`tagged` vs `supplementary`), what was deliberately excluded (activity checks, Pulumi state, grace period), edge cases deferred to Phase 4 human approval gate.
- [[Jira ticket creation]] — S12 (PLAT-1748) implementation notes: **DONE**, verified against code 2026-07-10. Shipped as a TypeScript script (`createTicket.ts`), not the shell/curl or agentic approaches originally scoped — OAuth 2.0 client credentials auth, CSV inventory attachment with retry, fail-loud exit-code redesign.
- [[jira-ticket-creation-approach-comparison]] — S12 design discussion: two agentic approaches compared (claude-tool vs claude-mcp); OPEN — awaiting senior decision. Confluence: 4816896013.

**Agent prompts:**
- [[subtask-1-workflow-platform-prompt|S1 — workflow platform evaluation prompt]] — for the agent working in the Ops Portal repo

**Expected docs (one per design decision + the implementation write-up):**
- Workflow platform decision (S1)
- Credential strategy (S2)
- Inventory persistence decision + schema (S3 + S4)
- Orphan identification logic (S5)
- Racing destroy safety (S6)
- Implementation notes (S7 + S8)
- Agentic attribution evaluation (S9)
- Official deliverable: `service-portal/ui/features/orphaned-resources/phase-2/discovery-workflow-design.md`
