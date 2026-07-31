# Discovery Approaches — Design Doc (Subtasks 2 + 3)

**Status: COMPLETE** — all 4 approaches evaluated + agentic dimension + recommendation. Built from [[Tagging coverage audit]] (2026-06-03) + [[Hybrid discovery script]] evidence. Official deliverable: `service-portal/ui/features/orphaned-resources/phase-1/discovery-design.md` (copy when ready for design review).

---

## Background

Cequence's tenant lifecycle leaves orphaned AWS resources behind when destroy pipelines fail, partially succeed, or silently skip steps (see [[failure-mode-analysis]]). An independent discovery job is required — the portal's own workflow status cannot be trusted as evidence of clean teardown (categories #2 and #3 both report SUCCESS while billing real money). This document evaluates the approaches for building that discovery job.

**Constraints going in:**
- Must be multi-region (defender secrets replicate across ~30 regions — [[pipeline-004-life360-saas-prod]])
- Must cover untagged resources (4.2% of orphaned resources in dev are systematically untagged — [[Tagging coverage audit]])
- Dev account only for Phase 1 evaluation (prod access not available)
- A working implementation already exists (`scripts/orphaned-resources-AWS/run.sh collect`)

---

## Options evaluated

### Option 1 — AWS Resource Explorer (tag-indexed)

| Dimension | Detail |
|---|---|
| How it works | Single paginated `resource-explorer-2:SearchCommand` with `tag.value:<tenant>`; returns all tagged resources across all regions in one call |
| Pros | Fast (1 API call per page); low IAM scope (`resource-explorer-2:Search` only); simplest to implement; multi-region by default |
| Cons | **Blind to untagged resources** — misses 4.2% of orphaned resources confirmed in dev (16/379); returns terminal-state echoes (deleted compute still indexed); tag-key not enforced (matches on value only) |
| IAM scope | `resource-explorer-2:Search` on the aggregator view ARN |
| Cost | Free (Resource Explorer has no per-query charge) |
| Time-to-implement | ~1 day (already partially implemented as the primary layer) |
| Untagged coverage | **None** — by design blind to anything not tagged with the tenant name |

**Verdict:** Insufficient alone. Covers 95.8% of orphaned resources but the 4.2% it misses are systematically untagged by design (KarpenterControllerPolicy IAM policies, VPC-child route tables and security groups) — not random gaps.

---

### Option 2 — AWS Config aggregator

| Dimension | Detail |
|---|---|
| How it works | AWS Config records every resource change and maintains a full configuration history; a Config aggregator collects this across accounts and regions |
| Pros | Authoritative inventory — covers **everything** regardless of tags; historical state available; supports complex queries (relationship traversal, config rules) |
| Cons | Significant setup overhead (Config must be enabled per region per account, aggregator configured); **higher cost** (~$0.003/config item recorded + $0.0012/rule evaluation); Config items lag real-time by minutes; requires broader IAM scope; overkill for orphan detection |
| IAM scope | `config:SelectAggregateResourceConfig`, `config:ListAggregateDiscoveredResources`, plus read permissions on all resource types tracked |
| Cost | ~$0.003 per configuration item recorded — for a large account this can reach $10s–$100s/month depending on resource churn |
| Time-to-implement | 3–5 days (Config setup + aggregator + query layer) |
| Untagged coverage | **Full** — Config records all resources regardless of tags |

**Verdict:** Powerful but disproportionate. The gap RE leaves is 16 resources in 3 well-understood resource types — Config's overhead (cost, setup, IAM scope) is not justified to cover a known, bounded set of untagged types.

---

### Option 3 — Cost Explorer + per-service SDK drill-down

| Dimension | Detail |
|---|---|
| How it works | Query Cost Explorer for resources with cost attributed to deleted tenants; drill into specific services via SDK calls to identify what's billing |
| Pros | Directly answers "what's costing money"; ties discovery to business impact; no need to enumerate every resource type |
| Cons | **Cost attribution lag** (up to 24–48 hours); Cost Explorer granularity is at service level, not resource level — needs SDK drill-down to identify specific resources; some resource types have no CE attribution (IAM roles, CFN stacks, route tables); cannot find $0-cost orphans (e.g. category #1 IAM role) |
| IAM scope | `ce:GetCostAndUsage`, `ce:GetResourceCostAndUsage`, plus per-service describe permissions |
| Cost | Cost Explorer API: $0.01 per API request; manageable at low query frequency |
| Time-to-implement | 3–4 days (CE query layer + per-service SDK handlers for each billable type) |
| Untagged coverage | **Partial** — finds cost-bearing untagged resources only; $0-cost orphans (IAM roles, CFN stacks, route tables) invisible |

**Verdict:** Useful as a *complement* (cost validation layer) but not as the primary discovery mechanism. Misses zero-cost orphans entirely and has attribution lag that makes it unreliable for immediate detection.

---

### Option 4 — Hybrid: Resource Explorer + per-service fallbacks (incumbent)

| Dimension | Detail |
|---|---|
| How it works | RE primary layer for all tagged resources; targeted SDK fallbacks for known systematically untagged types (VPC walk → route tables/SGs/IGWs/NAT/EIPs; EKS traversal → node groups; CFN traversal → stack children; name-prefix → ASGs/launch templates) |
| Pros | **Already built and tested** (379 resources, 19 tenants, 1m1s runtime); covers tagged + known untagged classes; multi-region by design; evidence-backed (16 resources confirmed found only by fallback); low IAM scope; deduplicates by ARN |
| Cons | Fallback logic must be maintained per new untagged resource type; currently misses ec2:instance/fleet/ENI (no fallback for those); name-based ASG attribution is best-effort (false positive risk on short tenant names) |
| IAM scope | `resource-explorer-2:Search` + targeted EC2/EKS/CFN/ASG describe permissions per fallback |
| Cost | Free (RE) + negligible (Describe calls) |
| Time-to-implement | **Already implemented** — ~0 days for current scope; incremental for new fallback types |
| Untagged coverage | **Partial but evidence-backed** — covers 3 confirmed systematically untagged classes; known gaps documented in [[Hybrid discovery script]] |

**Verdict:** Recommended. Only approach with live evidence in this environment; covers all confirmed untagged classes; lowest implementation cost; already integrated into ops-portal scripts.

---

## Comparison summary

| | RE only | Config | Cost Explorer | **Hybrid (rec.)** |
|---|---|---|---|---|
| Tagged coverage | 95.8% | 100% | Partial | 100% |
| Untagged coverage | None | Full | Partial (cost-bearing only) | **Confirmed classes** |
| Multi-region | Yes | Yes | Yes | Yes |
| Already built | Partial | No | No | **Yes** |
| IAM scope | Minimal | Broad | Moderate | Minimal + targeted |
| Cost | Free | $10s–$100s/mo | Low | **Free** |
| Time to implement | ~1 day | 3–5 days | 3–4 days | **0 days** |
| Handles $0-cost orphans | No | Yes | **No** | Yes |

---

## Agentic dimension (Subtask 3)

Could discovery be **agentic** — an agent that reasons about which APIs to call per tenant — rather than a deterministic scan?

### Evaluated approach
An LLM agent receives a tenant name + account, reasons about resource lineage (VPC → subnet → ENI → instance → volume), plans which APIs to call, and builds the inventory dynamically rather than via a fixed script.

| Dimension | Detail |
|---|---|
| Pros | Handles novel/unknown tag schemes; reasons about resource relationships without pre-coded fallback logic; potentially extensible without code changes; could explain findings in natural language |
| Cons | **Non-deterministic** — different runs may return different results for the same tenant; **not auditable** — hard to prove completeness for compliance/financial purposes; **LLM cost per run** — 19 tenants × N API calls + token cost at scheduled frequency; **false negatives are silent** — no way to know what was missed; **no replay-ability** — can't reproduce a prior scan's exact result |

### Decision: deterministic core first

The signals that point away from agentic discovery now:
1. **Auditability requirement** — orphan remediation involves approval-gated deletion of real AWS resources; a non-deterministic inventory that can't be replayed or verified is a liability, not an asset
2. **Known scope** — the untagged resource classes are already identified and bounded (3 confirmed types); an agent's flexibility isn't needed when the problem is well-understood
3. **Evidence exists** — the hybrid script is proven against real data; an agentic approach has no baseline

### What would change the decision
- If a new class of systematically untagged resources emerges that doesn't fit the relationship-traversal model (i.e., no anchor to walk from)
- If the "unattributable" bucket (resources with no tag AND no VPC/CF/EKS anchor) grows to a significant cost impact

### Where agentic *could* layer on top
The supervisor's guidance is correct: **the unattributable bucket** is the right place for an agent. Once the deterministic scan runs and produces its inventory, any resource with no tag and no relationship anchor could be handed to an agent with the question "which deleted tenant does this likely belong to?" — using naming patterns, creation timestamps, and resource relationships as signals.

### Decision log
A decision log entry capturing this reasoning will be added to [[Decision Logs]] (the vault template mandates an "AI / agentic angle" section).

---

## Evidence from Phase 0

- **Green workflow ≠ clean teardown** — categories #2/#3 in [[failure-mode-analysis]] bill real money while reporting SUCCESS → independent discovery is required regardless of approach chosen
- **RE overcount anomaly** — terminal-state compute echoes confirmed in [[pipeline-005-shub-test-dev]] (31 stale Karpenter rows) and #003's `ec2:fleet` rows → any RE-based option needs live-state verification layer
- **Multi-region mandatory** — defender secret replicas survive in 16–17 regions after destroy ([[pipeline-004-life360-saas-prod]], [[pipeline-005-shub-test-dev]]) → discovery must scan all regions, not just the tenant's primary

---

## Recommendation

**Adopt the hybrid approach (Option 4)** — Resource Explorer as the primary layer, per-service SDK fallbacks for confirmed systematically untagged resource classes.

Rationale:
1. Already built and tested (379 resources / 19 tenants / 1m1s runtime)
2. The only approach with live evidence in this environment
3. Covers all 3 confirmed systematically untagged classes (KarpenterControllerPolicy, route tables, security groups)
4. Free to run, minimal IAM scope, zero additional setup
5. Config's full coverage and Cost Explorer's cost-first approach don't justify their overhead given the gap is small, well-understood, and already handled

**Phase 2 additions recommended:**
- Add live-state verification (`describe-instances`, `describe-fleets`) to filter RE terminal-state echoes
- Add fallback for `ec2:network-interface` (ENIs from ELB/EKS — currently a known gap)
- Layer an agentic "unattributable bucket" classifier on top once the deterministic core is running

Decision to be formalized in [[ADR — discovery approach]] after the design review.
