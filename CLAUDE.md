# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not a code repository** — it is the user's **personal Obsidian vault** for tracking their work on the **Orphaned Resources Detection & Remediation** project at Cequence. All files are Markdown; there is no build, test, or lint step.

**Important:** This vault is personal work tracking. It is **not** a staging area for `service-portal/ui/features/orphaned-resources/` and its contents should not be treated as drafts destined for that repo. Anything that needs to ship to the Ops Portal repo (decision logs, root-cause docs, etc.) will be authored or copied there separately by the user. Do not propose moving, syncing, or restructuring files in this vault to match the Ops Portal layout.

## Project context (what the work is about)

Deleted tenants leave behind AWS resources (EBS, ELB, ECR, ENIs, RDS snapshots, etc.) that continue to incur cost. The project delivers, in phases:

1. **Discovery Job** — scheduled scan of both AWS accounts, tenant attribution via tags, structured inventory.
2. **Ops Portal "Orphans" tab** — per-tenant list, cost, last-deleted-at, breakdown by resource type.
3. **Approval-gated cleanup** — Slack + Portal approvals, **no fully-automated destroys**, audit trail.
4. **Monitoring** — Grafana dashboards: orphan count, $ saved, workflow success rates, tag coverage.
5. **Root-cause analysis** — Pulumi leak patterns (missing `forceDestroy`, RDS snapshot retention, ENI cleanup), upstream PRs to infra layers 0–5.

The Ops Portal repo path `service-portal/ui/features/orphaned-resources/` is the eventual home for deliverable artifacts (decision logs under `decision-logs/`, root-cause docs, etc.) — but that's handled separately, outside this vault.

## Vault structure

- `Index/Contents.md` — top-level table of contents; use this as the entry point when orienting.
- `Problem Statement - Orphaned Resources Detection & Remediation.md` — the canonical "what done looks like" spec from the supervisor.
- `Stories/` — phased story breakdowns (Phase 0 → Phase 6); Phases 0–5 are the committed path, Phase 6 is stretch.
- `Work Logs/` — daily logs (`YYYY-MM-DD.md`).
- `ToDo Lists/` — running task lists.
- `Decision Logs.md` — template + destination for decision-log entries. Each entry MUST include an **AI / agentic angle** section (whether AI was tried, prompt shape, how output was verified, or why AI wasn't the right tool).
- `Agentic Workflow/` — folder for agentic / AI-assisted work. Holds `Agentic System.md` (where AI/agentic approaches are high-fit vs. low-fit) plus per-experiment notes as they're created.
- `Jira Content - Orphaned Resources Project.md` — mirror of the Jira PLAT epic. Jira itself mirrors the Ops Portal folder `service-portal/ui/features/orphaned-resources/`, not this vault. This vault is the user's personal work tracking.
- `Pulumi state.md`, `Discovery Job.md`, `Approval Workflow.md`, `Escalation Plan.md`, `Risk Considerations.md`, `Key Questions for Team.md`, `Questions.md` — topic notes.
- `Auth Keys.md` — gitignored; never commit, never read into shared output.

## Conventions

- Obsidian wikilinks `[[Note Name]]` are used throughout — preserve them when editing; don't rewrite to relative paths.
- Filenames contain spaces and `&` — always quote paths in shell commands.
- Dates use ISO format in front-matter / decision logs (e.g. `2026-05-11`); work-log filenames use `YYYY-MM-DD (Weekday).md`.
- When adding a non-trivial decision, follow the template in `Decision Logs.md` exactly — Context, Options considered, AI / agentic angle, Decision, Open questions.
- `.gitignore` excludes `.obsidian/workspace*`, `.obsidian/cache/`, OS files, and `Auth Keys.md`.

## Working in this vault

- Prefer editing existing topic notes over creating new ones; if a new note is needed, add it to `Index/Contents.md`.
- For code-implementation questions (Pulumi, Ops Portal UI, discovery job code), this vault has only design notes — the code is in other repos and must be opened there.
