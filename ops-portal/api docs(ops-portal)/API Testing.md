---
title: API Testing Client Manager Integration
source: ops-portal/app/_lib/tenant/apiTestingClientManager.ts
updated: 2026-06-22
---

# API Testing Client Manager — Outbound Client

## Service Overview

`ApiTestingClientManager` manages API testing credentials (clients / API keys) on behalf of tenants. It calls an external API key management service to list, create, and delete per-tenant API keys used for testing. The class is implemented as a **singleton** (`ApiTestingClientManager.getInstance()`).

## Base URL

Read from `GetEnvironment().apiTestingClientUrl` at construction time.

**Env var:** `API_TESTING_CLIENT_URL`.

## SDK / HTTP Client

The global `fetch` API via the `fetchWithTimeout` wrapper (from `@/_lib/fetchWithTimeout`). No third-party HTTP library.

## Authentication

All requests carry a static API key in the `X-Api-Key` header:

```
X-Api-Key: {clientAccessToken}
```

**Env var:** `API_TESTING_CLIENT_TOKEN`.

The constructor throws if either `API_TESTING_CLIENT_URL` or `API_TESTING_CLIENT_TOKEN` is `null`.

## Required Environment Variables

| Variable | Description |
|---|---|
| `API_TESTING_CLIENT_URL` | Base URL of the API testing key-management service |
| `API_TESTING_CLIENT_TOKEN` | Service-level API key used to authenticate all management calls |

## Timeout

All requests time out after **30 000 ms** (`API_TESTING_CLIENT_TIMEOUT_MS`). All requests set `cache: 'no-store'` to bypass any HTTP cache.

## Methods and Endpoints

### `listClients()`

Returns all API testing clients registered in the service.

```
GET {base}/v1/manage/clients
Headers: X-Api-Key: {token}
```

Response is validated against `ListClientResponseSchema` (Zod). Each item contains:

```ts
{
  apiKey: string
  clientId: string
  clientName: string
  tenantId: string
  validUntil: number   // Unix timestamp ms
  createdBy: string
  createdAt: number
  status: string
  expired: boolean
  scopes: string[]
}
```

---

### `findClient(tenantId)`

Convenience wrapper over `listClients()`. Returns the first client whose `tenantId` matches the argument, or `undefined` if none is found. Does not make a separate API call.

---

### `addClient(tenantId)`

Creates a new API testing client for a tenant.

```
POST {base}/v1/manage/client
Headers: X-Api-Key: {token}
         Content-Type: (set by fetch default)
Body (JSON):
{
  name: "{tenantId}-api-key",
  tenantId: "{tenantId}",
  validUntil: <now + 5 years in ms>,
  role: "updates",
  notes: "Created by Saas Service Portal"
}
```

New clients are always created with role `"updates"` and a **5-year expiration**. The service allows multiple clients per tenant — the Ops Portal code is responsible for preventing duplicates (check with `findClient` first).

Response is validated against `ClientResponseSchema` (same shape as a single item from `listClients`).

---

### `deleteClient(clientId)`

Deletes an API testing client by its `clientId` (not `tenantId`).

```
DELETE {base}/v1/manage/client/{clientId}
Headers: X-Api-Key: {token}
         Content-Type: application/json
```

Throws if `clientId` is `null`. Logs the response status on success but returns `void`.

## Error Handling

All methods re-throw on error after logging with `logger.error`. There is no silent error swallowing. Zod schema validation (`parse`) will throw a `ZodError` if the service returns an unexpected shape.

## Notes

- The `clientId` (returned by the service) and the `tenantId` (the Cequence tenant) are different identifiers. The `clientId` must be stored in tenant info to support future deletion.
- The `addClient` endpoint does not deduplicate — callers must check `findClient` before calling `addClient`.
