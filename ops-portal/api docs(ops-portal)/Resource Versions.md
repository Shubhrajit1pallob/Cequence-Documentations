---
title: Resource Versions API
area: ops-portal/external-api
last-updated: 2026-06-22
source-files:
  - app/api/external/resource-versions/route.ts
  - app/api/external/resource-versions/[id]/route.ts
  - app/api/external/openapi.json/route.ts
---

# Resource Versions API

External API surface for managing resource versions in the Ops Portal. "Resource version" is the versioned entry for a deployable artifact (e.g. `cequence-asp 8.3.0`, `eck-operator 2.14.0`). These records drive tenant provisioning and UAP upgrade decisions.

All routes under `/api/external/resource-versions` require a Descope-issued Bearer token. The `/api/external/openapi.json` route is public.

---

## Inbound Routes

### 1. `GET /api/external/resource-versions`

**Purpose:** Lists all resource versions, with optional filtering by resource type and pagination.

**Auth:** `Authorization: Bearer <token>` — Descope JWT. Required permission: `resources:read` (`ServicePermission.ResourcesRead`).

**Query parameters:**

| Parameter  | Type   | Required | Default | Constraints                                                                                           |
|------------|--------|----------|---------|-------------------------------------------------------------------------------------------------------|
| `resource` | string | No       | —       | Must be one of: `cequence-asp`, `eck-operator`, `strimzi-operator`, `keycloak-theme`, `argo-parent-app` |
| `limit`    | number | No       | `100`   | Integer 1–1000                                                                                        |
| `offset`   | number | No       | `0`     | Integer >= 0                                                                                          |

**Response — 200 OK:**

```json
{
  "success": true,
  "versions": [
    {
      "id": "cequence-asp-8.3.0",
      "resource": "cequence-asp",
      "version": "8.3.0",
      "defaultVersion": true,
      "stable": true,
      "archived": false,
      "internalNotes": "GA release",
      "minimumKeycloakThemeTag": "8.3.0",
      "development": false,
      "eckOperatorVersion": "2.14.0",
      "strimziOperatorVersion": "0.44.0"
    }
  ],
  "total": 42,
  "limit": 100,
  "offset": 0
}
```

The `total` field reflects the count **before** pagination is applied. `eckOperatorVersion` and `strimziOperatorVersion` are only populated for `cequence-asp` entries.

**Error responses:**

| Status | When |
|--------|------|
| 400    | Invalid query parameters (Zod validation failure) |
| 401    | Missing or invalid Bearer token |
| 403    | Token valid but lacks `resources:read` permission |
| 500    | Internal error fetching versions |

---

### 2. `POST /api/external/resource-versions`

**Purpose:** Creates a new resource version record. For `cequence-asp`, also auto-creates the referenced `eck-operator` and `strimzi-operator` version records if they do not already exist, and triggers background cleanup of stale nightly/pre-release versions.

**Auth:** `Authorization: Bearer <token>` — Descope JWT. Required permission: `resources:manage` (`ServicePermission.ResourcesManage`).

**Request body (`application/json`):**

```json
{
  "resource": "cequence-asp",
  "version": "8.3.0",
  "defaultVersion": false,
  "stable": false,
  "archived": false,
  "development": false,
  "internalNotes": "Optional notes, max 5000 chars",
  "minimumKeycloakThemeTag": "8.3.0",
  "eckOperatorVersion": "2.14.0",
  "strimziOperatorVersion": "0.44.0"
}
```

| Field                   | Type    | Required | Default | Constraints |
|-------------------------|---------|----------|---------|-------------|
| `resource`              | string  | Yes      | —       | Enum: `cequence-asp`, `eck-operator`, `strimzi-operator`, `keycloak-theme`, `argo-parent-app` |
| `version`               | string  | Yes      | —       | Semver: `\d+\.\d+\.\d+(-prerelease)?(+build)?` |
| `defaultVersion`        | boolean | No       | `false` | Cannot be `true` when `development` is `true` |
| `stable`                | boolean | No       | `false` | — |
| `archived`              | boolean | No       | `false` | — |
| `development`           | boolean | No       | `false` | — |
| `internalNotes`         | string  | No       | `""`    | Max 5000 characters |
| `minimumKeycloakThemeTag` | string | No      | *(defaults to `version` value for `cequence-asp`)* | Must be valid semver |
| `eckOperatorVersion`    | string  | No       | *(default `eck-operator` version)* | Must be valid semver. Only meaningful for `cequence-asp` |
| `strimziOperatorVersion` | string | No       | *(default `strimzi-operator` version)* | Must be valid semver. Only meaningful for `cequence-asp` |

**`cequence-asp`-specific side effects:**
- If `eckOperatorVersion` or `strimziOperatorVersion` are omitted, the system looks up the current default version for each operator. Returns `400` if no default is configured.
- If the resolved `eck-operator-{version}` or `strimzi-operator-{version}` record does not exist, it is auto-created with `defaultVersion: false`, `stable: false`, `archived: false`.
- `minimumKeycloakThemeTag` defaults to the submitted `version` string if not provided.
- On any release version (semver with optional pre-release tag): fires-and-forgets a background job to archive old nightly builds for the same major/minor.
- On a clean stable release (`X.Y.Z` with no pre-release tag): additionally fires-and-forgets a background job to archive old beta/RC versions older than 3 months.

**Response — 201 Created:**

```json
{
  "success": true,
  "message": "Resource version created successfully",
  "version": {
    "id": "cequence-asp-8.3.0",
    "resource": "cequence-asp",
    "version": "8.3.0",
    "defaultVersion": false,
    "stable": false,
    "archived": false,
    "internalNotes": "",
    "minimumKeycloakThemeTag": "8.3.0",
    "development": false,
    "eckOperatorVersion": "2.14.0",
    "strimziOperatorVersion": "0.44.0"
  }
}
```

**Audit log:** Every successful create writes an `AuditActions.CREATE` / `AuditTargets.VERSION` entry (status `YES`). Failures write a `NO` entry.

**Error responses:**

| Status | When |
|--------|------|
| 400    | Zod validation failure, or no default operator version configured (for `cequence-asp`) |
| 401    | Missing or invalid Bearer token |
| 403    | Token valid but lacks `resources:manage` permission |
| 409    | Version record `{resource}-{version}` already exists |
| 500    | Internal error saving version |

---

### 3. `PUT /api/external/resource-versions/{id}`

**Purpose:** Partially updates an existing resource version identified by its ID (`{resource}-{version}`, e.g. `cequence-asp-8.3.0`). Only fields present in the request body are changed; all others retain their current values.

**Auth:** `Authorization: Bearer <token>` — Descope JWT. Required permission: `resources:manage` (`ServicePermission.ResourcesManage`).

**Path parameter:**

| Parameter | Type   | Required | Constraints |
|-----------|--------|----------|-------------|
| `id`      | string | Yes      | Format `{resource}-{version}`, e.g. `cequence-asp-8.3.0`. Must match pattern `^[a-z0-9-]+-\d+\.\d+\.\d+.*$` |

**Request body (`application/json`) — all fields optional:**

```json
{
  "defaultVersion": true,
  "stable": true,
  "archived": false,
  "development": false,
  "internalNotes": "Promoted to stable",
  "minimumKeycloakThemeTag": "8.3.0",
  "eckOperatorVersion": "2.14.0",
  "strimziOperatorVersion": "0.44.0"
}
```

| Field                   | Type    | Required | Constraints |
|-------------------------|---------|----------|-------------|
| `defaultVersion`        | boolean | No       | Cannot be `true` if `development` is also `true` in the same request |
| `stable`                | boolean | No       | — |
| `archived`              | boolean | No       | — |
| `development`           | boolean | No       | — |
| `internalNotes`         | string  | No       | Max 5000 characters |
| `minimumKeycloakThemeTag` | string | No      | Must be valid semver |
| `eckOperatorVersion`    | string  | No       | Must be valid semver |
| `strimziOperatorVersion` | string | No       | Must be valid semver |

Note: `resource` and `version` are not updatable — they are read from the existing record and derived from the path `id`.

**`cequence-asp`-specific side effects (same pattern as POST):**
- If the record being updated is `cequence-asp`, any provided (or currently-set) `eckOperatorVersion` / `strimziOperatorVersion` is resolved via the same default-lookup logic.
- Auto-creates operator version records if they do not already exist.

**Response — 200 OK:**

```json
{
  "success": true,
  "message": "Resource version updated successfully",
  "version": {
    "id": "cequence-asp-8.3.0",
    "resource": "cequence-asp",
    "version": "8.3.0",
    "defaultVersion": true,
    "stable": true,
    "archived": false,
    "internalNotes": "Promoted to stable",
    "minimumKeycloakThemeTag": "8.3.0",
    "development": false,
    "eckOperatorVersion": "2.14.0",
    "strimziOperatorVersion": "0.44.0"
  }
}
```

**Audit log:** Every successful update writes an `AuditActions.UPDATE` / `AuditTargets.VERSION` entry with both `initial` and `payload` snapshots. Failures write a `NO` entry.

**Error responses:**

| Status | When |
|--------|------|
| 400    | Zod validation failure, invalid `id` format, or no default operator version configured |
| 401    | Missing or invalid Bearer token |
| 403    | Token valid but lacks `resources:manage` permission |
| 404    | No record found for the given `id` |
| 500    | Internal error saving version |

---

### 4. `GET /api/external/openapi.json`

**Purpose:** Serves the live OpenAPI specification for the entire external API surface as JSON. The spec is generated from Zod schemas at first request and cached in process memory for the lifetime of the server (i.e. until the next deploy). Response headers also set `Cache-Control: public, max-age=3600`.

**Auth:** None — this endpoint is public. No Bearer token required.

**Request params:** None.

**Response — 200 OK:**

```http
Content-Type: application/json
Cache-Control: public, max-age=3600
```

Body is the full OpenAPI 3.x document generated by `generateOpenAPIDocument()` from `@/_lib/openapi`.

**Error responses:** None defined (failures would be unhandled 500s from the document generator).

---

## Shared Notes

### Resource type enum

All four routes use the same `ResourceTypeEnum`:

| Value             | Human name        | Notes                                      |
|-------------------|-------------------|--------------------------------------------|
| `cequence-asp`    | Cequence ASP      | Main UAP product version (`uapResourceName`) |
| `eck-operator`    | ECK Operator      | Elastic Cloud on Kubernetes operator       |
| `strimzi-operator`| Strimzi Operator  | Kafka operator                             |
| `keycloak-theme`  | —                 | —                                          |
| `argo-parent-app` | —                 | —                                          |

### Version ID format

The synthetic primary key for a resource version is `{resource}-{version}` (e.g. `cequence-asp-8.3.0`, `eck-operator-2.14.0`). This is what the `[id]` path segment and the duplicate-check logic use.

### Auth mechanism

All authenticated routes use `withDescopeAuth()` from `@/_lib/auth/externalApiAuth`. It:
1. Requires `Authorization: Bearer <token>` header.
2. Validates the JWT against Descope.
3. Checks that the token's permissions include the required scope (`resources:read` or `resources:manage`).
4. Attaches `serviceIdentifier` (extracted from the token payload) to the request for logging and auditing.

### Required env vars

None are explicitly read in these route files. Auth validation depends on Descope configuration (project ID / public key) which is consumed by `validateDescopeToken` in `@/_lib/auth/descopeAuth.ts` — see that file for its env var requirements.
