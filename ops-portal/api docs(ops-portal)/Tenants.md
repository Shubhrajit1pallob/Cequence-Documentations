---
title: Tenants API — External Routes
source_base: ops-portal/app/api/external/tenants
last_reviewed: 2026-06-22
---

# Tenants API — Inbound Routes

All routes live under `/api/external/tenants/`. They are part of the Ops Portal's **external** API surface, intended for machine-to-machine callers (CI pipelines, automation services, etc.).

---

## Auth overview

Two distinct auth wrappers are used across these routes.

**`authenticateRequest` (read routes)**
Used by `GET /tenants` and `GET /tenants/[tenant]` and `GET /tenants/[tenant]/status`. Validates a Descope service token from the `Authorization: Bearer <token>` header. Returns the decoded Descope JWT payload. Permissions are checked inline after this call.

**`withDescopeAuth(handler, ServicePermission.TenantsManage)` (write routes)**
A higher-order wrapper used by all `POST`, `PUT`, and `DELETE` routes. Validates the Descope service token and injects `request.serviceIdentifier` and `request.descopePayload` into the handler. All write routes require `ServicePermission.TenantsManage` at the wrapper level as a minimum gate; some routes also perform a secondary environment-aware permission check with `hasTenantPermission`.

**Tenant ID format** (enforced on every route that takes `[tenant]` as a path param):
```
{name}-{environment}
```
- `name`: lowercase alphanumeric with hyphens, no consecutive hyphens, 1–20 chars (e.g. `acme`, `my-company`).
- `environment`: `dev` or `prod`.
- Full examples: `acme-prod`, `my-company-dev`.

**Common error response shape** (all routes):
```json
{
  "status": "<HTTP status text>",
  "message": "<human-readable message>",
  "code": "<ErrorCode enum value>"
}
```

---

## Routes

### 1. List Tenants

**`GET /api/external/tenants`**

Returns a paginated list of tenants. Results are filtered to environments the caller has read permission for.

**Auth:** `authenticateRequest`. Required permission: any of:
- `tenants:dev:read` — returns dev tenants only
- `tenants:prod:read` — returns prod tenants only
- `tenants:read` — returns all environments
- `tenants:manage` — returns all environments

If the caller has no read permission for any environment, returns `403`.
If the `environment` query param is set to an environment the caller cannot read, returns `403`.

**Query parameters:**

| Parameter | Type | Default | Constraints | Description |
|-----------|------|---------|-------------|-------------|
| `limit` | integer | `100` | 1–1000 | Max number of results |
| `offset` | integer | `0` | ≥ 0 | Pagination offset |
| `environment` | string | — | `dev` or `prod` | Filter to a single environment |
| `customer` | string | — | min length 1 | Filter by customer name |

**Success response (200):**
```json
{
  "success": true,
  "tenants": [ /* array of tenant objects */ ],
  "total": 42,
  "limit": 100,
  "offset": 0
}
```

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Invalid query parameter (Zod validation failure) |
| 403 | No read permission for any environment, or `environment` param requests an environment the caller cannot read |
| 500 | Unexpected internal error |

---

### 2. Create Tenant

**`POST /api/external/tenants`**

Creates a new tenant record, optionally kicking off the provisioning workflow.

**Auth:** `authenticateRequest`. Required permission:
- `tenants:{environment}:create` (where `{environment}` matches the request body `environment` field), OR
- `tenants:manage`

**Request body:**

```json
{
  "customerName": "Acme Corp",
  "name": "acme",
  "environment": "prod",
  "region": "us-west-2",
  "accountStage": "customer",
  "branding": "cequence",
  "defenderType": "ASG",
  "uapVersion": "2.10.0",
  "eckOperatorVersion": "2.1.0",
  "strimziOperatorVersion": "0.35.0",
  "clusterVersion": "1.33",
  "karpenterVersion": "0.36.0",
  "geo": "us",
  "upgradeGroup": 1,
  "arrTier": 2,
  "internalNotes": "Optional free text, max 5000 chars",
  "additionalNotificationSlackChannel": "#acme-alerts",
  "ebsEncryptionKmsKeyArn": "arn:aws:kms:us-west-2:123456789012:key/...",
  "spyderEnabled": false,
  "wafEnabled": false,
  "customerFirstName": "Jane",
  "customerLastName": "Doe",
  "customerEmail": "jane.doe@acme.com",
  "startProvisioning": false
}
```

**Required fields:** `customerName`, `name`, `environment`.

**Field constraints:**
- `name`: 1–20 chars, lowercase alphanumeric + hyphens, no consecutive hyphens, must start/end with alphanumeric.
- `environment`: `"dev"` or `"prod"`.
- `region`: valid AWS region string (default `"us-west-2"`).
- `accountStage`: `"internal"`, `"pov"`, or `"customer"` (optional).
- `branding`: `"cequence"` or `"ultraapi"` (default `"cequence"`).
- `defenderType`: `"ASG"`, `"EKS"`, or `"Hybrid"` (default `"ASG"`).
- `uapVersion`, `eckOperatorVersion`, `strimziOperatorVersion`, `karpenterVersion`: valid semver (e.g. `2.10.0`, `1.0.0-beta.1`).
- `clusterVersion`: `major.minor` or `major.minor.patch` EKS format (e.g. `"1.33"`, `"1.33.0"`).
- `wafEnabled`: can only be `true` when `defenderType` is `"ASG"`.
- `spyderEnabled = true` requires `customerFirstName`, `customerLastName`, and `customerEmail` to also be set.
- `ebsEncryptionKmsKeyArn`: KMS key ARN whose region must match `region`.
- `startProvisioning`: when `true`, the provisioning workflow is started immediately (default `false`).

**Success response (201):**
```json
{
  "success": true,
  "message": "Tenant created successfully",
  "tenant": { /* tenant object */ },
  "provisioning": { /* provisioning status, present if startProvisioning=true */ }
}
```

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Validation failure (Zod), or `CUSTOMER_NOT_FOUND` |
| 403 | Insufficient permissions for the target environment |
| 409 | `TENANT_EXISTS` — tenant with that name/environment already exists |
| 500 | Unexpected internal error |

---

### 3. Get Tenant Details

**`GET /api/external/tenants/[tenant]`**

Returns full details for a single tenant. Sensitive internal fields are excluded.

**Path param:** `tenant` — tenant ID in `{name}-{environment}` format.

**Auth:** `authenticateRequest`. Required permission: any of `tenants:{environment}:read`, `tenants:read`, `tenants:manage` (environment derived from the tenant ID).

**Success response (200):**
```json
{
  "success": true,
  "tenant": { /* full tenant details object, sensitive fields excluded */ }
}
```

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID |
| 403 | No read permission for the tenant's environment |
| 404 | Tenant not found |
| 500 | Unexpected internal error |

---

### 4. Get Tenant Status

**`GET /api/external/tenants/[tenant]/status`**

Returns the operational status of a tenant including its provisioning workflow state.

**Path param:** `tenant` — tenant ID in `{name}-{environment}` format.

**Auth:** `authenticateRequest`. Required permission: any of `tenants:{environment}:read`, `tenants:read`, `tenants:manage`.

**Success response (200):** (response is the raw status object, not wrapped in `{ success: true }`)
```json
{
  "tenant": "acme-prod",
  "provisionStatus": {
    "status": "SUCCESS",
    "jobs": {
      "someJobName": {
        "state": "SUCCESS",
        "jobId": 12345,
        "message": "optional message",
        "link": "optional link"
      }
    }
  },
  "paused": false,
  "deleted": false,
  "archived": false
}
```

Job `state` values: `PENDING`, `RUNNING`, `SUCCESS`, `FAILURE`, `SKIPPED`.
Workflow `status` values: `PENDING`, `RUNNING`, `SUCCESS`, `FAILURE`.

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID |
| 403 | No read permission for the tenant's environment |
| 404 | Tenant not found |
| 500 | Unexpected internal error |

---

### 5. Start Synthetic Lab Traffic

**`POST /api/external/tenants/[tenant]/traffic/start`**

Starts the synthetic lab traffic workflow for a tenant.

**Path param:** `tenant` — tenant ID in `{name}-{environment}` format.

**Auth:** `withDescopeAuth` requiring `ServicePermission.TenantsManage`. Secondary check: `hasTenantPermission(principal, env, 'traffic')`.

**Pre-conditions (all must be true):**
- Tenant exists.
- Tenant is not deleted, archived, or paused.
- Tenant is fully provisioned (provision workflow in `SUCCESS` state).
- Synthetic traffic is not already started (`syntheticTrafficStarted === false`).
- Tenant has no currently running workflows.

**Request body (optional):**
```json
{
  "defenderMode": "reuse"
}
```
- `defenderMode`: `"reuse"` or `"dedicated"` (optional). If omitted, defaults to `"reuse"` when the tenant has EKS defender enabled, otherwise `"dedicated"`.

**Success response (200):**
```json
{
  "success": true,
  "message": "Start synthetic traffic workflow initiated",
  "tenant": "acme-prod",
  "workflow": "<startTenantSyntheticTrafficWorkflowName>"
}
```

**Side effects:** Sets `defenderMode` on the tenant record, starts `START_TENANT_SYNTHETIC_TRAFFIC_WORKFLOW`, clears the status of `STOP_TENANT_SYNTHETIC_TRAFFIC_WORKFLOW`. Writes an audit log entry.

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID, validation error, tenant is deleted/archived/paused, or tenant not fully provisioned, or synthetic traffic already started |
| 403 | Insufficient permissions |
| 404 | Tenant not found |
| 409 | Tenant has active workflows |
| 500 | Unexpected internal error |

---

### 6. Stop Synthetic Lab Traffic

**`POST /api/external/tenants/[tenant]/traffic/stop`**

Stops the synthetic lab traffic workflow for a tenant.

**Path param:** `tenant` — tenant ID in `{name}-{environment}` format.

**Auth:** `withDescopeAuth` requiring `ServicePermission.TenantsManage`. Secondary check: `hasTenantPermission(principal, env, 'traffic')`.

**Pre-conditions (all must be true):**
- Tenant exists.
- Synthetic traffic is currently started (`syntheticTrafficStarted === true`).
- Tenant has no currently running workflows.

**Request body:** None.

**Success response (200):**
```json
{
  "success": true,
  "message": "Stop synthetic traffic workflow initiated",
  "tenant": "acme-prod",
  "workflow": "<stopTenantSyntheticTrafficWorkflowName>"
}
```

**Side effects:** Starts `STOP_TENANT_SYNTHETIC_TRAFFIC_WORKFLOW`, clears the status of `START_TENANT_SYNTHETIC_TRAFFIC_WORKFLOW`. Writes an audit log entry.

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID, or synthetic traffic is not currently started |
| 403 | Insufficient permissions |
| 404 | Tenant not found |
| 409 | Tenant has active workflows |
| 500 | Unexpected internal error |

---

### 7. Initiate UAP Minor Upgrade

**`POST /api/external/tenants/[tenant]/upgrade`**

Starts a UAP minor upgrade workflow for a tenant.

**Path param:** `tenant` — tenant ID in `{name}-{environment}` format.

**Auth:** `withDescopeAuth` requiring `ServicePermission.TenantsManage`.

**Pre-conditions (all must be true):**
- Tenant exists.
- Tenant is not deleted, archived, or paused.
- Tenant is fully provisioned.
- Tenant has no currently running workflows.
- No active system freeze window applies to the tenant.
- `targetVersion` exists in the resource versions table (validated as `uap-{targetVersion}`).

**Request body:**
```json
{
  "targetVersion": "2.11.0",
  "gitlabCallbackUrl": "https://gitlab.example.com/callback/123"
}
```
- `targetVersion` (required): valid semver string.
- `gitlabCallbackUrl` (optional): valid URL. Stored on the tenant; used by GitLab CI to receive a callback when the upgrade completes. Stored as empty string if omitted.

**Success response (200):**
```json
{
  "success": true,
  "message": "UAP upgrade workflow started",
  "tenant": "acme-prod",
  "targetVersion": "2.11.0",
  "currentVersion": "2.10.0",
  "workflow": "TenantUapMinorUpgrade"
}
```

**Side effects:** Saves `targetVersion` and `gitlabCallbackUrl` to the tenant record, starts `UAP_MINOR_UPGRADE_WORKFLOW`. Writes an audit log entry on both success and failure.

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID, validation failure, target UAP version does not exist, or tenant is deleted/archived/paused/not provisioned |
| 404 | Tenant not found |
| 409 | Tenant has active workflows, or a system freeze window is active |
| 500 | Unexpected internal error |

---

### 8. Retry a Workflow Job

**`POST /api/external/tenants/[tenant]/workflows/[workflow]/jobs/[job]/retry`**

Retries a single job within a workflow by resetting its state to `PENDING`.

**Path params:**
- `tenant` — tenant ID in `{name}-{environment}` format.
- `workflow` — workflow name (must exist in `workflowPermissionMap`).
- `job` — job name within the workflow.

**Auth:** `withDescopeAuth` requiring `ServicePermission.TenantsManage`. Secondary check: `hasTenantPermission(principal, env, workflowPermissionMap[workflow])`.

**Pre-conditions:**
- Tenant exists.
- Workflow name is registered in `workflowPermissionMap`.
- Job exists and its current state is `FAILURE`, `RUNNING`, or `PENDING`.

**Request body:** None.

**Success response (200):**
```json
{
  "success": true,
  "message": "Job someJobName retry initiated",
  "tenant": "acme-prod",
  "workflow": "TenantProvisioning",
  "job": "someJobName",
  "previousState": "FAILURE",
  "newState": "PENDING"
}
```

**Side effects:** Sets the job's state to `PENDING` with `jobId: -1`. Sets workflow status to `RUNNING`. Writes an audit log entry.

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID, invalid workflow, or job not in a retryable state |
| 403 | Insufficient permissions |
| 404 | Tenant not found |
| 500 | Unexpected internal error or DB update failure |

---

### 9. Skip a Workflow Job

**`POST /api/external/tenants/[tenant]/workflows/[workflow]/jobs/[job]/skip`**

Skips a single job within a workflow, marking it as `SKIPPED`.

**Path params:**
- `tenant` — tenant ID in `{name}-{environment}` format.
- `workflow` — workflow name (must exist in `workflowPermissionMap`).
- `job` — job name within the workflow.

**Auth:** `withDescopeAuth` requiring `ServicePermission.TenantsManage`. Secondary check: `hasTenantPermission(principal, env, workflowPermissionMap[workflow])`.

**Pre-conditions:**
- Tenant exists.
- Workflow name is registered in `workflowPermissionMap`.
- Job exists and its current state is `FAILURE`, `RUNNING`, or `PENDING`.

**Request body (optional):**
```json
{
  "reason": "Manually skipped because X"
}
```
- `reason` (optional): free-text string. Stored as job message and included in the audit log. If omitted, the message is set to `"Skipped by user"`.

**Success response (200):**
```json
{
  "success": true,
  "message": "Job someJobName skipped",
  "tenant": "acme-prod",
  "workflow": "TenantProvisioning",
  "job": "someJobName",
  "previousState": "FAILURE",
  "newState": "SKIPPED",
  "reason": "Manually skipped because X"
}
```

**Side effects:** Sets the job's state to `SKIPPED` with `jobId: -1`. Writes an audit log entry (includes `reason` in payload).

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID, invalid workflow, or job not in a skippable state |
| 403 | Insufficient permissions |
| 404 | Tenant not found |
| 500 | Unexpected internal error or DB update failure |

---

### 10. Stop a Running Workflow Job

**`POST /api/external/tenants/[tenant]/workflows/[workflow]/jobs/[job]/stop`**

Force-stops a running job by marking it as `FAILURE`.

**Path params:**
- `tenant` — tenant ID in `{name}-{environment}` format.
- `workflow` — workflow name (must exist in `workflowPermissionMap`).
- `job` — job name within the workflow.

**Auth:** `withDescopeAuth` requiring `ServicePermission.TenantsManage`. Secondary check: `hasTenantPermission(principal, env, workflowPermissionMap[workflow])`.

**Pre-conditions:**
- Tenant exists.
- Workflow name is registered in `workflowPermissionMap`.
- Job exists and its current state is `RUNNING` (stricter than retry/skip — only `RUNNING` is accepted).

**Request body (optional):**
```json
{
  "reason": "Stopping runaway job"
}
```
- `reason` (optional): stored as the job message. Defaults to `"Stopped by user"` if omitted.

**Success response (200):**
```json
{
  "success": true,
  "message": "Job someJobName stopped",
  "tenant": "acme-prod",
  "workflow": "TenantProvisioning",
  "job": "someJobName",
  "previousState": "RUNNING",
  "newState": "FAILURE",
  "reason": "Stopping runaway job"
}
```

**Side effects:** Sets the job's state to `FAILURE` with `jobId: -1`. Writes an audit log entry (includes `reason` in payload).

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID, invalid workflow, or job not in `RUNNING` state |
| 403 | Insufficient permissions |
| 404 | Tenant not found |
| 500 | Unexpected internal error or DB update failure |

---

### 11. Resume a Failed Workflow

**`POST /api/external/tenants/[tenant]/workflows/[workflow]/resume`**

Resumes a workflow that is in `FAILURE` state. Finds the first job in `FAILURE` state and retries it (sets it to `PENDING`). This is a convenience wrapper over the individual job retry — it picks the failed job automatically.

**Path params:**
- `tenant` — tenant ID in `{name}-{environment}` format.
- `workflow` — workflow name (must exist in `workflowPermissionMap`).

**Auth:** `withDescopeAuth` requiring `ServicePermission.TenantsManage`. Secondary check: `hasTenantPermission(principal, env, workflowPermissionMap[workflow])`.

**Pre-conditions:**
- Tenant exists.
- Workflow name is registered in `workflowPermissionMap`.
- Workflow status exists on the tenant record.
- Workflow overall status is `FAILURE`.
- At least one job within the workflow has state `FAILURE`.

**Request body:** None.

**Success response (200):**
```json
{
  "success": true,
  "message": "Workflow TenantProvisioning resumed, retrying job someJobName",
  "tenant": "acme-prod",
  "workflow": "TenantProvisioning",
  "resumedJob": "someJobName"
}
```

**Side effects:** Sets the workflow overall status to `RUNNING`. Sets the first failed job's state to `PENDING` with `jobId: -1`. Writes an audit log entry.

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID, or invalid workflow name |
| 403 | Insufficient permissions |
| 404 | Tenant not found, or workflow status not found on tenant |
| 409 | Workflow is not in `FAILURE` state, or no failed job was found despite `FAILURE` workflow state |
| 500 | Unexpected internal error or DB update failure |

---

### 12. Set Tenant EOL Date

**`PUT /api/external/tenants/[tenant]/eol`**

Sets or updates the expected end-of-life (EOL) date for a tenant. Accepts either an explicit ISO 8601 datetime or a duration in days from now.

**Path param:** `tenant` — tenant ID in `{name}-{environment}` format.

**Auth:** `withDescopeAuth` requiring `ServicePermission.TenantsManage`.

**Pre-conditions:**
- Tenant exists.
- Tenant is not deleted or archived.

**Request body:** Exactly one of `expectedEolDate` or `durationDays` must be provided (not both).

```json
{ "expectedEolDate": "2027-01-01T00:00:00.000Z" }
```
or
```json
{ "durationDays": 365 }
```

- `expectedEolDate`: ISO 8601 datetime string (validated by Zod `z.string().datetime()`).
- `durationDays`: positive integer, max 3650 (10 years). EOL is computed as `now + durationDays * 24h`.

**Success response (200):**
```json
{
  "success": true,
  "tenant": "acme-prod",
  "expectedEolDate": "2027-01-01T00:00:00.000Z"
}
```

**Side effects:** Sets `expectedEolDate` on the tenant document. Writes an audit log entry.

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID, validation failure (neither or both fields provided, `durationDays` > 3650), or tenant is deleted/archived |
| 404 | Tenant not found |
| 500 | Unexpected internal error |

---

### 13. Clear Tenant EOL Date

**`DELETE /api/external/tenants/[tenant]/eol`**

Removes the expected EOL date from a tenant record.

**Path param:** `tenant` — tenant ID in `{name}-{environment}` format.

**Auth:** `withDescopeAuth` requiring `ServicePermission.TenantsManage`.

**Pre-conditions:**
- Tenant exists.
- Tenant is not deleted or archived.

**Request body:** None.

**Success response (200):**
```json
{
  "success": true,
  "tenant": "acme-prod",
  "expectedEolDate": null
}
```

**Side effects:** Unsets both `expectedEolDate` and `lastEolNotificationDate` on the tenant document. Writes an audit log entry.

**Error responses:**

| Status | Condition |
|--------|-----------|
| 400 | Malformed tenant ID, or tenant is deleted/archived |
| 404 | Tenant not found |
| 500 | Unexpected internal error |

---

## Error code reference

| `code` value | Meaning |
|---|---|
| `VALIDATION_ERROR` | Request body or query param failed schema validation |
| `INSUFFICIENT_PERMISSIONS` | Caller lacks the required permission |
| `TENANT_NOT_FOUND` | No tenant found for the given ID |
| `RESOURCE_NOT_FOUND` | Referenced resource (e.g. UAP version) does not exist |
| `RESOURCE_CONFLICT` | Conflict — e.g. tenant already exists |
| `TENANT_INVALID_STATE` | Tenant or job is in a state that prevents the requested operation |
| `WORKFLOW_CONFLICT` | Tenant has active workflows or an active freeze window blocks the operation |
| `INTERNAL_SERVER_ERROR` | Unhandled server-side error |
