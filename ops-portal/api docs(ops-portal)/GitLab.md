---
title: GitLab API Integration
service: GitLab
base_url: https://gitlab.com
created: 2026-06-22
source_files:
  - app/_lib/GitlabApi.ts
  - app/_lib/GitlabAirflowDagAccessTokenManager.ts
  - app/_lib/GitlabReleaseTokenManager.ts
---

# GitLab API Integration

The Ops Portal integrates with GitLab for three distinct purposes, each handled by a dedicated client class:

| Class | Purpose |
|---|---|
| `GitlabApi` | Trigger and inspect CI/CD pipelines and jobs |
| `GitlabAirflowDagAccessTokenManager` | Manage group-scoped access tokens for Airflow DAG reads |
| `GitlabReleaseTokenManager` | Manage group-scoped deploy tokens for container registry reads |

All three clients target `https://gitlab.com`. The two token-manager classes call the REST API directly (`https://gitlab.com/api/v4`); `GitlabApi` uses the [`@gitbeaker/rest`](https://github.com/jdalrymple/gitbeaker) SDK wrapper.

---

## 1. `GitlabApi` — Pipeline & Job Control

**Source:** `app/_lib/GitlabApi.ts`

### Auth

Constructor-injected personal/project access token passed as `token`. The `@gitbeaker/rest` `Gitlab` client sends it as the `PRIVATE-TOKEN` header on every request.

### Required env vars

`GitlabApi` itself does not read `process.env` — the token is injected by the caller. Callers source it from `GetEnvironment().gitlabToken` → `GITLAB_TOKEN`.

| Env var | Description |
|---|---|
| `GITLAB_TOKEN` | Personal or project access token used by callers to construct `GitlabApi` |

### SDK

`@gitbeaker/rest` — wraps the GitLab REST API v4. All calls are additionally wrapped in `instrumentGitlabRequest()` for Datadog tracing.

### Retry behaviour

Most methods accept an optional `retryAttempt` parameter and retry up to `RETRY_MAX = 5` times on error before returning `undefined`.

### Methods

#### `startPipeline(repositoryDetails, variables, retryAttempt?)`

Creates (triggers) a new pipeline run on a project's default branch.

- **SDK call:** `Pipelines.create(projectId, branch, { variables })`
- **Portal trigger:** Tenant provisioning and upgrade workflows that need to kick off a GitLab CI pipeline.
- **Variables injected** (via `getPipelineEnvVariables`):

```
TENANT       — tenant.name
ENVIRONMENT  — tenant.environment
TRIGGERED_BY — "saas_portal"
```

---

#### `runJob(projectId, jobId, tenant)`

Plays (re-triggers) a specific manual job within a project.

- **SDK call:** `Jobs.play(projectId, jobId, { jobVariablesAttributes })`
- **Portal trigger:** Ops Portal UI action to manually re-run a failed or skipped pipeline job for a tenant.
- **Variables injected:** same three as above (`TENANT`, `ENVIRONMENT`, `TRIGGERED_BY`).

---

#### `getPipelineById(projectId, pipelineId, retryAttempt?)`

Fetches full pipeline details by numeric ID.

- **SDK call:** `Pipelines.show(projectId, pipelineId)`
- **Portal trigger:** Status polling for an in-progress or completed pipeline.

---

#### `getJobs(projectId, pipelineId)`

Lists all jobs belonging to a pipeline.

- **SDK call:** `Jobs.all(projectId, { pipelineId })`
- **Portal trigger:** Displaying per-job status in the tenant pipeline view.

---

#### `findPipelineByCommit(projectId, sha, expectedTenantVar, retryAttempt?)`

Finds the pipeline for a specific commit SHA that also has a `TENANT` variable matching `expectedTenantVar`. Needed because multiple tenants' pipelines can share the same `master` SHA (see PLAT-1547 comment in source).

- **SDK calls:**
  1. `Pipelines.all(projectId, { sha })` — list pipelines for the commit
  2. `Pipelines.allVariables(projectId, pipelineId)` — check the `TENANT` variable on each candidate (401/403/404 responses are treated as "not ours")
  3. `getPipelineById(...)` — fetch full details once the matching pipeline is found
- **Portal trigger:** Looking up the current pipeline state from a tenant's stored commit SHA.

---

#### `getArtifactFromJob(projectId, pipelineId, jobName, artifactName)`

Downloads a specific artifact file from a named job in a pipeline.

- **SDK calls:**
  1. `Jobs.all(projectId, { pipelineId, includeRetried: false })` — find the job by name
  2. `JobArtifacts.downloadArchive(projectId, { jobId, artifactPath })` — stream the artifact
- **Returns:** artifact file contents as a string, or `''` if the job is not found.
- **Portal trigger:** Reading job output artifacts (e.g., generated config files) after a pipeline completes.

---

#### `fileExists(projectId, filePath, ref?, retryAttempt?)`

Checks whether a file exists in the repository at a given ref (defaults to `master`). A 404 response returns `false`; other errors trigger retries.

- **SDK call:** `RepositoryFiles.showRaw(projectId, filePath, ref)` — HEAD-style existence check without downloading file content.
- **Returns:** `boolean`
- **Portal trigger:** Pre-flight checks before triggering pipelines that depend on a specific file being present in the repo.

---

## 2. `GitlabAirflowDagAccessTokenManager` — Group Access Tokens (Airflow DAG)

**Source:** `app/_lib/GitlabAirflowDagAccessTokenManager.ts`

Singleton. Manages **group access tokens** scoped to `read_repository` for Airflow DAG access. Token names follow the convention `dag-read-<tenantName>`.

### Auth

`PRIVATE-TOKEN: <token>` header on all requests, using `GetEnvironment().gitlabToken`.

### Required env vars

| Env var | Description |
|---|---|
| `GITLAB_TOKEN` | Personal access token with permission to manage group access tokens |
| `GITLAB_DAG_GROUP_ID` | Numeric GitLab group ID whose access tokens are managed |

### HTTP client

Native `fetch` wrapped in `fetchWithTimeout` (30 s timeout, configurable for tests). No SDK.

### Response schema (Zod)

```typescript
GroupAccessTokenResponseSchema = {
  id: number
  name: string
  revoked: boolean
  created_at: string       // ISO 8601
  scopes: string[]
  user_id: number
  last_used_at?: string | null
  token?: string           // only present on creation
  active: boolean
  expires_at: string       // ISO 8601 date
  access_level: number
}
```

### Methods

#### `getAccessTokensForGroup()`

Paginates through all access tokens for the configured group (100 per page, follows `x-next-page` header).

- **Endpoint:** `GET /groups/{groupId}/access_tokens?per_page=100&page={n}`
- **Portal trigger:** Called internally before creating or deleting a token; also available for listing tokens in the UI.

---

#### `getAccessTokensForTenant(tenantName)`

Filters the full group token list to tokens whose `name` equals `dag-read-<tenantName>`.

- **Portal trigger:** Displaying Airflow DAG tokens for a specific tenant.

---

#### `createAccessTokenForGroup(tenantName)`

Creates a new group access token for a tenant.

- **Endpoint:** `POST /groups/{groupId}/access_tokens`
- **Portal trigger:** Tenant provisioning — creating the Airflow DAG read credential.
- **Request body:**

```json
{
  "name": "dag-read-<tenantName>",
  "access_level": 20,
  "expires_at": "<today + 11 months, yyyy-MM-dd>",
  "scopes": ["read_repository"]
}
```

- **Returns:** `GroupTokenResponse` (includes the plaintext `token` field, only present at creation time).

---

#### `deleteAccessToken(tenantName, tokenId)`

Deletes a specific access token by numeric ID, after first confirming the token belongs to the given tenant.

- **Endpoint:** `DELETE /groups/{groupId}/access_tokens/{tokenId}`
- **Portal trigger:** Tenant deprovisioning or token rotation.

---

## 3. `GitlabReleaseTokenManager` — Group Deploy Tokens (Container Registry)

**Source:** `app/_lib/GitlabReleaseTokenManager.ts`

Singleton. Manages **group deploy tokens** scoped to `read_registry` for pulling container images. Token `username` is set to the tenant name, and token `name` is `Customer - <tenantName>`.

### Auth

`PRIVATE-TOKEN: <token>` header on all requests, using `GetEnvironment().gitlabToken`.

### Required env vars

| Env var | Description |
|---|---|
| `GITLAB_TOKEN` | Personal access token with permission to manage group deploy tokens |
| `GITLAB_RELEASE_GROUP_ID` | Numeric GitLab group ID whose deploy tokens are managed |

### HTTP client

Native `fetch` (no timeout wrapper, no SDK).

### Response schema (Zod)

```typescript
TokenResponseSchema = {
  id: number
  name: string
  username: string
  token?: string           // only present on creation
  expires_at: string | null
  revoked: boolean
  expired: boolean
  scopes: string[]
}
```

### Methods

#### `getActiveDeployTokensForGroup()`

Paginates through all deploy tokens for the configured group (100 per page, follows `x-next-page` header).

- **Endpoint:** `GET /groups/{groupId}/deploy_tokens?per_page=100&page={n}`
- **Portal trigger:** Called internally before creating or deleting a token.

---

#### `getDeployTokensForTenant(tenantName)`

Filters the full group deploy token list to tokens whose `username` equals `tenantName`.

- **Portal trigger:** Listing deploy tokens for a specific tenant.

---

#### `createDeployTokenForGroup(tenantName)`

Creates a new group deploy token for a tenant.

- **Endpoint:** `POST /groups/{groupId}/deploy_tokens/`
- **Portal trigger:** Tenant provisioning — creating the container registry pull credential.
- **Request body:**

```json
{
  "name": "Customer - <tenantName>",
  "username": "<tenantName>",
  "expires_at": null,
  "scopes": ["read_registry"]
}
```

- **Returns:** `TokenResponse` (includes the plaintext `token` field, only present at creation time).

---

#### `deleteDeployToken(tenantName, tokenId)`

Deletes a specific deploy token by numeric ID, after confirming the token belongs to the given tenant.

- **Endpoint:** `DELETE /groups/{groupId}/deploy_tokens/{tokenId}`
- **Portal trigger:** Tenant deprovisioning or token rotation.

---

## Summary: env vars required

| Env var | Used by |
|---|---|
| `GITLAB_TOKEN` | All three clients |
| `GITLAB_DAG_GROUP_ID` | `GitlabAirflowDagAccessTokenManager` |
| `GITLAB_RELEASE_GROUP_ID` | `GitlabReleaseTokenManager` |

Note: `GitlabApi` does not read env vars directly — it receives the token at construction time from the caller, which reads `GITLAB_TOKEN` via `GetEnvironment()`.
