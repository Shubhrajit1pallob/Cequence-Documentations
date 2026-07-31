# Tagging Coverage Audit (Subtask 1)

**Status: COMPLETE** — dev account only (545681961293); built from `run.sh collect` live run (2026-06-03, 19 tenants) + Phase 0 investigation evidence + Subtask 4 coverage matrix. Prod account (552447114887) is out of scope — no access.

Feeds the recommendation in [[Discovery approaches — design doc]].

---

## Scope & method

| Field | Value |
|---|---|
| Account | dev (545681961293) only — prod out of scope |
| Tenants | 19 pre-identified orphan candidates from `Dev_tennant_orphaned_resources.csv` |
| Run date | 2026-06-03 |
| Data source | `run.sh collect` live run (363 tagged + 16 supplementary = 379 total) |
| Tag keys in use | `Tenant` (confirmed — RE query matches on `tag.value:<tenant>`; key is not enforced, any tag whose *value* equals the tenant name matches) |
| RE echo caveat | Terminal-state compute entries (e.g. the 31 stale Karpenter rows in [[pipeline-005-shub-test-dev]]) excluded — these are index artefacts, not live resources |
| Denominator note | "Total" counts below are scoped to the 19 orphan tenants in the run, **not** a full account scan. Account-wide totals would require per-service describe calls across all regions — noted as a follow-up if needed |

---

## Coverage table — dev account, 19 orphan tenants

For each resource type, **Found by RE** = tagged (primary layer); **Found by fallback** = supplementary (untagged, relationship-discovered). Source: [[Hybrid discovery script]] coverage matrix + live run.

| Resource type | Found by RE | Found by fallback | Tagging status | Notes |
|---|---|---|---|---|
| secretsmanager:secret | Yes — all instances | No fallback | **Well-tagged** | All defender secrets found by tag query across all regions |
| iam:instance-profile | Yes | No | **Well-tagged** | Found by tag query |
| iam:policy (worker-role-policy-ecr-*) | Yes | No | **Well-tagged** | Found by tag query |
| iam:policy (KarpenterControllerPolicy) | No | Yes — CF stack traversal | **Systematically untagged** | Created inside CFN stack; not tagged with tenant name; **6 instances missed by RE** across 3 tenants |
| iam:role | Yes (if tagged) | Partial — CF stack traversal | Partially untagged | Standalone untagged roles not found |
| iam:oidc-provider | Yes | No | Well-tagged | Found by tag query |
| ec2:vpc | Yes | No | **Well-tagged** | Found by tag query |
| ec2:subnet | Yes (if tagged) | Yes — VPC walk | Partially untagged | VPC walk always runs; tagged ones also appear in RE |
| ec2:route-table | Yes (if tagged) | Yes — VPC walk | **Systematically untagged** | **5 instances found only by fallback** across 4 tenants — not tagged |
| ec2:security-group | Yes (if tagged) | Yes — VPC walk | **Systematically untagged** | **5 instances found only by fallback** across 2 tenants — not tagged |
| ec2:internet-gateway | Yes (if tagged) | Yes — VPC walk | Partially untagged | |
| ec2:network-acl | Yes (if tagged) | Yes — VPC walk | Partially untagged | |
| ec2:natgateway | Yes (if tagged) | Yes — VPC walk | Partially untagged | High cost driver (~$32–45/mo each) |
| ec2:elastic-ip | Yes (if tagged) | Yes — extracted from NAT response | Partially untagged | |
| ec2:volume (EBS) | Yes | No | Well-tagged | Found by tag query |
| ec2:launch-template | Yes (if tagged) | Yes — name-contains filter | Partially untagged | Name-based attribution; false-positive risk on short tenant names |
| ec2:instance | Yes (if tagged) | **No fallback** | **Gap — may miss untagged** | Karpenter-launched instances may not match tag query; no fallback exists |
| ec2:fleet | Yes (if tagged) | **No fallback** | **Gap** | No fallback discovery path |
| ec2:spot-instances-request | Yes (if tagged) | **No fallback** | **Gap** | No fallback |
| ec2:network-interface (ENI) | Yes (if tagged) | **No fallback** | **Gap** | ENIs from Lambda/ELB/EKS typically untagged; no fallback |
| eks:cluster | Yes | No | Well-tagged (when tagged) | If untagged, cluster + all child node groups missed entirely |
| eks:nodegroup | No | Yes — EKS traversal | **Systematically untagged** | Not in RE; always found via cluster traversal |
| autoscaling:autoScalingGroup | No | Yes — name-starts-with | Systematically untagged by RE | ASG not indexed by RE; found via name filter |
| cloudformation:stack | Yes | No | **Well-tagged** | Found by tag query |
| acm:certificate | Yes | No | Well-tagged | Found by tag query |
| elasticloadbalancing:loadbalancer | Yes | No | Well-tagged | Found by tag query |
| elasticloadbalancing:targetgroup | Yes | No | Well-tagged | Found by tag query |
| elasticloadbalancing:listener | Yes | No | Well-tagged | Found by tag query |
| events:rule | Yes | No | Well-tagged | Found by tag query |
| logs:log-group | Yes | No | Well-tagged | Found by tag query |
| sqs:queue | Yes | No | Well-tagged | Found by tag query |
| route53:healthcheck | Yes | No | Well-tagged | Found by tag query |

---

## Systematically untagged resource classes

These are resource types that are **never** (or rarely) tagged with the tenant name by design — they get created as children of other resources and inherit no tag:

| Class | Why untagged | How found | Count (this run) |
|---|---|---|---|
| `iam:policy` (KarpenterControllerPolicy) | Created inside a CFN stack; not given a tenant-value tag | CF stack traversal | 6 across 3 tenants |
| `ec2:route-table` | VPC child resource; Pulumi does not tag route tables with tenant name | VPC walk | 5 across 4 tenants |
| `ec2:security-group` (non-default) | VPC child resource; not tagged | VPC walk | 5 across 2 tenants |
| `eks:nodegroup` | Child of EKS cluster; not independently tagged | EKS cluster traversal | found via traversal (count varies) |
| `autoscaling:autoScalingGroup` | Not indexed by RE at all | Name-starts-with filter | found via name match |

**Suspected but not confirmed** (known script gaps — no fallback exists):
- `ec2:instance` (Karpenter-launched) — may or may not match tag query depending on Karpenter tag configuration
- `ec2:network-interface` (ENIs from ELB/EKS) — typically untagged, no fallback
- `ec2:fleet` — no fallback

---

## Run summary

| Metric | Value |
|---|---|
| Total resources found | 379 |
| Found by RE (tagged, primary layer) | 363 (95.8%) |
| Found by fallback (untagged, supplementary) | 16 (4.2%) |
| Tenants with fallback resources | 5 of 19 (26%) |
| Fallback resource types | ec2:route-table, ec2:security-group, iam:policy |

---

## Limitations

- **Dev account only** — prod account (552447114887) out of scope; no access
- **19 orphan-candidate tenants only** — not a full account scan; active tenants excluded
- **Denominator is scoped, not account-wide** — counts reflect the 19 tenants in the run, not all resources in the account
- **RE terminal-state echoes excluded** — stale compute rows (ec2:instance, ec2:fleet) that RE returns for deleted resources are not counted
- **Tag-key casing** — RE matches on tag *value* (any key whose value = tenant name); formal tag-key enforcement (`Customer`, `Tenant`, `Env`) not independently verified against live resources in this run

---

## What this means for the hybrid approach

The live run confirms that **RE alone would miss 16 resources (4.2%)** across 19 tenants — concentrated in 3 resource types that are systematically untagged by design: KarpenterControllerPolicy IAM policies (created inside CFN stacks and never given a tenant-value tag), and VPC-child route tables and security groups (provisioned by Pulumi without a tenant tag). These are not edge cases — they appear in 5 of 19 tenants (26%) and would be silently missed by any tag-only discovery approach. The fallback layer exists specifically for these classes and its contribution is confirmed by evidence. This directly justifies the hybrid approach over RE-alone as the discovery strategy for Phase 2.
