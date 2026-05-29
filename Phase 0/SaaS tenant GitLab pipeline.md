# SaaS Tenant GitLab Pipeline

Top-down walkthrough of `SaaS-Tenant.gitlab-ci.yml` — what the pipeline declares, what the shared fragments do when extended, and how everything fits at runtime.

---

## 1. The image every job runs on

`SaaS-Tenant.gitlab-ci.yml:19` pins:

```
image: registry.gitlab.com/cequence/saas/infrastructure/tenants/cicd-templates/master:latest
```

Built from `cicd-templates/Dockerfile` — base `pulumi/pulumi-nodejs:3.160.0`, plus AWS CLI v2, `kubectl`, `helm`, `eksctl`, `awsudo`, `python3` + `boto3` + `yaml`, `jq`. Every job has Pulumi, Node/npm, AWS auth, and K8s tooling pre-installed.

---

## 2. Pipeline-level setup (lines 1–62)

- **`include:`** — pulls in `.tenants.yml` (local) plus 5 shared fragments from `cequence/cicd-templates` at `ref: master`: `.pulumi-preview.yml`, `.pulumi-update.yml`, `.pulumi-destroy.yml`, `.pulumi-diff-api.yml`, `.assume-role.yml`. Their hidden jobs (`.pulumi_preview`, `.pulumi_update`, `.pulumi_destroy`, `.send-pulumi-diff`, `.assume-role`) become available to `extends:` / `!reference`.
- **`variables:`** — pipeline-wide defaults: `WORKLOAD_ACCOUNT=saas-tenants`, `TRIGGERED_BY=gitlab_ci` (portal overrides this), `PULUMI_K8S_ENABLE_SERVER_SIDE_APPLY=false`, Node heap bump.
- **`stages:`** — just two: `preview` then `update`.
- **`workflow.rules`** — skips detached MR pipelines (works around GitLab issue 34756); otherwise always runs.
- **`.install-dependencies`** — `cd $PULUMI_PROJECT_DIR && npm ci`. Pulled in by `before_script` so every job installs deps first.
- **`.stack-names`** (lines 45–56) — overridden in practice by `.stack-names.yml` from shared templates. Computes `PULUMI_PROJECT` from `Pulumi.yaml`, then `PULUMI_STACK="$PULUMI_ORG/$PULUMI_PROJECT/$TENANT"`, plus S3 variants.
- **`before_script:`** — `set -e; set -o pipefail; !reference [.install-dependencies, script]`. Applies to every job unless overridden.

---

## 3. The three local base templates (lines 64–86)

Hidden jobs that wrap the shared hidden jobs and add SaaS-tenant-specific bits:

| Template | Extends | Stage | Extra |
|---|---|---|---|
| `.tenant_pulumi_preview` | `.pulumi_preview` | `preview` | `environment: $TENANT`, `resource_group: $TENANT` |
| `.tenant_pulumi_update` | `.pulumi_update` | `update` | `environment: $TENANT`, `resource_group: $TENANT` |
| `.tenant_pulumi_destroy` | `.pulumi_destroy` | `update` | `PULUMI_K8S_DELETE_UNREACHABLE=true`, `needs: []` |

- **`resource_group: $TENANT`** — per-tenant mutex; GitLab serializes any job sharing the same resource group, so two pipelines can't update the same tenant simultaneously.
- **`environment: $TENANT`** — makes the tenant show up in GitLab's Environments UI.
- **`needs: []`** on destroy — detaches it from stage ordering so it can fire without waiting on preview.

---

## 4. The concrete jobs (lines 88–308)

Every concrete job: picks `extends:` from the three local bases, sets AWS creds + role ARN + Pulumi token for its env, sets `PULUMI_OPERATION_TYPE`, fans out via `parallel: matrix: TENANT: !reference [.{dev,prod}-cluster, stacks]`, and gates with `rules:`.

The matrix expands `TENANT` from `.tenants.yml`:
- `.dev-cluster.stacks` → ~18 dev tenant slugs.
- `.prod-cluster.stacks` → ~90 prod tenant slugs.

Each tenant is serialized within its own `resource_group: $TENANT`.

| Job | When | Creds / role | Op |
|---|---|---|---|
| Dev Preview | Manual; not portal-triggered, not destroy | `PREVIEW_*_SDLC` + `ROLE_PULUMI_ARN_DEV_PREVIEW` | preview |
| Dev Update | Manual; not portal-triggered, not destroy; retry 1 | `UPDATE_*_SDLC` + `ROLE_PULUMI_ARN_DEV_UPDATE` | apply |
| Prod Preview [MR] | Manual; non-default branches (MRs) | `PREVIEW_*_PROD` + `ROLE_PULUMI_ARN_PROD_PREVIEW` | preview |
| Prod Preview | Manual; default branch push | `UPDATE_*_PROD` + `ROLE_PULUMI_ARN_PROD_PREVIEW` | preview |
| Prod Update | Manual; default branch push; retry 1 | `UPDATE_*_PROD` + `ROLE_PULUMI_ARN_PROD_UPDATE` | apply |
| Dev Destroy | Manual; `DESTROY_TENANT=true` + `TENANT` + `ENVIRONMENT=dev` | `UPDATE_*_SDLC` + `ROLE_PULUMI_ARN_DEV_UPDATE` | destroy |
| Prod Destroy | Manual; same + `ENVIRONMENT=prod` + default branch | `UPDATE_*_PROD` + `ROLE_PULUMI_ARN_PROD_UPDATE` | destroy |
| Dev Tenant Update | Auto when `TRIGGERED_BY=saas_portal` + `ENVIRONMENT=dev` + not destroying | `UPDATE_*_SDLC` + `ROLE_PULUMI_ARN_DEV_UPDATE` | apply |
| Prod Tenant Update | Auto when `TRIGGERED_BY=saas_portal` + `ENVIRONMENT=prod` + default branch + not destroying | `UPDATE_*_PROD` + `ROLE_PULUMI_ARN_PROD_UPDATE` | apply |
| Dev Tenant Destroy | Auto when `TRIGGERED_BY=saas_portal` + `DESTROY_TENANT=true` + `ENVIRONMENT=dev` | `UPDATE_*_SDLC` + `ROLE_PULUMI_ARN_DEV_UPDATE` | destroy |
| Prod Tenant Destroy | Auto when `TRIGGERED_BY=saas_portal` + `DESTROY_TENANT=true` + `ENVIRONMENT=prod` + default branch | `UPDATE_*_PROD` + `ROLE_PULUMI_ARN_PROD_UPDATE` | destroy |

---

## 5. Two regimes, one pipeline

The `rules:` blocks express two non-overlapping regimes:

1. **`TRIGGERED_BY=gitlab_ci` (default):** Matrix-fanned jobs (Dev/Prod Preview/Update + Destroy) are eligible. Updates require a manual click. Destroys require explicit `DESTROY_TENANT=true` + a single `TENANT`.
2. **`TRIGGERED_BY=saas_portal`** (set by the portal when it kicks the pipeline with `TENANT` + `ENVIRONMENT`): All matrix jobs gate off (`when: never`). The single-tenant jobs (Dev/Prod Tenant Update/Destroy) run automatically (`when: always`). No matrix fan-out — just the one tenant the portal asked for.

The `[MR]` vs non-MR split for Prod Preview is the other axis: same job logic, different creds, gated by `$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH`.

---

## 6. What the extended hidden jobs actually do at runtime

When a job `extends: .tenant_pulumi_preview` → which extends `.pulumi_preview`, here is the script body:

1. `before_script` → `set -e`; `pipefail`; `cd $PULUMI_PROJECT_DIR`; `npm ci`.
2. **`.assume-role`** — validates `ROLE_PULUMI_ARN` is set, calls `aws sts get-caller-identity`, then `aws sts assume-role` (region `us-west-2`, 1h session), re-exports `AWS_ACCESS_KEY_ID` / `SECRET` / `SESSION_TOKEN` to the assumed role's creds.
3. **`.stack-names`** — exports `PULUMI_PROJECT` (from `Pulumi.yaml`), `PULUMI_STACK="$PULUMI_ORG/$PULUMI_PROJECT/$TENANT"`, S3 variants, and the S3 backend URL.
4. `pulumi stack init` (ignore-already-exists) + `pulumi stack select`.
5. `pulumi preview --json --show-replacement-steps > pulumi-preview-$TENANT.log` (wrapped in `set +e` so a non-zero exit doesn't kill the job before the diff is sent).
6. **`.send-pulumi-diff`** —
   - Runs `.get-descope-token` → curls Descope's OAuth2 client-credentials endpoint, exports `DESCOPE_ACCESS_TOKEN`.
   - POSTs the log to `$CEQ_PULUMI_DIFF_API?type=$PULUMI_OPERATION_TYPE` with the Descope bearer token and `x-pulumi-*` headers (env, triggered-by, branch, SHA, repo, backend).
   - Parses the response, assigns a badge color (🔴 destructive, 🟠 changes, 🟢 none, ❌ failure), builds a Markdown summary.
   - If `PULUMI_OPERATION_TYPE=preview` and there's a `CI_MERGE_REQUEST_IID`, posts the summary as an MR comment via the GitLab notes API.
7. Greps the log for `diagnosticEvent.severity == "error"`; if any, prints them and exits with the saved Pulumi exit code.
8. Uploads `pulumi-preview-$TENANT.log` as an artifact (`when: always`).

> `.pulumi_update` — same shape with `pulumi refresh --yes || true` + `pulumi update --skip-preview --yes --json` instead of preview.
>
> `.pulumi_destroy` — same shape with `pulumi destroy --yes --remove --refresh --skip-preview --json`, and forces `PULUMI_ORG=$PULUMI_ORG_S3` before computing stack names (so destroys target the S3-backed stack, not Pulumi Cloud).

---

## 7. Pulumi Cloud vs S3 backend split

`.send-pulumi-diff` chooses backend via:

```bash
[[ "$PULUMI_ORG" == "cequence" ]] && PULUMI_BACKEND="pulumi-cloud" || PULUMI_BACKEND="s3"
```

Preview/update default to Pulumi Cloud. Destroy is rewritten to S3 (via `PULUMI_ORG=$PULUMI_ORG_S3`). `.stack-names.yml` builds both `PULUMI_STACK` (cloud form: `cequence/project/tenant`) and `PULUMI_STACK_S3` (`organization/project/tenant`), plus the S3 backend URL.

---

## 8. Companion pipelines

- **`Workloads.gitlab-ci.yml`** — entirely separate (not invoked by the SaaS-Tenant pipeline). Two jobs (Workloads Bucket Permission Dev/Prod) that run on MR events to manage workload-account bucket permissions. Redefines local copies of `.assume-role` and a `.workloads-arn` helper that reads `ROLE_PULUMI_ARN` from a Pulumi stack output.
- **`.gitlab-ci.yml`** — this repo's own Auto-DevOps pipeline (tests disabled), for the meta-build of the runner image.
- Other templates in `Tenants/cicd-templates/` (`Helm-Chart.gitlab-ci.yml`, `K8s-Update-Image.gitlab-ci.yml`, `Auto-DevOps.gitlab-ci.yml`, etc.) are consumed by application repos, not by the SaaS-tenant pipeline. For SaaS tenant work, ignore those — your contract surface is `.pulumi_preview` / `.pulumi_update` / `.pulumi_destroy` + the env vars they require.

---

## 9. End-to-end runtime example — Prod portal-triggered update

1. SaaS portal hits GitLab's pipeline trigger API: `TRIGGERED_BY=saas_portal`, `TENANT=ulta-prod`, `ENVIRONMENT=prod`, on `master`.
2. `workflow.rules` allows the pipeline (not a detached MR).
3. All matrix jobs evaluate their rules → first clause (`TRIGGERED_BY=="saas_portal"` → `never`) → skipped.
4. **Prod Tenant Update**'s rule matches → `when: always` → runs immediately.
5. `before_script` installs deps. `.assume-role` swaps to `ROLE_PULUMI_ARN_PROD_UPDATE`. `.stack-names` computes `cequence/<project>/ulta-prod`. `pulumi refresh` + `pulumi update --skip-preview --yes --json` runs against that single stack.
6. `.send-pulumi-diff` POSTs the diff log to the portal's API (`backend=pulumi-cloud`), skips the MR-comment step (no MR IID).
7. Errors in the log fail the job; the log is uploaded as an artifact either way. If it fails once, `retry: 1` gives it a second shot.
