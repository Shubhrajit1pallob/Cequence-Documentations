# ADR — Discovery Approach (Subtask 5)

**Status: PROPOSED** — hybrid approach approved by supervisor (2026-06-03); pending formal sign-off at design review with mentor + senior engineer (Subtask 6). Official home: `service-portal/ui/features/orphaned-resources/phase-1/adr-discovery-approach.md`, pushed separately after review.

---

# ADR: Discovery Approach for Orphaned Resources

## Status

Proposed — pending design review sign-off

## Context

Cequence's tenant lifecycle leaves orphaned AWS resources behind after tenant deletion. The portal's own destroy workflow cannot be trusted as evidence of clean teardown — Phase 0 investigations confirmed that categories #2 and #3 both report `SUCCESS` while billing real money. An independent, scheduled discovery job is required to find orphaned resources across both AWS accounts.

The discovery job must:
- Find resources across all regions (defender secrets replicate to ~30 regions)
- Cover resources that are untagged or incompletely tagged
- Be auditable and reproducible (results must be explainable for approval-gated cleanup)
- Operate at low cost within the existing IAM footprint

A working implementation already exists: `scripts/orphaned-resources-AWS/run.sh collect`, used throughout Phase 0 to identify orphaned resources across 19 dev tenants.

## Options Considered

| Option | Summary |
|---|---|
| 1 — RE only | Single `resource-explorer-2:Search` query per tenant; fast and free but blind to systematically untagged resource types |
| 2 — AWS Config | Full authoritative inventory regardless of tags; $10s–$100s/month cost, 3–5 days setup, broad IAM scope |
| 3 — Cost Explorer + SDK | Cost-first; misses $0-cost orphans (IAM roles, CFN stacks, route tables) and has 24–48h attribution lag |
| 4 — Hybrid (RE + fallbacks) | RE primary layer for tagged resources; per-service SDK fallbacks for confirmed untagged classes; already built |

Full evaluation in [[Discovery approaches — design doc]].

## Decision

**Option 4 — Hybrid: Resource Explorer + per-service fallbacks.**

The existing `run.sh collect` script is approved as the discovery mechanism for Phase 2. It runs in two layers:

1. **Primary (Resource Explorer):** `tag.value:<tenant>` query returns all tagged resources across all regions in one call
2. **Fallback (per-service SDK):** Targeted describe calls for resources that are systematically untagged by design — VPC children (route tables, security groups, IGWs, NAT gateways, EIPs), EKS node groups, CloudFormation stack children, and ASGs/launch templates via name prefix

## Consequences

**Pros:**
- Already built, tested, and proven — 379 resources across 19 dev tenants, 1m1s runtime
- Free to run (RE + Describe calls have no meaningful cost)
- Minimal IAM scope — `resource-explorer-2:Search` + targeted EC2/EKS/CFN/ASG describe permissions
- Covers all 3 confirmed systematically untagged classes (KarpenterControllerPolicy IAM policies, route tables, security groups)
- Multi-region by design
- Deterministic and auditable — same input produces same output, results are explainable

**Cons / known limitations:**
- Fallback logic must be maintained as new untagged resource types are discovered
- Currently misses `ec2:instance`, `ec2:fleet`, `ec2:network-interface` (no fallback for these)
- Name-based ASG attribution is best-effort — false positive risk on short tenant names
- Dev account only validated in Phase 1 (prod access not available)

**What this means for future phases:**
- Phase 2 builds the scheduled discovery job on top of this script
- Recommended Phase 2 additions: live-state verification for RE echoes, ENI fallback
- Agentic layer deferred — an LLM agent may be layered on the "unattributable bucket" (resources with no tag and no relationship anchor) once the deterministic core is running

## Evidence

- **Tagging audit** ([[Tagging coverage audit]]): 16/379 resources (4.2%) found only by fallback across 5 of 19 tenants; 3 systematically untagged classes confirmed — `ec2:route-table` (5), `ec2:security-group` (5), `iam:policy/KarpenterControllerPolicy` (6)
- **Live run** ([[Hybrid discovery script]]): `run.sh collect` against 19 dev tenants, 2026-06-03 — 363 tagged + 16 supplementary = 379 total, 1m1s runtime
- **Phase 0 investigations**: Green workflow ≠ clean teardown (categories #2/#3 in [[failure-mode-analysis]]); RE returns terminal-state echoes ([[pipeline-005-shub-test-dev]]); replicas survive in 16–17 regions ([[pipeline-004-life360-saas-prod]])
