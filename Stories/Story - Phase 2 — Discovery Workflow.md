**Title:** Phase 2 — Discovery Workflow

**Description:**

Build the working scheduled job that produces an orphan inventory.

Key design questions to answer:

- Where should the workflow live and how should it be scheduled?

(Argo CronWorkflow, Lambda on EventBridge, ECS scheduled task, Portal-side cron)

- Which existing Argo library patterns are worth reusing vs writing fresh?
- How should AWS credentials be obtained? (IRSA via service account is common)
- How should the inventory be persisted? (S3, DocumentDB, Postgres, Portal'sexisting data store)

- How will discovery distinguish "orphan" from "legitimately still in use"?
- How to handle racing destroys?
- What does the inventory record need to contain?
- Error handling, retries, and notifications.

**AI lens:** The biggest agentic opportunity is untagged-resource attribution.

Build the deterministic core first; layer an agent on top for the

"unattributable" bucket.

**Gate criteria:**

- Demo a scheduled run against dev
- Walk through design write-up and inventory output
- Manual spot-checks against AWS Resource Explorer
- No destroys yet

**Documentation:**

- `service-portal/ui/features/orphaned-resources/phase-2/discovery-workflow-design.md`

**Labels:** `coop-2026`