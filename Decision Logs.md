Sample File structure- 

```
## 2026-05-11 — Project documentation strategy

**Context:** Starting the orphaned resources project. Need to decide 
where and how to track progress.

**Options considered:**
- A: Jira-only tracking (familiar, visible)
- B: Git markdown only (detailed, reviewable, LLM-readable)
- C: Both — git as source of truth, Jira as public mirror

**AI / agentic angle:** Used Claude to draft initial problem statement 
and structure deliverables. Verified against supervisor's "what done 
looks like" spec. AI was useful for organizing the scope but the 
cost baseline and tech stack details came from internal docs/screenshots.

**Decision:** Option C. Markdown in 
`service-portal/ui/features/orphaned-resources/` is the source of truth. 
Jira PLAT epic mirrors status for stakeholder visibility.

**Open questions / next review:** Need to confirm Ops Portal repo 
write access to start committing docs.
```


The file will in be in the following folder: 
```
ops-portal/service-portal/ui/features/orphaned-resources/decision-logs/
```


