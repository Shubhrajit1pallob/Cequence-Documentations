---
title: Salesforce API Integration
source_files:
  - app/_lib/customer/CustomerSalesforceService.ts
  - app/_lib/salesforce/OpportunityContractSyncService.ts
last_reviewed: 2026-06-22
---

# Salesforce API Integration

## Overview

The Ops Portal syncs three Salesforce object types — **Accounts**, **Opportunities**, and **Contracts** — into its own database. All calls are outbound, initiated by scheduled or manually triggered sync jobs inside the portal. There is no inbound webhook or callback from Salesforce.

---

## Authentication

**Mechanism:** OAuth 2.0 Client Credentials flow (machine-to-machine, no user involved).

**Token endpoint:**
```
POST https://{SFDC_INSTANCE}/services/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
client_id={SFDC_CLIENT_ID}
client_secret={SFDC_CLIENT_SECRET}
```

**Token caching:** Tokens are cached in a singleton `SalesforceTokenManager` (in-process, not persisted). The cache uses the `issued_at` timestamp from the token response and treats the token as valid for 1 hour from that time. A new token is only fetched when the cached one has expired or is absent.

**Timeouts:**
- Token request: 20 s (`SALESFORCE_TOKEN_TIMEOUT_MS`)
- Query requests: 20 s for account syncs; 30 s for opportunity/contract syncs

---

## Required Environment Variables

| Variable | Used for |
|---|---|
| `SFDC_INSTANCE` | Salesforce org hostname, e.g. `cequence.my.salesforce.com` |
| `SFDC_CLIENT_ID` | OAuth connected-app client ID |
| `SFDC_CLIENT_SECRET` | OAuth connected-app client secret |
| `SFDC_API_VERSION` | REST API version string, e.g. `v59.0` |

All four are read from `GetEnvironment()` (`app/_lib/Environment.ts`). The base URL for every call is `https://${SFDC_INSTANCE}`.

---

## HTTP Client

Raw `fetch` wrapped in a local `fetchWithTimeout` utility (`app/_lib/fetchWithTimeout.ts`). No Salesforce SDK or third-party client library is used. Responses are parsed and validated with **Zod** schemas.

---

## Outbound Calls

### 1. Describe — Account field discovery

**Endpoint:**
```
GET https://{SFDC_INSTANCE}/services/data/{SFDC_API_VERSION}/sobjects/Account/describe
```

**Trigger:** Called once per sync, before every Account SOQL query, to filter out custom fields that do not exist on the target org (e.g. a dev sandbox). Results are cached in-process for 1 hour.

**Purpose:** Prevents 400 errors on SOQL selects that reference fields missing from the org. Required fields (`Id`, `Name`, `CreatedDate`) cause a hard error if absent; all others are silently dropped from the SELECT.

---

### 2. Account sync — Full

**Trigger:** `syncCustomersFromSalesforceFull(performedBy)` — called by admin-initiated full-sync action in the portal, or as fallback when no prior incremental sync baseline exists.

**SOQL query (dynamic SELECT, WHERE is fixed):**
```sql
SELECT
  Id, Name, Description, Type, Active_PoV__c, Renewal_Date__c,
  Account_Type__c, Region__c, Deployment_Type__c, Products__c,
  Purchased_Products__c, CreatedDate, LastActivityDate, LastModifiedDate,
  Industry, Account_Status__c, Account_Churn_Date__c, Partner_Type__c,
  Closed_Won_Total__c, Account_ARR__c, Account_ARR_Rollup__c,
  Account_ARR_Sum__c, Account_Revenue__c, AnnualRevenue, ACV__c,
  Account_Subscription_Start_Date__c, Customer_Since__c,
  API_Discovery_Protection__c, API_Discovery_Risk_Monitoring__c,
  API_Edge_Protection__c, API_Testing_ARR__c, CQ_Prime_ARR__c,
  Other_ARR__c, Sentinel_ARR__c, Spartan_ARR__c, Spyder_ARR__c,
  Threat_Protection_ARR__c, UAP_Bundle_ARR__c
FROM Account
WHERE
(
  Type IN ('Customer', 'Distributor', 'Integrator', 'Partner', 'Reseller')
    AND (CreatedDate > LAST_N_YEARS:1
      OR LastModifiedDate > LAST_N_YEARS:2
      OR LastActivityDate > LAST_N_YEARS:2)
) OR
(
  Type = 'Prospect'
  AND (
    CreatedDate >= LAST_N_DAYS:365
    OR LastModifiedDate >= LAST_N_DAYS:60
    OR LastActivityDate >= LAST_N_DAYS:60
  )
)
ORDER BY CreatedDate DESC
```

**Note on SELECT:** The SELECT list is resolved at runtime against the describe cache. Any of the 35 candidate fields missing from the org are silently dropped. The query above shows all candidates; actual queries may have fewer columns.

**Post-sync side-effect:** Accounts present in the local DB (sourced from SFDC) but absent from the query result are marked "missing" (`CustomerRepo.markMissingSalesforce`). The full-sync timestamp (`lastFullSyncAt`) is written to the sync-state record.

---

### 3. Account sync — Incremental

**Trigger:** `syncCustomersFromSalesforceIncremental(performedBy)` — called by the scheduled background sync job. Falls back to a full sync if no `lastIncrementalSyncAt` baseline exists.

**SOQL query:**
```sql
SELECT
  <same candidate fields as full sync>
FROM Account
WHERE Type IN ('Customer','Distributor','Integrator','Partner','Reseller','Prospect')
  AND (
    CreatedDate >= {lastIncrementalSyncAt}
    OR LastModifiedDate >= {lastIncrementalSyncAt}
    OR LastActivityDate >= {lastIncrementalSyncAt_dateOnly}
  )
ORDER BY CreatedDate DESC
```

`lastIncrementalSyncAt` is formatted as ISO 8601 without milliseconds (e.g. `2026-06-20T10:00:00Z`) for datetime fields. `LastActivityDate` uses a date-only literal (`YYYY-MM-DD`) because it is a Date field, not a DateTime field.

**Post-sync side-effect:** Missing detection is **not** run on incremental syncs. The incremental sync timestamp (`lastIncrementalSyncAt`) is updated after completion.

---

### 4. Account upsert logic (common to full and incremental)

After fetching records, each Salesforce Account is:

1. Validated against `SalesforceAccountSchema` (Zod).
2. Transformed to `SalesforceAccountData` via `transformSalesforceAccount`.
3. Skipped if `Account_Status__c` is null, empty, or `"test"` (case-insensitive). This prevents Prospects and blank-status accounts from being written as paying customers, which would inflate margin reports.
4. Upserted into the local `Customer` collection via `CustomerRepo.upsertFromSalesforce`.

The sync returns a `SyncSummary`:
```ts
interface SyncSummary {
  created: number   // net-new accounts inserted
  updated: number   // existing accounts updated
  missing: number   // accounts marked missing (full sync only)
  missingNames: string[]
  errors: number    // per-record errors (sync continues)
}
```

---

### 5. Opportunity sync

**Source file:** `app/_lib/salesforce/OpportunityContractSyncService.ts`

**Trigger:** `syncOpportunities(performedBy, { sinceYears? })` — called by `syncOpportunityAndContractData`, which is invoked by the admin sync action. Default window is the last 3 years.

**SOQL query:**
```sql
SELECT
  Id,
  AccountId,
  Name,
  StageName,
  Amount,
  CloseDate,
  IsClosed,
  IsWon,
  ContractId,
  Commercial_Term_Years__c,
  Ceq_Annual_recurring_revenue__c,
  Ceq_Annual_Contract_value__c,
  Ceq_Total_Contract_Value__c,
  Ceq_NARR__c,
  ARR__c,
  CreatedDate
FROM Opportunity
WHERE IsClosed = true AND IsWon = true AND CloseDate >= LAST_N_YEARS:{sinceYears}
ORDER BY CloseDate DESC
```

Only Closed-Won opportunities are fetched. Opportunities with no `AccountId` or an unparseable `CloseDate` are skipped (logged as warnings, not errors).

**Post-sync side-effect:** After upsert, `SalesforceOpportunityRepo.markStaleWithinScope` marks local opportunities that are within the sync window but were not returned by Salesforce (i.e., they were deleted or no longer match). An audit log entry is written on both success and failure.

**Returns:**
```ts
interface SyncSummary {
  fetched: number
  created: number
  updated: number
  errors: number
}
```

---

### 6. Contract sync

**Source file:** `app/_lib/salesforce/OpportunityContractSyncService.ts`

**Trigger:** `syncContracts(performedBy, { sinceYears? })` — called together with opportunity sync by `syncOpportunityAndContractData`. Default window is the last 3 years.

**SOQL query:**
```sql
SELECT
  Id,
  AccountId,
  ContractNumber,
  Status,
  StartDate,
  EndDate,
  ContractTerm
FROM Contract
WHERE EndDate >= LAST_N_YEARS:{sinceYears} OR Status = 'Activated'
ORDER BY StartDate DESC
```

Fetches contracts that ended within the window **or** are currently activated (regardless of age). Contracts with no `AccountId` are skipped.

**Post-sync side-effect:** `SalesforceContractRepo.markStaleWithinScope` marks local contracts within the sync window that were not returned. An audit log entry is written on both success and failure.

**Returns:** same `SyncSummary` shape as opportunity sync.

---

## Pagination

All three query types (Account, Opportunity, Contract) use the same pagination pattern:

1. Issue initial GET to `/services/data/{SFDC_API_VERSION}/query?q=<encoded SOQL>`.
2. Check `done` flag in the response.
3. If `done = false` and `nextRecordsUrl` is present, follow `https://{SFDC_INSTANCE}{nextRecordsUrl}` for the next page.
4. Repeat until `done = true`. If `done = false` but `nextRecordsUrl` is missing, a warning is logged and iteration stops to avoid an infinite loop.

---

## Response Shape (Salesforce Query API)

```json
{
  "totalSize": 427,
  "done": false,
  "records": [ { "Id": "...", ... } ],
  "nextRecordsUrl": "/services/data/v59.0/query/01gXXX-2000"
}
```

Each page's `records` array is accumulated into a flat list before upsert.

---

## Audit Logging

Opportunity and Contract syncs write an entry to the portal's internal audit log (`Audit.log`) on both success and failure, using:
- `action: AuditActions.SYNC`
- `resourceType: AuditTargets.SALESFORCE_OPPORTUNITY` or `SALESFORCE_CONTRACT`
- `resourceId: 'sync'`
- `performedBy`: caller identity passed in by the invoking API route

Account syncs do not write an explicit audit log entry (sync state timestamps serve as the record).
