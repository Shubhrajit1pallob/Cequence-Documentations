---
title: Slack API Integration
scope: ops-portal
last-updated: 2026-06-22
---

# Slack API Integration

The Ops Portal has two inbound HTTP routes that Slack calls (slash commands and interactive components) and one outbound client used throughout the portal to post messages back into Slack.

---

## Inbound Routes

### 1. Slash Commands — `POST /api/external/slack/commands`

**Source:** `app/api/external/slack/commands/route.ts`

#### Purpose

Entry point for all Slack slash commands registered against the Ops Portal Slack app. Slack POSTs here when a user runs any of the supported commands from a Slack channel. The route dispatches to per-command handlers and returns a synchronous JSON response that Slack renders immediately.

#### Auth / Verification

Every request is verified with `verifySlackSignature()` before any business logic runs:

- Reads `x-slack-signature` and `x-slack-request-timestamp` headers from the Slack request.
- Rejects any request whose timestamp is more than 5 minutes old (replay-attack guard).
- Computes `HMAC-SHA256` over `v0:<timestamp>:<raw-body>` using `SLACK_SIGNING_SECRET` and compares it to the header value with a timing-safe equality check.
- Returns `401 { error: "Invalid signature" }` on failure.

No portal user session / Descope auth is involved — Slack's own signing mechanism is the sole auth layer for inbound Slack traffic.

#### Request

Slack sends an `application/x-www-form-urlencoded` body. The route reads the following fields:

| Field | Description |
|---|---|
| `command` | The slash command string, e.g. `/extend-tenant` |
| `channel_name` | Human-readable name of the channel where the command was run |
| `channel_id` | Slack channel ID |
| `user_name` | Slack username of the invoking user |
| `user_id` | Slack user ID of the invoking user |
| `team_id` | Slack workspace (team) ID |
| `text` | Any text argument provided after the command |

#### Supported Commands

##### `/extend-tenant [days]`

Extends the End-of-Life (EOL) date for tenant(s) linked to the invoking channel.

- `text` is parsed as the number of days to extend (default: `30`; must be 1–365).
- Looks up tenants linked to the channel by `channel_name`.
- **Single tenant:** extends immediately, then calls `SlackNotification.sendDirectMessage()` to post a public `:white_check_mark:` confirmation to the channel. Returns an ephemeral acknowledgement.
- **Multiple tenants:** returns an ephemeral Block Kit message with up to 20 buttons (chunked 5 per `actions` block). Each button carries `action_id: extend_eol_select:<tenantId>` and `value: <durationDays>`. The actual extension is deferred to the interactions endpoint when the user clicks a button.
- Validation errors and lookup failures return ephemeral error messages.

##### `/tenant-info`

Posts information about the tenant(s) linked to the invoking channel.

- **Single tenant:** immediately kicks off `postTenantInfo(tenantId, channelId)` as fire-and-forget, returns ephemeral `"Fetching info..."` text.
- **Multiple tenants:** returns an ephemeral Block Kit message containing a `static_select` dropdown (`action_id: tenant_info_select`) with up to 100 options. Selection is handled by the interactions endpoint.

##### `/create-tenant`

Returns an ephemeral Block Kit message with a primary-style button that opens the POV tenant creation form in the Ops Portal.

- The button URL is `<NEXTAUTH_URL>/tenants/pov?channel_name=<name>&channel_id=<id>&user_id=<userId>`.
- Does not perform any write operation itself; the form in the portal handles creation.
- `action_id: open_new_pov_page` (button is URL-type; interaction endpoint is not invoked).

##### `/attach`

Returns an ephemeral Block Kit message with a button opening the Ops Portal's channel-attach page.

- URL: `<NEXTAUTH_URL>/slack/attach?channel_name=<name>&channel_id=<id>`.
- `action_id: open_attach_page`.

##### `/delete-tenant-request`

Delegates entirely to `handleDeleteTenantRequest()` in `@/_lib/tenant/deletion-requests/slackDeletionRequestHandler`. Passes: `userId`, `userName`, `channelId`, `channelName`, `teamId`, `text`, and the portal's `baseUrl` (derived from `NEXTAUTH_URL`).

#### Response (synchronous Slack payload)

All responses are JSON with HTTP 200 (Slack requires 200 for command acknowledgement). Shape varies by command:

```jsonc
// Ephemeral plain-text response
{
  "response_type": "ephemeral",
  "text": "..."
}

// Ephemeral Block Kit response (buttons or select)
{
  "response_type": "ephemeral",
  "blocks": [
    { "type": "section", "text": { "type": "mrkdwn", "text": "..." } },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "..." },
          "action_id": "...",
          "value": "...",      // present on extend-tenant buttons
          "url": "...",        // present on link-type buttons
          "style": "primary"   // present on primary buttons
        }
      ]
    }
  ]
}
```

Error responses:

```jsonc
// Signature failure
HTTP 401  { "error": "Invalid signature" }

// Unexpected server error
HTTP 500  { "error": "Internal server error" }
```

#### Required Env Vars

| Variable | Purpose |
|---|---|
| `SLACK_SIGNING_SECRET` | HMAC signing secret used to verify Slack request signatures |
| `NEXTAUTH_URL` | Portal hostname; used to build button/link URLs sent back to Slack |

---

### 2. Interactive Components — `POST /api/external/slack/interactions`

**Source:** `app/api/external/slack/interactions/route.ts`

#### Purpose

Receives Slack interactive component callbacks — specifically `block_actions` events triggered when users click buttons or select dropdown options in Slack messages originally sent by the portal. All handlers return HTTP 200 immediately (Slack's 3-second timeout requirement); any work that needs to post a follow-up message does so inline before the response is returned (or fire-and-forget where the operation is async).

#### Auth / Verification

Same `verifySlackSignature()` mechanism as the commands route — `HMAC-SHA256` over the raw body using `SLACK_SIGNING_SECRET`, plus the 5-minute replay guard.

#### Request

Slack sends `application/x-www-form-urlencoded` with a single field:

| Field | Description |
|---|---|
| `payload` | URL-encoded JSON string containing the full interaction payload |

The parsed payload structure (relevant fields):

```jsonc
{
  "type": "block_actions",
  "user": {
    "id": "<slack-user-id>",
    "name": "<slack-username>"
  },
  "channel": {
    "id": "<channel-id>"
  },
  "message": {
    "ts": "<message-timestamp>"   // present for es_resize actions
  },
  "actions": [
    {
      "action_id": "<action-id>",
      "value": "<value>",                         // for button actions
      "selected_option": { "value": "<value>" }  // for select actions
    }
  ]
}
```

#### Handled Action IDs

##### `tenant_info_select`

Triggered when a user picks a tenant from the multi-tenant `/tenant-info` dropdown.

- Reads `action.selected_option.value` as `tenantId`.
- Calls `postTenantInfo(tenantId, channelId)` fire-and-forget.
- Returns `HTTP 200` (empty body).

##### `extend_eol_select:<tenantId>` / `extend_eol:<tenantId>[:<days>]`

Triggered when a user clicks a tenant button from the multi-tenant `/extend-tenant` response, or from older-format extend-EOL messages.

- `tenantId` is extracted from the `action_id` suffix.
- `durationDays` is parsed from `action.value`.
- Validates: tenantId present, durationDays 1–365. Logs a warning and returns 200 silently on invalid input.
- Calls `extendTenantEol(tenantId, durationDays, userId, userName)`.
- Posts a follow-up message to the channel via `SlackNotification.sendDirectMessage()`:
  - Success: `:white_check_mark: <@userId> extended EOL for <tenant-link> by N days. New EOL: <date>`
  - Failure: `:x: Failed to extend EOL for <tenantId>: <error>`
- Returns `HTTP 200` (empty body).

##### `es_resize_approve:<pendingId>` / `es_resize_decline:<pendingId>` / `es_resize_retry:<pendingId>`

Approval/decline/retry actions for Elasticsearch PVC resize requests.

- Extracts `pendingId` from the `action_id` suffix.
- Reads `payload.message.ts` as `messageTs` for threading.
- Delegates to `handleEsResizeAction({ actionId, pendingId, userId, userName, channelId, messageTs })` fire-and-forget.
- Returns `HTTP 200` (empty body).

#### Response

All responses are `HTTP 200`. The body is either empty (`null`) or a JSON error for unexpected cases:

```jsonc
// Normal acknowledgement (all handled action types)
HTTP 200  (empty body)

// Signature failure
HTTP 401  { "error": "Invalid signature" }

// Missing payload field
HTTP 400  { "error": "Missing payload" }

// Unexpected server error
HTTP 500  { "error": "Internal server error" }
```

#### Required Env Vars

| Variable | Purpose |
|---|---|
| `SLACK_SIGNING_SECRET` | HMAC signing secret for verifying Slack request signatures |
| `NEXTAUTH_URL` | Portal hostname; used by `buildTenantLink()` to construct Slack-linked tenant URLs |
| `SLACK_TOKEN` | Bot token used by `SlackNotification` to post follow-up messages |
| `SLACK_NOTIFICATIONS_CHANNEL` | Default channel used by `SlackNotification` constructor (fallback target) |

---

## Outbound Client

**Source:** `app/_lib/slack.ts`

### External Service

Slack Web API — `https://slack.com` (base URL handled internally by the `@slack/web-api` SDK).

### SDK

`@slack/web-api` — official Slack Node.js SDK. Specifically uses:

- `WebClient` — instantiated per `SlackNotification` instance, initialized with the bot token from `SLACK_TOKEN`.
- `KnownBlock` from `@slack/types` — for typed Block Kit block construction.

### Class: `SlackNotification`

```typescript
new SlackNotification(options?: { skipDryRun?: boolean })
```

- `skipDryRun`: when `true`, sends even when `DRY_RUN=true`. Defaults to `false` (i.e., skips in dry-run mode).
- The constructor reads `SLACK_TOKEN` (bot token) and `SLACK_NOTIFICATIONS_CHANNEL` (default channel) from env at instantiation time.

#### Dry-run behavior

When `DRY_RUN` is not `"false"` (case-insensitive) and `skipDryRun` is `false`, all outbound methods return a no-op stub (`{ ok: true, ... }`) without calling the Slack API. When `skipDryRun: true`, the instance always sends regardless of `DRY_RUN`.

For `sendMessage()`, if not in dry-run mode, a `:warning: *DRY RUN*` header block is prepended to any message that does not supply its own `blocks` array — making it visible in Slack that the portal is running in non-production mode.

#### Methods

##### `sendMessage(args)`

Posts a message to the default channel (`SLACK_NOTIFICATIONS_CHANNEL`).

- Optionally posts to additional channels supplied in `args.additionalChannels`. Failures on additional channels are tracked but do not fail the primary send.
- Threading: if `args.timestamp` is set, the message is posted as a reply (`thread_ts`). If additionally `args.respondInChannel` is `true`, `reply_broadcast: true` is added so the reply also appears in the main channel.
- Returns `SlackSendMessageResult` — extends `ChatPostMessageResponse` with:
  - `messageReferences: { channel, ts }[]` — references for all successfully posted messages (primary + additional channels).
  - `additionalChannelFailures: { channel, reason }[]` — channels that failed.

**Triggered by:** provisioning job completions, tenant lifecycle events, COGS alerts, and any other portal workflow that needs to broadcast to the default ops channel.

##### `updateMessage(reference, args)`

Edits a previously sent message in place using `chat.update`.

- `reference: { channel, ts }` identifies the message to update.
- `timestamp` and `respondInChannel` from `args` are ignored (not meaningful for updates).
- **Triggered by:** workflows that post an initial status message and then update it as steps progress (e.g., provisioning, ES resize approval flows).

##### `sendThreadReply(reference, args, broadcast)`

Posts a reply in the thread of an existing message using `chat.postMessage` with `thread_ts`.

- `reference: { channel, ts }` — the parent message to reply to.
- `broadcast: boolean` — if `true`, sets `reply_broadcast: true` so the reply surfaces in the channel as well.
- `additionalChannels` from `args` is stripped; thread replies go only to the reference channel.
- **Triggered by:** step-level updates in long-running jobs that should stay threaded under the initial notification.

##### `sendDirectMessage(channelId, args)`

Posts a message to an explicit channel ID rather than the default channel.

- Used when the target channel is known from the Slack interaction context (e.g., the channel where a slash command was invoked).
- **Triggered by:** commands route and interactions route when confirming EOL extensions back to the invoking channel.

#### `SlackNotificationArgs` interface

```typescript
interface SlackNotificationArgs {
  message: string             // Plain-text fallback and mrkdwn block content
  timestamp?: string          // thread_ts — reply to this message ts if set
  respondInChannel?: boolean  // reply_broadcast — also show thread reply in channel
  iconEmoji?: string          // Bot icon override, e.g. ":robot_face:"
  additionalChannels?: string[] // Extra channel IDs/names to fan out to (sendMessage only)
  blocks?: KnownBlock[]       // Custom Block Kit blocks; if omitted, a section block wrapping `message` is auto-built
}
```

### Auth Mechanism

Bot token (`xoxb-...`) passed to the `WebClient` constructor. Sourced from `SLACK_TOKEN`. The token must have the following Slack OAuth scopes (inferred from API calls made):

| Scope | Reason |
|---|---|
| `chat:write` | `chat.postMessage` and `chat.update` |

### Required Env Vars

| Variable | Purpose |
|---|---|
| `SLACK_TOKEN` | Slack bot token (`xoxb-...`) used to authenticate all outbound API calls |
| `SLACK_NOTIFICATIONS_CHANNEL` | Default channel ID/name for `sendMessage()`; also fallback for `SLACK_COGS_ALERT_CHANNEL` and `SLACK_ES_RESIZE_CHANNEL_ID` |
| `DRY_RUN` | When not `"false"`, outbound calls are suppressed (unless `skipDryRun: true`); a dry-run banner block is prepended to non-blocked messages |

#### Optional / Related Env Vars (configured in `Environment.tsx`, used alongside Slack)

| Variable | Purpose |
|---|---|
| `SLACK_COGS_ALERT_CHANNEL` | Dedicated channel for COGS/cost alerts; falls back to `SLACK_NOTIFICATIONS_CHANNEL` |
| `SLACK_ES_RESIZE_CHANNEL_ID` | Dedicated channel for Elasticsearch resize approval messages; falls back to `SLACK_NOTIFICATIONS_CHANNEL` |
| `NEXTAUTH_URL` | Used by routes and outbound helpers to build absolute portal URLs embedded in Slack messages |
