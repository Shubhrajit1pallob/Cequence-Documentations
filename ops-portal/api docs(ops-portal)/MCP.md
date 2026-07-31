---
title: MCP Routes — Ops Portal External API
created: 2026-06-22
---

# MCP Routes

These four routes expose Model Context Protocol (MCP) servers over HTTP using the **Streamable HTTP** transport from `@modelcontextprotocol/sdk`. Each route is stateless — there are no persistent sessions and no DELETE handler. Both `POST` (JSON-RPC request) and `GET` (SSE event stream) are supported on every endpoint.

All routes live under `/api/external/mcp/` and are authenticated via Descope bearer token (see [[Auth — External API]] for the `withDescopeAuth` wrapper). The caller must present a token that carries the required permission scope(s) listed per route below.

> **Transport note.** `sessionIdGenerator` is set to `undefined` on every transport, confirming the stateless posture — no session ID is issued and the server is re-instantiated per request.

---

## Inbound Routes

---

### 1. Accounts MCP

| Field | Value |
|---|---|
| **Path** | `/api/external/mcp/accounts` |
| **Methods** | `GET`, `POST` |
| **MCP server name** | `cequence-accounts` (v0.1.0) |
| **Required permission** | `tenants:prod:read` |
| **Source** | `app/api/external/mcp/accounts/route.ts` |
| **Server builder** | `app/_lib/customer/mcp/accountsMcpServer.ts` |

#### Purpose

Exposes a read-only view of Salesforce account data stored in MongoDB (`CustomerModel`). Intended for AI agent use to look up and count customer accounts by industry, region, and status.

#### MCP Tools

**`list_accounts_by_industry_and_region`**

Returns up to 500 account records. All filter parameters are optional.

```
Input:
{
  industry?:      string   // case-insensitive regex match
  region?:        string   // case-insensitive regex match
  accountStatus?: string   // case-insensitive regex match
}

Output (array of objects, max 500):
{
  name:               string
  sfdcId:             string
  industry:           string
  region:             string
  accountStatus:      string
  type:               string
  customerSinceDate:  string | null
  churnDate:          string | null
  activePov:          boolean
}
```

**`count_accounts_by_industry_and_region`**

Cheap count-only variant of the same filters. Does not return account records.

```
Input:
{
  industry?:      string
  region?:        string
  accountStatus?: string
}

Output:
{
  count: number
}
```

#### MCP Resources

| URI | Description |
|---|---|
| `accounts://industries` | Distinct sorted list of all industry values |
| `accounts://regions` | Distinct sorted list of all region values |
| `accounts://statuses` | Distinct sorted list of all accountStatus values |

#### Auth

Descope bearer token with permission `tenants:prod:read`.

#### Error Response (all MCP routes share this shape)

```json
HTTP 500
{
  "jsonrpc": "2.0",
  "error": { "code": -32603, "message": "Internal MCP error" },
  "id": null
}
```

---

### 2. Margins MCP

| Field | Value |
|---|---|
| **Path** | `/api/external/mcp/margins` |
| **Methods** | `GET`, `POST` |
| **MCP server name** | `cequence-margins` (v0.2.0) |
| **Required permission** | `margins:read` |
| **Source** | `app/api/external/mcp/margins/route.ts` |
| **Server builder** | `app/_lib/margin/mcp/marginMcpServer.ts` |

#### Purpose

Exposes gross margin reporting data for AI-assisted finance review. Provides monthly cost/ARR/MRR/margin data per customer, trend analysis, top-N rankings, and month-over-month comparisons. Consumed via the Cequence AI Gateway, which proxies to this endpoint using a service credential bound to the `ops-portal-margin-reader` role.

> **Sensitivity note (from source):** `margins:read` must not be bundled into broader permission roles. Treat as highly confidential.

#### MCP Tools

**`get_monthly_margin_report`**

Full per-customer monthly gross margin report.

```
Input:
{
  month: string   // e.g. "2026-05"
}

Output (array):
{
  customer:           string
  costUsd:            number
  arrUsd:             number
  monthlyRevenueUsd:  number   // MRR
  marginUsd:          number
  marginPct:          number
  revenueSource:      "contract" | "salesforce-account" | "none"
}
```

**`get_customer_margin_detail`**

12-month trend plus per-tenant cost breakdown for one customer.

```
Input:
{
  customerName: string
  month:        string   // anchor month, e.g. "2026-05"
}

Output:
{
  customer:    string
  trend:       Array<{ month, costUsd, arrUsd, marginUsd, marginPct }>
  tenants:     Array<{ tenantName, costUsd }>
}
```

**`top_n_margins`**

Top or bottom N customers ranked by a chosen metric.

```
Input:
{
  month:     string
  n:         number
  sortBy:    "marginUsd" | "marginPct" | "costUsd" | "monthlyRevenueUsd" | "arrUsd"
  direction: "asc" | "desc"
}

Output: Array<margin report row> (same shape as get_monthly_margin_report)
```

**`summarize_month`**

Headline totals and revenue-source bucket counts only — no per-customer rows.

```
Input:
{
  month: string
}

Output:
{
  month:                  string
  totalCostUsd:           number
  totalRevenueUsd:        number
  totalMarginUsd:         number
  avgMarginPct:           number
  customerCount:          number
  revenueSourceCounts:    { contract: number, "salesforce-account": number, none: number }
}
```

**`compare_months`**

Per-customer deltas between two months.

```
Input:
{
  monthA: string   // earlier month
  monthB: string   // later month
}

Output (array per customer):
{
  customer:       string
  deltaCostUsd:   number
  deltaMarginUsd: number
  deltaMarginPct: number
  deltaArrUsd:    number
}
```

**`list_customers_missing_arr`**

Customers that have production tenants with cost data but no ARR record.

```
Input:
{
  month: string
}

Output: Array<{ customer: string, costUsd: number }>
```

#### MCP Resources

| URI | Description |
|---|---|
| `margins://glossary` | Markdown definitions for ARR, ACV, MRR, TCV |
| `margins://schema/revenue-sources` | Revenue source enum values and precedence rules (JSON) |
| `margins://months/available` | Sorted list of months that have cost data |
| `margins://customers` | Canonical customer list with ARR snapshot |

Revenue source precedence: `contract` > `salesforce-account` > `none`.

#### MCP Prompts

| Prompt name | Description |
|---|---|
| `monthly-finance-review` | Structured end-of-month finance review |
| `customer-deep-dive` | 12-month QBR / churn-risk deep-dive for one customer |
| `compare-recent-months` | Auto-selects last two closed months and writes a MoM diff |

#### Auth

Descope bearer token with permission `margins:read`. Consumed via AI Gateway service credential (`ops-portal-margin-reader` role).

---

### 3. Audit MCP

| Field | Value |
|---|---|
| **Path** | `/api/external/mcp/audit` |
| **Methods** | `GET`, `POST` |
| **MCP server name** | `portal-audit-mcp` (v1.0.0) |
| **Required permission** | `audit:read` |
| **Source** | `app/api/external/mcp/audit/route.ts` |
| **Server builder** | `app/_lib/mcp/auditMcpServer.ts` |
| **Tool registration** | `app/_lib/mcp/auditTools.ts` |

#### Purpose

Exposes read-only access to the ops-portal audit log for AI-assisted investigation and compliance review. All four tools are registered via `registerAuditTools()`. Consumed via the Cequence AI Gateway using a service credential bound to the `ops-portal-audit-reader` role.

> **Sensitivity note (from source):** `audit:read` must not be bundled into broader permission roles. Audit log data is highly confidential.

#### MCP Tools

**`search_audit_events`**

Filtered search across audit events with optional pagination.

```
Input:
{
  start?:         string    // ISO-8601 datetime; defaults to 24h before end
  end?:           string    // ISO-8601 datetime; defaults to now
  actions?:       string[]  // AuditActions enum values
  resourceTypes?: string[]  // AuditTargets enum values
  statuses?:      string[]
  users?:         string[]
  resourceId?:    string
  search?:        string    // free-text search
  includeSystem?: boolean
  limit?:         number    // 1–10000, default 1000
}

Output:
{
  count:  number
  events: Array<audit event record>
}
```

**`get_audit_event`**

Fetch a single audit event by its MongoDB `_id`.

```
Input:
{
  id: string   // MongoDB _id
}

Output: audit event record, or error ToolResult if not found
```

**`list_audit_facets`**

Returns all distinct values useful for building filter UIs or prompts.

```
Input:
{
  start?: string   // ISO-8601
  end?:   string   // ISO-8601
}

Output:
{
  users:         string[]   // distinct users in time range
  resourceIds:   string[]   // distinct resourceIds in time range
  actions:       string[]   // all AuditActions enum values
  resourceTypes: string[]   // all AuditTargets enum values
  statuses:      string[]   // all status enum values
}
```

**`audit_stats`**

Grouped aggregate counts over a time range.

```
Input:
{
  start?:   string    // ISO-8601
  end?:     string    // ISO-8601
  groupBy:  string    // one of AUDIT_GROUP_BY_FIELDS enum: "action" | "status" | "user" | "resourceType"
}

Output:
{
  groupBy: string
  rows:    Array<{ key: string, count: number }>
}
```

#### Auth

Descope bearer token with permission `audit:read`. Consumed via AI Gateway service credential (`ops-portal-audit-reader` role).

---

### 4. Account Opportunities MCP

| Field | Value |
|---|---|
| **Path** | `/api/external/mcp/account-opportunities` |
| **Methods** | `GET`, `POST` |
| **MCP server name** | `cequence-account-opportunities` (v0.1.0) |
| **Required permission** | `tenants:prod:read` |
| **Source** | `app/api/external/mcp/account-opportunities/route.ts` |
| **Server builder** | `app/_lib/salesforce/mcp/accountOpportunitiesMcpServer.ts` |

#### Purpose

Exposes Salesforce Account and Opportunity data for AI-assisted sales analysis. Supports both direct lookup by Salesforce ID and fuzzy name search with disambiguation. Data is sourced from `SalesforceOpportunityRepo`.

#### MCP Tools

**`get_account_with_opportunities`**

Returns one Account record plus all its Opportunities. Accepts lookup by `sfdcId` (preferred) or case-insensitive `accountName`. If multiple accounts match the name, returns a candidate list without drilling into any of them.

```
Input:
{
  sfdcId?:         string    // preferred; Salesforce account ID
  accountName?:    string    // case-insensitive substring match fallback
  closedWonOnly?:  boolean   // filter to closed-won opportunities only
  sinceDate?:      string    // ISO date; only opps created on/after this date
}

Output (single match):
{
  account: {
    sfdcId:    string
    name:      string
    // ...other account fields
  }
  opportunities: Array<{
    sfdcId:              string
    name:                string
    stageName:           string
    amount:              number | null
    closeDate:           string
    isClosed:            boolean
    isWon:               boolean
    contractSfdcId:      string | null
    commercialTermYears: number | null
    ceqArrUsd:           number | null
    ceqAcvUsd:           number | null
    ceqTcvUsd:           number | null
    ceqNarrUsd:          number | null
    arrUsd:              number | null
    createdDate:         string
  }>
}

Output (ambiguous name match):
{
  candidates: Array<{ sfdcId: string, name: string }>
}
```

**`find_accounts`**

Substring search across `Customer.name` for disambiguation. Returns up to 20 candidates.

```
Input:
{
  query: string   // substring to search
}

Output:
{
  accounts: Array<{ sfdcId: string, name: string }>   // max 20
}
```

#### MCP Resources

| URI | Description |
|---|---|
| `account-opportunities://glossary` | Markdown definitions for Opportunity fields and ARR/ACV/TCV/ceq* terminology |

#### Auth

Descope bearer token with permission `tenants:prod:read`.

---

## Shared Error Response

All four routes return this JSON-RPC error shape on unhandled exceptions:

```json
HTTP 500
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Internal MCP error"
  },
  "id": null
}
```

Errors are also logged via `logger.error` with the original exception under the `err` key.

---

## Common Notes

- **Runtime:** All routes set `export const runtime = 'nodejs'` and `export const revalidate = 0` (no static caching).
- **Transport:** `WebStandardStreamableHTTPServerTransport` from `@modelcontextprotocol/sdk/server/webStandardStreamableHttp.js`. Each request builds a fresh server + transport — no shared state between requests.
- **Session management:** `sessionIdGenerator: undefined` — the transport issues no session IDs. Stateless by design; no DELETE endpoint exists on any route.
- **Auth wrapper:** `withDescopeAuth(handler, [permissions])` from `@/_lib/auth/externalApiAuth`. Validates the Descope bearer token and checks that it carries all listed permission scopes before invoking the handler.
- **Gateway consumption:** The `margins` and `audit` routes are specifically noted as being consumed via the Cequence AI Gateway, which proxies with service credentials. The `accounts` and `account-opportunities` routes share the `tenants:prod:read` scope and are presumably consumed the same way.
