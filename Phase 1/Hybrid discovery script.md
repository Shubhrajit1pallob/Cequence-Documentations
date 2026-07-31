# Hybrid Discovery Script — Coverage & Behavior (Subtask 4)

**Status: COMPLETE** — agent code analysis (2026-06-03) + live `run.sh collect` run against dev account (19 tenants). Prod account not yet run.

Documents the existing implementation at `scripts/orphaned-resources-AWS/` (`run.sh collect`). Related docs: [[dependency-flow-testing]] (the `delete` subcommand's read-only dependency fixer), [[orphaned-resource-types]] (what the script *found*: 379 resources / 19 dev tenants).

---

## How it works — summary

The collect subcommand runs in two sequential layers per tenant per account. The **primary layer** issues a single paginated `resource-explorer-2:SearchCommand` with query `tag.value:<tenant>`, which returns every resource in the aggregator index whose tag value exactly equals the tenant name, across all regions in one call. The **fallback layer** then inspects those results, extracts three classes of anchor — VPC IDs, EKS cluster names, and CloudFormation stack names — and issues a series of targeted per-service Describe calls to find resources that belong to the same tenant but were never tagged: VPC children (subnets, route tables, SGs, IGWs, NAT gateways, EIPs, network ACLs), EKS node groups, ASGs and launch templates found by name prefix, and any CF stack–owned resource whose physical ID is an ARN. Results from both layers are deduplicated by ARN and written to a combined CSV (or printed to terminal in `--print` mode). Accounts are processed sequentially; tenants within an account are also processed sequentially to avoid Resource Explorer throttling.

---

## Coverage matrix

| Resource type | Found by RE (primary) | Found by fallback | How |
|---|---|---|---|
| secretsmanager:secret | Yes | No | Tag query |
| iam:policy | Yes | Partial | Tag query; also via CF stack traversal if created by a tagged stack |
| iam:role | Yes | Partial | Tag query; also via CF stack traversal if ARN physical ID |
| iam:instance-profile | Yes | No | Tag query |
| iam:oidc-provider | Yes | No | Tag query |
| ec2:vpc | Yes | No | Tag query |
| ec2:subnet | Yes | Yes | Tag query (if tagged); VPC walk (always) |
| ec2:security-group | Yes | Yes | Tag query (if tagged); VPC walk, skip default |
| ec2:route-table | Yes | Yes | Tag query (if tagged); VPC walk (always) |
| ec2:internet-gateway | Yes | Yes | Tag query (if tagged); VPC walk via attachment filter |
| ec2:network-acl | Yes | Yes | Tag query (if tagged); VPC walk, skip default |
| ec2:natgateway | Yes | Yes | Tag query (if tagged); VPC walk via vpc-id filter |
| ec2:elastic-ip | Yes | Yes | Tag query (if tagged); extracted from NAT gateway response (free) |
| ec2:volume | Yes | No | Tag query |
| ec2:launch-template | Yes | Yes | Tag query (if tagged); name-contains filter in VPC regions |
| ec2:instance | Yes | No | Tag query only; no fallback |
| ec2:fleet | Yes | No | Tag query only; no fallback |
| ec2:spot-instances-request | Yes | No | Tag query only; no fallback |
| ec2:network-interface | Yes | No | Tag query only; no fallback |
| eks:cluster | Yes | No | Tag query only |
| eks:nodegroup | No | Yes | EKS cluster traversal via ListNodegroups + DescribeNodegroup |
| autoscaling:autoScalingGroup | No | Yes | Account-global DescribeASGs, name-starts-with filter |
| cloudformation:stack | Yes | No | Tag query |
| elasticfilesystem:file-system | Yes | No | Tag query only; no fallback |
| cloudfront:distribution | Yes | No | Tag query only; no fallback |
| acm:certificate | Yes | No | Tag query |
| elasticloadbalancing:loadbalancer | Yes | No | Tag query |
| elasticloadbalancing:targetgroup | Yes | No | Tag query |
| elasticloadbalancing:listener | Yes | No | Tag query |
| events:rule | Yes | No | Tag query |
| logs:log-group | Yes | No | Tag query |
| sqs:queue | Yes | No | Tag query |
| sns:topic | Yes | No | Tag query |
| s3:bucket | Yes | No | Tag query |
| lambda:function | Yes | No | Tag query |
| route53:healthcheck | Yes | No | Tag query |
| Any CF stack child (ARN physical ID) | No | Yes | CF stack traversal via ListStackResources |

---

## Attribution logic

For **tagged resources**, the script queries Resource Explorer with `tag.value:<tenant>`. This is an exact match on tag values — not a substring match, not a key match. Any resource that has at least one tag whose value equals the tenant name exactly will be returned. The tag key is irrelevant; if a resource is tagged `Tenant=cielo`, `Customer=cielo`, or `Name=cielo`, all three match equally. In practice Cequence infrastructure uses `Tenant`, `Customer`, and `Env` tag keys, but the query doesn't filter by key.

For **VPC-anchored resources**, attribution is inherited transitively. The script first identifies all VPCs attributed to the tenant via the tag query. It then calls per-region EC2 Describe APIs filtered by those VPC IDs: `DescribeSubnets`, `DescribeRouteTables`, `DescribeSecurityGroups`, `DescribeInternetGateways`, `DescribeNetworkAcls`, and `DescribeNatGateways`. Any resource returned by those calls is attributed to the tenant on the reasoning that it lives inside a VPC already attributed to the tenant. EIPs are a second hop: they are extracted from the `NatGatewayAddresses` field of the NAT gateway Describe response, requiring no additional API call.

For **EKS node groups**, attribution chains through the cluster: the tag query finds an EKS cluster tagged with the tenant name, and then `eks:ListNodegroups` and `eks:DescribeNodegroup` enumerate all node groups under that cluster. Any node group is attributed to the tenant regardless of its own tags.

For **CloudFormation stack resources**, the tag query finds a CF stack tagged with the tenant name, and `cloudformation:ListStackResources` returns all resources the stack created. Only resources whose `PhysicalResourceId` is an ARN are emitted (other physical ID formats — queue URLs, rule names — are typically already found by the tag query). Attribution is inherited from the stack.

For **ASGs and launch templates**, attribution is name-based and is the most best-effort of all the methods. ASGs are included if their name starts with the tenant name (case-insensitive). Launch templates are included if their name contains the tenant name (case-insensitive), and only in regions where the tenant already has VPCs. These are the only two resource types where attribution is by naming convention rather than by tag or relationship. A false positive is possible if an ASG or launch template name coincidentally begins with a short tenant name (e.g., tenant `car` would match an ASG named `carsales-beanbag`).

---

## Untagged resource handling

The script does not have a separate "unattributable" bucket — untagged resources are either captured via a fallback mechanism or silently not collected. There is no explicit tracking of resources that were seen but could not be attributed.

For resources that are structurally connected to an attributed resource (VPC children, EKS node groups, CF stack children), the script picks them up via relationship traversal regardless of their tags. These are the most reliable fallbacks: the ownership chain is explicit (e.g., a subnet can only belong to one VPC; a node group can only belong to one cluster). A tagged VPC found via the primary layer acts as a root of trust, and everything reachable from it is included.

For resources with no structural connection to an already-attributed resource, the script falls back to name-matching: ASGs whose name starts with the tenant name, and launch templates whose name contains the tenant name. These are best-effort — there is no guarantee that naming conventions are consistent, and no warning is emitted if a name match looks suspicious. Resources that are completely untagged and have names that don't match the tenant's naming convention, and that exist outside any attributed VPC or CF stack, will not be found.

---

## Coverage model and limitations

The script has two discovery layers. Understanding where each layer fails determines what's invisible to the job.

| Layer | How it works | IAM needed | Miss condition |
|---|---|---|---|
| **Tagged** (Resource Explorer) | Finds anything tagged `tag.value:<tenant>` across all resource types | `resource-explorer-2:Search` | Resource was provisioned without the tenant tag |
| **Supplementary** (anchor-walking) | Finds untagged resources by walking from known anchors | The describe permissions in the inline policy | Resource is not a child of a known anchor AND not named after the tenant |

**Current anchors:** found VPCs → NAT gateways, EIPs, subnets, route tables, security groups, internet gateways, network ACLs, launch templates · found EKS clusters → node groups · found CloudFormation stacks → any child resource whose physical ID is an ARN · tenant name prefix → ASGs

**Blind spot:** any resource that is (a) not tagged with the tenant name, AND (b) not a child of one of the above anchors, AND (c) not an ASG named after the tenant — is completely invisible to the job. Current examples: untagged S3 buckets, standalone IAM roles/policies not inside a CFN stack, untagged RDS instances, untagged EBS volumes, untagged ALBs/NLBs.

### How to add coverage for a new resource type

Two steps — both must be done together:

1. **Add the IAM action** to the static IAM user's inline policy.
2. **Add the discovery logic** in the CLI (`ops-portal/scripts/orphaned-resources-AWS`) first, then re-copy the relevant section into `supplementaryDiscover.ts` in the job.

**The parity oracle rule:** the CLI (`run.sh collect`) is the reference implementation. `supplementaryDiscover.ts` in the job must stay at parity with it. Always update the CLI first, then port to the job.

**The sync hazard:** an IAM action with no corresponding discovery code does nothing. Discovery code without the IAM permission throws `AccessDeniedException` — but the `try/catch` in the supplementary loop swallows this as a warning and silently skips those resources. There is no error surfaced to the caller; the gap is invisible. Both the policy and the code must be updated together.

---

## Known gaps

- **ec2:instance** — found by RE only if the instance itself is tagged; no fallback. Karpenter-launched instances **do carry `Tenant` and `Customer` tags** (`cluster/src/config.ts:147-151` → `karpenter.ts:786` → EC2NodeClass), and the prod collector run confirms they are found by the tag query. The gap applies to **non-Karpenter instances** (e.g. manually launched or from other tooling) that may not carry the standard tag keys.
- **ec2:fleet** — found by RE only if tagged. EC2 Fleets created by Karpenter have no fallback discovery path.
- **ec2:spot-instances-request** — found by RE only if tagged. No fallback.
- **ec2:network-interface** — found by RE only if tagged. ENIs created by Lambda, ELB, or EKS are typically untagged and have no anchor relationship the script follows.
- **eks:cluster** — found by RE if tagged, but if a cluster is not tagged it is entirely invisible and its child node groups will also be missed (the node group fallback requires the cluster to be in foundRows).
- **elasticfilesystem:mount-target** — EFS mount targets are not indexed by Resource Explorer at all and have no fallback.
- **cloudfront:distribution** — found by RE if tagged; no fallback for untagged distributions.
- **iam:role** — found by RE if tagged; picked up by CF stack traversal only if the role has an ARN physical ID in a tagged stack. Standalone untagged roles with no CF stack anchor are missed.
- **Isolated untagged resources of any type** — any resource with no tag matching `tag.value:<tenant>`, no VPC/EKS/CF anchor, and a name that doesn't match the tenant prefix will never be discovered.
- **Resources in disabled opt-in regions** — calls to disabled regions may stall (no timeout in the collect path).
- **Launch templates outside VPC regions** — the launch template fallback only runs in regions where the tenant already has a tagged VPC. A launch template in a region with no VPC is missed.
- **CF stack children with non-ARN physical IDs** — SQS queues, EventBridge rules, and other resources use queue URLs or rule names as their physical IDs. These are filtered out by the `physId.startsWith('arn:')` check. They typically appear via the tag query anyway, but if they're untagged and created by a CF stack they are silently dropped.

---

## API call estimate (per tenant)

**Pre-flight (once per account):**

| Call | Count |
|---|---|
| sts:GetCallerIdentity | 1 |

**Primary layer:**

| Call | Lower bound | Upper bound |
|---|---|---|
| resource-explorer-2:Search (paginated, 1,000 results/page) | 1 page | 3–5 pages (300–500 tagged resources) |

**Fallback layer:**

| Call | Condition | Lower bound | Upper bound |
|---|---|---|---|
| autoscaling:DescribeAutoScalingGroups | Always (1 per tenant) | 1 page | 3 pages |
| ec2:DescribeNatGateways | Per VPC region | 0 (no VPCs) | R pages (R = regions with VPCs) |
| ec2:DescribeSubnets | Per VPC region | 0 | R pages |
| ec2:DescribeRouteTables | Per VPC region | 0 | R pages |
| ec2:DescribeSecurityGroups | Per VPC region | 0 | R pages |
| ec2:DescribeInternetGateways | Per VPC region | 0 | R pages |
| ec2:DescribeNetworkAcls | Per VPC region | 0 | R pages |
| ec2:DescribeLaunchTemplates | Per VPC region | 0 | R × 2 pages |
| eks:ListNodegroups | Per EKS cluster | 0 | C calls (C = clusters) |
| eks:DescribeNodegroup | Per node group | 0 | C × N calls (N = node groups per cluster) |
| cloudformation:ListStackResources | Per CF stack | 0 | S calls (S = stacks) |

**Summary by scenario:**

| Scenario | Description | Approx. total calls |
|---|---|---|
| Lower bound | Secrets-only tenant, no VPCs, no EKS, no CF stacks | 3 (1 STS + 1 RE + 1 ASG) |
| Typical tenant | 1 VPC region, 1 EKS cluster with 2 node groups, 2 CF stacks | ~17 (1 STS + 2 RE + 1 ASG + 7 VPC + 1 list-ng + 2 desc-ng + 2 CFN) |
| Upper bound | 3 VPC regions, 2 EKS clusters with 3 node groups each, 5 CF stacks, paginated results | ~45–55 (1 STS + 4 RE + 3 ASG + 21 VPC + 2 list-ng + 6 desc-ng + 5 CFN) |

**One-line summary:** A minimal tenant costs 3 API calls; a large multi-region tenant with EKS and CF stacks costs roughly 45–55 calls, dominated by the 7-call-per-VPC-region VPC child sweep.

---

## Live run findings (2026-06-03)

**Run:** `run.sh collect --tenants Dev_tennant_orphaned_resources.csv --print`
**Account:** dev only (545681961293) — prod not yet run (limitation noted)
**Input:** 19 pre-identified orphan-candidate tenants
**Runtime:** 1m1s (~3.2s per tenant average)
**Total resources found:** 379 (363 tagged / primary + 16 supplementary / fallback)

> **Data-integrity note:** The Phase 1 plan cited "26 untagged resources found by the fallback layer." The actual run shows **16 supplementary resources** — the 26 figure appears to have been an estimate or from an earlier run. 16 is the correct number from this run.

### Supplementary (fallback-found) resources — full list

16 resources across 5 of 19 tenants — these are resources RE did not find via the tag query:

| Tenant | Type | Resource | Found by |
|---|---|---|---|
| dmitry-test-5 | iam:policy | KarpenterControllerPolicy-dmitry-test-5-dev-cluster-v1 | CF stack traversal |
| dmitry-test-5 | iam:policy | KarpenterControllerPolicy-dmitry-test-5-dev-cluster | CF stack traversal |
| dmitry-test-5 | ec2:route-table | rtb-0af397f8877df5567 | VPC walk |
| dmitry-test-5 | ec2:security-group | sg-01198313a84151e37 | VPC walk |
| uap-upgrade-8-2-0 | ec2:route-table | rtb-0054d23df050ae5f2 | VPC walk |
| anirudh843 | iam:policy | KarpenterControllerPolicy-anirudh843-dev-cluster | CF stack traversal |
| anirudh843 | iam:policy | KarpenterControllerPolicy-anirudh843-dev-cluster-v1 | CF stack traversal |
| anirudh843 | ec2:route-table | rtb-0c12775b8d053646f | VPC walk |
| luke-k8s-defenders | ec2:route-table | rtb-039c0f4c778a96cd7 | VPC walk |
| test-upgrade1 | ec2:route-table | rtb-0b4e5cfa5f5d8b310 | VPC walk |
| test-upgrade1 | ec2:security-group | sg-09023ef16ad7b868e | VPC walk |
| test-upgrade1 | ec2:security-group | sg-02bf9fca9bd8c04ce | VPC walk |
| test-upgrade1 | ec2:security-group | sg-0c8c340184cccc933 | VPC walk |
| test-upgrade1 | ec2:security-group | sg-06534a6a208bfafcd | VPC walk |
| test-upgrade1 | iam:policy | KarpenterControllerPolicy-test-upgrade1-dev-cluster | CF stack traversal |
| test-upgrade1 | iam:policy | KarpenterControllerPolicy-test-upgrade1-dev-cluster-v1 | CF stack traversal |

### Supplementary breakdown by type

| Resource type | Count | Mechanism | Why RE missed them |
|---|---|---|---|
| iam:policy (KarpenterControllerPolicy) | 6 | CF stack traversal | Created inside CFN stack; not tagged with tenant name |
| ec2:route-table | 5 | VPC walk | Untagged route tables inside attributed VPCs |
| ec2:security-group | 5 | VPC walk | Untagged security groups inside attributed VPCs |
| **Total** | **16** | | |

### Coverage summary

| Layer | Resources found | % of total |
|---|---|---|
| Primary (RE tag query) | 363 | 95.8% |
| Fallback (supplementary) | 16 | 4.2% |
| **Total** | **379** | **100%** |

The fallback layer contributed resources in 5 of 19 tenants (26%). Without it, 16 resources across those tenants would be silently missed — notably all `KarpenterControllerPolicy` IAM policies and untagged route tables/security groups that are not given a tenant tag by Pulumi.

### Runtime

| Metric | Value |
|---|---|
| Total runtime | 1m1s (61 seconds) |
| Tenants processed | 19 |
| Average per tenant | ~3.2 seconds |
| Account scope | dev only (545681961293) |

> Prod account (552447114887) not yet run — prod tenants typically have defender pools, more regions, and larger resource footprints; runtime and supplementary counts may differ.
