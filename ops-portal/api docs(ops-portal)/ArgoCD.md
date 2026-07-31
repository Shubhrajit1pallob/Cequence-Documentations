---
title: ArgoCD API Integration
source: ops-portal/app/_lib/argocd/argoCd.ts
updated: 2026-06-22
---

# ArgoCD — Outbound Client

## Service Overview

The Ops Portal connects to two separate ArgoCD instances — one for the **dev** environment and one for **prod** — to manage the lifecycle of tenant Argo `Application` objects (list, sync, delete, status polling).

## Base URLs

| Environment | Env var |
|---|---|
| Dev | `ARGOCD_URL_DEV` |
| Prod | `ARGOCD_URL_PROD` |

Both values are read from `GetEnvironment()` at construction time via `argoCdUrlDev` / `argoCdUrlProd`.

## SDK / HTTP Client

- **Custom generated SDK** — `@/_lib/argocd-client` (auto-generated from the ArgoCD OpenAPI spec). Provides strongly-typed `ApplicationServiceApi` wrappers.
- **Axios** is the underlying HTTP transport used by the generated client, and also imported directly for error-type checks (`axios.isAxiosError`).

## Authentication

Bearer token auth. Each environment has a separate **read-only** token and an optional **write** token. Tokens are injected into the generated client's `Configuration` at construction:

```ts
new argo.Configuration({
  basePath: url,
  baseOptions: {
    headers: { Authorization: `Bearer ${accessToken}` },
  },
})
```

Four client instances are created:

| Instance | Token env var | Purpose |
|---|---|---|
| `argoCdApiReadDev` | `ARGOCD_TOKEN_READONLY_DEV` | All read operations on dev |
| `argoCdApiWriteDev` | `ARGOCD_TOKEN_WRITE_DEV` | Sync / delete on dev (optional — if absent, write ops throw) |
| `argoCdApiReadProd` | `ARGOCD_TOKEN_READONLY_PROD` | All read operations on prod |
| `argoCdApiWriteProd` | `ARGOCD_TOKEN_WRITE_PROD` | Sync / delete on prod (optional) |

The correct client is selected at call time based on `tenantId.environment` (`'dev'` or `'prod'`) and whether the operation is `READ` or `WRITE`.

## Required Environment Variables

| Variable | Used for |
|---|---|
| `ARGOCD_URL_DEV` | Dev ArgoCD base URL |
| `ARGOCD_URL_PROD` | Prod ArgoCD base URL |
| `ARGOCD_TOKEN_READONLY_DEV` | Read token — dev |
| `ARGOCD_TOKEN_WRITE_DEV` | Write token — dev (optional) |
| `ARGOCD_TOKEN_READONLY_PROD` | Read token — prod |
| `ARGOCD_TOKEN_WRITE_PROD` | Write token — prod (optional) |

## Methods and What They Call

All application names sent to ArgoCD are prefixed with the tenant ID: `${tenantId}-${applicationName}`.

### `getApplications()`

Fetches all `Application` objects from both environments in parallel.

```
GET /api/v1/applications   (dev + prod, via applicationServiceList)
```

Wrapped in `withRetry` (3 attempts, exponential backoff ~1s / ~2s / ~4s + up to 500ms jitter). If one environment returns empty, a warning is logged but the other result is still returned. Called when the portal needs to display a full list of all tenant applications.

---

### `getApplication(tenantId, applicationName)`

Fetches a single application with a hard refresh.

```
GET /api/v1/applications/{name}?refresh=hard   (applicationServiceGet)
```

First calls `isApplicationMissing` to skip the request if the app does not exist. Triggered when the portal needs detailed application state for a specific tenant.

---

### `getApplicationOperationStatus(tenantId, applicationName)`

Polls the `operationState.phase` field of an application to determine if a sync is `Running`, `Succeeded`, `Failed`, or `Unknown`.

```
GET /api/v1/applications/{name}   (applicationServiceGet, no refresh param)
```

Used internally before sync and delete operations to avoid double-triggering.

---

### `isApplicationMissing(tenantId, applicationName)`

Checks existence by attempting a single-app lookup. Returns `true` for HTTP 403, 404, and `PermissionDenied` gRPC errors (ArgoCD v2.14+ returns 403/PermissionDenied for non-existent apps per security fix GHSA-2q5c-qw9c-fmvq).

```
GET /api/v1/applications/{name}   (applicationServiceGet)
```

---

### `syncApplication(tenantId, applicationName)`

Triggers a full application sync with `prune: true`. Before syncing, performs a hard refresh to avoid stale cache issues (especially for `defender`). Retries once on initial failure with a reduced backoff.

```
GET  /api/v1/applications/{name}?refresh=hard
POST /api/v1/applications/{name}/sync
```

Retry strategy sent to ArgoCD: `limit: 3`, backoff `20s` doubling, max `4m` (2m on retry). Called during tenant provisioning and upgrade workflows.

---

### `syncApplicationResources(tenantId, applicationName, resources)`

Scoped sync — reconciles only the listed Kubernetes resources, leaving the rest of the application untouched. Avoids a full-app restart and the resulting traffic dip.

```
GET  /api/v1/applications/{name}?refresh=hard
POST /api/v1/applications/{name}/sync   (body includes resource list, prune: false)
```

---

### `terminateOperation(tenantId, applicationName)`

Terminates an in-progress sync operation. Includes an explicit `Content-Type: application/json` header as a workaround for [argoproj/argo-cd#16923](https://github.com/argoproj/argo-cd/issues/16923).

```
DELETE /api/v1/applications/{name}/operation
```

Called automatically inside `deleteApplication` when an operation is in progress.

---

### `deleteApplication(tenantId, applicationName)`

Deletes an Argo `Application` object. Terminates any in-progress operation first. Also includes the `Content-Type` workaround header.

```
DELETE /api/v1/applications/{name}
```

---

### `getResourceTree(tenantId, applicationName)`

Returns the full resource tree for an application (all managed K8s objects and their relationships).

```
GET /api/v1/applications/{name}/resource-tree
```

---

### `getResourceLiveState(tenantId, applicationName, params)`

Returns the live K8s manifest of a specific resource managed by the application. Response is a JSON-parsed manifest string.

```
GET /api/v1/applications/{name}/resource
    ?namespace=…&name=…&version=…&group=…&kind=…
```

## Error Handling

`ArgoCdError` wraps all thrown errors, carrying `applicationName`, `operation`, HTTP `status`, and `url`. 403/404 and gRPC `PermissionDenied` responses are silently treated as "application not found" and do not throw. 5xx and other errors are always re-thrown.

`withRetry` is used around bulk list operations (not individual tenant operations).
