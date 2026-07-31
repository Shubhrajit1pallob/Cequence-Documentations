# ops-portal / Orphaned Resources — Docs

Reference and testing documentation for the `orphaned-resources-AWS` CLI under `scripts/orphaned-resources-AWS/`.

## Docs

- [[dependency-flow-testing]] — Testing guide for the read-only dependency fixer in the `delete` subcommand
- [[orphaned-resource-types]] — Master list of AWS resource types confirmed orphaned across destroy investigations (dev account; 379 resources / 19 tenants)
- [[failure-mode-analysis]] — Phase 0 consolidated failure-mode analysis (2 categories: CFN no-retry IAM role, multi-region secret replica); dev account, 19 tenants
- [[phase-0-gate-review]] — Phase 0 gate review write-up: lifecycle walkthrough, 5+ leak-prone resource types, categorized failure-mode list, decision-point options (vault assembly; official copy in ops-portal repo)

- [[phantom-tenant-cost-path-analysis]] — Root cause analysis for phantom tenant rows (e.g. `luke-5-28-26-dev`): AWS bad tag (id instead of bare name) + portal create-on-miss in `TenantCostUpdater` → doubled env suffix (`-dev-dev`). Includes Mongo fingerprint query and fix options.

## Pipeline Investigations

Manual deep-dives into specific failed destroy pipelines. Each doc covers root cause, orphaned resources left behind, and remediation steps. New docs follow [[_TEMPLATE]] — the pattern guide for the prompt-drafting agent (naming, sections, table format, failure-category numbering).

- [[pipeline-001-testreportfeature-dev]] — CFN no-retry: IAM role left orphaned after instance profile deleted (CloudFormation DELETE_FAILED), cluster repo. **REVISED** after AWS Console verification 2026-06-01.
- [[pipeline-002-anirudh843-dev]] — Silent orphan: false-negative `fileExists` check skipped the cluster destroy pipeline, portal reported SUCCESS; ~$40-52/mo still billing (NAT Gateway + EIPs + full VPC). **ACTIVE** — ops-portal code bug (commit `17fdd6a7`).
- [[pipeline-003-api-security-ng1-dev]] — Multi-region secret replica orphan: clean SUCCESS destroy, but the replicated defender secret's primary was deleted while replicas survived as standalone secrets (30 created; survivor count not cost-verified — amended after #004). **ACTIVE** (confirmed live 2026-06-01). Defect is in tenant IaC `tenant/src/defender.ts` (resource-in-`apply()` + all-region replicas) — NOT an ops-portal bug.
- [[pipeline-004-life360-saas-prod]] — Multi-region secret replica orphan (category #3, 2nd instance): clean SUCCESS destroy, but of 30 defender-secret replicas 13 were removed and 17 left stranded & undeletable (primary gone). ~$6.80/mo. **ACTIVE** — tenant IaC defect (`tenant/src/defender.ts`), proves the teardown is *partial*, prompting the #003 count correction.
- [[pipeline-005-shub-test-dev]] — Multi-region secret replica orphan (category #3, 3rd instance): clean SUCCESS destroy on an internal dev test tenant (account 545681961293); 30 defender-secret replicas → 14 removed → 16 surviving (primary us-east-2). ~$6.40/mo (estimate). 31 Karpenter/EKS compute rows are stale Resource-Explorer terminal-state echoes (not billable; pending `describe-instances`). **ACTIVE** — tenant IaC defect (`tenant/src/defender.ts`), no ops-portal bug.
- [[pipeline-006-ache-prod]] — **New failure category #4**: ASG surviving-instance DependencyViolation (prod, account 552447114887, us-west-1). Defender-pool destroy hard-FAILED (`maybeCorrupt:true`): a running EC2 instance survived a clean ASG deletion and its ENI blocked the defender SG delete. Correctly surfaced as FAILURE (contrast silent #2/#3). **ACTIVE** — several fields PENDING (tenant JSON, instance ID, other stacks); defect in defender-pool teardown (`src/defenderPool.ts`), not orchestration.
