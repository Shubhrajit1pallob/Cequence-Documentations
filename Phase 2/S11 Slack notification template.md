---
status: DONE — implemented 2026-06-24
updated: 2026-06-24
---

# S11 — Slack Notification Template (PLAT-1699)

**Status: DONE**

Related: [[Discovery job implementation]] (S7) · [[S8 error handling retries notifications]] (S8) · [[Phase 2/Index|Phase 2 Docs]]

---

## What was built

The existing cluster-wide `notification-tasks` WorkflowTemplate is a stub — it echoes to stdout and never calls Slack. Rather than fix it, a new project-owned WorkflowTemplate was created that actually delivers Slack notifications.

### New file: `manifests/slacknotificationtemplate.yaml`

- **Kind:** `WorkflowTemplate`
- **Name:** `slack-notification`
- **Namespace:** `argo-workflows`
- **Template:** `send-message`
- **Image:** `curlimages/curl`
- **Calls:** `POST https://slack.com/api/chat.postMessage` using the Slack bot token auth (not webhook)
- **Format:** Slack Block Kit attachments — green card (✅) on `Succeeded`, red card (❌) on any other status

**Parameters:**

| Parameter | Source | Notes |
|---|---|---|
| `SLACK_CHANNEL` | `{{workflow.parameters.slack-channel}}` | Channel ID (default: `C0BB3LL3343`) |
| `WORKFLOW_STATUS` | `{{workflow.status}}` | Argo workflow exit status |
| `WORKFLOW_NAME` | `{{workflow.name}}` | Argo workflow run name |
| `ENV_NAME` | Hardcoded per cluster | `"dev"` in workflowtemplate-dev.yaml, `"prod"` in workflowtemplate-prod.yaml |

**Why `ENV_NAME` is hardcoded (not a workflow parameter):** Argo raises a validation error if you try to pass `{{workflow.parameters.env-name}}` into an `onExit` handler via `templateRef` — the parameter context is not available in the exit handler scope. Hardcoding per cluster is the correct workaround.

**Auth:** Bot token from K8s secret `slack-credentials`, key: `token`. The token must be a valid Slack bot token (`xoxb-*`).

---

## How it is wired into the discovery workflow

Both `manifests/workflowtemplate-dev.yaml` and `manifests/workflowtemplate-prod.yaml` were updated:

```yaml
spec:
  onExit: notify
  ...
  templates:
    - name: discover
      ...
    - name: notify
      steps:
        - - name: slack
            templateRef:
              name: slack-notification
              template: send-message
            arguments:
              parameters:
                - name: SLACK_CHANNEL
                  value: "{{workflow.parameters.slack-channel}}"
                - name: WORKFLOW_STATUS
                  value: "{{workflow.status}}"
                - name: WORKFLOW_NAME
                  value: "{{workflow.name}}"
                - name: ENV_NAME
                  value: "dev"   # "prod" in workflowtemplate-prod.yaml
```

Argo calls the `notify` template on both success and failure exits.

---

## Other changes made in this session

### API write path removed

The `apiWriter.ts` write path was removed — there is no ops-portal API endpoint yet to write to. Changes:
- `src/output/apiWriter.ts` — deleted entirely
- `src/config.ts` — `dryRun` and `apiUrl` fields removed
- `src/index.ts` — `writeRows()` call and related code removed

The job now always logs discovered resources to stdout only. Note: this write-path removal is unrelated to the S13 external API — S13 ([[Targets API endpoint]]) is a read path (fetching the deleted-tenant list), not a write path for discovered resources. The original note above conflated the two; corrected here on 2026-07-10.

### Merge conflicts resolved

`src/index.ts` and `src/config.ts` had merge conflicts from the `s8-error-handling` branch. Resolved by keeping the more complete throttling regex from `s8-error-handling` and removing the duplicate `retryMaxAttempts` block from config.

---

## Current blocker

The `slack-credentials` secret exists in the `argo-workflows` namespace on the dev cluster but contains the **wrong token value** — it does not start with `xoxb-` and is not a valid Slack bot token.

**Action required (seniors):** Update the `slack-credentials` secret with the correct `slackToken` value from the `saas-service-portal` Pulumi config.

```bash
kubectl create secret generic slack-credentials \
  --from-literal=token=xoxb-<correct-token> \
  -n argo-workflows \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## Follow-up work — two new mechanisms (verified against code 2026-07-10)

Two additional Slack notification mechanisms were added on top of this WorkflowTemplate, as part of the S12 Jira ticket creation implementation:

### A. Workflow-level "what failed" detail (onExit handler)

The existing `notify` exit handler (fires on every workflow completion, success or failure) now also renders a `{{workflow.failures}}` detail block — only on the failure branch; the success message is byte-identical to before.

- The failures JSON is passed via an **env var**, not spliced directly into the shell script — specifically so quotes/backslashes/apostrophes in a failure message can't break the shell or the JSON payload. Verified with a fixture containing all three characters.
- Capped to 1,500 characters to keep the Slack message readable.

### B. Attach-failure fallback (separate path, not through this shared template)

A narrower mechanism, called directly from `createTicket.ts` — **not** routed through `slacknotificationtemplate.yaml`:

- Only fires when the CSV attach specifically fails after all retries are exhausted (see [[Jira ticket creation]] for the retry logic).
- Posts directly via `chat.postMessage` (same secret/scope this template already uses) — needed because this must fire mid-workflow, not just at the workflow's final exit.
- Kept deliberately separate from this generic handler, since the shared template only takes generic text params today and has no awareness of Jira/attach-specific context.

**Version bump:** `slacknotificationtemplate.yaml` → `1.1.0`.

**Note on the blocker above:** both new mechanisms share the same `slack-credentials` secret as the original template — if that secret still has the wrong token value, both mechanisms are silently affected the same way.

**Verification note (2026-07-10):** both mechanisms confirmed present and matching this description in `slacknotificationtemplate.yaml` and `createTicket.ts`.

---

## What not to do

- **Do not reference `notification-tasks`** from orphan-discovery workflows — it is a cluster-wide stub that only echoes to stdout. Use `slack-notification` instead.
- **Do not route attach-failure alerts through the shared `notify` handler** — it doesn't carry the Jira/attach-specific context; call `chat.postMessage` directly instead, as `createTicket.ts` does.
