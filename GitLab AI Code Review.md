---
status: PENDING SETUP — waiting on two secrets
updated: 2026-06-29
---

# GitLab AI Code Review Setup

**Status: PENDING SETUP** — will be configured when the two required secrets are available.

---

## What it does

Automatically posts a Claude AI code review as a comment on every GitLab MR. Uses `cequence/cicd-templates/.ai-review.yml` — already exists, no need to build from scratch.

---

## How to enable it for a repo

### Step 1 — Add the include to `.gitlab-ci.yml`

```yaml
include:
  - project: 'cequence/cicd-templates'
    ref: master
    file: '.ai-review.yml'
```

### Step 2 — Add two CI/CD variables in the repo's GitLab settings

| Variable | What it is |
|---|---|
| `ANTHROPIC_API_KEY` | Anthropic API key |
| `GITLAB_AI_REVIEW` | GitLab Project Access Token with `api` scope — reads MR diffs + posts the review comment |

That's it. Every MR will get an automated Claude review posted as a comment.

---

## Reference

- Template: `cequence/cicd-templates/.ai-review.yml`
- Model: `claude-sonnet-4-6`
- Timeout: 900s
- Only fires on `merge_request_event` pipelines
- Skips if a review from `cequence-ai-review` (bot account) already exists on the MR
- `allow_failure: true` — won't block merges if it fails

---

## Two missing pieces blocking setup

1. **`ANTHROPIC_API_KEY`** — needs to be obtained/confirmed
2. **`GITLAB_AI_REVIEW`** — needs a GitLab Project Access Token created with `api` scope
