**Title:** Phase 5 — Monitoring & Rollout

**Description:**

Build observability and roll out to production in stages.

**Dashboard metrics:** Orphans found, orphans destroyed, $ saved, workflow

success rate, untagged-orphan count.

**Alerts:** Workflow failure, orphan count spike, approval timeout.

**Rollout:** Internal dev → internal prod → customer prod (last).

Gate each stage with mentor.

**AI lens:** Weekly trend-summary agent posting Slack digest. Stretch: triage

agent that opens Jira tickets pre-populated with suspected cause.

**Gate criteria:**

- Stable stretch of clean runs in production

**Documentation:**

- `service-portal/ui/features/orphaned-resources/phase-5/monitoring-rollout.md`

**Labels:** `coop-2026`