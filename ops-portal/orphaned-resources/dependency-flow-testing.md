# Dependency Flow Testing — `orphaned-resources-AWS delete`

Testing guide for the read-only dependency fixer added to the `delete` subcommand.
Audience: engineer with `aws sso login` working and a recent collector CSV on hand.

---

## 0. Prerequisites

- SSO session active:
  ```bash
  aws sso login --profile dev-profile
  ```
  Re-run if the session lapsed (token expires after ~8 hours).

- Working directory for all commands:
  ```bash
  cd scripts/orphaned-resources-AWS/
  ```

- `envs/dev.env` must contain real values — no placeholders:
  ```
  AWS_PROFILE=dev-profile
  RESOURCE_EXPLORER_VIEW_ARN=arn:aws:resource-explorer-2:us-east-1::view/...
  ```

- A recent collector CSV must exist at `output/resources/orphaned-resources-<timestamp>.csv`.
  If none exists, generate one first:
  ```bash
  ./run.sh collect
  ```
  Output lands at `output/resources/orphaned-resources-<YYYYMMDD-HHmm>.csv`.

---

## 1. Dry-run Sanity Check (no AWS mutations)

```bash
./run.sh delete --input ./output/resources/orphaned-resources-<timestamp>.csv
```
_(← replace `<timestamp>` with your latest CSV filename)_

**Walk:**
1. Tenant picker appears → select any tenant.
2. Region picker → select any region.
3. Choose **View** → confirm the resource list renders and waits for Enter.
4. Choose **Delete ALL** → confirm with `yes`.
5. Observe `DRY-RUN: would delete …` lines in the output. No dependency prompts should appear.
6. Choose **Quit**.

**Expected artifacts:**

| Path | Expected content |
|---|---|
| `output/audit-logs/deletion-audit-<ts>.csv` | Only `Action=dry-run, Result=would-delete` rows |
| `output/dependency-reports/` | Empty or absent — no dependency failures occurred in dry-run |

---

## 2. Pick a Low-Risk Tenant for the Real Test

Choose a tenant where the remaining resources are clearly disposable.

**Concrete suggestion:** `test-upgrade1` — Karpenter/EKS leftovers in `eu-south-1`, a known orphan confined to a single region.

Before proceeding, skim the step 1 audit log to confirm what would be touched:

```bash
cat output/audit-logs/deletion-audit-*.csv
```

---

## 3. Real Run, Narrow Scope (`--execute`)

```bash
./run.sh delete --input ./output/resources/orphaned-resources-<timestamp>.csv --execute
```

Note the **red `EXECUTE` banner** in the preamble — confirms live deletes are enabled.

**Walk:**
1. Tenant picker → select `test-upgrade1`.
2. Region picker → select `eu-south-1` (narrow scope — single region).
3. Choose **Delete ALL** → confirm with `yes`.

**Expected:** each resource returns either:
- `✓ deleted` (green) — successfully removed.
- `✗ <AWS error>` (red) — failed; if the error is a dependency error, the dependency flow triggers.

For the EC2 network chain (VPCs, subnets, SGs), expect "has dependencies" failures that trigger the flow described in section 4.

---

## 4. Observe the Dependency Flow

When a delete is blocked, the terminal prints (in yellow):

```
⚠ SecurityGroup sg-0abc123def456789 is blocked by 2 dependency(s):
  • eni-0aabbccdd11223344  |  owner: i-0deadbeef  |  type: interface  |  status: in-use  |  attachment: attached
  • eni-0eeff00112233445  |  owner: eni-owner-id  |  type: interface  |  status: available  |  attachment: detached
```

Then the four-option prompt appears:

```
? SecurityGroup arn:aws:ec2:eu-south-1:123456789012:security-group/sg-0abc123def456789 is blocked by dependencies. What now?
  > Skip this resource (I will handle it manually / later)
    Retry the delete (I have cleared the blockers)
    Investigate — open the report at output/dependency-reports/dependencies-<ts>.md and I will wait
    Quit
```

---

## 5. Try All Three Actions

### Skip

On the first blocked resource, select **Skip this resource**.

- The script moves to the next resource in the list.
- The dependency report path is printed to the terminal.
- The blocked resource is recorded in the audit log as skipped.

### Investigate

On the next blocked resource, select **Investigate**.

- The script prints the report path and pauses:
  ```
  Report written to: output/dependency-reports/dependencies-<ts>.md
  Press Enter when you are ready to continue…
  ```
- Open a second terminal and inspect the report:
  ```bash
  cat output/dependency-reports/dependencies-*.md
  ```
  The report contains one Markdown section per blocked resource: ARN, type, region, the verbatim AWS error, and the full blocker list.
- Optionally: manually clear blockers in the AWS Console (e.g. detach/delete the ENIs, update SG rules).
- Return to the original terminal and press **Enter**.

### Retry

After Investigate (with or without manually clearing blockers), select **Retry the delete**.

- The script re-attempts **only** the originally-listed resource's own delete.
- If blockers remain: the delete fails again and the four-option menu re-appears.
- If blockers were cleared: the delete succeeds and prints `✓ deleted`.

The script never touches the blocking resources themselves (ENIs, load balancers, SG rules, bucket objects) regardless of which option is chosen.

---

## 6. Inspect the Artifacts

**`output/audit-logs/deletion-audit-<ts>.csv`**

Every attempted action is recorded, including:
- Dry-run attempts (`Action=dry-run`)
- Successful deletes (`Result=deleted`)
- Failures (`Result=failed`, AWS error verbatim)
- Retries (separate row per attempt)
- Skipped resources

**`output/dependency-reports/dependencies-<ts>.md`**

Human-readable Markdown. One section per blocked resource containing:
- ARN, resource type, region
- Verbatim AWS error that triggered the fixer
- Full blocker list (IDs, owners, types, statuses)

This file is only created if at least one delete was blocked. If all deletes succeeded or were dry-run, this file does not exist.

---

## 7. Regressions to Watch For

> **Warning:** The script must never, on its own, perform any of the following. If any of these occur, stop immediately and report.
>
> - Delete an ENI or network interface not explicitly listed in the input CSV.
> - Modify a load balancer, CloudFront distribution, or any ACM certificate consumer.
> - Strip or modify another security group's inbound/outbound rules.
> - Empty an S3 bucket — delete objects, versions, or delete markers.
>
> Additionally:
> - **Esc** anywhere must navigate back one level. At the top-level tenant picker, Esc is a no-op — it must never exit the script.
> - **Ctrl+C** must kill the process cleanly. Audit log rows and dependency report sections written before the kill must be preserved on disk.

---

## 8. Bailing Out

**Quit from any menu**
Clean exit. The audit log is finalized. Any dependency report written so far is closed and readable.

**Ctrl+C**
Hard kill. Everything written up to that point is preserved — both the audit log and the dependency report stream to disk as they go, so no in-flight data is lost.
