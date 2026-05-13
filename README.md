# Orphaned Cloud Resources — Project Notes

A personal working repository for an ongoing effort to **detect and remediate orphaned cloud resources** left behind by deleted tenants in a multi-tenant SaaS environment.

This repo is a project log, not the production code. The code lives elsewhere; here I track problem framing, design decisions, daily progress, and open questions as the work unfolds.

## The problem

When a tenant is deleted, infrastructure-as-code teardown often leaves residue — orphaned volumes, load balancers, container registries, network interfaces, database snapshots, and more. These resources keep accruing cost, clutter the cloud accounts, and obscure what's actually in use.

The goal: find them reliably, attribute them back to their owner, and remove them safely.

## Approach

The work is broken into phases, each shipped independently:

1. **Discovery** — scheduled scan across cloud accounts, tag-based tenant attribution, structured inventory.
2. **Visibility** — an internal UI surfacing orphans per tenant, with cost and age breakdowns.
3. **Approval-gated cleanup** — human-in-the-loop deletion via chat + UI approvals. No fully-automated destroys.
4. **Monitoring** — dashboards for orphan count, dollars saved, workflow health, and tag coverage.
5. **Root-cause fixes** — upstream patches to the IaC layers that produce orphans in the first place.

A sixth, stretch phase explores agentic / AI-assisted root-cause analysis and risk scoring.

## Repo layout

- `Stories/` — phase-by-phase story breakdowns.
- `Work Logs/` — dated daily logs.
- `ToDo Lists/` — running task lists.
- `Decision Logs.md` — design decisions with context, options, and rationale (each entry calls out where AI was or wasn't useful).
- `Agentic System.md` — where AI/agentic approaches fit and where they don't.
- Topic notes — discovery, approval workflow, escalation, risk, open questions.
- `Index/Contents.md` — entry point.

## Conventions

- Markdown-only; designed for [Obsidian](https://obsidian.md) but readable anywhere.
- Wikilinks (`[[Note Name]]`) used throughout.
- ISO dates (`YYYY-MM-DD`).
- Decision logs follow a fixed template: Context → Options → AI/agentic angle → Decision → Open questions.

## Status

In progress. Phases ship sequentially; see `Stories/` and the latest entries in `Work Logs/` for current state.
