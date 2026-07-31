# Inventory Persistence Decision (Subtask 3)

**Status: PIVOTED** — MongoDB/Mongoose dependency removed from the job. Current approach: results log to stdout (dry-run). Future write path: ops-portal API via `src/output/apiWriter.ts`, not a direct DB connection.

Related: [[Discovery job implementation]] (S7) · [[Inventory record schema]] (S4) · [[Phase 2/Index|Phase 2 Docs]]

---

## Current approach — dry-run

The job writes nothing to a database. Results are logged to stdout. `DRY_RUN=true` is the default.

The write path is wired but gated:

| Condition | Behaviour |
|---|---|
| `DRY_RUN=true` (default) | Results logged to stdout only |
| `DRY_RUN=false` + `API_URL` set | `src/output/apiWriter.ts` POSTs each inventory record to ops-portal |

---

## Original plan — DocumentDB (shelved)

The original plan was to write directly to the portal's DocumentDB (`orphanedresourcerepos` collection, `saas-portal` DB). That was shelved:

- MongoDB/Mongoose dependency removed from the job entirely
- Two-cluster topology (one cluster per account) means no shared DB network path is required for the scan itself
- Write path decoupled: job → ops-portal API → portal's own DB (portal owns its data model; separation of concerns)

The reasoning for DocumentDB over S3 or Postgres still stands — it's just accessed through the portal's API layer rather than directly from the job.

---

## Future write path — what's needed

| Item | Status |
|---|---|
| ops-portal `POST /api/external/orphaned-resources` | Not yet built |
| ops-portal `GET /api/external/orphaned-resources/targets` | Not yet built — will replace `TENANT_TARGETS` env var |

Until those endpoints exist, the job runs in dry-run mode and `TENANT_TARGETS` is set inline in the WorkflowTemplate.

---

## Related

- [[Discovery job implementation]] — `src/output/apiWriter.ts`
- [[Inventory record schema]] — the schema the write path will use once the API endpoints are built
