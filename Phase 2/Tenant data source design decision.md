---
status: DECIDED — Option A shipped
updated: "2026-07-10"
---

# Orphaned Resource Discovery — Tenant Data Source: Design Decision

**Status: DECIDED** — Option A implemented and verified against current code on 2026-07-10.

Related: [[Targets API endpoint]] (S13) · [[Discovery job implementation]] (S7) · [[Phase 2/Index|Phase 2 Docs]]

---

## Background

The orphaned-resource discovery job is an Argo CronWorkflow that runs on the central-ops Kubernetes cluster and scans AWS accounts for resources left behind by deleted tenants. Currently the job gets its list of tenants from a hardcoded `TENANT_TARGETS` environment variable in the WorkflowTemplate YAML. Every time a new tenant is deleted, a manual YAML edit and redeploy is required.

S13 (PLAT-1730) replaces this with a dynamic API call to the ops-portal. This document presents the investigation findings and two implementation options for senior/lead engineers to review and decide.

---

## Investigation Findings

The existing endpoint `GET /api/external/tenants?environment=dev` was tested live against `https://api.ops-dev.int.cequence.ai` with a valid Descope Bearer token.

**What it returns today:**
- 322 dev tenants, paginated at 100 per page (default)
- Deleted tenants ARE included in the response, mixed with active ones (`deleted: true/false`)
- Fields returned per tenant: `id`, `name`, `environment`, `customer`, `accountStage`, `region`, `branding`, `paused`, `deleted`, `archived`, `upgradeGroup`

**What it does NOT return:**
- `deletedAt` — deliberately absent
- Any cost data — explicitly excluded by design per the OpenAPI spec: *"cost data is excluded from the response"*
- `awsAccountId` — not a blocker; the job can derive this from its own `ACCOUNT_VIEW_MAP` config

**Requirements confirmed for the discovery job (from senior):**

| Field | Purpose | In existing endpoint? |
|---|---|---|
| `tenant` | Tenant name for AWS Resource Explorer search | ✅ (`name`) |
| `deletedAt` | Defines the scan window — monthly run scans tenants deleted in the last 30 days | ❌ |
| `costSinceDeletion` | Total AWS spend since deletion (sum of `dailyCosts[].cost` where `date > deletedAt`) — only scan tenants where cost > 0 to skip already-clean tenants | ❌ |
| `customerName` | Inventory record attribution (nice to have) | ✅ (`customer`) |

---

## Option A — Extend the existing `GET /api/external/tenants` endpoint

Add the missing fields as optional additions to the existing response.

**Changes required:**
- Add `deleted=true` as an optional query param filter in `TenantService.listTenants()`
- Add `deletedAt` to the DB `.select()` and response mapping
- Add `costSinceDeletion` (computed: sum of `dailyCosts[].cost` where `date > deletedAt`) to the response — introduces cost data to an endpoint that currently explicitly excludes it
- Update `tenantsListQuerySchema` Zod schema with the new query param
- Update `TenantsListResponseSchema` Zod schema with new optional fields
- Update OpenAPI spec and paths registration

**Pros:**
- Additive changes only — existing callers unaffected
- Reuses established auth, error handling, and OpenAPI patterns
- No new endpoint surface area to maintain
- Fewer files touched

**Cons:**
- Cost data was deliberately excluded from this endpoint — adding it overrides an explicit design decision
- A general-purpose tenant list endpoint gains fields specific to one internal operational job
- PR reviewers must understand the change doesn't break the ~322-tenant general list behaviour
- The `costSinceDeletion` computation (filtering `dailyCosts` array server-side) adds query complexity to a frequently-called endpoint
- Future changes to discovery requirements mean touching the general API again

---

## Option B — New purpose-built endpoint (recommended approach per S13 ticket)

Create `GET /api/external/orphaned-resources/targets?environment=dev|prod`

**Changes required:**
- New file: `app/api/external/orphaned-resources/targets/route.ts`
- New OpenAPI paths file: `app/_lib/openapi/paths/orphanedResources.ts`
- Register in `app/_lib/openapi/paths/index.ts`
- Add to `openapi.yaml`

**Response shape:**
```json
{
  "targets": [
    {
      "tenant": "shub-test",
      "awsAccountId": "545681961293",
      "tenantDeletedAt": "2026-05-05T00:00:00Z",
      "customerName": "Cequence",
      "costSinceDeletion": 12.40
    }
  ]
}
```

**Pros:**
- Purpose-built — the response shape exactly matches what the discovery job needs, nothing more
- Cost data belongs here conceptually (internal ops tooling endpoint, not a general consumer API)
- No risk of affecting existing consumers of `GET /api/external/tenants`
- Future discovery requirements are isolated to one file
- Easier to justify cost data inclusion in PR review (clear, single-purpose use case)
- Clean OpenAPI documentation — the endpoint's intent is self-evident

**Cons:**
- New file to create, maintain, and document
- More upfront work (OpenAPI paths file, schema registration)
- One more endpoint for seniors to be aware of

---

## Open Questions for Senior/Lead Engineers

1. Should `costSinceDeletion` be exposed via the external API at all, given it was deliberately excluded from `GET /api/external/tenants`? Is there a policy concern about cost data in external endpoints?
2. Which option better fits the team's API design philosophy — purpose-built endpoints or extending general ones?
3. Is `costSinceDeletion > 0` the right scan filter, or should there be a minimum threshold (e.g. > $1.00) to avoid scanning tenants with negligible orphaned resources?
4. Should the scan window be configurable (e.g. `deletedWithin=30d` query param) or hardcoded to 30 days?

---

## Decision

**Option A — extend the existing endpoint.** Shipped as `GET /api/external/tenants?environment=<env>&deleted=true` — the existing tenant list endpoint with a `deleted` query filter, not a new purpose-built endpoint. No new OpenAPI paths file, no new route.

**What actually shipped, verified against `scripts/discover/tenants/apiTargets.ts` on 2026-07-10:**
- The discovery job calls the existing endpoint with `environment=<env>&deleted=true`
- `TENANT_TARGETS` remains as a fallback when `API_URL`/`DESCOPE_ACCESS_KEY` are not configured
- Auth: Descope access key is exchanged for a session JWT; the exchange itself requires `Authorization: Bearer <PROJECT_ID>:<DESCOPE_ACCESS_KEY>` (project ID prefixed — discovered via a 401 `E011007` when the key was sent alone)
- Cost data: the response already includes a cost field — but it's named `recentCostSinceDeletion`, not `costSinceDeletion`. The job's `apiTargets.ts` reads `costSinceDeletion`, which does not exist on the response shape, so the "skip zero-cost tenants" filter is currently a silent no-op. **This is a live, unpatched bug** — see [[Targets API endpoint]] for tracking.

**Why Option A over Option B in practice:** cost concerns raised in Option B's pros (cost data belongs in a purpose-built endpoint) turned out moot — the existing endpoint already carried a cost field (`recentCostSinceDeletion`) by the time this was implemented, so extending it required less net-new work than standing up a new route.

Full implementation notes: [[Targets API endpoint]]. Decision log entry: [[Decision Logs]].
