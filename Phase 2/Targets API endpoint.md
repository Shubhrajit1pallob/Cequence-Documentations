---
status: DONE — verified against code 2026-07-10
updated: "2026-07-10"
---

# S13 — External API Tenant Discovery (PLAT-1730)

**Status: DONE** — implemented, shipped, and verified against the actual codebase on 2026-07-10.

Related: [[Tenant data source design decision]] · [[Discovery job implementation]] (S7) · [[Phase 2/Index|Phase 2 Docs]]

---

## What was built

The discovery job's tenant list source was replaced with a live call to the ops-portal external API, implemented in `scripts/discover/tenants/apiTargets.ts` in the `orphaned-resource-discovery` repo.

**This is not the originally-scoped new endpoint** (`GET /api/external/orphaned-resources/targets`, Option B). The team shipped **Option A** instead — extending the existing `GET /api/external/tenants` endpoint with a `deleted` query filter. See [[Tenant data source design decision]] for the full option comparison and why A was chosen in practice.

### Flow

1. Descope access key → exchanged for a session JWT
2. `GET /api/external/tenants?environment=<env>&deleted=true` called with the JWT
3. Response `tenants[]` parsed into `DiscoveryTarget[]`
4. `TENANT_TARGETS` env var remains as a fallback when `API_URL`/`DESCOPE_ACCESS_KEY` are not set — no breaking change for any environment not yet configured for the live API

### Auth detail

The Descope token exchange requires the project ID prefixed to the access key in the Bearer header:

```
Authorization: Bearer <PROJECT_ID>:<DESCOPE_ACCESS_KEY>
```

Discovered via a `401 E011007` ("missing bearer token") when the access key was sent alone. `PROJECT_ID` is treated as public config — passed as a plain manifest value, not a `secretKeyRef`.

`config.ts` has a fail-fast guard requiring `PROJECT_ID` whenever `DESCOPE_ACCESS_KEY`/`API_URL` are set, applied consistently in both dry-run and live modes (previously the two modes had inconsistent requirements).

### Cost filter behavior

Tenants with `costSinceDeletion === 0` are skipped (already cleaned up, no need to scan). `null`/`undefined` cost is **not** skipped — treated as "scan unconditionally" rather than assuming zero cost.

---

## Bugs found during implementation

### 1. `api.` host requirement (fixed)

`ops-dev.int.cequence.ai` / `ops.int.cequence.ai` are VPN-IP-allowlisted at the ingress level. A separate `api.ops-dev.int.cequence.ai` / `api.ops.int.cequence.ai` host routes through the same load balancer to `/api/external` with no IP allowlist. `API_URL` was corrected to the `api.` host in both `workflowtemplate-dev.yaml` and `workflowtemplate-prod.yaml`, plus `.env.example`. This is documented precedent elsewhere in the ops-portal repo.

### 2. `costSinceDeletion` field-name mismatch (confirmed live bug, NOT yet patched)

`apiTargets.ts` reads a field called `costSinceDeletion` from the API response. The actual field the ops-portal API returns is **`recentCostSinceDeletion`** (`ops-portal/app/_lib/tenant/TenantService.ts`). Because the field name doesn't match, `costSinceDeletion` is always `undefined` on the real response — which silently defeats the "skip $0-cost deleted tenants" filter (it never matches `=== 0`, so it falls into the "scan unconditionally" path instead of skipping). Every tenant with recent cost data is currently scanned regardless of actual cost. Tracked as a follow-up bug ticket — see below.

### 3. Upstream ops-portal bug — tenants incorrectly flagged deleted (not this job's bug)

`GET /api/external/tenants?deleted=true` returned tenants (`dm-n-uap-waf-1/2/3`, `dm-n-uap-nowaf-1`) whose own live tenant documents show `deleted: false`, no `deletedAt`, no `destroyStatus` — i.e., these were freshly-provisioned test tenants that were never actually deleted. Reproduced via a direct `curl` against the API and cross-checked against direct DB record inspection.

Ruled out (with code evidence) before landing on the likely cause: duplicate tenant documents (impossible — `tenantExists()`/`getTenantByNameAndEnvironment()` don't filter on `deleted`, so both `createTenant` and the Pulumi tenant updater refuse/reuse any existing record regardless of its `deleted` flag), a stale local dry-run DB (ruled out — reproduced against the live API directly), and reactivation of a prior delete (ruled out — `internalNotes` proves these were first-ever documents).

Most likely cause: DocumentDB read-replica lag (`ops-portal` connects with `family: 4 // IPv4 only` for DocumentDB support; no explicit `readPreference` override visible in the code reachable from this repo). This is an upstream ops-portal issue, not a defect in the discovery job or this endpoint extension. Reported to have since been resolved on the ops-portal side — which side fixed it was not confirmed in this session.

---

## Verification (2026-07-10)

All implementation claims above were independently verified against the current code in `saas/orphaned-resource-discovery` — `apiTargets.ts`, `config.ts`, both `workflowtemplate-*.yaml`, `.env.example`. Every claim CONFIRMED except the field-name bug, which was CONFIRMED as still present and unpatched.

---

## What not to do

- **Don't assume `costSinceDeletion` works today** — it's always `undefined` due to the field-name bug; every deleted tenant is currently scanned regardless of cost.
- **Don't remove `TENANT_TARGETS` fallback** — it's the safety net for any environment not yet configured with `API_URL`/`DESCOPE_ACCESS_KEY`.

---

## Open follow-ups

- Patch `apiTargets.ts` to read `recentCostSinceDeletion` instead of `costSinceDeletion`.
- Confirm with ops-portal team whether the DocumentDB read-replica lag theory for the deleted-tenant bug was the actual root cause and whether a fix has been deployed.
