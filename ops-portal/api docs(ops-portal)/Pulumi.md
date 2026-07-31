---
title: Pulumi S3 Bridge API Integration
source: ops-portal/app/_lib/pulumi/pulumiSyncServiceClient.ts
updated: 2026-06-22
---

# Pulumi S3 Bridge — Outbound Client

## Service Overview

`PulumiSyncServiceClient` is an HTTP client for the **Pulumi S3 Bridge** — an internal sidecar/pod running in the same Kubernetes cluster as the Ops Portal. The bridge aggregates Pulumi stack state from two backends (Pulumi Cloud and S3) and stores it in MongoDB. The Ops Portal uses this client to query stack state, trigger syncs, inspect diffs, and browse projects.

A **singleton instance** is exported: `export const pulumiSyncServiceClient = new PulumiSyncServiceClient()`.

## Base URL

Read from `GetEnvironment().pulumiS3BridgeService` at construction time. Defaults to the Kubernetes in-cluster service name (no external hostname needed).

**Env var:** `PULUMI_S3_BRIDGE_SERVICE` (name resolved via `GetEnvironment()`).

## SDK / HTTP Client

The global `fetch` API via a custom `fetchWithTimeout` wrapper (from `@/_lib/fetchWithTimeout`). No third-party HTTP library is used.

## Authentication

None. All calls go over the Kubernetes cluster network to an internal service. No `Authorization` header is set on any request.

## Timeouts

| Constant | Value | Applied to |
|---|---|---|
| `PULUMI_SYNC_TIMEOUT_MS` | 45 000 ms | All sync and data calls |
| `PULUMI_HEALTH_TIMEOUT_MS` | 5 000 ms | Health check only |

## Required Environment Variables

| Variable | Description |
|---|---|
| `PULUMI_S3_BRIDGE_SERVICE` | In-cluster service URL for the Pulumi S3 Bridge |

## Methods and Endpoints

### `getStatus()`

Returns counts of stack states and diffs stored in MongoDB.

```
GET {base}/api/pulumi/sync
```

Response shape:
```ts
{
  mongodb: {
    stackStates: number
    previewDiffs: number
    applyDiffs: number
    destroyDiffs: number
  }
  message: string
}
```

---

### `triggerManualSync()`

Triggers a full resync of all stacks from both Pulumi Cloud and S3 backends into MongoDB.

```
POST {base}/api/pulumi/sync
```

Response includes per-stack success/failure counts and an array of per-stack error details.

---

### `refreshStack(params)`

Refreshes a single stack — fetches fresh data from the specified backend and updates MongoDB.

```
POST {base}/api/pulumi/sync
Body: { backend, projectName, stackName, fullStackName, organization?, environment? }
```

`backendType` must be `'pulumi-cloud'` or `'s3'`.

---

### `healthCheck()`

Checks whether the Pulumi S3 Bridge service is up. Returns `true` if the response is `ok`, `false` otherwise (does not throw).

```
GET {base}/health
```

---

### `getStackDiffs(stackId, params?)`

Returns diffs for a stack identified by its MongoDB stack ID.

```
GET {base}/api/pulumi/stacks/{stackId}/diffs
    [?type=preview|apply|destroy&limit=…&offset=…]
```

Response includes paginated `diffs[]` and a `meta` block with full stack name and project name.

---

### `getDiffById(diffId)`

Returns a single diff record by its MongoDB ID.

```
GET {base}/api/pulumi/diffs/{diffId}
```

---

### `listProjects(params?)`

Lists all projects with their stack counts and per-stack metadata. Optionally filtered by backend type.

```
GET {base}/api/pulumi/projects
    [?backendType=pulumi-cloud|s3]
```

---

### `listProjectStacks(projectName, params?)`

Lists stacks within a specific project, with optional pagination and filtering by environment or backend type.

```
GET {base}/api/pulumi/projects/{projectName}/stacks
    [?environment=…&backendType=…&limit=…&offset=…]
```

---

### `getStackByProjectAndName(projectName, stackName, backendType?)`

Fetches a single stack record by project + stack name.

```
GET {base}/api/pulumi/projects/{projectName}/stacks/{stackName}
    [?backendType=…]
```

---

### `getStackResourcesByProjectAndName(projectName, stackName, backendType?)`

Returns the resource list for a stack (all Pulumi-tracked resources).

```
GET {base}/api/pulumi/projects/{projectName}/stacks/{stackName}/resources
    [?backendType=…]
```

---

### `getStackDiffsByProjectAndName(projectName, stackName, params?)`

Returns diffs for a stack identified by project + stack name rather than a MongoDB ID.

```
GET {base}/api/pulumi/projects/{projectName}/stacks/{stackName}/diffs
    [?type=…&limit=…&offset=…&backendType=…]
```

## Error Handling

All methods throw on non-`ok` responses (HTTP 4xx/5xx). 404 responses are converted to descriptive `Error` objects (`'Stack not found'`, `'Diff not found'`). Errors are logged with `logger.error` before re-throwing. `healthCheck` is the only method that swallows errors and returns `false` instead of throwing.
