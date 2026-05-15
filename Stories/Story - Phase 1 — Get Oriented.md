**Working Docs:** [[Phase 1/Index|Phase 1 Docs]]

**Title:** Phase 1 — Discovery Design

**Description:**

Design the discovery approach. This phase is design-only — no building.

Produce a written design doc with 2–3 discovery approaches and their

tradeoffs for review.

**Key activities:**

- Audit AWS tagging coverage in both accounts. Quantify % of resources

with Customer + Tenant + Env tags, broken out by service.

- Bring 2–3 discovery approaches to the design review as a written doc.

Starting points to evaluate:

- AWS Resource Explorer (tag-indexed; simplest)
- AWS Config aggregator (authoritative; heavier)
- Cost Explorer + per-service SDK drill-down (cost-first)
- Hybrid (Resource Explorer + per-service fallbacks for untagged classes)
- For each option: evaluate pros/cons, IAM scope, cost, time-to-implement,

coverage of untagged resources.

- AI lens: evaluate whether discovery could be agentic instead of a

deterministic scan. Capture the reasoning in the decision log.

**Gate criteria:**

- Design review with mentor and one senior engineer
- One approach picked
- ADR-style write-up documenting the decision and the "why"

**Documentation:**

- `service-portal/ui/features/orphaned-resources/phase-1/discovery-design.md`

**Labels:** `coop-2026`