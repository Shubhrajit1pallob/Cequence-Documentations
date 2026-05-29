## 1. Unblock yourself

- [x]  Test write access to Ops Portal repo — try creating and pushing a test branch. If it works, delete the branch and start committing docs. If it fails, you know you need Developer role and can follow up with team lead.
- [x]  Follow up on pending access requests (Descope ops-portal-admin, AWS read access, DocumentDB read access). Check if team lead has responded since you sent the justification.

## 2. Jira setup

- [x]  Create Jira stories for Phase 0 through Phase 6 under the Epic (use jira-content.md as copy-paste source)
- [x]  Create the 9 Phase 0 subtasks under the Phase 0 story
- [x]  Tag everything with coop-2026
- [x]  Move Phase 0 story to "In Progress"
- [x]  Mark subtasks 1 (clone repos) and 2 (access requests) as in progress

## 3. Research the code

Start reading the code that matters for Phase 0. The goal is to understand how tenant creation and destruction actually works.

### Provisioning side (working backwards from these helps the destroy side)

- [x] End-to-end provisioning workflow → [[Tenant provisioning workflow]]
- [x] GitLab pipeline trigger + status tracking → [[GitLab pipeline trigger + tracking]]
- [x] Job-to-job transitions / Coordinator polling loop → [[Job transitions]]
- [x] ArgoCD sync jobs — how the sync steps fit into provisioning

### Destroy side

- [ ] **TenantDestroyCoordinator.ts** — orchestration logic for tenant deletion. Find it in the Ops Portal repo. Read through it and note:
  - What triggers the destroy?
  - In what order are layers destroyed?
  - What error handling exists?
  - Where could a failure leave resources behind?
- [x] **Layer 1 (Cluster) repo** — Pulumi code that creates EKS clusters and VPCs. Look at the `.gitlab-ci.yml` to understand the destroy pipeline stages. Check the Pulumi config for how the state backend is configured (this answers your Pulumi state question).
- [x] **Layer 3 (Tenant) repo** — what gets created per tenant (namespace, ingresses, CDN, Keycloak). These are the resources at the tenant layer that might leak.

### Pipeline-level

- [x] **`.gitlab-ci.yml` files** across repos — pipeline structure for destroy jobs. Look for manual gates (the "Blocked" pipelines), retry logic, and error handling.

Don't try to read everything at once. Start with `TenantDestroyCoordinator.ts` next. Take notes as you go.

## 4. Documentation

- [x]  If write access confirmed: commit initial docs via MR (problem-statement.md, access-requests.md, decisions.md, work-log.md)
- [x]  If write access not yet confirmed: keep docs locally, ready to push
- [x]  Update work-log.md at end of day with what you actually did
- [x]  Add any new decisions to decisions.md

## 5. Explore agentic approaches for Phase 0

Your supervisor's AI Lens for Phase 0 calls for **both** tasks below. Do them sequentially — start with the one whose input data is already in hand (usually Task B, since the `.ts` file is in the cloned repo).

### Setup

- [x]  Read [[Agentic Workflow/Index|Agentic Workflow]] to ground yourself in where AI is/isn't a fit for this project
- [ ]  Decide where AI experiment inputs and outputs will live (suggest: `Agentic Workflow/` folder with one sub-file per experiment — e.g. `pipeline-log-analysis.md`, `destroy-coordinator-walkthrough.md`)

### Task A — Pipeline log analysis

- [ ]  Gather a batch of failed destroy pipeline logs (50+ if available) — save the raw logs to a file
- [ ]  Draft a prompt: what task (categorize failure modes), what input format, what output format you want (e.g. table of category → count → representative example)
- [ ]  Run on a small sample first (1–2 logs) to sanity-check the output
- [ ]  Run on the full batch
- [ ]  **Verify a sample manually** — pick ~5 logs and check whether the AI's categorization matches what you'd say
- [ ]  Capture the result in `Decision Logs.md` using the AI / agentic angle template — note where AI sped you up and where it missed something a human caught

### Task B — `TenantDestroyCoordinator.ts` walkthrough

- [ ]  Locate `TenantDestroyCoordinator.ts` in the cloned Ops Portal repo
- [ ]  Draft a prompt asking Claude Code to walk through the file and produce a sequence diagram of the destroy flow
- [ ]  Run it; read the AI's output alongside the source file
- [ ]  Compare the diagram against your own reading — note discrepancies
- [ ]  Capture the result in `Decision Logs.md` using the AI / agentic angle template — note where AI sped you up and where it missed something a human caught

---

## Notes

- Standup is today — key talking points prepared in standup-2026-05-12.md
- Don't try to finish everything today. Phase 0 has multiple subtasks spread across days/weeks.
- If access is still blocked, focus on what you CAN do: reading code, understanding the pipeline structure.
