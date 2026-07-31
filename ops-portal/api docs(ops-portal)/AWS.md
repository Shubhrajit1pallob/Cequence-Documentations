---
title: AWS API Integrations — Ops Portal
created: 2026-06-22
sources:
  - app/_lib/awsCostInfo.ts
  - app/_lib/eks/AwsEksClient.ts
  - app/_lib/awsEc2Data.ts
---

# AWS API Integrations

The Ops Portal makes outbound calls to three AWS services: **Cost Explorer**, **EKS**, and **EC2**. All three use AWS SDK v3 with static IAM credentials supplied via environment variables. None of these modules expose inbound API routes — they are purely server-side library modules called by route handlers and background jobs.

---

## 1. AWS Cost Explorer

**Source:** `app/_lib/awsCostInfo.ts`

### Service details

| Property | Value |
|---|---|
| Service | AWS Cost Explorer |
| AWS endpoint | `https://ce.us-east-1.amazonaws.com` (SDK default; region override via `AWS_REGION`) |
| SDK client | `CostExplorerClient` (`@aws-sdk/client-cost-explorer`) |
| SDK command | `GetCostAndUsageCommand` |

### Auth

Static IAM credentials read from environment variables at module load time. A single `CostExplorerClient` instance is created as a module-level singleton:

```ts
const costExplorer = new CostExplorerClient({
  region: GetEnvironment().awsRegion,
  credentials: {
    accessKeyId: GetEnvironment().costExplorerAwsAccessKey,
    secretAccessKey: GetEnvironment().costExplorerAwsSecretKey,
  },
})
```

### Required env vars

| Env var | Maps to |
|---|---|
| `AWS_REGION` | Region for the Cost Explorer endpoint |
| `COST_EXPLORER_AWS_ACCESS_KEY` | IAM access key ID |
| `COST_EXPLORER_AWS_SECRET_KEY` | IAM secret access key |

### AWS accounts queried

| Account ID | Role |
|---|---|
| `552447114887` | SaaS tenant workloads — prod |
| `545681961293` | SaaS tenant workloads — SDLC (dev) |
| `930532394465` | CDN Administrator |
| `155873538454` | CDN Staging |

Accounts `545681961293` and `155873538454` are treated as dev environments; all others as prod.

### API calls made

#### `GetCostAndUsageCommand` — monthly costs per tenant (all tenants)

Triggered by `getSaasWorkloadsCostsPerTenant()` and `getCdnAdminCostsPerCustomer()`, which are called by the public exports `getAwsCostsPerTenant()`, `getCdnAdminCosts()`, and `getSaaSWorkloadCosts()`.

**SaaS workloads query shape:**

```ts
{
  TimePeriod: { Start: "YYYY-MM-DD", End: "YYYY-MM-DD" },  // rolling 12-month window
  Granularity: "MONTHLY",
  Filter: {
    Dimensions: {
      Key: "LINKED_ACCOUNT",
      Values: ["552447114887", "545681961293"]
    }
  },
  GroupBy: [
    { Type: "TAG",       Key: "Tenant" },
    { Type: "DIMENSION", Key: "LINKED_ACCOUNT" }
  ],
  Metrics: ["NetAmortizedCost"]
}
```

**CDN admin query shape** — same structure with `CUSTOMER_TAG` (`"Customer"`) and accounts `["155873538454", "930532394465"]`.

**Response fields consumed:**

```
ResultsByTime[].TimePeriod.Start              -- month bucket date (ISO string)
ResultsByTime[].Groups[].Keys[0]             -- "Tenant$<tenant-name>" or "Customer$<customer-name>"
ResultsByTime[].Groups[].Keys[1]             -- linked account ID
ResultsByTime[].Groups[].Metrics.NetAmortizedCost.Amount  -- cost string
```

Missing months in the response are zero-filled so every tenant always has a full 12-month series.

#### `GetCostAndUsageCommand` — monthly costs by service for a single tenant

Triggered by `getTenantCostByService(tenantName, month)`.

```ts
{
  TimePeriod: { Start: "YYYY-MM-01", End: "YYYY-MM-01" },  // single calendar month
  Granularity: "MONTHLY",
  Filter: {
    And: [
      { Dimensions: { Key: "LINKED_ACCOUNT", Values: [/* all 4 accounts */] } },
      { Tags: { Key: "Tenant", Values: [tenantName] } }
    ]
  },
  GroupBy: [
    { Type: "DIMENSION", Key: "SERVICE" },
    { Type: "DIMENSION", Key: "LINKED_ACCOUNT" }
  ],
  Metrics: ["NetAmortizedCost"]
}
```

Returns `TenantServiceCost[]` — one entry per `(service, accountId)` pair, sorted descending by cost, rounded to 2 decimal places. Zero-cost entries are dropped.

**Return shape:**

```ts
interface TenantServiceCost {
  service: string    // e.g. "Amazon Elastic Compute Cloud - Compute"
  accountId: string  // linked account ID
  cost: number       // rounded to 2dp
}
```

#### `GetCostAndUsageCommand` — daily costs for a single tenant

Triggered by `getDailyTenantCost(tenantName, startDate, endDate)`.

```ts
{
  TimePeriod: { Start: "YYYY-MM-DD", End: "YYYY-MM-DD" },
  Granularity: "DAILY",
  Filter: {
    And: [
      { Dimensions: { Key: "LINKED_ACCOUNT", Values: [/* all 4 accounts */] } },
      { Tags: { Key: "Tenant", Values: [tenantName] } }
    ]
  },
  GroupBy: [
    { Type: "TAG",       Key: "Tenant" },
    { Type: "DIMENSION", Key: "LINKED_ACCOUNT" }
  ],
  Metrics: ["NetAmortizedCost"]
}
```

Missing days are zero-filled per account. Returns `AwsDailyCost[]` sorted ascending by date.

**Return shape:**

```ts
interface AwsDailyCost {
  date: Date
  accountId: string
  cost: number  // rounded to 2dp
}
```

### Tenant-to-customer tag mapping

Several tenants have multiple CDN `Customer` tag values (e.g. `esteelauder` maps to 10 customer tags). The mapping is a hardcoded lookup table `tenantToCustomerMap` inside `awsCostInfo.ts`. CDN costs for all matching customer tags are summed and merged with the SaaS workload costs to produce a total cost per tenant.

---

## 2. AWS EKS

**Source:** `app/_lib/eks/AwsEksClient.ts`

### Service details

| Property | Value |
|---|---|
| Service | Amazon Elastic Kubernetes Service (EKS) |
| AWS endpoint | Regional EKS endpoint (determined by `AWS_REGION`) |
| SDK client | `EKSClient` (`@aws-sdk/client-eks`) |
| SDK command | `DescribeClusterVersionsCommand` |

### Auth

Static IAM credentials via `saasTenantWorkloadsAwsAccessKey` / `saasTenantWorkloadsAwsSecretKey`. A new `EKSClient` instance is created per `AwsEksClient` constructor call (not a singleton).

```ts
this.client = new EKSClient({
  region: env.awsRegion,
  credentials: {
    accessKeyId: env.saasTenantWorkloadsAwsAccessKey,
    secretAccessKey: env.saasTenantWorkloadsAwsSecretKey,
  },
})
```

### Required env vars

| Env var | Maps to |
|---|---|
| `AWS_REGION` | Region for the EKS endpoint |
| `SAAS_TENANT_WORKLOADS_AWS_ACCESS_KEY` | IAM access key ID |
| `SAAS_TENANT_WORKLOADS_AWS_SECRET_KEY` | IAM secret access key |

### API calls made

#### `DescribeClusterVersionsCommand` — list all available EKS versions

Triggered by `AwsEksClient.getAvailableVersions()`. Used by the portal to display EKS version lifecycle information (e.g. when standard/extended support ends).

**Request:**

```ts
{
  includeAll: true,
  maxResults: 100,
  nextToken: string | undefined  // pagination token
}
```

Pagination: the call loops until `response.nextToken` is undefined, accumulating all pages.

**Response fields consumed:**

```
clusterVersions[].clusterVersion          -- version string, e.g. "1.30"
clusterVersions[].releaseDate             -- Date | null
clusterVersions[].endOfStandardSupportDate -- Date | null
clusterVersions[].endOfExtendedSupportDate -- Date | null
clusterVersions[].versionStatus           -- "standard-support" | "extended-support" | "unsupported" | unknown
clusterVersions[].defaultVersion          -- boolean
response.nextToken                        -- pagination cursor
```

Unknown `versionStatus` values default to `"standard-support"`.

**Return shape:**

```ts
interface EksVersionInfo {
  version: string
  releaseDate: Date | null
  standardSupportEndDate: Date | null
  extendedSupportEndDate: Date | null
  versionStatus: 'standard-support' | 'extended-support' | 'unsupported'
  isDefault: boolean
}
```

---

## 3. AWS EC2

**Source:** `app/_lib/awsEc2Data.ts`

### Service details

| Property | Value |
|---|---|
| Service | Amazon EC2 |
| AWS endpoint | Regional EC2 endpoint (per-request region for AZ queries; `AWS_REGION` for region enumeration) |
| SDK client | `EC2Client` (`@aws-sdk/client-ec2`) |
| SDK commands | `DescribeRegionsCommand`, `DescribeAvailabilityZonesCommand` |

### Auth

Static IAM credentials via `saasTenantWorkloadsAwsAccessKey` / `saasTenantWorkloadsAwsSecretKey`. A new `EC2Client` is instantiated per call inside `initRegions()` and `getAvailabilityZones()`. Credentials are validated before use — both values must be non-empty or an error is thrown.

### Required env vars

| Env var | Maps to |
|---|---|
| `AWS_REGION` | Region used for `DescribeRegions` |
| `SAAS_TENANT_WORKLOADS_AWS_ACCESS_KEY` | IAM access key ID |
| `SAAS_TENANT_WORKLOADS_AWS_SECRET_KEY` | IAM secret access key |

### Singleton pattern

`AwsEc2Data` is a lazy singleton. `AwsEc2Data.getInstance()` returns the shared instance, initialising it (and calling `DescribeRegions`) only on the first call. It is wrapped in `withSsrTiming` for performance tracing.

### API calls made

#### `DescribeRegionsCommand` — enumerate all AWS regions

Triggered once during `initRegions()`, which runs as part of `getInstance()` on first access.

**Request:**

```ts
{ AllRegions: true }
```

**Response fields consumed:**

```
Regions[].RegionName     -- region name string, e.g. "us-east-1"
Regions[].Endpoint       -- fallback if RegionName is absent
Regions[].OptInStatus    -- "opted-in" | "opt-in-not-required" | "not-opted-in"
```

`enabled` is `true` when `OptInStatus !== "not-opted-in"`.

**Return shape (cached on instance):**

```ts
interface AwsRegion {
  name: string
  enabled: boolean
}
```

#### `DescribeAvailabilityZonesCommand` — list AZs for a region

Triggered by `getAvailabilityZones(region)` and `getFirstTwoAzs(region)`. A fresh `EC2Client` is created per call, scoped to the requested region.

**Request:**

```ts
{
  Filters: [
    { Name: "state",          Values: ["available"] },
    { Name: "zone-type",      Values: ["availability-zone"] },  // excludes Local Zones, Wavelength Zones
    { Name: "opt-in-status",  Values: ["opt-in-not-required"] } // excludes account-level opt-in zones
  ]
}
```

The filters deliberately exclude Local Zones (e.g. `us-east-1-bos-1a`) and Wavelength Zones because Pulumi/EKS cannot provision into them.

**Response fields consumed:**

```
AvailabilityZones[].ZoneName   -- AZ name string, e.g. "us-east-1a"
```

Returns AZ names sorted alphabetically. `getFirstTwoAzs()` returns only the first two entries from the sorted list.

---

## Credential summary

| Credential pair | Env vars | Used by |
|---|---|---|
| Cost Explorer IAM user | `COST_EXPLORER_AWS_ACCESS_KEY` / `COST_EXPLORER_AWS_SECRET_KEY` | `awsCostInfo.ts` only |
| SaaS tenant workloads IAM user | `SAAS_TENANT_WORKLOADS_AWS_ACCESS_KEY` / `SAAS_TENANT_WORKLOADS_AWS_SECRET_KEY` | `AwsEksClient`, `AwsEc2Data` |

All credentials are static IAM key pairs (not IRSA, not assumed roles, not instance profiles). They are read from `process.env` through `GetEnvironment()` (`app/_lib/Environment.tsx`).
