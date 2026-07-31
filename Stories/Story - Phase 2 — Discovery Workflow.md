**Working Docs:** [[Phase 2/Index|Phase 2 Docs]]

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

### Phase 2 Subtasks

All subtasks belong under the "Phase 2 — Discovery Workflow" story.

**Phase output:** A working scheduled job that produces an orphan inventory.

**Note:** The discovery script (hybrid approach: Resource Explorer + per-service fallbacks) already exists from Phase 1. Phase 2 is about turning that into a production-grade scheduled workflow with persistence, safety mechanisms, and error handling.

---

### Subtask 1

**Title:** Design decision — where should the workflow live and how should it be scheduled?

**Description:**

Evaluate where the discovery workflow should run and how it should be triggered. Options to consider:

- **Argo CronWorkflow** under `infra/saas/service-portal/ui/argo-workflows/` — runs inside K8s, aligns with existing team patterns
- **Lambda on EventBridge** — serverless, cheap, but separate deployment target
- **ECS scheduled task** — containerized, managed by AWS
- **Portal-side cron** — runs within the Ops Portal application itself

For each: evaluate operational complexity, team familiarity, credential management, logging/observability, and how it fits with existing infrastructure.

**Action:** Skim the Argo patterns section in the existing Argo library. Check which patterns fit naturally and what gaps you'd need to fill. See `argo-workflows/` in the Ops Portal repo.

**Acceptance criteria:**
- Options evaluated with tradeoffs documented
- Decision made and captured in decision log
- Chosen platform confirmed with mentor

---

### Subtask 2

**Title:** Design decision — how should AWS credentials be obtained?

**Description:**

Determine how the discovery workflow authenticates to AWS. The common pattern in existing workflows is IRSA (IAM Roles for Service Accounts) via a Kubernetes service account.

**Questions to answer:**
- Is IRSA the right fit here?
- The discovery job needs to scan across both AWS accounts — does IRSA support cross-account access, or do you need something different (e.g., AssumeRole)?
- What IAM permissions does the discovery job need? (Resource Explorer, EC2 describe, EKS describe, ELB describe, etc.)
- How do existing workflows in the Argo library handle credentials?

**Acceptance criteria:**
- Credential strategy documented
- IAM policy scoped to minimum required permissions
- Cross-account access approach confirmed

---

### Subtask 3

**Title:** Design decision — how should the inventory be persisted?

**Description:**

The discovery job produces an orphan inventory. Where should it be stored? The Portal UI (Phase 3) and cleanup workflow (Phase 4) both need to read from it.

**Options to weigh:**
- **S3** — simple, cheap, good for JSON/CSV dumps, but not queryable
- **DocumentDB** — the Portal's existing data store, queryable, familiar to the team. See `documentdbProxy/` and `app/_lib/`
- **Postgres table** — relational, queryable, but may not exist in the current stack
- **Portal's existing data store** — whatever the Portal already uses for similar long-running data

**Questions to answer:**
- What does the Portal already use for similar long-running data?
- Does DocumentDB fit, or do you need something else?
- What's the read pattern? (Portal UI polling, API endpoint, direct query)
- How much data? (estimate record count and size)
- Retention policy? (keep last N runs, keep all, TTL?)

**Acceptance criteria:**
- Storage option chosen with justification
- Schema/data model designed
- Decision captured in decision log

---

### Subtask 4

**Title:** Define the inventory record schema

**Description:**

Design what each inventory record needs to contain to be useful for the Portal UI (Phase 3) and the cleanup workflow (Phase 4).

**Minimum fields to consider:**
- Tenant name
- Customer name
- Tenant deletion date
- AWS account and region
- Resource type (EC2, EBS, ELB, ENI, S3, etc.)
- Resource ID (ARN or AWS resource ID)
- Tags present on the resource
- Estimated monthly cost
- Discovery run timestamp
- Attribution confidence (tagged vs inferred via fallback)
- Age of orphan (time since tenant deletion)

**Also consider:**
- How to represent untagged-but-suspected orphans
- How to flag resources that were found by the fallback layer vs Resource Explorer
- What metadata the cleanup workflow needs to execute a delete
- What the Portal UI needs for display and filtering

**Acceptance criteria:**
- Schema documented with field descriptions
- Minimum vs optional fields defined
- Schema reviewed with mentor

---

### Subtask 5

**Title:** Design decision — how to distinguish "orphan" from "legitimately still in use"

**Description:**

The most reliable signal: tenant is marked deleted in the Portal but resources tagged for that tenant still exist in AWS.

**Additional signals to evaluate:**
- Pulumi state — is there an active stack for this tenant? Is cross-referencing the state backend feasible, and does it add accuracy?
- Resource activity — is the resource actually being used? (e.g., network traffic on an ENI, reads/writes on an EBS volume)
- Time since deletion — resources found immediately after deletion might be mid-destroy, not orphaned
- Deletion pipeline status — did the destroy pipeline pass, fail, or block?

**Questions to answer:**
- What combination of signals gives you confidence that a resource is truly orphaned?
- How do you handle edge cases? (tenant deleted but re-provisioned, shared resources across tenants, resources created outside of Pulumi)
- What's the false positive tolerance?

**Acceptance criteria:**
- Orphan identification logic documented
- Edge cases identified and handled
- False positive strategy defined

---

### Subtask 6

**Title:** Design decision — how to handle racing destroys

**Description:**

A discovery run that fires while a destroy is mid-flight will produce false positives — it'll find resources that are about to be deleted.

**Safety mechanisms to evaluate:**
- **Tenant-keyed mutexes** — see `argo-workflows/docs/WORKFLOW_CONCURRENCY.md` for existing patterns
- **Maintenance window** — only run discovery during a known quiet period
- **Freshness check** — after discovery finds resources, re-check before flagging as orphan (e.g., "was this tenant's destroy pipeline still running when I found this?")
- **Grace period** — don't flag resources as orphaned until X days after tenant deletion

**Acceptance criteria:**
- Concurrency safety mechanism chosen
- Documented in design write-up
- Decision captured in decision log

---

### Subtask 7

**Title:** Implement the scheduled discovery workflow

**Description:**

Build the production workflow using the chosen platform (from Subtask 1). Integrate the existing discovery script into the workflow framework.

**The workflow should:**
1. Fetch the list of deleted tenants from the Portal/DocumentDB
2. For each tenant, run the hybrid discovery (Resource Explorer + fallbacks)
3. Attribute resources to tenants (tagged + inferred)
4. Calculate estimated cost per resource
5. Store the inventory in the chosen data store (from Subtask 3)
6. Handle errors gracefully (see Subtask 8)

**Reuse existing patterns** from the Argo library where they fit.

**Acceptance criteria:**
- Workflow runs successfully on schedule in dev
- Produces a complete orphan inventory
- Results persisted and queryable

---

### Subtask 8

**Title:** Implement error handling, retries, and notifications

**Description:**

Design how the workflow handles failure modes:

**Expected failure modes:**
- AWS API throttling
- IAM permission errors
- Partial-account scans (one region fails, others succeed)
- Network timeouts
- Data store write failures

**For each, decide:**
- Retry strategy (exponential backoff, max retries)
- Should partial results be saved or discarded?
- Should failures page someone or just log?
- How to surface which parts of the scan succeeded vs failed (coverage reporting)

**Also implement:**
- `continueOn: { failed: true }` for non-blocking failures where appropriate
- Notifications on workflow failure (Slack, email, or PagerDuty)
- Logging sufficient for debugging

**Acceptance criteria:**
- Error handling implemented for all expected failure modes
- Retry logic in place
- Failure notifications configured
- Partial failure doesn't lose successful results

---

### Subtask 9

**Title:** Evaluate agentic approach for untagged-resource attribution

**Description:**

The biggest agentic opportunity in the project: untagged-resource attribution.

The deterministic scan finds resources but can't always tell you whose they were if they're untagged. An agent reasoning over lineage can often guess:
- VPC → subnet → ENI → instance → volume
- LB → target group → K8s service annotations

**Evaluate:**
- Build the deterministic core first (already done)
- Can an agent layer on top for the "unattributable" bucket?
- How would you handle agreement / disagreement / low-confidence answers?
- What's the cost-per-run of the agent approach?
- How do you verify the agent's attributions?

**Capture in the decision log:**
- Did you try an agentic approach? What was the result?
- If not, why not? What would make it worth trying?

**Acceptance criteria:**
- Agentic attribution evaluated
- Decision documented in decision log
- If built: accuracy measured against known-tagged resources

---

### Subtask 10

**Title:** Phase 2 gate review — demo scheduled run against dev

**Description:**

Demonstrate the discovery workflow to mentor. Present:
- Design write-up covering all decisions (workflow platform, credentials, persistence, orphan identification, concurrency safety, error handling)
- Live demo of the scheduled workflow running against dev environment
- Inventory output showing discovered orphans with tenant attribution
- Manual spot-checks comparing inventory output against AWS Resource Explorer to verify accuracy

**Gate criteria:**
- Demo a scheduled run against dev
- Walk through design write-up and inventory output
- Manual spot-checks pass
- No destroys yet

**Acceptance criteria:**
- Gate review completed with mentor
- All spot-checks verified
- Phase 2 gate passed