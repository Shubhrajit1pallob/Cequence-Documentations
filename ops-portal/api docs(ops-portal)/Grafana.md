---
title: Grafana API Integration
last_updated: 2026-06-22
sources:
  - app/api/external/grafana/alerts/route.ts
  - app/_lib/grafana.ts
  - app/_lib/grafanaWorkloadCostInfo.ts
---

# Grafana API Integration

The Ops Portal both **receives** webhook events from Grafana (inbound) and **calls** the Grafana Cloud API (outbound). These are entirely separate flows; the sections below cover each.

---

## Inbound Routes

### `POST /api/external/grafana/alerts`

**Source:** `app/api/external/grafana/alerts/route.ts`

**Purpose:** Receives alert-state webhooks pushed by Grafana Alerting. Each call carries a batch of one or more alerts that have just fired or resolved. The handler verifies the request signature, validates the payload shape, and dispatches alerts to the internal `dispatchGrafanaAlerts` handler (which routes them to downstream actions — e.g. Slack notifications, ticket creation).

#### Auth mechanism

HMAC-SHA256 signature verification. Grafana attaches a hex digest in the `X-Grafana-Signature` header; the handler recomputes `HMAC-SHA256(secret, rawBody)` using `GRAFANA_WEBHOOK_SECRET` and compares with `timingSafeEqual`. If the header is absent or the digest does not match, the request is rejected with `401`.

No JWT / session auth is involved — this endpoint is meant to be called by Grafana, not by a browser user.

**Required env var:**

| Variable | Description |
|---|---|
| `GRAFANA_WEBHOOK_SECRET` | Shared secret configured in the Grafana webhook contact point. Must match exactly. |

#### Request

```
POST /api/external/grafana/alerts
Content-Type: application/json
X-Grafana-Signature: <hex-encoded HMAC-SHA256 of body>
```

Body (`GrafanaWebhookPayload`):

```jsonc
{
  "alerts": [
    {
      "status": "firing",           // "firing" | "resolved"
      "labels": {                   // arbitrary key-value labels from Grafana
        "alertname": "HighErrorRate",
        "tenant": "acme",
        "environment": "prod"
      },
      "annotations": {              // arbitrary annotations
        "summary": "Error rate above 5%"
      },
      "values": {                   // optional — numeric metric values at alert time
        "errorRate": 0.07
      },
      "fingerprint": "abc123",      // unique alert identity string
      "startsAt": "2026-06-01T00:00:00Z",
      "endsAt": "2026-06-01T01:00:00Z"  // optional; absent while still firing
    }
  ]
}
```

The `alerts` array **must** be present and must be an array; otherwise the handler returns `400 Malformed payload`.

#### Responses

| Status | Condition |
|---|---|
| `200` (empty body) | Signature valid, payload accepted, dispatch completed successfully. |
| `400 {"error":"Malformed payload"}` | `alerts` field missing or not an array. |
| `400 {"error":"<message>"}` | `dispatchGrafanaAlerts` threw a `BadRequestError`. |
| `401 {"error":"Invalid signature"}` | Missing or mismatched `X-Grafana-Signature`. |
| `500 {"error":"Internal error"}` | Unexpected error during dispatch. |

---

## Outbound Client

### `grafana.ts` — `Grafana` class

**Source:** `app/_lib/grafana.ts`

**External service:** Grafana Cloud (`https://cequence.grafana.net`)

**HTTP client:** `axios` (`AxiosInstance`), constructed once per `Grafana` instance with a 10 000 ms timeout.

**Auth mechanism:** Bearer token — `Authorization: Bearer <GRAFANA_API_KEY>` on every request.

**Required env var:**

| Variable | Description |
|---|---|
| `GRAFANA_API_KEY` | Grafana Cloud service-account API key with read access to the Prometheus datasource and read/write access to Alertmanager silences. |

All requests pass through `instrumentGrafanaRequest(method, path, fn)` for observability (timing/tracing).

#### Methods and endpoints called

---

##### `getAverageMessagesPerSecondPerDay()`

**Trigger:** Scheduled sync / tenant dashboard data load — fetches weekly TPS history for the tenants list.

**Endpoint:** `POST /api/ds/query`

Runs a Prometheus range query via the Grafana datasource proxy. The PromQL expression is:

```
sum by (tenant, environment) (
  rate(kafka_server_brokertopicmetrics_messagesin_total{
    topic="cq.sensor.data.logs", tenant !=""
  }[1d])
)
```

Time window: last 7 days (up to end of yesterday). Interval: `1d`. Returns `Record<string, number[]>` keyed by `"<tenant>-<environment>"`, each value being an array of up to 7 daily TPS averages.

---

##### `getAlertManagerSilences(tenantName, environment)`

**Trigger:** Tenant detail page load — shows active downtime/silences for a tenant.

**Endpoint:** `GET /api/alertmanager/grafana/api/v2/silences?filter=environment=<env>&filter=tenant=<tenant>`

Returns non-expired silences mapped to `{ name, details: Downtime }[]`. Returns `[]` on error (non-throwing).

---

##### `createTenantScheduledAlertSilence(tenantLabel, tenantName, environment, message, createdBy, start, end, silenceTrafficOnly?)`

**Trigger:** User creates a downtime window for a tenant in the portal.

**Endpoint:** `POST /api/alertmanager/grafana/api/v2/silences`

Request body sent to Grafana:

```jsonc
{
  "comment": "<message>",
  "createdBy": "<createdBy>",
  "startsAt": "<ISO date>",
  "endsAt": "<ISO date>",        // omitted if end is falsy
  "matchers": [
    { "isEqual": true, "isRegex": false, "name": "<tenantLabel>", "value": "<tenantName>" },
    { "isEqual": true, "isRegex": false, "name": "environment",   "value": "<environment>" },
    // added only when silenceTrafficOnly=true:
    { "isEqual": true, "isRegex": false, "name": "component",     "value": "infra" }
  ]
}
```

`tenantLabel` is either `"tenant"` or `"customer"` depending on which Grafana label schema is in use for that tenant.

---

##### `removeSilence(silenceId)`

**Trigger:** User ends a downtime window early, or `removeAllTenantAlertSilences` calls it per-silence.

**Endpoint:** `DELETE /api/alertmanager/grafana/api/v2/silence/<silenceId>`

Throws on error.

---

##### `removeAllTenantAlertSilences(tenantName, environment)`

**Trigger:** Tenant deletion / bulk-cancel downtime.

Fetches active silences via the GET silences endpoint (same as `getAlertManagerSilences`), then calls `removeSilence` on each using `Promise.allSettled`. Partial failures are logged as warnings but do not throw. Full list of results (`PromiseSettledResult[]`) is returned.

---

##### `getTenantUapVersion()`

**Trigger:** Tenant list / version dashboard — reads the currently installed UAP version for each tenant from Kubernetes labels in Prometheus.

**Endpoint:** `POST /api/ds/query`

PromQL (instant, `now-5m` to `now`):

```
group by (tenant, label_app_kubernetes_io_version)(
  kube_deployment_labels{tenant=~".*", label_app_kubernetes_io_component="ui"}
)
```

Returns `Array<{ name: string; uapVersionInstalled: string }>`. Returns `[]` on error.

---

##### `checkEksNodeVersions(tenantName, environment, expectedEksVersion?)`

**Trigger:** UAP upgrade workflow — verifies all EKS nodes in a tenant's cluster are on the expected Kubernetes version before proceeding.

**Endpoint:** `POST /api/ds/query`

PromQL (instant, `now-5m` to `now`):

```
kube_node_info{tenant="<tenantName>", environment="<environment>"}
```

Returns:

```ts
{
  allNodesOnSameVersion: boolean,
  allNodesMatchExpected: boolean | null,   // null if expectedEksVersion not supplied
  nodeVersions: Array<{ nodeName: string; kubeletVersion: string }>,
  uniqueVersions: string[]
}
```

Throws on error.

---

##### `listDataSources()`

**Trigger:** Internal tooling / diagnostic calls — enumerates datasources configured in the Grafana org.

**Endpoint:** `GET /api/datasources`

Returns `Array<{ id, uid, name, type, url?, isDefault? }>`. Returns `[]` on error.

---

##### `searchPrometheusMetricNames(datasourceUid, pattern)`

**Trigger:** Internal tooling — discovers which OpenCost / kube-state metrics are available.

**Endpoint:** `GET /api/datasources/proxy/uid/<uid>/api/v1/label/__name__/values`

Filters the full metric name list by the supplied `RegExp`. Returns `string[]`. Returns `[]` on error.

---

##### `getPrometheusLabelValues(datasourceUid, labelName, metric?)`

**Trigger:** Internal tooling — fetches all values for a given Prometheus label (e.g. `cluster`, `tenant`), optionally scoped to a specific metric.

**Endpoint:**
- With metric: `GET /api/datasources/proxy/uid/<uid>/api/v1/label/<label>/values?match[]=<metric>`
- Without metric: `GET /api/datasources/proxy/uid/<uid>/api/v1/label/<label>/values`

Returns `string[]`. Returns `[]` on error.

---

### `grafanaWorkloadCostInfo.ts` — workload cost fetcher

**Source:** `app/_lib/grafanaWorkloadCostInfo.ts`

**External service:** Grafana Cloud (`https://cequence.grafana.net`) — same org, same Prometheus datasource as `grafana.ts` but a separate module with its own cached `AxiosInstance`.

**HTTP client:** `axios`, 60 000 ms timeout (longer than the main client because cost queries can be expensive). The instance is module-level cached and reused across calls as long as `apiKey` and `baseUrl` are unchanged.

**Auth mechanism:** Bearer token — same `GRAFANA_API_KEY` as the main client.

**Feature flag:** The entire module is gated on `GRAFANA_COST_ENABLED=true`. If the flag is absent, `false`, or any value other than the case-insensitive string `"true"`, all functions return early without making network calls. `isGrafanaCostConfigured()` can be used by callers to check this at runtime.

**Required env vars:**

| Variable | Default | Description |
|---|---|---|
| `GRAFANA_API_KEY` | — | Grafana Cloud API key (shared with main client). |
| `GRAFANA_COST_ENABLED` | `"false"` | Must be exactly `"true"` (case-insensitive) to enable. |
| `GRAFANA_COST_DATASOURCE_UID` | `"grafanacloud-prom"` | UID of the Prometheus datasource that has OpenCost metrics. |

#### Functions

---

##### `isGrafanaCostConfigured(): boolean`

Returns `true` if all three env vars are present and `GRAFANA_COST_ENABLED=true`. No network call.

---

##### `getWorkloadCostsForMonths(months: Date[]): Promise<GrafanaWorkloadCostRow[]>`

**Trigger:** Scheduled cost-sync job — called once per billing cycle to populate per-workload cost data for one or more calendar months.

**Endpoint:** `POST /api/ds/query` (one request per month supplied)

For each month, an instant PromQL query is executed at the end of the month window (capped at `now` so future months are handled safely). The query computes total monthly cost per workload by combining CPU and memory allocation metrics with per-node hourly cost rates:

```promql
(
  sum by (cluster, namespace, workload, workload_type) (
    avg_over_time(container_cpu_allocation[<Nd>])
    * on (cluster, instance) group_left
    avg_over_time(node_cpu_hourly_cost[<Nd>])
  )
) + (
  sum by (cluster, namespace, workload, workload_type) (
    (avg_over_time(container_memory_allocation_bytes[<Nd>]) / (1024 * 1024 * 1024))
    * on (cluster, instance) group_left
    avg_over_time(node_ram_hourly_cost[<Nd>])
  )
)
```

Where `<N>` is the number of days in the window (1–31, rounded to nearest day). The PromQL result is an average $/hr value; the code multiplies by the actual hours covered to produce a total dollar cost for the window.

Rows with non-finite or zero cost are silently dropped.

**Return type:**

```ts
interface GrafanaWorkloadCostRow {
  month: Date        // first day of the month (UTC midnight)
  cluster: string
  namespace: string
  workload: string
  workloadType: string
  cost: number       // total USD for the window, rounded to 2 decimal places
}
```

Per-month query failures are logged and skipped; partial results are returned rather than throwing.

---

##### `clusterNameMatchesTenant(clusterName: string, tenantName: string): boolean`

**No network call.** Pure helper used by the cost-sync caller to match Grafana `cluster` label values (which follow an `<tenant>-eks` naming convention) back to tenant names. Strips a trailing `-eks` suffix and compares case-insensitively.

```ts
clusterNameMatchesTenant('acme-eks', 'acme')   // true
clusterNameMatchesTenant('acme-eks', 'ACME')   // true
clusterNameMatchesTenant('acme', 'acme')        // true
clusterNameMatchesTenant('acme-eks', 'other')   // false
```

---

## Environment variable summary

| Variable | Used by | Required | Default |
|---|---|---|---|
| `GRAFANA_API_KEY` | `grafana.ts`, `grafanaWorkloadCostInfo.ts` | Yes | `""` |
| `GRAFANA_WEBHOOK_SECRET` | `webhookAuth.ts` (inbound) | Yes | `""` |
| `GRAFANA_COST_ENABLED` | `grafanaWorkloadCostInfo.ts` | No | `"false"` |
| `GRAFANA_COST_DATASOURCE_UID` | `grafanaWorkloadCostInfo.ts` | No | `"grafanacloud-prom"` |
