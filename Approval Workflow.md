
==(Needs to change)==

**What is it?** When the discovery job finds orphaned resources for a deleted tenant, someone needs to approve before those resources get deleted. That's the "human says yes" step.

**Where can it live?** A few options:

- **Slack**: A bot posts "Tenant `carsales` has $2,576/month in orphaned resources — approve cleanup?" with approve/reject buttons. Quick and visible to the team.
- **Ops Portal**: An approval UI in the Orphans tab itself — a reviewer clicks "Approve Cleanup" per tenant. More formal, auditable.
- **Both**: Slack notification triggers attention, Portal is where the actual approval action happens. This is probably the best approach.
- **GitLab MR-based**: A pipeline generates a cleanup MR, someone merges it to approve. Fits your existing GitOps flow.

**What about escalation?** Imagine the bot posts an approval request on Monday and nobody responds for 5 days. Meanwhile `carsales` costs another $430. Escalation just means "if nobody approves within X days, remind again or notify a manager." It's not strictly required for v1 — it's a nice-to-have. You can skip it initially and add it if approvals start getting stuck. This is actually a good decision log entry.

