---
tags:
  - ops-portal
  - plat-981
  - testing
  - local-testing
created: 2026-05-26
status: verified-locally
related: "[[PLAT-981 — UAP Operator Version Auto-Update]]"
---

# PLAT-981 — Local Testing

> Implementation note: [[PLAT-981 — UAP Operator Version Auto-Update]]

---

## Environment Setup

- **Mode:** `DRY_RUN=true` in `.env.local` — git clones and commits happen for real, push runs as `git push --dry-run` (validates auth without writing to remote)
- **Database:** Local MongoDB via Docker (`npm run mongo-start`)
- **Server:** `npm run dev`
- **Login:** `ENABLE_TEST_USER=true`, email `test@example.com` / password `test`
- **GitLab token:** Personal Access Token in `GITLAB_TOKEN` — requires `read_repository` + `write_repository` scopes. SSO session must be active (`https://gitlab.com/groups/cequence/-/saml/sso`)

---

## Test Approach

The UAP upgrade workflow runs sequentially. Since most early jobs require external services (ArgoCD, Datadog, Grafana) that don't exist locally, the first four jobs were pre-seeded as `SUCCESS` directly in MongoDB. The coordinator then picked up `UpdateClusterServicesOperatorVersionsJob` as the next `PENDING` job and executed it automatically.

### Why not click "Start UAP Upgrade" in the UI?

`uapMinorUpgradeStatus` is `null` by default — MongoDB cannot dot-path into a null field. Seeding the entire status object directly avoids this and lets us skip straight to the job under test.

### Key discovery: jobs need explicit PENDING state

If a job has no entry in the `jobs` map at all, the coordinator's switch statement has no `default` case — it silently falls through without executing anything. The job must be explicitly set to `state: 'PENDING'` for the coordinator to call `start()` on it.

---

## Seed Commands Used

### ResourceVersion (target operator versions)

```javascript
db.resourceversionrepos.updateOne(
  { id: 'cequence-asp-2024.2.0' },
  { $set: {
    id: 'cequence-asp-2024.2.0',
    resource: 'cequence-asp',
    version: '2024.2.0',
    defaultVersion: true,
    stable: true,
    archived: false,
    development: false,
    eckOperatorVersion: '2.17.0',
    strimziOperatorVersion: '0.45.0',
    internalNotes: 'Test version for local dry-run'
  }},
  { upsert: true }
)
```

### Tenant with pre-seeded workflow status

Tenant: `pmdemo1-dev` — chosen because `Pulumi.pmdemo1-dev.yaml` exists in the real `cluster-services` remote repo and the portal's `PORTAL_ENVIRONMENT=dev` filter requires a dev environment tenant.

```javascript
db.tenantrepos.updateOne(
  { name: 'pmdemo1', environment: 'dev' },
  { $set: {
    id: 'pmdemo1-dev',
    customer: 'PM Demo 1',
    region: 'us-west-2',
    uapVersion: '2024.1.0',
    targetUapUpgradeVersion: '2024.2.0',
    provisionStatus: { status: 'SUCCESS', jobs: {} },
    uapMinorUpgradeStatus: {
      status: 'RUNNING',
      jobs: {
        VerifyUapApplicationSyncJob:              { jobId: 'skip-001', state: 'SUCCESS' },
        CreateDatadogDowntimeJob:                 { jobId: 'skip-002', state: 'SUCCESS' },
        CreateGrafanaSilenceTrafficJob:           { jobId: 'skip-003', state: 'SUCCESS' },
        UpdateApplicationsUapVersionJob:          { jobId: 'skip-004', state: 'SUCCESS' },
        UpdateClusterServicesOperatorVersionsJob: { jobId: 'pending-005', state: 'PENDING' }
      }
    }
  }},
  { upsert: true }
)
```

---

## Test Cases Verified

### Case 1 — GitLab token SSO expired (`dry-run-tenant-dev`)

**Setup:** Token existed but SSO session was expired.
**Result:** Job started, clone attempted, 403 returned immediately.
**Verified by:** Dev server log showing `remote: Cannot find valid SSO session` and UI showing `FAILURE` with `{}` error (later fixed to show actual message).
**Conclusion:** Job correctly propagates git errors as `FAILURE`.

---

### Case 2 — Pulumi file not found (`dry-run-tenant-dev`, valid token)

**Setup:** Token refreshed. Tenant `dry-run-tenant` does not exist in the real `cluster-services` repo.
**Result:** Clone succeeded, file lookup failed.
**Verified by:** Dev server log:
```
Performing a git clone for ...cluster-services.git... ✅
Error: Pulumi.dry-run-tenant-dev.yaml not found in cluster-services
```
UI showing `FAILURE` with the exact message from the job.
**Conclusion:** File-not-found case returns a clear, actionable `FAILURE` message.

---

### Case 3 — Versions differ, commit succeeds (`pmdemo1-dev`) ← MAIN TEST

**Setup:**
- Real Pulumi file exists in `cluster-services`: `Pulumi.pmdemo1-dev.yaml`
- Current values in file: `eckVersion: 2.16.1`, `strimziVersion: 0.44.0`
- Target from `ResourceVersion`: `eckVersion: 2.17.0`, `strimziVersion: 0.45.0`

**Result:** Clone succeeded, file found, versions compared, file updated, commit made, push failed (token missing `write_repository` scope).

**Verified by dev server logs:**
```
Starting UpdateClusterServicesOperatorVersionsJob for tenant pmdemo1-dev
Performing a git clone for ...cluster-services.git ✅
luke-local commit: 'Update ECK/Strimzi operator versions for pmdemo1-dev (UAP 2024.2.0)' ✅
remote: You are not allowed to upload code. 403 ❌ (token scope issue)
```

**Verified by inspecting the temp clone:**
```
cat /tmp/git-repo/provision/cluster-services-uap-update/869e58497c7264d741eef8a6/Pulumi.pmdemo1-dev.yaml
```
Output confirmed:
```yaml
saas-tenant-cluster-services:strimziVersion: 0.45.0   # was 0.44.0
saas-tenant-cluster-services:eckVersion: 2.17.0        # was 2.16.1
saas-tenant-cluster-services:awsLoadBalancerControllerVersion: 1.4.5  # untouched
saas-tenant-cluster-services:vpaEnabled: 'true'        # untouched
```

**Verified by UI:** Workflow dialog showed the job at the correct position (step 5, after _Update UAP version in applications project_) with the `FAILURE` message showing the raw git push error.

**Conclusion:** All job logic is proven correct end-to-end. The push failure is a local token permissions issue, not a code issue.

---

## What Was NOT Tested Locally (Needs Remote/Staging)

| Item | Why it needs remote |
|---|---|
| `git push` actually writing to `cluster-services` | Needs `write_repository` token scope and a real deployment |
| `WaitForClusterServicesPipelineJob` completing | Needs a real GitLab pipeline to trigger and run |
| `SyncTenantFromSourcesJob` reconciling operator versions back to tenant record | Needs a real Pulumi state in the cluster |
| No-change path on a real tenant (versions already match) | Can only happen naturally in staging |

---

## Unit Test Results

All 13 tests passing:

```
PASS __tests__/_lib/tenant/provisioning/upgrade/uap/UpdateClusterServicesOperatorVersionsJob.test.ts
  FAILURE cases (5 tests) ✅
  SUCCESS — versions already match, no commit (1 test) ✅
  SUCCESS — versions differ, commit + push (5 tests) ✅
  static properties (2 tests) ✅
```

---

## Gotchas for Future Reference

- `PORTAL_ENVIRONMENT=dev` in `.env.local` filters all coordinator queries to dev tenants only — use a dev environment Pulumi file when testing locally.
- `uapMinorUpgradeStatus: null` in MongoDB cannot be dot-pathed — always set the full object in one `$set`.
- Coordinator switch has no `default` case — jobs must have an explicit `state: 'PENDING'` entry to be executed.
- Dry-run temp directories accumulate in `/tmp/git-repo/` since `removeDirectory()` is skipped — glob patterns (`*`) will match multiple dirs from separate runs.
- `git push --dry-run` still requires `write_repository` scope on GitLab — even a simulated push gets a 403 if the token only has read access.
