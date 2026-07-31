---
status: IN REVIEW — submitted 2026-06-17
updated: 2026-06-17
---

# S8 — Error Handling, Retries, and Notifications (PLAT-1649)

**Status: IN REVIEW**

Related: [[Discovery job implementation]] (S7) · [[Phase 2/Index|Phase 2 Docs]]

---

## What was built

Five changes across the job codebase, no structural changes to the account/tenant error resilience model already in place.

---

## 1. Retry configuration — `RETRY_MAX_ATTEMPTS`

**New env var:** `RETRY_MAX_ATTEMPTS` (integer, default: `5`)

Added to `JobConfig` in `src/config.ts`. `loadConfig()` reads it, validates it's a positive integer (throws clearly if not), and defaults to 5 if unset. Threaded through to every AWS SDK client in the job:

| File | Client(s) |
|---|---|
| `src/aws/credentials.ts` | `STSClient` in `verifyProvider()` |
| `src/discovery/resourceExplorer.ts` | `ResourceExplorer2Client` in `createClient()` |
| `src/discovery/supplementaryDiscover.ts` | EC2, EKS, AutoScaling, CloudFormation clients |

All clients use `retryMode: 'adaptive'` alongside `maxAttempts` — adaptive mode backs off automatically on throttling responses (`ThrottlingException`, 429s).

Both WorkflowTemplates have the env var documented as a commented placeholder:

```yaml
# - name: RETRY_MAX_ATTEMPTS
#   value: "5"   # default; increase if seeing ThrottlingException in logs
```

---

## 2. Error type classification

**New helper:** `classifyError()` in `src/index.ts`

Classifies each caught tenant-level error into one of four categories by inspecting the error message:

| Category | Matches |
|---|---|
| `throttled` | `ThrottlingException`, `RequestLimitExceeded`, `Throttling`, `Rate exceeded` |
| `permission` | `AccessDenied`, `UnauthorizedOperation`, `AuthFailure`, `is not authorized` |
| `timeout` | `TimeoutError`, `ETIMEDOUT`, `ECONNRESET`, `socket hang up` |
| `other` | Everything else |

Replaces the single `tenantErrors` counter with a `tenantErrorCounts` object tracking counts per category.

---

## 3. API write error tracking

`writeRows()` in `src/output/apiWriter.ts` already throws on non-2xx. Previously that throw fell into the per-tenant try/catch, making API write failures indistinguishable from discovery failures.

Fixed with a separate inner try/catch around `writeRows()` and a dedicated `apiWriteErrors` counter. API write failures are non-fatal and never counted as tenant discovery errors.

---

## 4. Improved summary block

Both new counters surface in the summary. Lines only appear when their counts are > 0:

```
── Summary ──
  accounts:      1 (0 failed)
  tagged rows:   12
  supplementary: 4
  total written: 16
  tenant errors: 3 (throttled: 1, permission: 1, other: 1)
  api write errors: 1 (non-fatal)
```

---

## What is NOT in scope (deferred)

| Item | Where |
|---|---|
| Slack notifications on workflow exit | S11 (PLAT-1699) — `manifests/notificationtemplate.yaml` |
| Argo `continueOn: { failed: true }` | Not applicable — job is a single container, not multi-step |

---

## Invariants preserved

- Sequential processing maintained — no parallelism added
- Account-level failures still flip exit code to 1 but do not abort the batch
- Tenant-level errors remain non-fatal
- Discovery modules (`resourceExplorer.ts`, `supplementaryDiscover.ts`) received only minimal targeted changes (maxAttempts param added); no logic refactored
- TypeScript compiles clean throughout
