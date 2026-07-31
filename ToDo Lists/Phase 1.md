# Phase 1 — Discovery Design

Design-only phase. Subtask details in [[Story - Phase 1 — Discovery Design|the Phase 1 story]]; working docs in [[Phase 1/Index|Phase 1 Docs]].

## 0. Carry-over from Phase 0

- [ ] Get the Phase 0 gate-review write-up signed off (mentor + reviewer) — write-up + PDF ready
- [ ] Jira: create the Phase 1 subtasks under the Phase 1 story (use the story file as copy-paste source); move story to "In Progress"

## 1. Tagging coverage audit (Subtask 1) ✓

- [x] Settle the method questions in [[Tagging coverage audit]] (source = run.sh collect output; RE echoes excluded; tag key = value-match on tenant name)
- [x] Run the audit on the dev account (545681961293) — derived from live run 2026-06-03
- [x] Run the audit on the prod account — out of scope, no access
- [x] Fill the per-service coverage tables; identify systematically untagged classes

## 2. Design doc (Subtask 2) ✓

- [x] Fill the evaluation rubric in [[Discovery approaches — design doc]] (4 options × pros/cons, IAM scope, cost, time-to-implement, untagged coverage)
- [x] Fold in the tagging-audit data to support the recommendation
- [x] Draft recommendation (hybrid approach — Option 4)

## 3. Agentic evaluation (Subtask 3) ✓

- [x] Evaluate agentic vs deterministic discovery (pros/cons in the design doc)
- [x] Decision-log entry in [[Decision Logs]]: signals, what would change the decision, where agentic layers on top (unattributable bucket)

## 4. Hybrid script documentation (Subtask 4) ✓

- [x] Complete the coverage matrix in [[Hybrid discovery script]]
- [x] Document attribution logic, untagged handling, performance (runtime, API calls)
- [x] Write up the fallback-found resources (16 supplementary — note: plan cited 26, actual is 16)
- [x] List known NOT-covered resource types

## 5. Review & ADR (Subtasks 5 + 6)

- [ ] Schedule the design review (mentor + one senior engineer)
- [ ] Present: audit results, design doc, agentic eval, script docs, recommendation
- [ ] Pick one approach with the reviewers
- [ ] Fill in [[ADR — discovery approach]] and commit to the ops-portal repo
- [ ] Phase 1 gate passed
