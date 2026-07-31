# Agent Prompt — Phase 2 Subtask 1: Workflow Platform Evaluation

**Purpose:** Feed this prompt to the agent working in the Ops Portal repo to gather the evidence needed for the S1 design decision: **where should the discovery workflow live and how should it be scheduled?** Work through tasks in order — complete each fully before moving to the next.

**Repo context:** Ops Portal repo (`service-portal`). Key paths: `infra/saas/service-portal/ui/argo-workflows/` (the Argo library), `argo-workflows/docs/WORKFLOW_CONCURRENCY.md`.
**Output destination:** Workflow platform decision doc in the vault (S1 working doc), then the decision log.

**The four candidate platforms:**
1. **Argo CronWorkflow** — runs inside K8s, aligns with existing team patterns
2. **Lambda on EventBridge** — serverless, cheap, but separate deployment target
3. **ECS scheduled task** — containerized, managed by AWS
4. **Portal-side cron** — runs within the Ops Portal application itself

**Evaluation dimensions (use these consistently):** operational complexity, team familiarity, credential management, logging/observability, fit with existing infrastructure.

---

## Task 1 — Map the existing Argo library

Read through `argo-workflows/` in the Ops Portal repo. Identify:
- Every existing **CronWorkflow** (scheduled workflow) — name, schedule, what it does
- The reusable **templates/patterns** the library provides (retry wrappers, notification steps, concurrency guards, etc.)
- How workflows are **deployed** (ArgoCD sync? GitLab pipeline? manual apply?)
- Which patterns would fit a "fetch tenant list → scan AWS → write inventory" job naturally, and which gaps would need filling

Do not produce output yet — just read and build your mental model. When done, give me a one-paragraph summary of how the Argo library is organized and a bullet list of existing CronWorkflows.

---

## Task 2 — Document how existing workflows get AWS credentials

This feeds both S1 (credential management dimension) and S2 (credential strategy decision). Answer specifically:
- Do existing workflows use **IRSA** (IAM Roles for Service Accounts)? Point to the service account + role annotation in the manifests.
- Is there any existing **cross-account** access pattern (AssumeRole into the other AWS account)? The discovery job must scan **both** AWS accounts.
- Where are the IAM roles/policies for workflows defined (which infra layer / Pulumi stack)?

Output: 2–3 short paragraphs + file references.

---

## Task 3 — Document logging, observability, and failure handling for existing workflows

- Where do workflow logs go? (Argo UI, CloudWatch, Grafana/Loki, somewhere else?)
- Is there an existing **failure notification** pattern (Slack, PagerDuty) wired into any workflow?
- How would an operator notice a scheduled workflow silently stopped running?
- Skim `argo-workflows/docs/WORKFLOW_CONCURRENCY.md` and summarize the concurrency mechanisms available (this also feeds S6 — racing destroy safety — so note anything tenant-mutex-shaped).

Output: short paragraphs per question + a 3–5 bullet summary of the concurrency doc.

---

## Task 4 — Check for precedent for the other three options

Search the repo (and note anything you know about sibling infra repos) for evidence of:
- **Lambda** deployments (EventBridge rules, SAM/Pulumi Lambda definitions, serverless config)
- **ECS** scheduled tasks (task definitions, EventBridge → ECS targets)
- **Portal-side cron** — does the Ops Portal app already run any in-process scheduled jobs? (Look for cron libraries, `setInterval`-style schedulers, or job-runner patterns in the portal codebase, e.g. around the coordinator/job infrastructure.)

For each: state whether precedent exists, where, and how mature it looks. "No precedent found" is a valid and useful answer — it scores against team familiarity.

Output: one short subsection per option.

---

## Task 5 — Build the comparison matrix

Using Tasks 1–4, produce a markdown table:

| Dimension | Argo CronWorkflow | Lambda + EventBridge | ECS scheduled task | Portal-side cron |

Rows = the five evaluation dimensions plus: **runtime limits** (the hybrid scan may run long — per-tenant scans across 2 accounts × many regions; Lambda's 15-min cap matters), **access to Portal data** (the job needs the deleted-tenant list from DocumentDB), and **deployment/iteration speed**.

Fill every cell with a brief evidence-backed assessment, not generic pros/cons — cite what you found in the repo wherever possible. Mark anything you could not verify as `UNVERIFIED`.

---

## Task 6 — Recommendation and gaps

Give:
- A recommended platform with a 3–5 sentence justification grounded in the matrix
- The runner-up and the condition under which it would win instead
- A bullet list of **gaps to fill** if the recommendation is adopted (e.g. "no existing cross-account AssumeRole pattern — needs new IAM role in account X", "no Slack notification template — write one")

This is input to the decision, not the decision itself — the human confirms with the mentor and writes the decision log entry.

---

## What this prompt cannot cover

- **Mentor confirmation** — the recommendation must be reviewed with the mentor before it becomes the decision (S1 acceptance criterion).
- **Decision log entry** — written in the vault (`Decision Logs.md` template, including the AI / agentic angle section) after mentor sign-off.
- **Anything about the other AWS account's infra** that isn't visible from this repo — flag it as a question rather than guessing.
