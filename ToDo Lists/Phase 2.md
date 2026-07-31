# Phase 2 — Discovery Workflow

Build-phase. Subtask details in [[Story - Phase 2 — Discovery Workflow|the Phase 2 story]]; working docs in [[Phase 2/Index|Phase 2 Docs]].

**Prerequisite:** Phase 1 gate passed (design review sign-off + ADR committed).

## Design decisions (Subtasks 1–6)

- [ ] **S1** — Workflow platform: evaluate Argo CronWorkflow vs Lambda vs ECS vs Portal-side cron; confirm with mentor; decision log entry
  - 2026-06-04: evaluation done (repo-agent investigation) + decision drafted (**Argo CronWorkflow**) + vault decision-log entry written ([[Workflow platform decision]]). Repo `decisions_logs.md` entry = repo agent's task. **Remaining: mentor confirmation** (5 open questions in the working doc).
- [ ] **S2** — AWS credentials: evaluate IRSA + AssumeRole for cross-account; scope IAM policy to minimum permissions
- [ ] **S3** — Inventory persistence: evaluate S3 vs DocumentDB vs Postgres; design schema; decision log entry
- [ ] **S4** — Inventory record schema: minimum fields, attribution confidence field, cleanup metadata
  - 2026-06-09: schema defined; `tenantDeletedAt` + `customerName` being added; cost excluded (Phase 3 joins separately); mentor review pending. See [[Inventory record schema]].
- [ ] **S5** — Orphan identification logic: signals (deleted-at + resource exists), edge cases, false positive strategy
- [ ] **S6** — Racing destroy safety: evaluate mutexes / maintenance window / grace period; decision log entry

## Build (Subtasks 7–8)

- [ ] **S7** — Implement the scheduled discovery workflow
  - 2026-06-08: core implementation done (`orphaned-resource-discovery-job/`); dev account local run COMPLETE; **prod account run PENDING** (next step); cq-pulumi login change held pending senior decision. See [[Discovery job implementation]]. (fetch tenants → hybrid scan → attribute → cost → persist)
- [ ] **S8** — Implement error handling, retries, and Slack/PagerDuty failure notifications

## Agentic evaluation (Subtask 9)

- [ ] **S9** — Evaluate agentic untagged-resource attribution on top of deterministic core; decision log entry

## Gate review (Subtask 10)

- [ ] **S10** — Schedule demo with mentor; live run against dev; spot-checks vs Resource Explorer; no destroys
- [ ] Phase 2 gate passed
