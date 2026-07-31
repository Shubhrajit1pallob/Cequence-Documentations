# Pipeline Investigation #001 — testreportfeature-dev (cluster)

**Status: REVISED** — updated after AWS Console verification on 2026-06-01

## Overview

| Field | Value |
|---|---|
| Tenant | testreportfeature-dev |
| Environment | dev |
| AWS Account | 545681961293 |
| AWS Region | us-east-1 |
| Repo | 1 - Cluster (gitlab.com/cequence/saas/infrastructure/tenants/cluster) |
| Pipeline | #2501429362 |
| Commit | b1e9082a — "Removing tenant files for atacadao-prod" |
| Failed job | Dev Tenant Destroy |
| Duration | 16 minutes 26 seconds |
| Investigated | 2026-05-31 |
| AWS Console verified | 2026-06-01 |

---

## How this pipeline was found

In GitLab CI/CD → Pipelines for the cluster repo, filtered by Source=api and Status=Failed. Destroy pipelines are identified by name — "Removing tenant files for X" indicates a destroy run. Provisioning pipelines say "Add cluster for X" — those are ignored for this investigation.

---

## What the failed job does

The "Dev Tenant Destroy" job in the cluster repo's `.gitlab-ci.yml` runs `scripts/destroy-cluster.sh`. That script:

1. Removes Karpenter managed nodes (provisioners, nodepools, EC2 instances)
2. Waits for EC2 instances to fully terminate
3. Runs `pulumi destroy` with `PULUMI_K8S_DELETE_UNREACHABLE=true`
4. Saves full Pulumi output to `pulumi-destroy-<tenant>.log` as a GitLab artifact
5. Parses the log for errors — exits with code 1 if any errors found

---

## The artifact

Every destroy pipeline saves a `pulumi-destroy-<tenant>.log` file as a GitLab pipeline artifact. This is a JSON array of Pulumi events. To find errors:
- Filter events where `diagnosticEvent.severity == "error"`
- Key fields: `urn` (which AWS resource), `message` (full AWS error with HTTP status code)

The artifact for this pipeline: `pulumi-destroy-testreportfeature-dev.log`
- Total events: 781
- Total errors: 3
- Summary: 0 create, 0 update, 98 delete attempted
- `maybeCorrupt: true` — Pulumi state partially out of sync with AWS after this failure

---

## Root cause

**Category: CloudFormation does not retry IAM role deletion after blocker is cleared**

Full error from artifact:

```
stack status (DELETE_FAILED): The following resource(s) failed to delete: [KarpenterNodeRole].
Cannot delete entity, must remove roles from instance profile first.
(Service: Iam, Status Code: 409, Request ID: 95c01089-3317-4575-b22c-8709a86ce582)
```

### What actually happened (confirmed in AWS Console)

CloudFormation stack `testreportfeature-dev-cluster-karpenterInfra` Resources tab shows:

| Resource | Type | Status |
|---|---|---|
| InstanceStateChangeRule | AWS::Events::Rule | DELETE_COMPLETE |
| KarpenterControllerPolicy | AWS::IAM::ManagedPolicy | DELETE_COMPLETE |
| KarpenterInterruptionQueue | AWS::SQS::Queue | DELETE_COMPLETE |
| KarpenterInterruptionQueuePolicy | AWS::SQS::QueuePolicy | DELETE_COMPLETE |
| KarpenterNodeInstanceProfile | AWS::IAM::InstanceProfile | DELETE_COMPLETE |
| **KarpenterNodeRole** | **AWS::IAM::Role** | **DELETE_FAILED** |
| RebalanceRule | AWS::Events::Rule | DELETE_COMPLETE |
| ScheduledChangeRule | AWS::Events::Rule | DELETE_COMPLETE |
| SpotInterruptionRule | AWS::Events::Rule | DELETE_COMPLETE |

8 out of 9 resources deleted successfully. Only `KarpenterNodeRole` failed.

The actual sequence:

1. CloudFormation tried to delete `KarpenterNodeRole`
2. AWS IAM returned 409 — role still attached to instance profile at that moment
3. CloudFormation continued and successfully deleted `KarpenterNodeInstanceProfile`
4. CloudFormation did NOT go back to retry deleting the role after the blocker was cleared
5. Stack stuck in DELETE_FAILED with only the role remaining

This is NOT a deletion ordering bug — the ordering was correct. The root cause is a known CloudFormation limitation: it does not retry failed resource deletions within the same delete operation.

---

## Why this happens — the code

Karpenter infrastructure is deployed via `pulumi-k8s/src/karpenter/cloudformation.yaml`. Both the role and instance profile are declared in the same CFN template:

```yaml
KarpenterNodeInstanceProfile:
  Type: "AWS::IAM::InstanceProfile"
  Properties:
    Roles:
      - Ref: "KarpenterNodeRole"    # ← association declared here
KarpenterNodeRole:
  Type: "AWS::IAM::Role"
  ...
```

Instantiated by `pulumi-k8s/src/karpenter/karpenterInfra.ts` via `new aws.cloudformation.Stack(...)`. Pulumi manages the entire CFN stack as one resource — no control over internal CFN deletion retry behavior.

---

## Orphaned resources confirmed in AWS

| Resource | Type | Name | Status |
|---|---|---|---|
| KarpenterNodeRole | IAM Role | Kptr-testreportfeature-dev-cluster | Still exists — orphaned |
| KarpenterNodeInstanceProfile | IAM Instance Profile | (already deleted) | DELETE_COMPLETE — not orphaned |
| karpenterInfra CFN stack | CloudFormation Stack | testreportfeature-dev-cluster-karpenterInfra | DELETE_FAILED — stuck |

Only ONE resource is truly orphaned: the IAM Role. The instance profile was successfully deleted.

### Additional finding

`urvashi-sentiel-dev-cluster-karpenterInfra` also shows DELETE_FAILED in the same CFN stacks list (created 2026-04-29). Likely the same failure pattern — needs separate investigation.

---

## How to remediate (do not action without team approval)

1. CloudFormation → find `testreportfeature-dev-cluster-karpenterInfra`
2. Click "Retry delete" — the 409 blocker (instance profile) is already gone
3. Verify stack is removed and IAM role `Kptr-testreportfeature-dev-cluster` is gone

---

## Failure category (for analysis table)

| # | Tenant | Layer | Repo | Pipeline | Failed resource | Category |
|---|---|---|---|---|---|---|
| 1 | testreportfeature-dev | Layer 1 (cluster) | cluster | #2501429362 | aws:cloudformation/stack::karpenterInfra → KarpenterNodeRole | CFN no-retry: IAM role left orphaned after instance profile deleted |
