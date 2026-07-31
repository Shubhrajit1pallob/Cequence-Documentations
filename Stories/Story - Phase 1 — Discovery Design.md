**Working Docs:** [[Phase 1/Index|Phase 1 Docs]]

**Title:** Phase 1 — Discovery Design

**Description:**

Design the discovery approach. This phase is design-only — no building.

Produce a written design doc with 2–3 discovery approaches and their tradeoffs for review.

**Key activities:**

- Audit AWS tagging coverage in both accounts. Quantify % of resources with Customer + Tenant + Env tags, broken out by service.
- Bring 2–3 discovery approaches to the design review as a written doc. Starting points to evaluate:
  - AWS Resource Explorer (tag-indexed; simplest)
  - AWS Config aggregator (authoritative; heavier)
  - Cost Explorer + per-service SDK drill-down (cost-first)
  - Hybrid (Resource Explorer + per-service fallbacks for untagged classes)
  - For each option: evaluate pros/cons, IAM scope, cost, time-to-implement, coverage of untagged resources.
- AI lens: evaluate whether discovery could be agentic instead of a deterministic scan. Capture the reasoning in the decision log.

**Gate criteria:**

- Design review with mentor and one senior engineer
- One approach picked
- ADR-style write-up documenting the decision and the "why"

**Documentation:**

- `service-portal/ui/features/orphaned-resources/phase-1/discovery-design.md`

**Labels:** `coop-2026`

### Phase 1 Subtasks

All subtasks belong under the "Phase 1 — Discovery Design" story.

---

### Subtask 1

**Title:** Audit AWS tagging coverage across both accounts

**Description:**

Quantify how well resources are tagged with `Customer` + `Tenant` + `Env` across both AWS accounts, broken out by service.

For each service type (EC2, ELB, EBS, ENI, S3, RDS, EKS, NAT Gateway, VPC, etc.):
- Total resource count
- Count with all three tags (Customer + Tenant + Env)
- Count with partial tags
- Count with no tags
- Tagging coverage percentage

The gaps found here justify the fallback layer in the hybrid approach and identify where future tagging improvements are needed.

**Note:** The existing discovery script already revealed gaps (26 untagged resources found by the fallback layer that Resource Explorer missed). This subtask formalizes that into a quantified report.

**Acceptance criteria:**
- Tagging coverage report by service type for both accounts
- Clear identification of systematically untagged resource classes
- Data supports the discovery approach recommendation

---

### Subtask 2

**Title:** Document 2–3 discovery approaches with tradeoff analysis

**Description:**

Write a design doc evaluating 2–3 discovery approaches. For each option, document: pros/cons, IAM scope required, cost, time-to-implement, and coverage of untagged resources.

**Approaches to evaluate (pick what makes sense, propose your own):**

1. **AWS Resource Explorer** (tag-indexed; simplest)
   - Pros: fast, low IAM scope, simple to implement
   - Cons: blind to untagged resources

2. **AWS Config aggregator** (authoritative; heavier)
   - Pros: authoritative resource inventory, covers everything
   - Cons: heavier setup, higher cost, more IAM permissions

3. **Cost Explorer + per-service SDK drill-down** (cost-first)
   - Pros: directly tied to cost, answers "what's costing money"
   - Cons: cost attribution lag, may not identify specific resources

4. **Hybrid: Resource Explorer + per-service fallbacks** (already built)
   - Pros: covers both tagged and untagged, evidence-backed (26 resources
     found by fallbacks that RE missed)
   - Cons: need to maintain fallback logic per service, may miss new
     resource types

**Note:** A working hybrid implementation already exists. The design doc should present it alongside alternatives with data from the tagging audit to justify the choice.

**Acceptance criteria:**
- Written design doc with 2–3 approaches
- Each option has pros/cons, IAM scope, cost, time-to-implement, untagged coverage
- Tagging audit data used to support the recommendation

**Documentation:**
- `service-portal/ui/features/orphaned-resources/phase-1/discovery-design.md`

---

### Subtask 3

**Title:** Evaluate agentic discovery as an alternative approach

**Description:**

Add a fourth dimension to the tradeoff analysis: could discovery be agentic instead of deterministic?

**Evaluate:**
- An agent that plans which APIs to call per tenant rather than enumerating exhaustively
- Pros: handles novel tag schemes, reasons about resource lineage (VPC → subnet → ENI → instance → volume), potentially faster/cheaper
- Cons: non-deterministic (different results each run), harder to audit, cost-per-run (LLM API calls), false negatives, no replay-ability

**Capture in the decision log:**
- Which signals led you to pick deterministic vs agentic
- What test or evidence would change your mind
- Where agentic might layer on top (e.g., untagged resource attribution)

**Note:** The supervisor's guidance suggests building the deterministic core first and potentially layering an agent on top for the "unattributable" bucket. This subtask is about evaluating and documenting the reasoning, not necessarily building an agent.

**Acceptance criteria:**
- Agentic approach evaluated with pros/cons in the design doc
- Decision log entry capturing the deterministic vs agentic reasoning
- Clear statement on what would change the decision

---

### Subtask 4

**Title:** Document the hybrid discovery script — what it does and what it covers

**Description:**

Document the existing discovery script implementation:

**Primary layer — Resource Explorer:**
- Uses `tag.value:<tenant>` to find all tagged resources
- Covers resources properly tagged with tenant name

**Fallback layer — per-service SDK calls:**
- VPC → subnets, route tables, security groups, IGWs, NAT gateways, EIPs
- EKS cluster → node groups
- CloudFormation stack → all resources it created
- Tenant name → ASGs, launch templates

**Document:**
- What resource types are covered by each layer
- What resource types are NOT yet covered (known gaps)
- How the script attributes resources to tenants
- How it handles untagged resources
- Performance characteristics (runtime, API call count)
- The 26 previously-missing resources found by the fallback layer

**Acceptance criteria:**
- Script functionality documented
- Coverage matrix showing which resource types are found by which layer
- Known gaps identified

---

### Subtask 5

**Title:** Write ADR for discovery approach decision

**Description:**

After the design review with mentor and senior engineer, document the chosen approach as an Architecture Decision Record (ADR).

**ADR format:**

```
# ADR: Discovery Approach for Orphaned Resources

## Status
[Proposed / Accepted / Superseded]

## Context
[What problem are we solving? What constraints exist?]

## Options Considered
[List each approach with brief summary]

## Decision
[Which approach was chosen and why]

## Consequences
[Pros and cons of the chosen approach]
[What this means for future phases]

## Evidence
[Tagging audit data, script results, 26 untagged resources finding]
```

**Acceptance criteria:**
- ADR written and committed to git
- Captures the reasoning so future team members understand the "why"
- Reviewed by mentor and senior engineer

**Documentation:**
- `service-portal/ui/features/orphaned-resources/phase-1/adr-discovery-approach.md`

---

### Subtask 6

**Title:** Phase 1 gate review — design review with mentor and senior engineer

**Description:**

Schedule and present the design review. Bring:
- Tagging audit results
- Design doc with 2–3 approaches and tradeoffs
- Agentic vs deterministic evaluation
- Existing script documentation and evidence (26 untagged resources)
- Recommendation with supporting data

Pick one approach with the reviewers. Write the ADR afterward.

**Acceptance criteria:**
- Design review completed with mentor + one senior engineer
- One approach picked and agreed upon
- ADR written documenting the decision
- Phase 1 gate passed
