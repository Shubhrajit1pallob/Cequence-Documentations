---
status: IN REVIEW — submitted 2026-06-17
updated: 2026-06-17
---

# Agentic Attribution Evaluation (S9 / PLAT-1650)

**Status: IN REVIEW**

Related: [[Discovery job implementation]] (S7) · [[Orphan identification logic]] (S5) · [[Phase 2/Index|Phase 2 Docs]]

---

## The question

Can an LLM agent attribute resources in the discovery job's blind spot — resources that are neither tagged with a tenant name nor reachable from a known structural anchor — and is it worth building?

---

## What the current job covers

`src/discovery/supplementaryDiscover.ts` walks from four anchor types:

| Anchor | Resources walked |
|---|---|
| Tagged VPC | Subnets, route tables, security groups, IGWs, NACLs, NAT gateways, EIPs, launch templates |
| Tagged EKS cluster | Node groups |
| ASG name prefix | ASGs whose name starts with the tenant name |
| Tagged CFN stack | All child resources whose physical ID is an ARN |

---

## The blind spot

Resources that are **not tagged** AND **not reachable from any anchor above**:

| Resource type | Blind spot? | Naming pattern | Cost if orphaned |
|---|---|---|---|
| ALB / NLB | ✅ Yes — VPC walked but LBs not described | Strong (`<tenant>-alb`, `<env>-<tenant>-lb`) | **HIGH** |
| ElastiCache (Redis) | ✅ Yes — in tenant VPC, no describe step | Strong (`<tenant>-redis`, `<tenant>-cache`) | **MEDIUM-HIGH** |
| CloudFront distributions | ✅ Yes — global, no VPC anchor | Moderate (origin URL contains tenant name) | **MEDIUM** |
| S3 buckets (Pulumi-direct) | ✅ Yes — created outside CFN | Strong (`<tenant>-<purpose>`) | **VARIABLE** |
| IAM roles (Pulumi-direct) | ✅ Yes — not inside a CFN stack | Moderate (`<tenant>-<role-name>`) | **ZERO** (security/hygiene only) |
| ACM certificates | ✅ Yes — global, no VPC anchor | Strong (domain contains tenant name) | **ZERO** |
| Route53 hosted zones | ✅ Yes — global, no VPC anchor | Strong (zone name IS the tenant domain) | **LOW** |
| Secrets Manager replicas | ✅ Yes — known Category #3 orphan | n/a | **LOW** |

**Key observation:** ALB/NLB and ElastiCache — the two highest-cost blind spots — are both VPC-scoped. They are deterministically solvable by adding `DescribeLoadBalancers` (ELBv2) and `DescribeCacheClusters` (ElastiCache) to the existing VPC anchor-walking loop in `supplementaryDiscover.ts`. No LLM needed for those.

---

## What the agentic approach would look like

For resources that remain after the deterministic expansion:

**Inputs per resource:** ARN, type, name, region, account ID, creation timestamp, partial tags, related ARNs where discoverable

**Prompt shape:** batch resources by type, supply the list of deleted tenants + deletion dates, ask for attribution with confidence level and reasoning. Return JSON: `{ arn, tenant|"none", confidence: "high|medium|low", reasoning }`.

**Confidence handling:**
- `high` → include in inventory automatically
- `medium` → include with `needsReview: true` flag for Phase 4 human gate
- `low` → discard, log to stdout for operator awareness

**Cost per run:** Negligible at any reasonable model price given the volumes involved (tens of resources per run in dev, hundreds in prod). LLM cost is not a factor in this decision.

**Accuracy measurement:** Run agent against resources that ARE tagged (ground truth known), strip their tags, measure true positive / false positive / false negative rates. Target: >90% true positive, <5% wrong-tenant attribution before trusting in production.

---

## Recommendation: Defer — expand deterministic layer first

The right next step is **not** an LLM agent. It is adding `DescribeLoadBalancers` and `DescribeCacheClusters` to the VPC anchor-walking loop — a deterministic, ~30-line addition with zero false-positive risk that closes the two highest-cost blind spots.

After that expansion, the remaining blind spot consists almost entirely of free or near-free resources (IAM roles, ACM certs, Route53 zones, Secrets Manager replicas). There is no cost justification for an agentic layer targeting those.

The agentic approach becomes worth revisiting when:
1. The deterministic VPC expansion is done (prerequisite)
2. Real production runs exist (2–4 weeks of data)
3. That data shows the remaining blind spot is generating meaningful orphan spend that the deterministic layer cannot reach

Until those conditions are met, the added complexity and false-positive burden of an LLM layer is not justified.

---

## Decision

**Defer to Phase 3 or later.**

Prerequisite action item: add ALB/NLB (`DescribeLoadBalancers`) and ElastiCache (`DescribeCacheClusters`) to the existing VPC anchor-walking loop in `supplementaryDiscover.ts`. This is a deterministic fix and should be tracked as a separate task.

See also: [[Decision Logs]] entry 2026-06-17 — Agentic attribution evaluation.
