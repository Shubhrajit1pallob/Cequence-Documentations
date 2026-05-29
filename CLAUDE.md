# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not a code repository** — it is the user's **personal Obsidian vault** and second brain. It tracks all work done at Cequence: research notes, design docs, daily work logs, to-do lists, decision logs, and implementation notes. All files are Markdown; there is no build, test, or lint step.

**This vault is the source of truth for context.** Before asking the user to re-explain something, check the relevant notes here first. The user does not want to repeat themselves — read the vault to get up to speed.

**Important:** This vault is personal work tracking. It is **not** a staging area for `service-portal/ui/features/orphaned-resources/` and its contents should not be treated as drafts destined for that repo. Anything that needs to ship to the Ops Portal repo (decision logs, root-cause docs, etc.) will be authored or copied there separately by the user. Do not propose moving, syncing, or restructuring files in this vault to match the Ops Portal layout.

## Active projects being tracked

### 1. Orphaned Resources Detection & Remediation
Deleted tenants leave behind AWS resources (EBS, ELB, ECR, ENIs, RDS snapshots, etc.) that continue to incur cost. The project delivers, in phases:

1. **Discovery Job** — scheduled scan of both AWS accounts, tenant attribution via tags, structured inventory.
2. **Ops Portal "Orphans" tab** — per-tenant list, cost, last-deleted-at, breakdown by resource type.
3. **Approval-gated cleanup** — Slack + Portal approvals, **no fully-automated destroys**, audit trail.
4. **Monitoring** — Grafana dashboards: orphan count, $ saved, workflow success rates, tag coverage.
5. **Root-cause analysis** — Pulumi leak patterns (missing `forceDestroy`, RDS snapshot retention, ENI cleanup), upstream PRs to infra layers 0–5.

The Ops Portal repo path `service-portal/ui/features/orphaned-resources/` is the eventual home for deliverable artifacts — handled separately, outside this vault.

### 2. PLAT-981 — UAP Upgrade: Auto-update ECK/Strimzi Operator Versions
Adds two new jobs to the existing UAP minor upgrade workflow (`TenantUapMinorUpgradeCoordinator.ts`) so that ECK and Strimzi operator versions in `cluster-services` are updated automatically during a UAP upgrade. Status: **shipped**. Design doc and local testing notes are in `UAP-Workflow/`.

## Vault structure

Always check `Index/Contents.md` first — it is the canonical entry point and lists everything currently in the vault.

- `Index/Contents.md` — top-level table of contents.
- `Problem Statement - Orphaned Resources Detection & Remediation.md` — the canonical spec from the supervisor.
- `Phase N/` (0–6) — one folder per phase, each with `About.md` and `Index.md`. Phase 0 has the most content (research docs on provisioning workflow, GitLab pipelines, job transitions, ArgoCD sync jobs, SaaS tenant pipeline, coordinator-and-workflow).
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
