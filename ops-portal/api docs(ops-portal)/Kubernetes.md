---
title: Kubernetes API Integration
source: ops-portal/app/_lib/kubernetes/kubernetesApiClient.ts
updated: 2026-06-22
---

# Kubernetes — Outbound Client

## Service Overview

`KubernetesApiClient` talks directly to a Kubernetes cluster API server over HTTPS to query pod status. It is used to get defender pod counts for a given namespace. The client is constructed per-cluster, not a singleton — callers instantiate it with the cluster URL, CA cert, and OIDC credentials for that specific cluster.

## Base URL

Passed to the constructor as `clusterApiUrl`. There is no centralised env var read inside this file; the caller is responsible for supplying the correct cluster API server URL.

## SDK / HTTP Client

Plain **Axios** (`axios.create()`). No Kubernetes SDK is used. All API calls go directly to the Kubernetes REST API using standard `axios.get()`.

## Authentication — Two-Step OIDC Flow

The client uses **OIDC client_credentials** to obtain a short-lived bearer token, then attaches it to all Kubernetes API requests.

### Step 1 — Exchange OIDC credentials for a token

```
POST {oidcConfig.issuerUrl}/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
client_id={oidcConfig.clientId}
client_secret={oidcConfig.clientSecret}
scope=openid
```

Timeout: 10 000 ms. The response carries `access_token` and `expires_in` (seconds).

### Step 2 — Use the token on K8s API calls

```
Authorization: Bearer {access_token}
```

The token is cached in memory. It is refreshed automatically **1 minute before expiry** (`now >= tokenExpiresAt - 60000`). When a refresh happens the Axios instance is torn down and recreated with the new token.

### TLS

The cluster CA certificate (`clusterCa`) is supplied as a base-64–encoded string. The client decodes it and passes it to an `https.Agent` with `rejectUnauthorized: true`. Self-signed / private-CA clusters are handled correctly this way.

## Constructor Parameters (not env vars)

`KubernetesApiClient` takes all config through its constructor — there are no direct `process.env` reads in this file:

| Parameter | Type | Description |
|---|---|---|
| `clusterApiUrl` | `string` | K8s API server base URL |
| `clusterCa` | `string` | Base-64 encoded cluster CA certificate |
| `oidcConfig.issuerUrl` | `string` | OIDC issuer URL (token endpoint = `{issuerUrl}/token`) |
| `oidcConfig.clientId` | `string` | OIDC client ID |
| `oidcConfig.clientSecret` | `string` | OIDC client secret |

The caller (not shown in this file) is expected to source these values from environment variables or secrets.

## Methods and What They Call

### `getDefenderPodCounts(namespace, labelSelector?)`

Public entry point. Lists pods in a namespace and returns a summary count broken down by status.

Internally calls `listPods`, then iterates over results:

- `Running` + `Ready=True` condition → `runningPods`
- `Running` without `Ready=True`, `Pending`, or unknown phase → `pendingPods`
- `Failed` → `failedPods`
- `Succeeded` (completed Job pods) → ignored

Returns `null` if the pod list cannot be retrieved (errors are caught and logged, not thrown).

---

### `listPods(namespace, labelSelector?)` *(private)*

```
GET /api/v1/namespaces/{namespace}/pods
    [?labelSelector={encoded}]
```

Timeout: 30 000 ms (set on the Axios instance). Returns a `K8sPodListResponse` or `null` on error.

---

### `getAuthenticatedClient()` *(private)*

Manages token lifecycle and Axios instance creation. Returns a configured `AxiosInstance` ready to call the K8s API. Triggers `getOidcToken()` when the token is missing or within 1 minute of expiry.

---

### `getOidcToken()` *(private)*

Performs the OIDC client_credentials exchange described above. Returns the raw `OidcTokenResponse` or `null` on failure.

## Interfaces Exported

```ts
interface OidcConfig {
  issuerUrl: string
  clientId: string
  clientSecret: string
}

interface PodStatus {
  totalPods: number
  runningPods: number
  pendingPods: number
  failedPods: number
}
```

## Error Handling

All public methods return `null` rather than throwing. Errors are logged via `logger.error` with context (namespace, label selector, HTTP status). Axios errors are checked with `instanceof AxiosError` to extract the response status.
