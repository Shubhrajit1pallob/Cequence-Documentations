### Overview

Deleted tenants continue to incur AWS infrastructure costs due to orphaned resources that persist after tenant deletion. This project will build automated discovery, visibility, and approval-gated cleanup workflows to eliminate ongoing waste and prevent future resource leakage.

### What "Done" Looks Like

#### 1. Discovery Job

**Deliverable:** Scheduled job (cron/Lambda/K8s CronJob) that:

- Scans both AWS accounts for orphaned resources
- Attributes resources to originating tenant via tags
- Catches untagged resources in a fallback bucket
- Produces structured inventory (JSON/DB)

**Output:** Queryable inventory with tenant attribution and metadata

---

#### 2. Visibility - Ops Portal Integration

**Deliverable:** New "Orphans" tab in Ops Portal showing:

- Per-tenant orphaned resource list
- Cost estimate per resource/tenant
- Last-deleted-at timestamp
- Resource breakdown by type (EBS, ELB, ECR, etc.)

**Output:** UI component with filtering/sorting capabilities

---

#### 3. Approval-Gated Cleanup Workflow

**Deliverable:** Human-in-the-loop cleanup system with:

- Per-tenant batch approval requests
- Slack integration for approval notifications
- Portal UI for approval actions
- **No fully-automated destroys** - requires human confirmation
- Audit trail of all cleanup actions

**Output:** Workflow orchestration (Step Functions/Argo Workflows/custom)

---

#### 4. Monitoring & Alerting

**Deliverable:** Observability stack showing:

- Orphan count over time
- $ saved (cumulative and per-cleanup)
- Discovery workflow success/failure rates
- Coverage metrics (% of resources tagged/discovered)
- Alerts on workflow failures

**Output:** Grafana dashboards + alerting rules

---

#### 5. Root Cause Analysis

**Deliverable:** Written report identifying:

- Top 3-5 Pulumi leak patterns (e.g., missing `forceDestroy`, RDS snapshot retention, ENI cleanup failures)
- Why these patterns occur (code review + log analysis)
- 2-3 PRs to upstream Pulumi repos fixing common issues
    - Example: Add `forceDestroy: true` to resource defaults
    - Example: RDS snapshot lifecycle policies
    - Example: ENI cleanup improvements

**Output:**

- `root-cause-analysis.md`
- Pull requests to infrastructure repos (layers 0-5)

---

#### 6. Cost Reduction

**Deliverable:** Measurable monthly cost reduction

- Baseline: Current monthly cost from orphaned resources
- Target: Quantified $ reduction post-cleanup
- Tracking: Cost Explorer snapshots before/after

**Output:** Cost savings report with metrics

---

#### 7. Decision Log

**Deliverable:** Running markdown log in git tracking every non-trivial decision

**Location:** `service-portal/ui/features/orphaned-resources/decisions/`

**Format:**

markdown

```markdown
## YYYY-MM-DD — <decision title>

**Context:** what problem are you solving?

**Options considered:** 
- Option A: [brief pros/cons]
- Option B: [brief pros/cons]
- Option C: [brief pros/cons]

**AI / agentic angle:** 
- Did you try an agentic approach? Result?
- If you used AI: what prompt shape, how did you verify the output?
- If you didn't: why not? (capture why AI wasn't the right tool)

**Decision:** what you picked and why

**Open questions / next review:** anything to revisit later
```

**Output:** Git-tracked decision history in MRs