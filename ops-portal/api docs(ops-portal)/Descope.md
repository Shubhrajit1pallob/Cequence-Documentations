---
title: Descope Auth Integration
source: ops-portal/app/_lib/auth/descopeAuth.ts
updated: 2026-06-22
---

# Descope — Outbound Client

## Service Overview

The Ops Portal uses **Descope** as its identity/auth layer for validating inbound JWT bearer tokens on API routes. This file contains no inbound server logic — it is purely an outbound client that calls the Descope service to validate tokens, exchange access keys, and look up role/permission assignments via the Descope Management API.

The Descope SDK is initialised lazily (first call to `getDescopeSDK()`), then reused as a module-level singleton.

## SDK

**`@descope/node-sdk`** (`createSdk`). This is the official Descope Node.js SDK, not a raw HTTP client.

## Authentication — Descope Project Credentials

```ts
createSdk({
  projectId: env.descopeProjectId,
  managementKey: env.descopeManagementKey || undefined,
})
```

`managementKey` is optional but required for any Management API calls (access-key lookup, role search).

## Required Environment Variables

| Variable | Description |
|---|---|
| `DESCOPE_PROJECT_ID` | Descope project ID — required for all SDK operations. SDK init throws if absent. |
| `DESCOPE_MANAGEMENT_KEY` | Descope management API key — required for `lookupAccessKeyAuth`. Optional otherwise. |
| `DESCOPE_TENANT_ID` | The Cequence Descope tenant ID. Required for extracting tenant-scoped roles/permissions from a validated JWT. Validation returns `valid: false` if absent. |

## Exported Functions

### `validateDescopeToken(token)`

Main entry point used by API route middleware. Accepts a raw bearer value — either a Descope session JWT or a raw access key — and returns a `TokenValidationResult`.

**Flow:**

1. **Detect token type.** If the value does not have exactly 3 dot-separated segments, it is treated as a raw access key (not a JWT).

2. **Access key path** — exchanges the raw key for a JWT via `sdk.exchangeAccessKey(token)`. Fails fast with `valid: false` if the key is invalid or revoked.

3. **JWT validation** — calls `sdk.validateSession(sessionToken)`. Fails with `valid: false` if the session is expired or invalid.

4. **Extract tenant-scoped claims.** Looks up `tokenData.tenants[DESCOPE_TENANT_ID]` and extracts `permissions` and `roles` from that tenant entry only. Tokens that do not belong to the configured tenant return `valid: false`.

5. **Fallback for project-scoped RBAC.** If an access key was exchanged but the resulting JWT contains no role/permission claims (project-level roles are not embedded in JWTs by default), the Management API is called via `lookupAccessKeyAuth` to retrieve them. Results are cached for 5 minutes.

Returns:

```ts
interface TokenValidationResult {
  valid: boolean
  payload?: DescopeTokenPayload  // decoded token claims + extracted roles/permissions
  error?: string
  serviceIdentifier?: string     // value of the `sub` claim
}
```

---

### `lookupAccessKeyAuth(sdk, accessKeyId)` *(module-private)*

Calls the Descope Management API to resolve role and permission assignments for an access key. Used only when an exchanged JWT is missing claims (project-scoped RBAC).

```
sdk.management.accessKey.load(accessKeyId)    // fetch key's roleNames + keyTenants[].roleNames
sdk.management.role.search({ roleNames: … })  // expand roles → permissionNames
```

Results are cached in an in-memory `Map<string, CachedAccessKeyAuth>` with a **5-minute TTL** per access key. Not called on standard session JWTs.

---

### `hasRole(payload, allowedRoles)`

Checks whether the decoded `payload.roles` array contains at least one of the `allowedRoles`. Returns `false` if `payload.roles` is empty. Logs a debug message on failure.

---

### `hasPermission(payload, requiredPermissions)`

Checks whether the decoded `payload.permissions` array satisfies a required permission. Implements four matching rules in order:

| Rule | Example |
|---|---|
| 1. Exact match | `tenants:manage` matches `tenants:manage` |
| 2. Environment-scoped equivalence | `tenants:dev:manage` grants `tenants:manage` |
| 3. Env-scoped action grants `:manage` | `tenants:dev:reset` grants `tenants:manage` (any non-`:read` action) |
| 4. `:manage` implies `:read` | `tenants:manage` grants `tenants:read` (rules 1–3 also apply transitively) |

Pattern for environment-scoped permissions: `{resource}:(dev|prod):{action}`.

---

### `getServiceIdentifier(payload)`

Returns `payload.sub` or `'unknown-service'` if `sub` is empty. Used for logging and audit purposes.

## Role Constants

```ts
export const SERVICE_ROLE_READ_ONLY = 'api-read-only'
export const SERVICE_ROLE_WRITE     = 'api-write'
```

## Access Key Cache

```ts
const accessKeyAuthCache = new Map<string, CachedAccessKeyAuth>()
const ACCESS_KEY_CACHE_TTL_MS = 5 * 60 * 1000   // 5 minutes
```

Cached per `accessKeyId` (the `sub` claim from the exchanged JWT). The cache is module-level (process lifetime), so it persists across requests in the same Node.js process. Not distributed — each pod maintains its own cache.

## Notes

- Descope permissions are **tenant-scoped**. Permissions from other tenants in the JWT are intentionally ignored to prevent privilege escalation.
- Raw access keys (paste-in style, e.g. from MCP config) are fully supported via the exchange flow without requiring callers to pre-convert them.
- The `managementKey` is only exercised on the project-scoped RBAC fallback path — it is not needed for standard session-JWT validation.
