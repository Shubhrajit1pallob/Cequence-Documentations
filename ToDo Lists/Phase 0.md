## Priority 1: Unblock yourself

- [ ]  Test write access to Ops Portal repo — try creating and pushing a test branch. If it works, delete the branch and start committing docs. If it fails, you know you need Developer role and can follow up with team lead.
- [ ]  Follow up on pending access requests (Descope ops-portal-admin, AWS read access, DocumentDB read access). Check if team lead has responded since you sent the justification.

## Priority 2: Jira setup

- [x]  Create Jira stories for Phase 0 through Phase 6 under the Epic (use jira-content.md as copy-paste source)
- [x]  Create the 9 Phase 0 subtasks under the Phase 0 story
- [x]  Tag everything with coop-2026
- [x]  Move Phase 0 story to "In Progress"
- [x]  Mark subtasks 1 (clone repos) and 2 (access requests) as in progress

## Priority 3: Start researching the repos

Start reading the code that matters for Phase 0. The goal is to understand how tenant creation and destruction actually works. Focus areas:

- [ ]  **TenantDestroyCoordinator.ts** — this is the orchestration logic for tenant deletion. Find it in the Ops Portal repo. Read through it and note: - What triggers a destroy? - What order are layers destroyed? - What error handling exists? - Where could a failure leave resources behind?
- [ ]  **Layer 1 (Cluster) repo** — look at the Pulumi code that creates EKS clusters and VPCs. Look at the .gitlab-ci.yml to understand the destroy pipeline stages. Check the Pulumi config for how the state backend is configured (this answers your Pulumi state question).
- [ ]  **Layer 3 (Tenant) repo** — look at what gets created per tenant (namespace, ingresses, CDN, Keycloak). These are the resources at the tenant layer that might leak.
- [ ]  **.gitlab-ci.yml files** across repos — understand the pipeline structure for destroy jobs. Look for manual gates (the "Blocked" pipelines you saw), retry logic, and error handling.

Don't try to read everything today. Start with TenantDestroyCoordinator.ts and one layer repo. Take notes as you go.

## Priority 4: Documentation

- [ ]  If write access confirmed: commit initial docs via MR (problem-statement.md, access-requests.md, decisions.md, work-log.md)
- [ ]  If write access not yet confirmed: keep docs locally, ready to push
- [ ]  Update work-log.md at end of day with what you actually did
- [ ]  Add any new decisions to decisions.md

## Priority 5: Plan the pairing session

- [ ]  Identify which team member(s) can walk you through a provision and destroy in dev. Consider: - Who has done tenant provisioning/deletion recently? - Who maintains the Pulumi code for the tenant layers? - Who has dev environment access?
- [ ]  Reach out to schedule the session — suggest specific times
- [ ]  Before the session, prepare a list of questions so you make the most of the time (what to watch for, what logs to check, etc.)

## Priority 6: Explore agentic approaches for Phase 0

Your supervisor's AI Lens for Phase 0 suggests two specific things:

- [ ]  **Pipeline log analysis**: Can Claude Code ingest a batch of failed destroy pipeline logs and produce a categorized failure-mode summary? Think about: - How would you get the logs into Claude Code? (copy-paste, file, API) - What would the prompt look like? - How would you verify the output? (manually check a sample) - Is this faster than reading logs yourself? Note: you need pipeline log access first, which you have.

- [ ]  **TenantDestroyCoordinator.ts walkthrough**: Can Claude Code walk you through the file and produce a sequence diagram? Think about: - You can start this today if you have the file from the cloned repo - Feed the file to Claude Code and ask for a sequence diagram - Compare the output against your own reading of the code - Log the result in your decision log: where AI helped, where it didn't, what you had to correct

Don't build anything agentic today. Just explore whether these two suggestions from your supervisor are feasible and note your thinking. The decision log entry matters more than the output.

---

## Notes

- Standup is today — key talking points prepared in standup-2026-05-12.md
- Don't try to finish everything today. Phase 0 has multiple subtasks spread across days/weeks.
- If access is still blocked, focus on what you CAN do: reading code, understanding the pipeline structure, planning the pairing session.