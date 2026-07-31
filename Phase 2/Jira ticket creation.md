---
status: DONE — verified against code 2026-07-10
updated: "2026-07-10"
---

# Jira Ticket Creation — Post-Discovery Run (S12 / PLAT-1748)

**Status: DONE** — implemented, shipped, and verified against the actual codebase on 2026-07-10.

Related: [[Discovery job implementation]] (S7) · [[jira-ticket-creation-approach-comparison]] · [[Phase 2/Index|Phase 2 Docs]]

---

## What it does

After every discovery run completes, `scripts/orphan-jira-creator/createTicket.ts` creates one Jira issue in the CLOPS project summarising the run, with an optional CSV inventory attachment. Operators see a ticket appear in their board, can view the full resource list via the attached CSV, and use it as a lightweight audit trail.

Key design constraints (unchanged from original scope):
- **One ticket per run, only when resources are found**
- **Disabled by default** — required secrets must exist in the cluster
- **Non-fatal in the ways that matter** — see fail-loud redesign below for what's now intentionally NOT silent

---

## Architecture evolution

| Phase | Approach | Status |
|---|---|---|
| Original | Shell + `jiratickettemplate.yaml` using `curlimages/curl` directly | Superseded — could not produce detailed enough ADF-formatted ticket from shell |
| Agentic candidates | `claude-tool` vs `claude-mcp` (see [[jira-ticket-creation-approach-comparison]]) | Comparison drafted for senior discussion |
| **Shipped** | TypeScript script (`createTicket.ts`) with direct Jira REST calls, OAuth 2.0 auth, CSV attachment + retry | **Done** |

The team ultimately shipped a TypeScript implementation rather than either agentic approach from the comparison doc — Claude/agentic formatting was not needed once the description was redesigned as a summary rather than a full per-resource table (see the 32,767-character bug below).

---

## Auth — OAuth 2.0 client credentials (not Basic Auth)

`createTicket.ts` uses OAuth 2.0 client credentials, not Basic Auth or a personal API token:

1. `client_id`/`client_secret` → `POST api.atlassian.com/oauth/token` (client_credentials grant) → access token
2. `GET /oauth/token/accessible-resources` → resolve the Atlassian `cloudId`
3. All subsequent calls go through `api.atlassian.com/ex/jira/<cloudId>/...`

**Why OAuth over Basic Auth — what was tried and rejected:**
- Basic Auth with a scoped API token — failed, restricted API/project visibility
- Bearer auth with a personal/service token — failed, 403 (Bearer is only valid for OAuth JWTs)
- Basic Auth with a personal classic token — worked, but not sustainable for a shared service account
- **Landed on OAuth 2.0 client credentials** once the team's Jira admin provisioned a proper OAuth app — verified end-to-end (ticket PLAT-1782 created with correct browse link)

**Site resolution fix:** `fetchCloudId` matches the correct Atlassian site by checking `url.includes('xangent')`, falling back to `sites[0]` only if no match is found — guards against the OAuth app someday having access to more than one Atlassian org.

**Browse link fix:** the ticket browse link is built from the dynamically-resolved site URL. The earlier hardcoded `xangent.atlassian.net` / `JIRA_URL` constant was deleted entirely.

---

## Reusability

This WorkflowTemplate is shared/callable by other jobs, not exclusive to orphan discovery:

- `JIRA_PROJECT_KEY` / `JIRA_ISSUE_TYPE` are configurable via env vars, defaulting to `CLOPS` / `Operate on Customer Environment` — not hardcoded, so other Argo jobs can reuse the script without editing it.
- `JIRA_ATTACH_DOCUMENT` (boolean, default off) gates the entire CSV-attachment feature — deliberately opt-in, since this template is reusable by jobs that won't have an orphan-resource summary shape to attach. (Renamed from an earlier `JIRA_ATTACH_INVENTORY` to the more generic name, per explicit direction — the toggle concept is generic even though the CSV content it builds is intentionally specific to this job's resource-inventory shape.)

---

## The 32,767-character description bug (root cause of the CSV feature)

**Original design:** a full ADF table (one row per discovered resource) directly in the ticket description. At scale (hundreds of resources), this hit Jira's hard 32,767-character description limit → HTTP 400.

**Fix:** `buildAdf()` rewritten to a summary — intro paragraph (run ID, total count, tenant count), a disclaimer ("discovery only, verify before deleting"), a per-tenant resource-count breakdown (capped at 50 lines + overflow note), and a closing note pointing to where the full list lives (the CSV attachment). Verified with a 600-resource synthetic fixture: description stayed at 1,268 characters.

---

## CSV attachment feature

- **`buildInventoryCsv(summary)`** — proper RFC 4180 CSV escaping (quotes fields containing commas/quotes/newlines, doubles embedded quotes). Verified with fixtures containing commas and embedded quotes.
- **`attachToIssue(...)`** — uploads via `POST /issue/<key>/attachments` using Node's built-in `fetch`/`FormData`/`Blob` (deliberate deviation from this file's usual `node:https` style — hand-rolled multipart boundary encoding is error-prone).
- **Retry logic (`attachToIssueWithRetry`)** — up to 3 attempts. Transient errors (429, 5xx, network-level exceptions like timeouts) retry with backoff (1s, then 2s); terminal errors (400, 403, 413, etc.) fail immediately, no wasted retries. 15s per-attempt timeout via `AbortController`. Verified with fixtures covering all three shapes (terminal / transient-then-success / network-exhausted).
- **Fixed artifact path, always written:** the CSV is written to a fixed local path (`/tmp/orphan_inventory.csv`, not run-ID-scoped) unconditionally whenever the feature is enabled — independent of whether the Jira attach call succeeds. This is what lets it be declared as a static Argo output artifact (`inventory`, `optional: true`) on the `create-jira-ticket` step — always recoverable from the Argo UI regardless of attach outcome.
- **Dropped Slack file-upload API:** originally scoped to upload the CSV directly to Slack via the modern 3-step flow (`files.getUploadURLExternal` → upload → `files.completeUploadExternal`), which needs a `files:write` bot scope. Changed direction to: post a plain-text Slack message (reusing the same `chat.postMessage` mechanism the existing notify handler already uses, same `slack-credentials` secret / `chat:write` scope — no new Slack permissions needed) that points at the ticket and the Argo artifact, rather than carrying the file.
- **`notifyAttachFailure(...)`** — fires only after all 3 attach attempts are exhausted. Fully best-effort: skips silently if `SLACK_TOKEN`/`SLACK_CHANNEL` are not set, never throws, never affects the step's exit code — because the CSV is already safely recoverable via the Argo artifact regardless of whether this notification succeeds.

---

## Fail-loud redesign (exit-code semantics)

**Original design:** every Jira failure logged a warning and exited 0 — silent by design, "non-fatal."

**Changed to fail-loud after review:** OAuth-exchange failure, `cloudId`-resolution failure, and a non-2xx/network error creating the issue now `console.error` + exit 1 — so the workflow shows Failed and the `notify` exit handler's ❌ Slack alert fires. Intentional skips (no resources found, `MOCK=true`, creds not set) still exit 0 — these aren't failures.

**The CSV-attach failure path specifically does NOT follow this** — per explicit direction ("we will not fail the entire workflow for this one blip"), an attach failure (after retries) logs, notifies Slack, and lets the run succeed, since the ticket itself was created successfully and the CSV is recoverable via the artifact regardless.

---

## Supporting manifest/CI changes

- `jiratickettemplate.yaml`: added `attach-document` / `slack-channel` optional input parameters (defaults `"false"` / `""`), `SLACK_TOKEN` mounted from `slack-credentials` with `optional: true` (never blocks pod startup if the secret/key is absent), `inventory` output artifact `optional: true`.
- `workflowtemplate-dev.yaml` / `workflowtemplate-prod.yaml`: set `attach-document: "true"` and thread it + the existing `slack-channel` parameter down to the `create-jira-ticket` step.
- Version labels bumped: `jiratickettemplate.yaml` and both `workflowtemplate-*.yaml` → `1.2.0`; `slacknotificationtemplate.yaml` → `1.1.0`.
- New Typecheck CI job added — a gate that was lost when the Dockerfile switched from compiled JS to running `tsx` directly.
- Docs updated: `.env.example`, `README.md` (new "Auth flows" + "Fail-loud behavior" sections), repo-local `CLAUDE.md`.

---

## Verification (2026-07-10)

All claims above were independently verified against the current code (`createTicket.ts`, `jiratickettemplate.yaml`, both `workflowtemplate-*.yaml`, `.env.example`, `README.md`) — every claim CONFIRMED. One stale note found: the repo-local `CLAUDE.md` still lists the `JIRA_URL` hardcoded issue as open even though the code already fixed it — flagged as a doc discrepancy to clean up.

---

## Config env vars

| Env var | Required | Default | Notes |
|---|---|---|---|
| `JIRA_PROJECT_KEY` | No | `CLOPS` | Configurable — reusable by other Argo jobs |
| `JIRA_ISSUE_TYPE` | No | `Operate on Customer Environment` | Configurable |
| `JIRA_ATTACH_DOCUMENT` | No | `false` | Gates the CSV attachment feature |
| `SLACK_TOKEN` | If attach-failure alerts wanted | — | Mounted `optional: true` |
| `SLACK_CHANNEL` | If attach-failure alerts wanted | — | — |

---

## What not to do

- **Don't reintroduce a full per-resource ADF table in the description** — it will exceed the 32,767-character limit at scale. Use the CSV attachment for the full list.
- **Don't make CSV-attach failures fatal** — explicit direction was to keep this non-blocking.
- **Don't hardcode the Jira site URL** — always resolve dynamically via `fetchCloudId`.

---

## Open questions / deferred items

- Should CLOPS tickets be linked to a parent epic? Still TBD.
- Repo-local `CLAUDE.md` needs a pass to remove stale "still open" items that are actually already fixed in code (found: JIRA_URL hardcoded item, and others per verification pass).
- Cron schedule still `*/1 * * * *` (every-minute, testing) — not yet restored to daily pending a cadence discussion; `suspend: true` protects against accidental firing.
- Dockerfile tags unpinned (`node:20-slim`, global `tsx` install) — deferred, revisit only if the image misbehaves in Argo.
- `CreateContainerConfigError` hang risk: a missing required secret currently makes the pod retry-loop indefinitely instead of failing fast. Fix identified (`optional: true` on the `secretKeyRef`s + let the script itself report the missing config) — not yet implemented.
- Prod full rollout still gated on: confirming `saas-workloads-prod-read` secret key names, filling in the prod tenant list, confirming prod CI vars exist.
