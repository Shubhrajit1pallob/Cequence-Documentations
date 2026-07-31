# IRSA Credential Strategy — S2 Investigation

**Status: EVALUATED — NOT CHOSEN** — researched fully; seniors blocked it due to setup overhead. Decision went to static IAM user. This doc remains as a reference if IRSA is revisited.

Related: [[Credential strategy]] (S2 working doc) · [[Discovery job implementation]] (S7)

---

## Background

The original S2 decision used cq-pulumi with a Descope access key (`orphan-discovery-creds` secret) to obtain AWS credentials. This approach was blocked because cq-pulumi grants permissions that are too broad for a pod whose sole function is read-only AWS resource discovery. The principle of least privilege requires a credential path scoped specifically to the job's read operations.

IRSA (IAM Roles for Service Accounts) is the correct Kubernetes-native solution: the pod assumes a scoped IAM role through the cluster's OIDC identity provider, receiving temporary credentials automatically — no static keys, no over-permissioned auth system.

---

## What IRSA does

IRSA lets a Kubernetes pod assume an IAM role with scoped, short-lived credentials via the cluster's OIDC identity provider. The pod receives a projected JWT (`ProjectedServiceAccountToken`) which it exchanges via `STS AssumeRoleWithWebIdentity` for temporary AWS credentials. The AWS SDK picks these up automatically through the standard credential provider chain.

The EKS mutating webhook injects two environment variables into every pod using the annotated service account:

| Env var | Value |
|---|---|
| `AWS_ROLE_ARN` | The IAM role ARN |
| `AWS_WEB_IDENTITY_TOKEN_FILE` | `/var/run/secrets/eks.amazonaws.com/serviceaccount/token` |

The SDK's `fromNodeProviderChain()` reads both and calls `STS AssumeRoleWithWebIdentity`. No code changes needed beyond removing the cq-pulumi credential path.

---

## IRSA vs alternatives

| | IRSA | Static keys (K8s secret) | Node IAM role |
|---|---|---|---|
| Scope | Per service account | Per secret mount | All pods on the node |
| Credentials | Temporary (auto-rotated) | Long-lived (manual rotation) | Temporary |
| Least privilege | ✅ Yes | ⚠️ Only if tightly scoped | ❌ Shared by all pods |
| Secret in cluster | ❌ None | ✅ Lives in etcd | ❌ None |
| Admin overhead | One-time OIDC setup | None | None |

---

## Setup required (platform team)

### Step 1 — Register the OIDC provider (one-time per cluster)

Verify first — the provider may already exist:

```bash
aws iam list-open-id-connect-providers | grep <cluster-oidc-id>
```

If not present:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster <cluster-name> \
  --approve
```

### Step 2 — Create a scoped IAM role

Trust policy — restricts assumption to exactly the `argo-workflow` service account in the `argo-workflows` namespace:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:argo-workflows:argo-workflow"
        }
      }
    }
  ]
}
```

The `:sub` condition is critical — without it, any service account in the cluster can assume the role.

### Step 3 — Attach the minimum IAM permissions

Attach AWS managed policy:
`arn:aws:iam::aws:policy/AWSResourceExplorerReadOnlyAccess`

This covers: `resource-explorer-2:Get*`, `resource-explorer-2:List*`, `resource-explorer-2:Search`, `resource-explorer-2:BatchGetView`, `ec2:DescribeRegions`.

Plus an inline policy for the supplementary discovery layer:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSubnets",
        "ec2:DescribeNatGateways",
        "ec2:DescribeRouteTables",
        "ec2:DescribeNetworkAcls",
        "ec2:DescribeInternetGateways",
        "ec2:DescribeLaunchTemplates",
        "autoscaling:DescribeAutoScalingGroups",
        "eks:ListNodegroups",
        "eks:DescribeNodegroup",
        "cloudformation:ListStackResources"
      ],
      "Resource": "*"
    }
  ]
}
```

### Step 4 — Annotate the Kubernetes service account

```bash
kubectl annotate serviceaccount argo-workflow \
  -n argo-workflows \
  eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
```

---

---

## Open questions (if IRSA is revisited)

1. Is the OIDC provider already registered for `central-ops-dev`? (`aws iam list-open-id-connect-providers`)
2. Is IMDSv2 hop-count set to `1` on the node groups? (If not, pods can also reach the node IAM role, which undermines least privilege)

---

## What to ask the platform team

> For the orphaned-resource discovery job, IRSA setup is needed on the `argo-workflow` service account in the `argo-workflows` namespace on `central-ops-dev`. The job requires read-only AWS access only:
> - Attach managed policy `arn:aws:iam::aws:policy/AWSResourceExplorerReadOnlyAccess`
> - Plus inline: `ec2:Describe*` (the specific actions listed above), `autoscaling:DescribeAutoScalingGroups`, `eks:ListNodegroups`, `eks:DescribeNodegroup`, `cloudformation:ListStackResources`
> - Trust policy scoped to `system:serviceaccount:argo-workflows:argo-workflow` via `:sub` OIDC condition
> - Annotate the service account with `eks.amazonaws.com/role-arn: <ARN>`
