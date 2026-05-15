**Working Docs:** [[Phase 6/Index|Phase 6 Docs]]

**Title:** Phase 6 — Root-cause Fixes (STRETCH)

**Description:**

If Phases 0–5 complete with time left. Fix the underlying Pulumi patterns

so future destroys leave fewer orphans.

- Write a short report ranking leak patterns by frequency, weighted by $
- Pick top 3–5 and open PRs against upstream Pulumi repos

(e.g., forceDestroy: true, RDS snapshot lifecycle, ENI cleanup)

- Each PR reviewed by Cloud Eng

**AI lens:** Claude Code for focused refactors. Log how much code came from

AI vs you, what you rewrote, what it missed.

**Gate criteria:**

- PRs merged
- Re-run discovery shows fixed patterns measurably dropped

**Documentation:**

- `service-portal/ui/features/orphaned-resources/phase-6/root-cause-report.md`

**Labels:** `coop-2026`