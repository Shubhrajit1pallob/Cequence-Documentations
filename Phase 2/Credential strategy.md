# Credential Strategy (Subtask 2)

**Status: DECIDED** — static IAM user with minimum read-only permissions. Access keys stored as k8s secrets in the `argo-workflows` namespace, mounted as `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` env vars in each cluster's WorkflowTemplate.

Related: [[Workflow platform decision]] (S1) · [[Discovery job implementation]] (S7) · [[IRSA credential strategy]] (IRSA evaluated but not chosen) · [[Phase 2/Index|Phase 2 Docs]]

---

## Decision

**Static IAM user.** `chainProvider()` in `src/aws/credentials.ts` picks up `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` from the environment automatically — no explicit credential construction needed in the job code.

---

## History — three approaches tried in sequence

1. **cq-pulumi (original choice)** — blocked. Required admin privileges seniors were not willing to grant to a read-only discovery pod.
2. **IRSA + AssumeRole** — researched fully (see [[IRSA credential strategy]]). Seniors blocked it due to setup overhead: OIDC provider registration and IAM trust policy creation per target account.
3. **Static IAM user** — chosen. Simpler setup; adequate for a read-only job; requires a key rotation policy.

---

## How it works

Each cluster has its own k8s secret holding AWS access keys. The WorkflowTemplate mounts them as env vars:

| Cluster | Secret name | AWS account |
|---|---|---|
| Dev (central-ops dev) | `saas-workloads-sdlc-read` | 545681961293 |
| Prod (central-ops prod) | `saas-workloads-prod-read` | 552447114887 |

`chainProvider()` resolves `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` from the environment — no changes to `credentials.ts` needed beyond removing the IRSA and cq-pulumi paths.

---

## IAM permissions required

Attach AWS managed policy: `arn:aws:iam::aws:policy/AWSResourceExplorerReadOnlyAccess`

Plus inline policy for the supplementary discovery layer (see [[IRSA credential strategy]] for the full JSON):
- `ec2:Describe*`
- `autoscaling:DescribeAutoScalingGroups`
- `eks:ListNodegroups`, `eks:DescribeNodegroup`
- `cloudformation:ListStackResources`

---

## Open items

- **Key names inside secrets** — confirm the exact key names inside `saas-workloads-sdlc-read` and `saas-workloads-prod-read` with seniors before wiring `secretKeyRef.key` in the WorkflowTemplate
- **Key rotation policy** — long-lived credentials require a rotation schedule; not yet defined
- **Prod secret** — `saas-workloads-prod-read` existence on prod cluster unconfirmed

---

## What was ruled out

- [[IRSA credential strategy]] — the right long-term approach (temporary creds, no secrets in cluster), but seniors blocked it for current scope
- **cq-pulumi** — required admin privileges; removed entirely from the job

---

## Related

- S1 working doc: [[Workflow platform decision]]
- Implementation: [[Discovery job implementation]]
