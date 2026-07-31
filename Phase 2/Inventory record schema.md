# Inventory Record Schema (Subtask 4)

**Status: SCHEMA DEFINED** — mentor review pending. Two fields added (`tenantDeletedAt`, `customerName`, 2026-06-09); cost deliberately excluded.

Canonical source: `app/_lib/orphanedResources/orphanedResourceRepo.ts` (ops-portal repo).
Job write path: `src/output/apiWriter.ts`.
Related: [[Inventory persistence decision]] (S3) · [[Discovery job implementation]] (S7)

---

## Data sources

Every inventory document is assembled from four sources:

### Source 1 — `TENANT_TARGETS` env var (temporary)

Parsed by `loadTargetsFromEnv()` in `index.ts`. Currently a name-only string array (e.g. `["shub-test", "dmitry-test-5"]`). The `awsAccountId` is derived from the single entry in `ACCOUNT_VIEW_MAP`.

| Field | Comes from | Currently available? |
|---|---|---|
| `tenantName` | tenant name string | ✅ Yes |
| `awsAccountId` | derived from `ACCOUNT_VIEW_MAP` | ✅ Yes |
| `environment` | not in current TENANT_TARGETS format | ❌ Pending API |
| `tenantDeletedAt` | not in current TENANT_TARGETS format | ❌ Pending API |
| `customerName` | not in current TENANT_TARGETS format | ❌ Pending API |

`tenantDeletedAt`, `customerName`, and `environment` will be available once `GET /api/external/orphaned-resources/targets` is built to replace `TENANT_TARGETS`.

### Source 2 — AWS Resource Explorer (`searchTenant()`)

One API call per tenant: `tag.value:<tenantName>`. Finds resources explicitly tagged with the tenant name.

| Field | Comes from |
|---|---|
| `identifier` | `Resource.Arn` (full ARN) |
| `service` | `Resource.Service` (e.g. `"ec2"`, `"secretsmanager"`) |
| `resourceType` | `Resource.ResourceType` (e.g. `"ec2:security-group"`) |
| `region` | `Resource.Region` |
| `tags` | `Resource.Properties` → flattened `"Key=Value;Key=Value"` |
| `discoveryMethod` | hardcoded `"tagged"` |

### Source 3 — AWS service-specific APIs (`supplementaryDiscover()`)

EC2 Describe, Secrets Manager List, etc. — only called for untagged relatives of already-attributed resources.

| Field | Comes from |
|---|---|
| `identifier` | ARN constructed from describe results |
| `service` | hardcoded per handler |
| `resourceType` | hardcoded per handler |
| `region` | from the describe API response |
| `tags` | from describe results (if available) |
| `discoveryMethod` | hardcoded `"supplementary"` |

### Source 4 — Job runtime (set at startup)

| Field | Comes from |
|---|---|
| `runId` | `RUN_ID` env var → set by Argo to `{{workflow.name}}`; locally defaults to `"local-2026-06-09T..."` |
| `discoveredAt` | `new Date()` at the start of `main()` — one timestamp per run, shared by all docs in that run |

### Flow diagram

```
TENANT_TARGETS env var (name-only strings, temporary)
  ["shub-test", "dmitry-test-5", ...]
  awsAccountId derived from ACCOUNT_VIEW_MAP
          │
          ▼
    for each tenant
          │
          ├──▶ Resource Explorer  ──▶  tagged rows
          │    tag.value:<tenant>       (identifier, service, type, region, tags)
          │    discoveryMethod = "tagged"
          │
          └──▶ EC2/SecretsMgr/etc ──▶  supplementary rows
               walk from anchors        (same fields)
               discoveryMethod = "supplementary"
          │
          ▼
    writeInventoryRows()
    + runId (env)
    + discoveredAt (new Date())
    + tenantName, awsAccountId (from TENANT_TARGETS / ACCOUNT_VIEW_MAP)
          │
          ▼
    stdout (DRY_RUN=true, default)
    OR apiWriter.ts → POST /api/external/orphaned-resources (DRY_RUN=false)
```

---

## Field definitions

| Field | Type | Required | Story field | Notes |
|---|---|---|---|---|
| `tenantName` | String | ✅ Required | Tenant name | Matches the tag value used for RE query |
| `customerName` | String | ✅ Required | Customer name | **Added** (2026-06-09) — tenant→customer join at discovery time |
| `tenantDeletedAt` | Date | ✅ Required | Tenant deletion date | **Added** (2026-06-09) — enables orphan-age computation and Phase 4 confirmation |
| `awsAccountId` | String | ✅ Required | AWS account | Numeric account ID (e.g. `545681961293`) |
| `region` | String | ✅ Required | Region | AWS region code (e.g. `us-east-1`) |
| `resourceType` | String | ✅ Required | Resource type | Full type string (e.g. `secretsmanager:secret`, `ec2:natgateway`) |
| `service` | String | ✅ Required | Resource type | Service portion of the type (e.g. `secretsmanager`, `ec2`) |
| `identifier` | String | ✅ Required | Resource ID / ARN | Full AWS ARN |
| `tags` | String | Optional | Tags | Flattened `Key=Value;Key=Value` string; empty string if untagged |
| `discoveryMethod` | String | ✅ Required | Attribution confidence | `tagged` (found by RE tag query) or `supplementary` (found by fallback layer) — covers "untagged-but-suspected orphans" and "fallback vs RE" in one field |
| `discoveredAt` | Date | ✅ Required | Discovery run timestamp | Timestamp of the scan run that found this resource |
| `runId` | String | ✅ Required | Discovery run timestamp | Groups all resources from a single scan run |
| `environment` | String | Optional | — | Not in the story list but present in implementation; useful for dev/prod filtering |

---

## Deliberately excluded fields

### Estimated monthly cost

**Not stored.** Reasons:

The job has a hard constraint: `DISCOVERY ONLY — this job never reads, stores, or logs any cost figure`. Cost data (from Cost Explorer) is 24–48 hours lagged; the inventory is near-real-time. Mixing them in one document couples two concerns that change at different rates and have different accuracy properties. The portal already has Cost Explorer access wired (`costExplorerAwsAccessKey`) and the tenant's `awsCostPerMonth` field. Phase 3 (Orphans UI) should join cost separately at query time rather than storing potentially stale figures in the inventory.

### Age of orphan

**Not stored — derivable.** Once `tenantDeletedAt` is added, age = `discoveredAt - tenantDeletedAt`. Computing at query time is free and always accurate. Storing it would mean it gets stale with every subsequent scan.

---

## Story questions — answered

| "Also consider" question | Answer |
|---|---|
| How to represent untagged-but-suspected orphans? | `discoveryMethod: supplementary` — these are resources found by the fallback layer (VPC walk, CF stack traversal, name-prefix) without a matching tag |
| How to flag fallback layer vs RE? | Same `discoveryMethod` field — `tagged` = RE, `supplementary` = fallback |
| What metadata does Phase 4 need to execute a delete? | `identifier` (ARN) + `region` + `awsAccountId` — sufficient to call the delete API for any resource type. Dependency ordering (e.g. delete subnets before VPC) is a Phase 4 orchestration problem, not a schema field |
| What does Phase 3 UI need for display and filtering? | `tenantName`, `customerName`, `awsAccountId`, `resourceType`, `region`, `discoveryMethod`, `discoveredAt`, `tenantDeletedAt` — all present |

---

## Live validation — shub-test confirms Phase 0 category #3

The job's live output for the `shub-test-dev` tenant included this row:

```
"region": "ap-northeast-1",
"identifier": "arn:aws:secretsmanager:ap-northeast-1:...:secret:saas/shub-test/defender-RLG5qK",
"discoveryMethod": "tagged",
"tags": "Customer=shub-test;Env=dev;Tenant=shub-test"
```

Resource Explorer found this because the replica secret has the tag `Tenant=shub-test`. The region `ap-northeast-1` (Tokyo) is one of the ~30 regions the defender secret was replicated to — exactly the category #3 orphan pattern confirmed in Phase 0 (investigation #005). The job's output proves the pipeline is finding what the failure analysis predicted.

---

## Open items

- `customerName`, `tenantDeletedAt`, `environment` — not populated by current `TENANT_TARGETS` (name-only strings). These become available once `GET /api/external/orphaned-resources/targets` is built and `index.ts` is updated to call it. Repo agent's task.
- **Mentor review** — this schema should be walked through at the same session as the S1/S2/S3 decisions.
