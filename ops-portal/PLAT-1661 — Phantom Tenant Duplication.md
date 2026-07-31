---
status: INVESTIGATION COMPLETE — fix pending
jira: PLAT-1661
updated: 2026-06-17
---

# Phantom Tenant Duplication — Root Cause Analysis (PLAT-1661)

**Jira:** PLAT-1661 · **Status:** To Do · **Assignee:** Shubhrajit Pallob

Related: [[phantom-tenant-cost-path-analysis]] (earlier investigation notes — simpler framing, overlapping scope)

---

## Executive Summary

The platform's standard convention is to tag every tenant AWS resource with `Tenant = <bare tenant name>` (e.g. `luke-5-28-26`). The Edge Management Service (EMS) instead tags its resources with `Tenant = <tenant id>` (the env-suffixed form, e.g. `luke-5-28-26-dev`). The ops-portal cost updater reads the `Tenant` tag from AWS Cost Explorer and reconstructs a tenant id by appending the environment (`${tag}-${env}`). For a bare-name tag this yields the correct id; for an EMS tag it yields a doubled-env id (`luke-5-28-26-dev-dev`) that matches no existing tenant. On that lookup miss, the updater creates a brand-new tenant row with `accountStage: 'pov'` and `region: 'unknown'` — the phantom. Several compounding defects make this possible and repeatable.

---

## Systems Involved

| Repo | GitLab path | Local path | Role in this bug |
|---|---|---|---|
| **ops-portal** | `cequence/orcx-team/portal` | `saas/ops-portal` | Where the duplicate row gets created. Contains the cost updater. |
| **edge-management-service** | `cequence/orcx-team/edge-management-service` | `saas/edge-management-service` | Runtime microservice that tags AWS resources with the env-suffixed tenant name. |
| **tenant** | `cequence/saas/infrastructure/tenants/tenant` | `saas/tenant` | Where the `-dev`/`-prod` suffix on the EMS tenant identity is originally minted at provisioning. |
| **edge-management-infra** | `cequence/saas/infrastructure/central-ops/edge-management-infra` | `saas/edge-management-infra` | Deploys EMS (IAM/IRSA, SQS, EventBridge, k8s). **Not the bug source** — included to prevent misattribution. |

### Key terms

- **Bare name:** the tenant name without environment suffix — e.g. `luke-5-28-26`. This is what the platform's provisioning IaC (cluster, defender, tenant Pulumi stacks) writes into the `Tenant` AWS tag.
- **Tenant id:** the env-suffixed form — e.g. `luke-5-28-26-dev`. This is the primary key in the portal DB (`tenantrepos` collection) and what Descope stores as the tenant identity.
- **Phantom / duplicate tenant:** a portal DB row created by `TenantCostUpdater` on a lookup miss — it has `region: 'unknown'`, `accountStage: 'pov'`, and was never provisioned through the normal workflow.

---

## End-to-End Chain

### Step 1 — The `-dev` suffix is minted at provisioning

**Repo:** `tenant` · **File:** `src/tenant.ts:380`

When a tenant with EMS/Descope integration is provisioned, `EdgeManagementDescopeInt` is instantiated with `tenantName` set to the full id:

```typescript
// tenant/src/tenant.ts:380
new EdgeManagementDescopeInt(
  'descope-edge-management',
  {
    tenantName: `${args.tenantName}-${args.environment}`,  // "luke-5-28-26" + "-" + "dev" = "luke-5-28-26-dev"
    managementKey: args.edgeManagement.descope.descopeManagementKey,
    projectId: args.edgeManagement.descope.projectId,
    parentTenantId: args.edgeManagement.descope.parentTenantId,
    roles: args.edgeManagement.descope.roles,
  },
  { parent: this },
)
```

**Repo:** `tenant` · **File:** `src/edge-management/edgeManagementDescopeInt.ts:17-25`

The Descope tenant is created with `name: args.tenantName` — so Descope stores the tenant's name as `luke-5-28-26-dev` (the id form). This name later appears in the JWT claim `nsec.tenantName`.

```typescript
// tenant/src/edge-management/edgeManagementDescopeInt.ts:17-25
const tenant = new DescopeTenant(
  'descope-tenant',
  {
    projectId: args.projectId,
    managementKey: args.managementKey,
    name: args.tenantName,          // "luke-5-28-26-dev" — the id form
    parentTenantId: args.parentTenantId,
  },
  { parent: this },
)
```

---

### Step 2 — EMS reads the id-form name verbatim from the JWT

**Repo:** `edge-management-service` · **File:** `src/middleware/auth.ts:92`

EMS reads `tenantName` directly from the Descope JWT (`nsec.tenantName`). It never strips the env suffix — nor can it, because:

```typescript
// edge-management-service/src/middleware/auth.ts:92
const tenantName = nsec?.tenantName || topLevelTenantName;
```

**Repo:** `edge-management-service` · **File:** `src/utils/tenantEnv.ts:7-11`

`parseTenantEnv` only reads the suffix — it does not strip it:

```typescript
// edge-management-service/src/utils/tenantEnv.ts:7-11
export function parseTenantEnv(tenantId: string): TenantEnv | undefined {
  if (tenantId.endsWith('-dev')) return 'dev';
  if (tenantId.endsWith('-prod')) return 'prod';
  return undefined;
}
```

**Repo:** `edge-management-service` · **File:** `src/middleware/auth.ts:144-160`

If `parseTenantEnv(tenantName)` returns `undefined` (i.e. the name doesn't end in `-dev` or `-prod`), the request is rejected with HTTP 400. This means **every resource EMS creates necessarily has an env-suffixed `tenantName`** — stripping the suffix is not an option at the middleware layer without breaking auth.

```typescript
// edge-management-service/src/middleware/auth.ts:144-160
const tenantEnv = parseTenantEnv(tenantName);

if (!tenantEnv) {
  res.status(400).json({
    error: {
      code: ERROR_CODES.INVALID_TENANT_ENV,
      message: `Tenant name '${tenantName}' must end with '-dev' or '-prod'`,
      requestId: req.requestId,
    },
  });
  return;
}
```

---

### Step 3 — EMS writes the id-form name into the AWS `Tenant` tag ← ROOT DEFECT

**Repo:** `edge-management-service` · **File:** `src/utils/awsTags.ts:6-11`

```typescript
// edge-management-service/src/utils/awsTags.ts:6-11
export function buildResourceTags(ctx: TenantContext): AwsTag[] {
  return [
    { Key: 'ManagedBy', Value: config.SERVICE_NAME },   // "edge-management-service"
    { Key: 'Tenant',    Value: ctx.tenantName },         // ← "luke-5-28-26-dev" (id-form, NOT bare name)
    { Key: 'ChangedBy', Value: ctx.actorId },
  ];
}
```

`ctx.tenantName` flows from `auth.ts:92` above. This tag is applied when EMS creates AWS resources:

- **ACM certificates:** `src/services/certificateService.ts:259, :628, :890`
- **CloudFront distributions:** `src/services/cdnAliasPromotion.ts:250`
- **Pools:** `src/services/poolService.ts:2374`
- **VPC/NAT (Argo/Pulumi path):** `src/services/vpcService.ts` → `argoClient` — shares the same id-form value but was not individually line-verified; treat as probable.

The fingerprint of an EMS-tagged resource in AWS: three specific tag keys — `ManagedBy = edge-management-service`, `Tenant = <tenant-id>`, `ChangedBy = <Descope access key id>`.

---

### Step 4 — ops-portal pulls cost grouped by `Tenant` tag with no creator filter

**Repo:** `ops-portal` · **File:** `instrumentation.ts:35`

On startup the scheduler fires `TenantUpdater.startIntervalUpdate()`:

```typescript
// ops-portal/instrumentation.ts:35
tenantUpdater.default.startIntervalUpdate().finally()
```

**Repo:** `ops-portal` · **File:** `app/_lib/tenant/TenantUpdater.ts:39, :63`

```typescript
// ops-portal/app/_lib/tenant/TenantUpdater.ts:39
await tenantCostUpdater.updateTenantsWithCostInfo()

// ops-portal/app/_lib/tenant/TenantUpdater.ts:63-67
async startIntervalUpdate() {
  await this.update()
  setInterval(() => {                // no overlap guard — concurrent runs possible
    this.update().catch(...)
  }, this.intervalMs)
}
```

**Repo:** `ops-portal` · **File:** `app/_lib/awsCostInfo.ts:136-147`

```typescript
// ops-portal/app/_lib/awsCostInfo.ts:136-147  (constants: lines 15, 18, 20, 22)
const AWS_ACCOUNT_SAAS_TENANT_WORKLOADS_SDLC = '545681961293'   // :18
const AWS_ACCOUNT_SAAS_TENANT_WORKLOADS_PROD = '552447114887'   // :15
const DEV_ACCOUNTS = [AWS_ACCOUNT_SAAS_TENANT_WORKLOADS_SDLC, ...]  // :20
const TENANT_TAG = 'Tenant'                                      // :22

async function getSaasWorkloadsCostsPerTenant(...) {
  const params = getCostQuery({
    groupByTag: TENANT_TAG,   // groups by the "Tenant" tag — no ManagedBy filter
    accounts: [AWS_ACCOUNT_SAAS_TENANT_WORKLOADS_SDLC, AWS_ACCOUNT_SAAS_TENANT_WORKLOADS_PROD],
    ...
  })
  return executeCostQuery(params, startDate, endDate)
}
```

The Cost Explorer query groups by `TAG "Tenant"` across both accounts with **no filter on `ManagedBy` or `ChangedBy`**. An EMS-tagged bucket is indistinguishable from a legitimately-tagged one. EMS resources are in scope because they live in the same accounts (confirmed by ARNs in account 545681961293).

---

### Step 5 — ops-portal reconstructs a doubled-env id (Mistake #1) and treats $0 as real cost (Mistake #3)

**Repo:** `ops-portal` · **File:** `app/_lib/awsCostInfo.ts:163-181`

```typescript
// ops-portal/app/_lib/awsCostInfo.ts:163-181
result?.Groups?.forEach((group) => {
  const tenantName = group?.Keys ? group?.Keys[0]?.split('$')[1] : 'unknown'
  // :163 ^ tag value, verbatim — e.g. "luke-5-28-26-dev"

  if (tenantName && tenantName.length > 0) {
    const accountNumber = group?.Keys ? group?.Keys[1] : 'unknown'
    const environment = DEV_ACCOUNTS.includes(accountNumber) ? 'dev' : 'prod'  // :167
    const tenantId = `${tenantName}-${environment}`  // :168 — MISTAKE #1
    // ^ "luke-5-28-26-dev" + "-" + "dev" = "luke-5-28-26-dev-dev"

    const tenantCost = tenantCostPerMonth.get(tenantId) || []
    try {
      if (group?.Metrics?.NetAmortizedCost?.Amount) {   // :172 — MISTAKE #3
        // Amount = "0" (string) is truthy — $0 resources enter the Map
        tenantCost.push({ date: ..., cost: Math.round(parseNumberOrZero(Amount)) })
      }
    } catch ...
    tenantCostPerMonth.set(tenantId, tenantCost)  // :181 — set runs unconditionally
  }
})
```

- **Mistake #1** (line 168): appends env to a tag value that already contains it → `luke-5-28-26-dev-dev`.
- **Mistake #3** (line 172): JavaScript string `"0"` is truthy → even $0-cost resources produce a cost row that enters the Map and triggers the create-on-miss path downstream.
- The `set` at line 181 runs **unconditionally** — the tenantId lands in the Map regardless of cost amount.

---

### Step 6 — ops-portal looks up that id and creates on miss (Mistake #2)

**Repo:** `ops-portal` · **File:** `app/_lib/tenant/TenantCostUpdater.ts:35, :298-331`

The Map flows to `updateTenantCost` for each entry:

```typescript
// ops-portal/app/_lib/tenant/TenantCostUpdater.ts:35
for (const [tenantId, tenantCosts] of Array.from(awsCostPerMonth)) {
  await this.updateTenantCost(tenantId, costs, 'aws')
}
```

```typescript
// ops-portal/app/_lib/tenant/TenantCostUpdater.ts:298-331
async updateTenantCost(tenantId: string, tenantCosts: TenantCosts, provider: 'aws' | 'gcp' = 'aws') {
  const tenant = await Tenant.getTenantById(tenantId)        // getTenantById("luke-5-28-26-dev-dev")

  if (tenant && !tenant.tenantHasActivelyRunningWorkflows()) {
    // ✅ Normal path: update costs on existing tenant
    tenant.awsCostPerMonth = tenantCosts
    await tenant.save()
  } else if (tenant) {
    // Tenant exists but has running workflows → skip
    logger.info(`Skipping cost update for tenant ${tenantId} - has actively running workflows`)
  } else {
    // tenant === null: NOT FOUND → MISTAKE #2: create instead of warn-and-skip
    logger.warn(`Unable to find tenant for cost data ${tenantId}`)
    const nameAndEnv = TenantId.fromIdString(tenantId)   // splits on last '-'
    if (shouldSkipForEnvironment(nameAndEnv.environment)) return
    if (nameAndEnv.name && nameAndEnv.name.length > 0) {
      await new Tenant({
        name: nameAndEnv.name,            // "luke-5-28-26-dev"  (split on last '-')
        customer: nameAndEnv.name,
        environment: nameAndEnv.environment,  // "dev"
        region: 'unknown',
        accountStage: povAccountStage,    // 'pov'
        internalNotes: 'Tenant cost data created by TenantCostUpdater',
        lastCostUpdateDate: new Date(),
      }).save()
    }
  }
}
```

**Repo:** `ops-portal` · **File:** `app/_lib/tenant/tenant.ts:123-127`

`TenantId.fromIdString` splits on the **last** `-`, so `luke-5-28-26-dev-dev` → `name = "luke-5-28-26-dev"`, `environment = "dev"`:

```typescript
// ops-portal/app/_lib/tenant/tenant.ts:123-127
static fromIdString(id: string): TenantId {
  const name = id.substring(0, id.lastIndexOf('-'))      // "luke-5-28-26-dev"
  const environment = id.substring(id.lastIndexOf('-') + 1)  // "dev"
  return new TenantId(name, environment)
}
```

**Note — GCP exposure:** The same `updateTenantCost` logic runs for GCP cost at `TenantCostUpdater.ts:47`. Any GCP tenant with a similar id-form tagging issue would trigger the same phantom creation.

**Note — on-demand trigger:** Besides the timer, `app/api/tenants/[tenant]/refresh-costs/route.ts:52` calls `getAwsCostsPerTenant()` directly, providing an additional trigger surface.

---

### Result — the phantom row

A duplicate tenant is created in the portal DB:

```
id:            "luke-5-28-26-dev-dev"
name:          "luke-5-28-26-dev"
environment:   "dev"
region:        "unknown"
accountStage:  "pov"
internalNotes: "Tenant cost data created by TenantCostUpdater"
provisionStatus: <absent>
```

On subsequent runs, `TenantCostUpdater.updateDailyCosts()` finds the now-existing phantom and keeps feeding it daily cost from the same `luke-5-28-26-dev` tag. Human operators who subsequently try to tidy the phantom may add `customer`, `owner`, `additionalNotificationSlackChannel` — but the durable origin fingerprints remain `region: "unknown"` and the `internalNotes` string.

---

## End-to-End Flowchart

```
tenant/src/tenant.ts:380
  tenantName = "luke-5-28-26-dev"  (bare + env)
        │
        ▼
tenant/src/edge-management/edgeManagementDescopeInt.ts:22
  Descope tenant name = "luke-5-28-26-dev"
  → JWT claim nsec.tenantName = "luke-5-28-26-dev"
        │
        ▼
edge-management-service/src/middleware/auth.ts:92
  ctx.tenantName = "luke-5-28-26-dev"  (verbatim from JWT)
        │
        ▼
edge-management-service/src/utils/awsTags.ts:9   ← ROOT DEFECT
  AWS tag: Tenant = "luke-5-28-26-dev"  (id-form, not bare name)
        │
        ▼
AWS Cost Explorer
  groups by Tenant tag → bucket key = "luke-5-28-26-dev"
  (no ManagedBy filter — indistinguishable from legit tag)
        │
        ▼
ops-portal/app/_lib/awsCostInfo.ts:163
  tenantName = "luke-5-28-26-dev"
        │
        ▼
ops-portal/app/_lib/awsCostInfo.ts:168    ← MISTAKE #1
  tenantId = "luke-5-28-26-dev" + "-" + "dev" = "luke-5-28-26-dev-dev"
        │
  (if Amount = "0" → truthy → cost row pushed)   ← MISTAKE #3
        │
        ▼
ops-portal/app/_lib/tenant/TenantCostUpdater.ts:300
  getTenantById("luke-5-28-26-dev-dev") → null
        │
        ▼
ops-portal/app/_lib/tenant/TenantCostUpdater.ts:318   ← MISTAKE #2
  CREATE new Tenant:
    name = "luke-5-28-26-dev"
    accountStage = "pov"
    region = "unknown"
    internalNotes = "Tenant cost data created by TenantCostUpdater"
```

---

## The Four Defects

| # | Repo / File | Location | Defect |
|---|---|---|---|
| **ROOT** | `edge-management-service` · `src/utils/awsTags.ts:9` | Value comes from `auth.ts:92`, minted at `tenant/src/tenant.ts:380` | Tags `Tenant` with the env-suffixed id, not the bare name — collides with the platform-wide bare-name convention |
| **#1** | `ops-portal` · `app/_lib/awsCostInfo.ts:168` | `tenantId = \`${tenantName}-${environment}\`` | Reconstructs id by appending env to a tag that already contains it → doubled `…-dev-dev` |
| **#2** | `ops-portal` · `app/_lib/tenant/TenantCostUpdater.ts:311-329` | `else { ... new Tenant(...).save() }` | On lookup miss, creates a tenant instead of warn-and-skip |
| **#3** | `ops-portal` · `app/_lib/awsCostInfo.ts:172` | `if (group?.Metrics?.NetAmortizedCost?.Amount)` | String `"0"` is truthy → $0 resources produce a cost row and trigger defects #1 and #2 |

---

## Evidence & Verification

1. **Phantom DB records examined (real examples):** `ems-day1-dev-dev` (twin of real `ems-day1`), `luke-5-28-26-dev-dev` (twin of real `luke-5-28-26`). Each phantom had `region: "unknown"`, `accountStage: "pov"/"internal"`, and `internalNotes: "Tenant cost data created by TenantCostUpdater"`. The real twins had full `provisionStatus`, correct region, and bare-name ids.

2. **AWS Tag Editor CSV exports** (us-east-1, account 545681961293) for `Tenant = road-runner-dev` and `Tenant = luke-5-28-26-dev` showed resources tagged with the id form, carrying `ManagedBy = edge-management-service` and `ChangedBy = <Descope key>` (ACM certs + CloudFront), and for `luke` — a full us-east-1 VPC + 2 NAT Gateways + EIPs + subnets. The three tag keys `ManagedBy`/`Tenant`/`ChangedBy` are the exact fingerprint of `buildResourceTags` (`awsTags.ts:6-11`).

3. **Cost Explorer screenshots** (grouped by service, filtered by id-form `Tenant` tag) confirmed real current spend under the id-form tag for live (not deleted) tenants — e.g. `road-runner` ~$46, `luke` ~$35 — concentrated May–June 2026.

4. **Code audit:** every `Tenant:` tag written by ops-portal provisioning and the cluster/tenant/defender Pulumi stacks uses the bare name (`CreateClusterEntryJob.ts:54`, `CreateTenantEntryJob.ts:59`, `CreateDefenderPoolEntryJob.ts:52`, `WriteNewPoolConfigJob.ts:184/208/227`), and the IaC `getTenantName()` strips the env. Only EMS produces id-form `Tenant` tags.

---

## Important Corrections (Dead Ends)

- An early hypothesis that the duplicates came from orphaned resources left behind after tenant destroys was **disproven**: three of four affected tenants (road-runner, luke-5-28-26, ems-day1) are live/not deleted, and the id-form tag carries current cost. The trigger is live mistagging by EMS, not post-destroy orphans.
- `edge-management-infra` is **not the bug source** — it only deploys EMS. The cert/CloudFront/pool tagging lives in `edge-management-service`. The VPC/NAT tagging path (`vpcService.ts → argoClient`) shares the same id-form value but was not line-level verified — treat as probable.

---

## Recommended Fixes

### Option A — Upstream fix (preferred — stops new phantoms at source)

In `edge-management-service/src/utils/awsTags.ts:9`, emit the **bare tenant name** for the `Tenant` tag by stripping the `-dev`/`-prod` suffix.

```typescript
// Proposed change to edge-management-service/src/utils/awsTags.ts:9
{ Key: 'Tenant', Value: ctx.tenantName.replace(/-(dev|prod)$/, '') },  // bare name
```

**Caveats before implementing:**
- Verify EMS doesn't use the id-form `Tenant` tag for any internal lookups (e.g. `tenantWipeService.ts`, tag-based resource discovery).
- Confirm no cost dashboards or external tooling keys on the current id-form tag value — this is a coordinated change across teams, not a blind one-liner.
- Changing the suffix at `tenant.ts:380` or in Descope is **not advised** — that name is an auth identity claim and EMS's env guard (`auth.ts:144-160`) requires it.

### Option B — Defensive fixes in ops-portal (protects against any future tag drift)

1. **Warn-and-skip instead of create** in `TenantCostUpdater.ts:311-329` — log a warning when a tenant is not found and return, never create.
2. **Detect doubled env suffix** — before building `tenantId` at `awsCostInfo.ts:168`, check if `tenantName` already ends with `-dev` or `-prod`; if so, skip or log.
3. **Add a unique compound index** on `{name, environment}` in `app/_lib/tenant/tenantRepo.ts` to prevent duplicate rows at the DB layer.
4. **Stop treating `"0"` as real cost** at `awsCostInfo.ts:172` — parse to number first: `if (parseFloat(Amount ?? '0') > 0)`.
5. **Add an overlap guard** on `TenantUpdater.startIntervalUpdate` to prevent concurrent runs if an update cycle takes longer than `intervalMs`.

### Cleanup (separate, explicitly-approved step)

Identify and remove/repair existing phantom rows and re-tag the mistagged AWS resources. Must follow the org's audit-logging requirements. Do not bundle into the code fix PR.

**Fingerprint query for existing phantoms:**

```js
db.tenantrepos.find({
  region: "unknown",
  internalNotes: /TenantCostUpdater/
}, {
  _id: 0, name: 1, id: 1, region: 1, accountStage: 1, internalNotes: 1, createdAt: 1
})
```
