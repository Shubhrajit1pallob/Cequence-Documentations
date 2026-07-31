# Agent Prompt — Subtask 4: Hybrid Discovery Script Documentation

**Purpose:** Feed this prompt to the scripts agent to generate the materials needed to fill in [[Hybrid discovery script]]. Work through tasks in order — complete each fully before moving to the next.

**Script location:** `scripts/orphaned-resources-AWS/`
**Entry point:** `run.sh collect`
**Output destination:** [[Hybrid discovery script]] in the vault

---

## Task 1 — Read and map the script

Read the full source of the collect subcommand. Identify:
- Every AWS API call made (e.g. `describe-vpcs`, `list-secrets`, Resource Explorer search)
- Which calls are part of the **primary layer** (Resource Explorer tag query) vs the **fallback layer** (per-service SDK calls)
- Which resource types each call returns

Do not produce output yet — just read and build your mental model. Tell me when you're done and give me a one-paragraph summary of how the script works overall.

---

## Task 2 — Build the coverage matrix

Using what you learned in Task 1, produce a markdown table with these columns:

| Resource type | Found by RE (primary) | Found by fallback | How (brief) |

Cover every resource type the script can discover. Mark `yes`, `no`, or `partial` in each layer column. In the "How" column, one phrase explaining the mechanism (e.g. "VPC walk", "tag query", "CFN stack traversal").

---

## Task 3 — Document the attribution logic

Explain in plain language: how does the script decide that a given AWS resource belongs to a specific tenant?

Answer these specifically:
- For **tagged resources**: what tag key and value does it look for?
- For **fallback resources**: what is the chain of reasoning? (e.g. "finds VPC by X, then derives subnets from VPC ID")
- Are there any resources where attribution is ambiguous or best-effort?

Output: 3–5 short paragraphs, suitable to paste directly into the doc.

---

## Task 4 — Document untagged resource handling

Explain what the script does when it encounters a resource with no tenant tag:
- Does it skip it entirely?
- Does it try to match it by naming convention?
- Does it pick it up only if it's a child of an already-attributed resource?
- Is there a separate "unattributable" bucket?

Output: 2–3 short paragraphs.

---

## Task 5 — Identify known gaps

Based on your reading of the script, list the resource types it **cannot** currently discover — either because there's no RE coverage and no fallback for that type, or because the fallback is incomplete. For each gap, note why it's missed.

Output: a bullet list of gaps.

---

## Task 6 — Estimate API call count

Without running the script, estimate the number of AWS API calls made per tenant during a full collect run. Walk through each layer and count the calls. Give a lower bound (minimal tenant, few resources) and upper bound (large tenant, many resources).

Output: a short breakdown table + a one-line summary.

---

## What this prompt cannot cover (needs a live run)

The following two items require a real `run.sh collect` execution against the AWS accounts — the agent cannot produce them from code reading alone:

- **The 26-resource finding** — the actual list of resources the fallback found that RE missed. Once you have a real collect run output CSV, paste it back and this section can be filled in.
- **Real runtime performance** — actual wall-clock time and API call count measured from a run.

Once those are available, add them to [[Hybrid discovery script]] manually or run a follow-up prompt.
