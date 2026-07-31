# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not a code repository** — it is the user's **personal Obsidian vault** and second brain. It tracks all work done at Cequence: research notes, design docs, daily work logs, to-do lists, decision logs, and implementation notes. All files are Markdown; there is no build, test, or lint step.

**This vault is the source of truth for context.** Before asking the user to re-explain something, check the relevant notes here first. The user does not want to repeat themselves — read the vault to get up to speed.

**Important:** This vault is personal work tracking. It is **not** a staging area for `service-portal/ui/features/orphaned-resources/` and its contents should not be treated as drafts destined for that repo. Anything that needs to ship to the Ops Portal repo (decision logs, root-cause docs, etc.) will be authored or copied there separately by the user. Do not propose moving, syncing, or restructuring files in this vault to match the Ops Portal layout.

## Active projects being tracked

### 1. Orphaned Resources Detection & Remediation
Deleted tenants leave behind AWS resources (EBS, ELB, ECR, ENIs, RDS snapshots, etc.) that continue to incur cost. This project covers Phase 0 (investigation), Phase 1 (discovery design), and Phase 2 (discovery workflow). No further phases are tracked in this vault.

The Ops Portal repo path `service-portal/ui/features/orphaned-resources/` is the eventual home for deliverable artifacts — handled separately, outside this vault.

### 2. PLAT-981 — UAP Upgrade: Auto-update ECK/Strimzi Operator Versions
Adds two new jobs to the existing UAP minor upgrade workflow (`TenantUapMinorUpgradeCoordinator.ts`) so that ECK and Strimzi operator versions in `cluster-services` are updated automatically during a UAP upgrade. Status: **shipped**. Design doc and local testing notes are in `UAP-Workflow/`.

## Vault structure

Always check `Index/Contents.md` first — it is the canonical entry point and lists everything currently in the vault.

- `Index/Contents.md` — top-level table of contents.
- `Problem Statement - Orphaned Resources Detection & Remediation.md` — the canonical spec from the supervisor.
- `Phase 0/`, `Phase 1/`, `Phase 2/` — one folder per phase, each with `About.md` and `Index.md`. Phase 0 has the most content (research docs on provisioning workflow, GitLab pipelines, job transitions, ArgoCD sync jobs, SaaS tenant pipeline, coordinator-and-workflow). Phase 1 holds the Discovery Design working docs (all 6 subtasks complete; submitted for review). Phase 2 is the active phase (Discovery Workflow — S1–S11, S13 all tracked with working docs).
- `Stories/` — narrative story breakdowns per phase; each story file has a `Working Docs:` link at the top pointing to the phase's index.
- `ops-portal/provisioning/` — detailed reference docs for the ops-portal provisioning subsystem (generated from agent research). Contains: `Index.md`, `tenant-provisioning-workflow.md`, `provisioning-jobs.md`, `coordinator-and-workflow.md`, `gitlab-pipeline-integration.md`, `cicd-templates-repo.md`, `argocd-sync-jobs.md`. Remaining topics (failure/retry semantics, other workflows catalog, data model) tracked in the folder index.
- `UAP-Workflow/` — all docs related to the UAP upgrade workflow and PLAT-981:
  - `UAP Workflow Documentation and Docs.md` — index for this folder.
  - `UAP Update Workflow.md` — the existing 13-job workflow (coordinator, entry points, guardrails, tenant fields).
  - `What we are Building.md` — high-level summary of the two new jobs added by PLAT-981.
  - `PLAT-981 — UAP Operator Version Auto-Update.md` — full shipped design doc (problem, what was built, workflow order, decisions, edge cases, tests). Has YAML frontmatter.
  - `PLAT-981 — Local Testing.md` — local dry-run setup, seed commands, verified test cases, gotchas. Has YAML frontmatter.
- `Work Logs/` — daily logs named `YYYY-MM-DD.md` (no weekday suffix). Format: Focus, Time, Lunch, What I did, Blockers.
- `Work Logs.md` — index of all daily log entries; update when adding a new log.
- `ToDo Lists/` — running task lists:
  - `Phase 0.md` — project tasks for Phase 0.
  - `Phase 1.md` — project tasks for Phase 1 (Discovery Design). All 6 subtasks complete; submitted for review.
  - `Phase 2.md` — project tasks for Phase 2 (Discovery Workflow). 10 subtasks; S1 IN PROGRESS.
  - `Personal TODO s.md` — personal and work-support tasks (aliases, tooling, bugs to fix, etc.).
- `TODO Lists.md` — index of to-do lists.
- `Agentic Workflow/` — AI/agentic work documentation. Contains `Index.md`, `About.md`, and `Agentic System.md`.
- `Questions I need to ask (Myself, to claude, or/` — folder for questions grouped by audience:
  - `Claude.md` — questions to ask Claude.
- `Investigation Checklist.md` — structured checklist of candidate causes for orphaned resources to investigate.
- `Everyday Tasks.md` — standing daily reminders.
- `Notes.md` — scratch/general notes.
- `Checklist.md` — ad-hoc miscellaneous checklist.
- `Decision Logs.md` — template + destination for decision-log entries. Each entry MUST include an **AI / agentic angle** section.
- `Jira Content - Orphaned Resources Project.md` — mirror of the Jira PLAT epic. Jira itself mirrors `service-portal/ui/features/orphaned-resources/`, not this vault.
- `Pulumi state.md`, `Escalation Plan.md`, `Risk Considerations.md`, `Key Questions for Team.md` — topic notes.
- `Auth Keys.md` — gitignored; **never commit, never read into shared output**.

## Conventions

- Obsidian wikilinks `[[Note Name]]` are used throughout — preserve them when editing; don't rewrite to relative paths.
- Wikilink alias syntax: `[[target|display text]]`.
- Filenames contain spaces and `&` — always quote paths in shell commands.
- Dates use ISO format (e.g. `2026-05-11`); work-log filenames use `YYYY-MM-DD.md`.
- New docs that carry metadata (design docs, decision logs) use YAML frontmatter with `---` delimiters.
- When adding a non-trivial decision, follow the template in `Decision Logs.md` exactly — Context, Options considered, AI / agentic angle, Decision, Open questions.
- `.gitignore` excludes `.obsidian/workspace*`, `.obsidian/cache/`, OS files, and `Auth Keys.md`.

## Working in this vault

- **Read first.** This vault is the second brain. Check relevant notes before asking the user for context — the answer is likely already here.
- Prefer editing existing topic notes over creating new ones. If a new note is needed, add it to `Index/Contents.md` and link it from the relevant folder index.
- New docs go in the folder that matches their topic: UAP work → `UAP-Workflow/`, Phase 0 research → `Phase 0/`, etc. If unsure of placement, ask before creating.
- For code-implementation questions (Pulumi, Ops Portal UI, discovery job code), this vault has design notes only — the code lives in other repos.
- Work log entries: wait for the user to provide the content of what they did. Do not pre-fill with session activity. Keep bullets short (one sentence + doc link where relevant).

## Session State (resume point)

This section is working memory ("RAM") for cross-session continuity. Keep it current: at the end of meaningful work, overwrite it with the latest state so a fresh session can resume with zero re-explaining. Record only the live state — not a running history.

- **Last updated:** 2026-06-17
- **Active thread:** **S10 and S11 remaining** — S1–S5, S7–S9 In Review; S6 Declined; S10 (gate review) To Do; S11 (notificationtemplate.yaml) In Progress. — error handling, retries, and Slack notifications (PLAT-1649). S7 In Review; MR #3 fixes applied (schedule flip + NPM_TOKEN confirmed).
- **Phase 2 subtask status (PLAT-1642 through PLAT-1651, all sub-tasks of PLAT-1558):**
  - S1 PLAT-1642 — Argo CronWorkflow decision — In Review. Vault: `Phase 2/Workflow platform decision.md`. Confluence: 4788092935.
  - S2 PLAT-1643 — Credential strategy (static IAM user) — working doc written. Vault: `Phase 2/Credential strategy.md`. Confluence: 4787830813.
  - S3 PLAT-1644 — Inventory persistence (stdout/apiWriter, MongoDB removed) — working doc written. Vault: `Phase 2/Inventory persistence decision.md`. Confluence: 4788191246.
  - S4 PLAT-1645 — Inventory record schema — In Review. Vault: `Phase 2/Inventory record schema.md`. Confluence: 4789960721.
  - S5 PLAT-1646 — Orphan identification logic — **In Review**. Vault: `Phase 2/Orphan identification logic.md`. Confluence: 4800544836.
  - S6 PLAT-1647 — Racing destroy safety — Declined.
  - S7 PLAT-1648 — Discovery job implementation — **In Review** (dev cluster verified 2026-06-11; prod pending seniors). Vault: `Phase 2/Discovery job implementation.md`. Confluence: 4787732508 (v12).
  - S8 PLAT-1649 — Error handling, retries, notifications — **In Review**. Vault: `Phase 2/S8 error handling retries notifications.md`. Confluence: 4800577556.
  - S9 PLAT-1650 — Agentic attribution evaluation — **In Review** (defer; expand deterministic VPC layer first). Vault: `Phase 2/Agentic attribution evaluation.md`. Confluence: 4800544911.
  - S10 PLAT-1651 — Gate review — NOT STARTED.
  - S11 **PLAT-1699** — Slack notification template — **Done**. `manifests/slacknotificationtemplate.yaml` created (`slack-notification` WorkflowTemplate, `curlimages/curl`, bot token from `slack-credentials/token`). `onExit: notify` re-enabled in both WorkflowTemplates. Blocker: wrong token in `slack-credentials` secret (needs `xoxb-*` value from Pulumi config). Vault: `Phase 2/S11 Slack notification template.md`. Confluence: 4797693969.
  - S12 **PLAT-1748** — Jira ticket creation step — **In Progress**. Argo `create-jira-ticket` step using `curlimages/curl`; fires only when `totalResources > 0`; project CLOPS, issue type "Operate on Customer Environment". Blocked on `jira-credentials` secret. Vault: `Phase 2/Jira ticket creation.md`. Confluence: 4814766100.
  - S13 **PLAT-1730** — Build GET /api/external/orphaned-resources/targets endpoint — To Do. Confluence: 4803690670.
- **Confluence map:** Space Dotstar (id 4173037584, cloudId xangent.atlassian.net). Sibling folders: Phase 0 = 4783210497, Phase 1 = 4783243265, Phase 2 = 4782850054. Phase 2 parent page = **4788125698** (PLAT-1558 linked). All S1–S4 and S7 pages are children of this parent; S8 page to be created here too. Active sprint: **PLAT 20** (id 3479, ends 2026-06-22).
- **S7 key facts (needed for S8 context):**
  - Two-cluster topology: dev cluster = acct 545681961293 / secret `saas-workloads-sdlc-read`; prod cluster = acct 552447114887 / secret `saas-workloads-prod-read`.
  - Credentials: static IAM user (not IRSA, not cq-pulumi). `chainProvider()` in `src/aws/credentials.ts` picks up `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` from env.
  - Manifest split: shared `cronworkflow.yaml` + per-cluster `workflowtemplate-<dev|prod>.yaml`. Argo cannot substitute `{{workflow.parameters}}` into `secretKeyRef.name` — literal required.
  - TENANT_TARGETS env var: currently name-only strings (e.g. `["shub-test"]`); will be replaced by `GET /api/external/orphaned-resources/targets` once that API is built.
  - Write path: `DRY_RUN=true` → stdout (default); `DRY_RUN=false` + `API_URL` → `apiWriter.ts` → `POST /api/external/orphaned-resources`.
  - Prod open questions (not blocking S8): secret key names inside `saas-workloads-prod-read`, prod tenant list, prod regcred, who deploys to prod cluster.
  - Schedule flip (`*/1` → `0 6 * * *`) applied in MR #3 fixes.
- **Phase 1 status:** S1–S6 ✅ ALL DONE. Submitted for review. Official deliverable: `service-portal/ui/features/orphaned-resources/phase-1/discovery-design.md`.
- **Phase 0 carry-overs still open:** (a) close #006 PENDING items (ache tenant JSON, `describe-instances --filters group-id=sg-05bb690f3deeafe80` us-west-1, `saas/ache/defender` secret check); (b) atacadao-prod write-up (#007 next unused); (c) `urvashi-sentiel-dev` (cat #1 2nd instance); (d) `pipeline-002` vault doc not yet updated to ANOMALY TENANT (pending user decision). Jira: PLAT-1629/1630/1631 exist for failure cats #1/#3/#4 — `Jira Content - Orphaned Resources Project.md` note about "NOT yet created" is stale.
- **Failure-category catalog:** #1 = CFN no-retry IAM-role (#001, $0). #2 = false-negative `fileExists` skip → silent orphan (#002, cost). #3 = multi-region secret replica (#003/#004/#005, cost). #4 = ASG surviving-instance DependencyViolation (#006, prod, hard FAILURE + `maybeCorrupt`).
