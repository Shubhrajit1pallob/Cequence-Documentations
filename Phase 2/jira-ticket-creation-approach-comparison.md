---
status: OPEN — both approaches in active local testing
updated: "2026-07-01"
---

# Jira Ticket Creation — Agentic Approach Comparison

**Status: OPEN** — comparison prepared for senior discussion. Decision pending.

Related: [[Jira ticket creation]] (S12 / PLAT-1748) · [[Phase 2/Index|Phase 2 Docs]]

---

## Context

The orphaned-resource discovery job (Phase 2) is an Argo CronWorkflow running on the central-ops Kubernetes cluster. It discovers orphaned AWS resources for deleted tenants and writes a JSON summary containing: `runId`, `discoveredAt`, `totalResources`, and a `resources` array (each entry has `tenant`, `arn`, `service`, `resourceType`, `region`, `accountId`, `discoveryType`).

After discovery, a second Argo workflow step must create a single Jira ticket in the CLOPS project (issue type: Operate on Customer Environment) listing all orphaned resources found in that run. The ticket must be detailed enough for an engineer to locate and destroy the specific resources.

A pure shell/curl approach (no AI) was evaluated and **superseded** — it could not produce a detailed enough ADF-formatted resource table from a shell script. Both agentic approaches below are currently being **built and tested locally in parallel** to provide real evidence for the senior decision. No choice has been made yet.

---

## Approach 1 — claude-tool

### How it works

1. The Argo step calls the Anthropic Messages API (`POST /v1/messages`) with the discovery summary JSON as the prompt
2. A `create_jira_ticket` tool is defined inline in the API request as a JSON schema — it describes the parameters Claude should fill in (ticket title, ADF description with resource table, resource count)
3. Claude reads the summary, decides to call the tool, and returns a `tool_use` block with the filled-in parameter values
4. The Argo step reads that output and executes the actual Jira REST API call (`POST /rest/api/3/issue`) using Claude's generated values
5. Claude handles the formatting and judgment; the Argo step executes the Jira call

**API used:** Anthropic Messages API — `tool_use` is stable (no beta header required)

**Secrets required in cluster:**

| Secret | Key | Purpose |
|---|---|---|
| `anthropic-api-key` | `apiKey` | Call the Claude API |
| `jira-credentials` | `email`, `apiToken` | Call the Jira REST API directly |

### Pros

- **API stability:** `tool_use` is a stable, production-grade Anthropic API feature. No beta risk.
- **Team owns the Jira call:** The Argo step makes the final `POST /rest/api/3/issue` call directly — the team can inspect, log, retry, and adjust the Jira payload independently of any external system.
- **Debuggable:** Two distinct HTTP calls (Claude API → Jira API) with separate response objects. If Jira rejects the ticket, the error is clear and local.
- **Testable:** The tool schema can be validated offline. Claude's `tool_use` output can be mocked in unit tests without a live Anthropic connection.
- **No OAuth lifecycle:** `jira-credentials` uses a long-lived Jira API token (email + apiToken). No refresh loop.
- **Extensibility is incremental:** Adding more tools (Slack notification, PagerDuty alert) means adding more tool definitions in the same request — each is independent and the pattern is well understood.
- **No external MCP dependency:** Works entirely within the stable Anthropic API surface. Not affected by MCP versioning or Atlassian MCP server availability.

### Cons

- **Tool schema maintenance:** The `create_jira_ticket` JSON schema must be written and kept in sync with what Jira actually accepts (ADF format, field names, project key). If Jira's API changes, the schema and prompt must be updated.
- **Two secret types:** Both `anthropic-api-key` and `jira-credentials` must be present and rotated. More k8s secrets to manage.
- **Two HTTP calls:** Claude API call + Jira REST API call. A failure in either must be handled separately; retry logic applies to both.
- **ADF correctness depends on Claude:** The team trusts Claude to produce valid ADF JSON. Malformed ADF will cause Jira to reject the ticket. This needs a test run to verify ADF output quality before production.
- **Prompt sensitivity:** Changes to the discovery job's summary JSON shape (new fields, renamed keys) must be reflected in the prompt. The prompt is not strongly typed.

---

## Approach 2 — claude-mcp

### How it works

1. The Argo step calls the Anthropic Messages API with the `anthropic-beta: mcp-client-2025-11-20` header
2. The request includes an `mcp_servers` array pointing at the Atlassian remote MCP server URL, with an OAuth Bearer token
3. The Anthropic backend connects to the Atlassian MCP server, discovers its tools (including `createJiraIssue`), and Claude calls them directly
4. The Anthropic backend executes the Jira call — the Argo step makes **one HTTP call total** and never directly calls Jira
5. No tool schema needs to be written or maintained

**API used:** Anthropic Messages API with `mcp-client-2025-11-20` beta header — currently in public beta. One breaking version change has already occurred (`mcp-client-2025-04-04` deprecated). Not available on Amazon Bedrock or Google Cloud. Not ZDR eligible.

**Secrets required in cluster:**

| Secret | Key | Purpose |
|---|---|---|
| `anthropic-api-key` | `apiKey` | Call the Claude API |
| `atlassian-mcp-oauth` | `token` | OAuth Bearer token for the Atlassian MCP server |
| `jira-credentials` | `email`, `apiToken` | May still be needed depending on how the Atlassian MCP server handles auth internally — TBC |

### Pros

- **No tool schema to write or maintain:** The Atlassian MCP server exposes `createJiraIssue` (and other tools) natively — Claude calls them directly with no JSON schema definition required from the team.
- **Fewer direct integrations:** The Argo step makes one API call (Anthropic). The Anthropic backend manages the Atlassian connection. The team owns less integration surface.
- **Extensibility via MCP ecosystem:** Adding Slack, PagerDuty, or other services in future means adding MCP server entries — no new tool schemas if those services expose MCP servers.
- **Anthropic manages the MCP client protocol:** Connection handling, tool discovery, and retries at the MCP layer are Anthropic's responsibility.
- **One HTTP call from the pod:** Simplifies the Argo step container logic — one `curl` to the Anthropic API, done.

### Cons

- **Beta API risk:** The `mcp-client` beta header has already had one breaking version change. A scheduled production CronWorkflow depending on a beta feature is a real operational risk — a breaking change requires urgent manifest updates.
- **Not on Bedrock or GCP:** If the team ever migrates Claude usage to Amazon Bedrock or Google Cloud Vertex AI, this feature is not available. Locks the workflow to the Anthropic-hosted API.
- **Not ZDR eligible:** Data exchanged with the MCP server (tool definitions, execution results, resource inventory) is retained under Anthropic's standard retention policy. Cannot be opted out.
- **OAuth token lifecycle:** The Atlassian MCP server requires an OAuth Bearer token. The team must handle token acquisition and refresh — either manually rotating the `atlassian-mcp-oauth` secret or building an automated refresh mechanism. Long-lived tokens may expire mid-operation.
- **Atlassian MCP server is an unknown:** The URL, available tools, auth requirements, and reliability of the Atlassian MCP server are not yet confirmed for this environment. It may require the same `jira-credentials` secret anyway, negating the secret-reduction benefit.
- **Less debuggable:** When the Jira ticket creation fails, the error path crosses the Anthropic backend → Atlassian MCP server boundary. Diagnosing whether the failure is in the Claude API call, the MCP connection, or the Jira write is harder than Approach 1.
- **Harder to test offline:** Testing requires a live Anthropic connection with the MCP beta header + an accessible Atlassian MCP server. Cannot be mocked as easily as a `tool_use` schema.
- **Migration burden when MCP stabilises:** When the beta graduates to stable (or if another version bump occurs), the manifest and step code must be updated under a live production workflow.

---

## Local simulation design (Phase 1)

Both approaches are being validated locally using `scripts/simulate-workflow.sh`, which mirrors the two-step Argo workflow without a live cluster. `claudeTool.ts` supports four modes:

| Mode | Claude called? | Jira called? | Use case |
|---|---|---|---|
| `--print-request` | No | No | Inspect the full Claude API request payload before sending |
| `--mock` | No | Yes | Test the Jira write path without calling Claude |
| `--response-file <path>` | No | Yes | Replay a saved `tool_use` block against real Jira |
| *(default)* | Yes | Yes | Full production path — real Claude + real Jira |

No Docker image changes in Phase 1 — all local runs use `npm run dev` / `tsx`.

---

## Local simulation design (Phase 1)

Both approaches are being validated locally using `scripts/simulate-workflow.sh`, which mirrors the two-step Argo workflow without a live cluster. `claudeTool.ts` supports four modes:

| Mode | Claude called? | Jira called? | Use case |
|---|---|---|---|
| `--print-request` | No | No | Inspect the full Claude API request payload before sending |
| `--mock` | No | Yes | Test the Jira write path without calling Claude |
| `--response-file <path>` | No | Yes | Replay a saved `tool_use` block against real Jira |
| *(default)* | Yes | Yes | Full production path — real Claude + real Jira |

No Docker image changes in Phase 1 — all local runs use `npm run dev` / `tsx`.

---

## Open questions

These need senior input before a decision can be made:

1. **What is the URL of the Atlassian MCP server** connected to the team's Claude Code environment? (Required to even prototype Approach 2.)
2. **Who manages the OAuth token lifecycle** for the Atlassian MCP server — is there a service account, or does the token need to be manually refreshed? How long do tokens live?
3. **Is the team comfortable taking a dependency on a public beta Anthropic feature** for a scheduled production CronWorkflow? (One breaking change has already occurred; another is possible before GA.)
4. **Are there any data retention concerns?** Note: Approach 2 (claude-mcp) is explicitly not ZDR eligible. Approach 1 (claude-tool) does not pass resource inventory through the MCP path — only the summary JSON is sent to the Claude API under standard Anthropic retention policy.
5. **Anthropic API key vs Bedrock** — does the team have an Anthropic API key available, or should the implementation use AWS Bedrock (Claude on Bedrock via existing IRSA/SSO credentials) as the primary path? Note: Approach 2 (claude-mcp) is not available on Bedrock.

---

## Summary comparison

| Dimension | Approach 1 — claude-tool | Approach 2 — claude-mcp |
|---|---|---|
| **API stability** | Stable (`tool_use` is GA) | Public beta — one breaking change already |
| **Secrets needed** | `anthropic-api-key` + `jira-credentials` | `anthropic-api-key` + `atlassian-mcp-oauth` (+ possibly `jira-credentials`) |
| **Who executes the Jira call** | Argo pod (direct Jira REST API) | Anthropic backend (via Atlassian MCP server) |
| **Tool schema maintenance** | Team owns and maintains JSON schema | No schema — Atlassian MCP server provides it |
| **HTTP calls from pod** | 2 (Claude API + Jira REST) | 1 (Claude API only) |
| **OAuth lifecycle** | None — Jira API token is long-lived | Required — OAuth token for Atlassian MCP server |
| **ZDR eligible** | Standard retention (not ZDR) | No — explicitly excluded |
| **Extensibility** | Add tool definitions per service | Add MCP server entries per service |
| **Testability** | Mockable offline; schema testable | Requires live Anthropic + Atlassian MCP connection |
| **Debuggability** | Two clear failure points, both local | Failure spans Anthropic backend → MCP server |
| **Bedrock / GCP compatible** | Yes | No |
| **Unblocked today** | Yes — no unknowns | No — Atlassian MCP URL + OAuth unconfirmed |
| **Prod risk** | Low | Depends on MCP graduating to stable |
