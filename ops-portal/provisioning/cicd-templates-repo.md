# cicd-templates Repo

The external `cicd-templates` GitLab repo and its role in the provisioning system.

---

## Repo Identity

| Field | Value |
|---|---|
| URL | `https://gitlab.com/cequence/saas/infrastructure/tenants/cicd-templates.git` |
| GitLab project ID | `25561822` |
| Default branch | `master` |
| `hasPipeline` | `false` — portal never triggers a pipeline here |

---

## What the Portal Touches

One file only: `.tenants.yml` at the repo root.

### File Structure

```yaml
.cluster-stacks-prod: [tenantId, tenantId, ...]
.cluster-stacks-beta: [tenantId, ...]
.tenant-stacks-prod:  [tenantId, tenantId, ...]
.tenant-stacks-beta:  [tenantId, ...]
.tenant-stacks-dev:   [tenantId, ...]
```

One key per `type × environment` combination. Keys prefixed with `.` follow the GitLab CI hidden/anchor templates convention.

These lists are consumed by CI pipelines in other repos (`cluster`, `cluster-services`, `tenant`, `tenant-apps`) to know what to provision/destroy.

---

## Multi-tenant vs Normal Tenant

| Tenant type | Added to |
|---|---|
| Normal | `cluster-stacks-{env}` **and** `tenant-stacks-{env}` |
| Multi-tenant | `tenant-stacks-{env}` only (cluster is already shared) |

---

## Operations

### Create — `CreateCICDEntryJob`
1. Clone repo
2. Append `tenantId` to the correct key(s)
3. Sort the list
4. Commit: `"Add <tenantId>"`
5. Push → GitLab CI auto-triggers in downstream repos

### Destroy — `DestroyCICDEntryJob`
1. Clone repo
2. Splice `tenantId` out of the correct key(s)
3. Commit: `"Removing tenant <tenantId>"`
4. Push

---

## Role in the System

`cicd-templates` is a **declarative registry** of "what tenants exist in what environment."

```
Portal edits .tenants.yml
  → downstream CI pipelines read it
    → cluster repo CI: provisions/destroys EKS clusters for listed tenants
    → tenant repo CI: provisions/destroys tenant infra for listed tenants
```

The portal does not need to understand what those pipelines do — it just maintains the list.

---

## TL;DR

One YAML file, one repo, no pipeline. The portal appends or removes tenant IDs from keyed lists on create/destroy. Downstream GitLab CI in cluster and tenant repos reads these lists to decide what to run. `hasPipeline: false` — the portal never triggers a pipeline here, only commits.
