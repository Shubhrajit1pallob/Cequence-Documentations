**Working Docs:** [[Phase 3/Index|Phase 3 Docs]]

**Title:** Phase 3 — Portal "Orphans" UI

**Description:**

Add an "Orphans" tab/page to the Ops Portal (infra/saas/service-portal/ui).

Match the visual conventions of existing pages.

**Per-tenant row minimum fields:**

- Tenant, customer, last-deleted-at, # resources by type, estimated monthly

cost, age of orphans.

**Required row actions:**

- "View details" — drill into the resource list
- "Request Cleanup" — kicks off the approval flow (Phase 4)

**AI lens:** Two agent surfaces to evaluate:

(1) Per-tenant summarizer — raw inventory → plain English

(2) Natural-language search — "show me orphans > $100/mo in us-west-2"

**Gate criteria:**

- UX review with mentor and product/design touchpoint
- Page populated with real data from Phase 2 inventory

**Documentation:**

- `service-portal/ui/features/orphaned-resources/phase-3/portal-ui-design.md`

**Labels:** `coop-2026`