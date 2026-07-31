# Phase 0 — Failure Mode Analysis
Orphaned Resources Project · Dev Environment
Account: 545681961293 · Tenants reviewed: 19

> **Note on category numbering:** the category numbers in this document are local to this deliverable. **Category 1** corresponds to investigation [[pipeline-001-testreportfeature-dev]]; **Category 2** corresponds to investigations [[pipeline-003-api-security-ng1-dev]], [[pipeline-004-life360-saas-prod]], and [[pipeline-005-shub-test-dev]]. A third documented failure mode — a silent `fileExists`-skip orphan — is recorded separately in [[pipeline-002-anirudh843-dev]] and is intentionally **not** included in this deliverable.

## Executive Summary

19 deleted dev tenants were reviewed by reading historical GitLab destroy-pipeline logs, parsing Pulumi destroy artifacts, and confirming residual resources in the AWS Console and Resource Explorer. Two distinct orphan-producing failure modes were confirmed: a CloudFormation delete-retry limitation that strands a zero-cost Karpenter IAM role, and a fleet-wide multi-region secret-replica leak that bills recurring cost and leaves stale credentials across regions. Both can occur even when the portal/pipeline reports the destroy as successful, so neither raises an operational alert.

## Methodology

- Source: Historical GitLab destroy pipeline logs (failed pipelines in cluster, cluster-services, and tenant repos)
- Tenant JSON documents pulled from ops-portal (`destroyStatus` field)
- Pulumi artifact logs (`pulumi-destroy-<tenant>.log`) parsed for errors
- AWS Resource Explorer and CloudFormation Console used for confirmation
- Orphaned resource collector script run across 19 dev tenants
- Time period: tenant destroys spanning **2026-03-25 → 2026-06-01**; investigations conducted late May – early June 2026

## Failure Categories

### Category 1 — Karpenter CloudFormation no-retry → orphaned IAM role

| Field | Detail |
|---|---|
| Confirmed tenants | `testreportfeature-dev` (confirmed); `urvashi-sentiel-dev` (suspected, DELETE_FAILED 2026-04-29) |
| Layer affected | Layer 1 (cluster) |
| Repo | cluster |
| Destroy status shown | FAILURE — destroy job errors; CFN stack stuck in DELETE_FAILED |
| Detected by | Pipeline log + AWS CloudFormation Console |

**What happens:**
During the cluster destroy, the Karpenter CloudFormation stack tries to delete its IAM node role before detaching it from the instance profile, hits a 409 ("must remove roles from instance profile first"), then deletes the instance profile but never goes back to retry the role within the same operation. The stack is left in DELETE_FAILED with the IAM role still present. 8 of 9 stack resources delete successfully; only `KarpenterNodeRole` remains.

**Root cause:**
A known **CloudFormation limitation** — CFN does not retry a failed resource deletion within the same delete operation, so once the role delete fails it is not reattempted after the instance-profile blocker clears. Not a deletion-ordering bug. The role and instance profile are declared together in one CFN template and instantiated as a single `aws.cloudformation.Stack` via `pulumi-k8s/src/karpenter/karpenterInfra.ts`, so Pulumi has no control over CFN's internal retry behaviour.

**Resources orphaned:**

| Resource type | Example ID | Still exists? |
|---|---|---|
| `iam:role` | `Kptr-testreportfeature-dev-cluster` | Yes — orphaned |
| `cloudformation:stack` | `testreportfeature-dev-cluster-karpenterInfra` | Yes — stuck DELETE_FAILED |

**Estimated cost impact:**
**$0/month** — an IAM role and a stuck CloudFormation stack do not bill. Cleared manually via CloudFormation "Retry delete" once the blocker is gone.

---

### Category 2 — Multi-region defender-secret replica orphan

| Field | Detail |
|---|---|
| Confirmed tenants | `api-security-ng1-dev` (dev); `shub-test-dev` (dev, account 545681961293 — internal test tenant); `life360-saas-prod` (**prod**, account 552447114887 — manual investigation, outside the dev scan) |
| Layer affected | Layer 3 (tenant) |
| Repo | tenant |
| Destroy status shown | SUCCESS — genuinely clean, 0 errors across all stacks |
| Detected by | Collector script + AWS Resource Explorer + Pulumi destroy artifacts |

**What happens:**
Every tenant's "defender" secret in AWS Secrets Manager is created replicated to all enabled AWS regions (~30). On destroy, the destroy runs cleanly and reports SUCCESS with zero errors, but only the primary secret (and sometimes a subset of replicas) is removed — the remaining replicas survive as standalone secrets in their regions. Because the primary is gone, AWS then refuses to delete those replicas ("operation not permitted on a replica secret"), leaving them stranded and undeletable by normal means. Teardown can be partial and inconsistent across regions.

**Root cause:**
**Tenant IaC defect** (not ops-portal orchestration). `tenant/src/defender.ts:111-159` creates the secret `saas/<tenant>/defender` with `recoveryWindowInDays: 0` and `replicas: aws.getRegionsOutput()` (every enabled region), built inside a `pulumi.all().apply()` callback (a resources-in-apply anti-pattern). `tenant.ts:260-273` instantiates `Defender` unconditionally, so every tenant gets the multi-region secret and every destroyed tenant strands replicas fleet-wide.

**Resources orphaned:**

| Resource type | Example ID | Still exists? |
|---|---|---|
| `secretsmanager:secret` (replica) | `saas/life360-saas/defender-DRhi6w` | Yes — 30 created → 13 removed → **17 orphaned** (life360-saas-prod) |
| `secretsmanager:secret` (replica) | `saas/shub-test/defender-RLG5qK` | Yes — 30 created → 14 removed → **16 orphaned** (shub-test-dev) |
| `secretsmanager:secret` (replica) | `saas/api-security-ng1/defender-xVj1TG` | Yes — 30 created; survivor count not cost-verified (~17 visible) |

**Estimated cost impact:**
**$6.80/month confirmed** for life360-saas-prod (17 surviving replicas × $0.40), matching the tenant's post-deletion daily-cost data; **~$6.40/month estimated** for shub-test-dev (16 surviving replicas × $0.40, not independently cost-verified); **up to ~$12/month** per tenant if all 30 replicas survive. Plus a **security impact** — each orphan is a stale traffic-ingestion credential left live across many regions worldwide (up to 30 per tenant).

---

## Orphaned Resource Types — Full List

> This is the account-wide list produced by the collector script across the 19 dev tenants. It is **broader than the two categories detailed above** — several of the networking types below were chiefly evidenced by the separately-documented silent-skip case ([[pipeline-002-anirudh843-dev]]).

### Secrets Manager
- `secretsmanager:secret` — Defender replica secrets left across multiple AWS regions after tenant destroy. Pattern: `saas/<tenant>/defender-<suffix>`. Found in 17+ regions per tenant. Most widespread finding.

### Networking (EC2/VPC layer)
- `ec2:vpc` — Entire VPC left running
- `ec2:subnet` — Public and private subnets
- `ec2:natgateway` — NAT Gateway (primary cost driver, ~$32-45/month each)
- `ec2:elastic-ip` — Elastic IPs (~$3.60/month each when unattached)
- `ec2:internet-gateway` — Internet Gateway
- `ec2:route-table` — Route tables (public and private)
- `ec2:security-group` — Security groups
- `ec2:launch-template` — Karpenter EC2 launch templates
- `ec2:volume` — EBS volumes

### Load Balancing (Defender/ASG layer)
- `elasticloadbalancing:loadbalancer/app` — Application Load Balancer
- `elasticloadbalancing:listener/app` — ALB listener
- `elasticloadbalancing:targetgroup` — ALB target group

### DNS / Certificates
- `acm:certificate` — ACM SSL/TLS certificates
- `route53:healthcheck` — Route53 health checks

### Infrastructure / Orchestration
- `cloudformation:stack` — Karpenter CFN stacks stuck in DELETE_FAILED
- `events:rule` — EventBridge rules (Karpenter interruption handling)
- `sqs:queue` — Karpenter interruption queue

### IAM
- `iam:instance-profile` — EC2 instance profiles
- `iam:policy` — IAM managed policies (ECR access, Karpenter controller)

### Observability
- `logs:log-group` — CloudWatch log groups (EKS cluster logs)

## Summary Table

| Category | Tenants confirmed | Resources per tenant | Est. monthly cost | Priority |
|---|---|---|---|---|
| 1 — CFN no-retry IAM role | testreportfeature-dev (+1 suspected) | 1 IAM role + stuck CFN stack | $0 | Low |
| 2 — Multi-region secret replica | api-security-ng1-dev, shub-test-dev, life360-saas-prod | up to ~30 replica secrets (16–17 confirmed orphaned) | $6.40–12 | High |

## Scope

- Environment reviewed: dev (account 545681961293)
- Prod environment: not yet scanned (one prod tenant, `life360-saas-prod` in account 552447114887, was confirmed via a manual investigation but is not part of the dev collector-script run)
- Total orphaned resources found: 379 (dev) — collector-script aggregate; not independently reconciled against the per-investigation evidence
- Script coverage: tags-based discovery + supplementary relationship discovery

## Next Steps (Phase 1)

- Prod environment scan
- Ticket implementation for each failure category
- Automated remediation script
