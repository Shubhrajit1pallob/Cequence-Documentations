**Working Docs:** [[Phase 4/Index|Phase 4 Docs]]

**Title:** Phase 4 — Approval & Destruction

**Description:**

⚠️ Highest-risk phase. Slow down here.

**Cleanup workflow:**

1. Inputs: tenant-name, resource-list, dry-run flag
2. Dry-run first: Slack message with resource list + cost + Approve/Decline

(see PLAT-1429–1435 as precedent)

3. suspend: {} template with timeout (24h auto-decline)
4. On approve → execute deletes per-resource with retry + continueOn: { failed: true }
5. Post-execution report to Slack + Portal audit log

**Constraints:**

- Per-tenant batch approval granularity
- High-risk classes (RDS, S3 with data) get extra confirmation
- Don't modify Pulumi state — AWS-side only; humans handle pulumi state delete

**AI lens:** AI prepares the decision, never makes it. Surfaces: Slack-message

drafter, per-resource risk rater, pre-flight checker.

**Gate criteria:**

- Tabletop walkthrough + one prod dry-run with mentor
- Real destroys against internal Cequence test tenants only
- Not customer tenants until Phase 5 rollout

**Documentation:**

- `service-portal/ui/features/orphaned-resources/phase-4/approval-workflow-design.md`

**Labels:** `coop-2026`