# Pipeline Investigation — TEMPLATE & pattern guide

**This is not an investigation.** It is the canonical shape for the `pipeline-NNN-*.md` docs in this folder. The agent that drafts investigation content should follow this so every doc reads the same as `[[pipeline-001-testreportfeature-dev]]`, `[[pipeline-002-anirudh843-dev]]`, and `[[pipeline-003-api-security-ng1-dev]]`.

## Hard rules (don't deviate)

- **Filename:** `pipeline-NNN-<tenant>-<env>.md` — `NNN` zero-padded 3 digits, tenant name with environment suffix (e.g. `pipeline-004-foo-dev.md`). One investigation per file.
- **Numbering:** the investigation number (`#NNN`) is sequential and permanent once assigned. The **Failure category number** (the leading `#` in the final table) is its own catalog; so far it has matched the investigation number 1:1 — keep that alignment unless told otherwise.
- **Tables:** GitHub-flavored **markdown** tables only (`| col | col |`). Never paste terminal/ASCII box-drawing tables — convert them.
- **Code, diagrams, long lists** (code snippets, cascade flows, region lists): wrap in fenced code blocks (```), not tables.
- **Section separators:** a `---` rule between top-level sections, as in the existing docs.
- **Remediation/fix steps:** always carry a "do not action without team approval/discussion" caveat.
- **Data integrity:** never alter a fact to make a doc more consistent. Consistency is about *structure*, not *content* — if the data differs, the doc differs.

## Status banner (optional, second line)

Right under the H1, when applicable:
- `**Status: ACTIVE INVESTIGATION** — ...` (findings still open)
- `**Status: REVISED** — updated after <verification> on <date>` (corrected after the fact)
- Omit entirely for a clean, complete first-pass investigation (see #003).

## Section order (use the ones that apply)

1. `# Pipeline Investigation #NNN — <tenant-env> (<layer / scope descriptor>)`
2. *(optional status banner)*
3. `## Overview` — Field|Value table. Typical fields: Tenant, Environment, AWS Account, AWS Region, Repo/Layer, Pipeline(s), Commit, Failed job, Created→Deleted / Destroy date, Destroy status (portal), Investigated, AWS-verified, Failure category.
4. `## How this pipeline was found` *(if relevant — e.g. GitLab filter, or DocumentDB tenant doc)*
5. `## What the failed job does` **or** `## What the portal destroy workflow shows` **or** `## Destroy workflow result` — job/stack status table.
6. `## Root cause` — the category line + what actually happened (cite AWS Console / Pulumi state / source when verified).
7. `## Why this happens — the code` *(when a code/IaC cause is identified — name the file:line)*.
8. `## Pulumi state file analysis` *(if a state export was inspected)*.
9. `## Orphaned resources` (confirmed in AWS) — table of Resource | Type | ID/ARN | Status. Note cost impact and (if any) security impact.
10. `## Unresolved anomaly` *(anything that doesn't fit / needs follow-up)*.
11. `## How to remediate` **or** `## Recommended fix` — numbered steps, with the approval caveat.
12. `## Bugs to fix in ops-portal` *(when applicable)* — `#` | File (`path:line`) | Fix needed.
13. `## Failure category` — the analysis-table row (see below).

## Failure-category table (last section)

Always lead with a `#` column carrying the catalog number. Columns otherwise fit the case (they vary across #001–#003 by design — keep the data that matters for that investigation). Minimal shape:

```
| # | Tenant | Layer | Category | <case-specific cols: Repo / Pipeline / Cost / Detection signal …> |
|---|---|---|---|---|
| N | <tenant-env> | Layer N (<repo>) | <one-line category> | … |
```

## After creating a doc

- Add a one-line link under "## Pipeline Investigations" in `../Index.md`.
- Update the **Session State** section in the vault `CLAUDE.md` (done last / next step / failure-category catalog).
