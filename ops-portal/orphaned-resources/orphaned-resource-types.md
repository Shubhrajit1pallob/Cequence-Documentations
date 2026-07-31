# Orphaned Resource Types — Master List
Last updated: 2026-06-02
Source: Dev account (545681961293), 19 tenants investigated

## How this list is built
Resources are identified by the orphaned resources collector script
(`scripts/orphaned-resources-AWS/run.sh collect`) run against deleted tenants.
Resources are confirmed orphaned when they exist in AWS after the tenant's `deletedAt` date.

## Resource types confirmed orphaned

### Secrets Manager
- `secretsmanager:secret` — Defender replica secrets left across multiple AWS regions
  after tenant destroy. Pattern: `saas/<tenant>/defender-<suffix>`. Found in 17+ regions
  per tenant. Most widespread finding.

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

## Summary counts (dev account, 19 tenants)
- Total orphaned resources found: 379
- Most common type: `secretsmanager:secret` (majority of 379)
- Highest cost driver: `ec2:natgateway`
- Widest regional spread: `secretsmanager:secret` (17+ regions per tenant)

## Notes
- This list covers the dev environment only — built from the collector script run against
  account 545681961293.
- Prod has not yet been scanned by the script. Note that investigation #004
  ([[pipeline-004-life360-saas-prod]]) is a **manual** prod finding (account 552447114887),
  not part of this dev script run.
- **3 failure categories confirmed so far** (see [[Index]] and the per-investigation docs):
  #1 CFN no-retry IAM-role orphan, #2 silent `fileExists`-skip orphan, #3 multi-region
  secret replica orphan. Investigation #004 is the **2nd instance of category #3**, not a
  new category — there is no 4th category at this point.
- Script does not yet detect all resource types (e.g. IAM roles, EKS clusters)
  — supplementary discovery via AWS Resource Explorer fills some gaps.
