---
title: Defender Pools — External API
created: 2026-06-22
source: ops-portal/app/api/external/defender-pools/[tenant]/[poolId]/
---

# Defender Pools — External API

All routes live under `/api/external/defender-pools/[tenant]/[poolId]/`.

**Path parameters (all routes)**

| Parameter | Description |
|-----------|-------------|
| `tenant`  | Full tenant ID string (e.g. `acme-prod`). The environment (`dev`/`prod`) is derived from this via `TenantId.fromIdString()`. |
| `poolId`  | Identifier of the specific defender pool. |

---

## Authentication (all routes)

Every route goes through `authenticateRequest()` from `externalTenantAuth.ts`.

- Expects an `Authorization: Bearer <token>` header.
- The token is a **Descope JWT**, validated against `DESCOPE_PROJECT_ID` and scoped to `DESCOPE_TENANT_ID`.
- Required env vars: `DESCOPE_PROJECT_ID`, `DESCOPE_TENANT_ID`.
- On failure: `401 Unauthorized` with `code: MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, or `INVALID_TOKEN`.

After token validation, each route performs a **permission check** against the token payload. Permissions are environment-scoped (`dev`/`prod`) derived from the tenant ID in the path.

**Permission strings**

| Permission constant | String value |
|---------------------|-------------|
| `DefenderPoolsRead` | `defender-pools:read` |
| `DefenderPoolsManage` | `defender-pools:manage` |
| `DefenderPoolsDevRead` | `defender-pools:dev:read` |
| `DefenderPoolsProdRead` | `defender-pools:prod:read` |
| `DefenderPoolsDevManage` | `defender-pools:dev:manage` |
| `DefenderPoolsProdManage` | `defender-pools:prod:manage` |

**Read permission** (`hasDefenderPoolReadPermission`): granted if caller holds `defender-pools:<env>:read`, OR `defender-pools:read`, OR `defender-pools:manage` (manage implies read).

**Manage permission** (`hasDefenderPoolManagePermission`): granted if caller holds `defender-pools:<env>:manage`, OR `defender-pools:manage`.

---

## Common error response shape

```json
{
  "success": false,
  "status": "<HTTP status text>",
  "error": "<human-readable message>",
  "code": "<ErrorCode>"
}
```

Error codes used: `INSUFFICIENT_PERMISSIONS`, `RESOURCE_NOT_FOUND`, `VALIDATION_ERROR`, `WORKFLOW_CONFLICT`, `INTERNAL_SERVER_ERROR`.

---

## Inbound Routes

### 1. GET /api/external/defender-pools/[tenant]/[poolId]

*(no sub-path — this is the pool root)*

This route is defined in `route.ts` at the `[poolId]` level but only exports `DELETE`. There is no `GET` at the pool root.

---

### 2. DELETE /api/external/defender-pools/[tenant]/[poolId]

**Source:** `app/api/external/defender-pools/[tenant]/[poolId]/route.ts`

**Purpose:** Deletes the specified defender pool.

**Auth:** `defender-pools:manage` (env-scoped or unscoped).

**Request body:** None.

**Success response (200)**

```json
{
  "success": true,
  "message": "<string>",
  "operationId": "<string | null>",
  "dryRun": "<boolean>"
}
```

**Error responses**

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid Bearer token |
| 403 | Insufficient permissions |
| 404 | Pool or tenant not found |
| 409 | An operation is already in progress / running workflows exist |
| 400 | Other service-layer error |
| 500 | Unexpected server error |

---

### 3. GET /api/external/defender-pools/[tenant]/[poolId]/config

**Source:** `app/api/external/defender-pools/[tenant]/[poolId]/config/route.ts`

**Purpose:** Returns the current structured configuration for the pool.

**Auth:** `defender-pools:read` (env-scoped or unscoped).

**Request body:** None.

**Success response (200)**

```json
{
  "success": true,
  "config": { /* pool config object — shape determined by DefenderPoolService.getConfig() */ }
}
```

**Error responses**

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid Bearer token |
| 403 | Insufficient permissions |
| 404 | Pool or tenant not found |
| 400 | Other service-layer error |
| 500 | Unexpected server error |

---

### 4. POST /api/external/defender-pools/[tenant]/[poolId]/config

**Source:** `app/api/external/defender-pools/[tenant]/[poolId]/config/route.ts`

**Purpose:** Updates the configuration of the pool. Supports two modes: structured field update or raw YAML override.

**Auth:** `defender-pools:manage` (env-scoped or unscoped).

**Request body** (validated by `updateConfigRequestSchema`):

At least one field from either mode must be present. Standard mode and raw YAML mode can coexist in the same request.

```json
{
  "amiName": "string (optional)",
  "minReplicas": "integer >= 0 (optional)",
  "maxReplicas": "integer >= 1 (optional)",
  "volumeSize": "integer >= 1 (optional)",
  "instanceRefreshEnabled": "boolean (optional)",
  "rawYaml": "string, non-empty (optional)",
  "appsRawYaml": "string, non-empty (optional)"
}
```

Validation rules:
- At least one of the above fields must be present.
- If both `minReplicas` and `maxReplicas` are provided, `minReplicas <= maxReplicas` is enforced.

**Success response (200)**

```json
{
  "success": true,
  "message": "<string>",
  "commitSha": "<string | null>",
  "dryRun": "<boolean>"
}
```

**Error responses**

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid Bearer token |
| 403 | Insufficient permissions |
| 404 | Pool or tenant not found |
| 409 | An operation is already in progress / running workflows exist |
| 400 | Validation failure or other service-layer error |
| 500 | Unexpected server error |

---

### 5. GET /api/external/defender-pools/[tenant]/[poolId]/raw-config

**Source:** `app/api/external/defender-pools/[tenant]/[poolId]/raw-config/route.ts`

**Purpose:** Returns the raw YAML configuration file for the pool (as opposed to the parsed/structured config returned by `/config`).

**Auth:** `defender-pools:read` (env-scoped or unscoped).

**Request body:** None.

**Success response (200)**

```json
{
  "success": true,
  "yaml": "<string — raw YAML content>",
  "fileName": "<string — file name in the repo>"
}
```

**Error responses**

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid Bearer token |
| 403 | Insufficient permissions |
| 404 | Pool or tenant not found |
| 400 | Other service-layer error |
| 500 | Unexpected server error |

---

### 6. POST /api/external/defender-pools/[tenant]/[poolId]/verify-config

**Source:** `app/api/external/defender-pools/[tenant]/[poolId]/verify-config/route.ts`

**Purpose:** Validates a candidate YAML configuration for the pool without applying any changes. Useful for dry-run checks before calling `/config`.

**Auth:** `defender-pools:manage` (env-scoped or unscoped).

**Request body** (validated by `verifyConfigRequestSchema`):

```json
{
  "yaml": "string, non-empty (required)"
}
```

**Success response (200)**

This route always returns 200 even when the YAML is invalid — the `valid` field communicates the validation outcome.

```json
{
  "success": true,
  "valid": "boolean",
  "errors": ["string array of validation errors, or null/empty if valid"],
  "message": "<string>"
}
```

**Error responses**

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid Bearer token |
| 403 | Insufficient permissions |
| 400 | Malformed JSON body or Zod schema violation |
| 500 | Unexpected server error |

---

### 7. GET /api/external/defender-pools/[tenant]/[poolId]/operation-status

**Source:** `app/api/external/defender-pools/[tenant]/[poolId]/operation-status/route.ts`

**Purpose:** Returns the current operation status for the pool (e.g. whether a config update or delete is in flight).

**Auth:** `defender-pools:read` (env-scoped or unscoped).

**Request body:** None.

**Success response (200)**

```json
{
  "success": true,
  "operation": { /* operation status object — shape determined by DefenderPoolService.getOperationStatus() */ }
}
```

**Error responses**

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid Bearer token |
| 403 | Insufficient permissions |
| 404 | Pool or tenant not found |
| 400 | Other service-layer error |
| 500 | Unexpected server error |

---

### 8. GET /api/external/defender-pools/[tenant]/[poolId]/instance-refresh

**Source:** `app/api/external/defender-pools/[tenant]/[poolId]/instance-refresh/route.ts`

**Purpose:** Returns the status of the most recent (or in-progress) instance refresh for the pool.

**Auth:** `defender-pools:read` (env-scoped or unscoped).

**Request body:** None.

**Success response (200)**

```json
{
  "success": true,
  "status": "<string — e.g. Pending, InProgress, Successful, Failed, Cancelled>",
  "percentComplete": "<number | null>",
  "startTime": "<ISO datetime string | null>",
  "endTime": "<ISO datetime string | null>",
  "refreshId": "<string | null>"
}
```

**Error responses**

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid Bearer token |
| 403 | Insufficient permissions |
| 404 | Pool or tenant not found |
| 400 | Other service-layer error |
| 500 | Unexpected server error |

---

### 9. POST /api/external/defender-pools/[tenant]/[poolId]/instance-refresh

**Source:** `app/api/external/defender-pools/[tenant]/[poolId]/instance-refresh/route.ts`

**Purpose:** Starts a new instance refresh on the pool's Auto Scaling Group.

**Auth:** `defender-pools:manage` (env-scoped or unscoped).

**Request body** (optional, validated by `startInstanceRefreshRequestSchema`):

The body is optional — if the request body is empty, defaults are used.

```json
{
  "minHealthyPercentage": "integer 0–100 (optional)",
  "instanceWarmup": "integer >= 0 (optional, seconds)"
}
```

**Success response (200)**

```json
{
  "success": true,
  "message": "<string>",
  "refreshId": "<string>"
}
```

**Error responses**

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid Bearer token |
| 403 | Insufficient permissions |
| 404 | Pool or tenant not found |
| 409 | Refresh already in progress |
| 400 | Validation failure or other service-layer error |
| 500 | Unexpected server error |

---

### 10. DELETE /api/external/defender-pools/[tenant]/[poolId]/instance-refresh

**Source:** `app/api/external/defender-pools/[tenant]/[poolId]/instance-refresh/route.ts`

**Purpose:** Cancels an in-progress instance refresh.

**Auth:** `defender-pools:manage` (env-scoped or unscoped).

**Request body:** None.

**Success response (200)**

```json
{
  "success": true,
  "message": "<string>"
}
```

**Error responses**

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid Bearer token |
| 403 | Insufficient permissions |
| 404 | Pool or tenant not found |
| 400 | Other service-layer error (e.g. no refresh in progress) |
| 500 | Unexpected server error |

---

## Required environment variables

| Variable | Purpose |
|----------|---------|
| `DESCOPE_PROJECT_ID` | Descope project used to validate incoming Bearer tokens. Required by all routes. |
| `DESCOPE_TENANT_ID` | Descope tenant scope for extracting per-tenant permissions from the JWT. Required by all routes. |

---

## Notes

- **`sync` and `ami-versions` routes listed in the original spec do not exist** in the codebase as of 2026-06-22. The actual sub-routes under `[poolId]` are: `config`, `raw-config`, `verify-config`, `operation-status`, `instance-refresh`, and the pool root itself (DELETE only).
- The `CallerIdentity` passed to mutating service calls carries `{ displayName: <service identifier from token>, source: 'api' }` — this is used for audit logging downstream.
- `revalidate = 0` is set on all routes, disabling Next.js caching (each request is always dynamic).
