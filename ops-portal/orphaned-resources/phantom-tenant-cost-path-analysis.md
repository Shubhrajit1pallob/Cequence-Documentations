# Phantom Tenant Root Cause — Cost Path Doubling

**Status:** Analysis complete. Fix not yet implemented.
**Related tenants:** `luke-5-28-26-dev` (confirmed), `ems-day1` (same pattern), `shub-test` (stack-path variant)
**Date:** 2026-06-17

---

## Plain English: what this is saying

There are phantom tenant rows appearing in the portal — tenant records that look real (they have a customer name, owner, daily cost) but were never provisioned through the normal workflow. This analysis explains exactly how they get created.

**It happens in two steps:**

**Step 1 — AWS has a bad tag.**
A real, live tenant (e.g. `luke-5-28-26`) has at least one AWS resource that is incorrectly tagged. Instead of being tagged `Tenant = luke-5-28-26` (the bare name, which is what all IaC code produces), that one resource is tagged `Tenant = luke-5-28-26-dev` (the full tenant id, which includes the environment suffix). This is a tagging leak — probably a single networking resource like a NAT Gateway, ALB, or EIP, based on the ~$2.16/day flat daily cost. AWS Cost Explorer then reports this as a separate cost line under a different "tenant."

**Step 2 — The portal doesn't recognise it and creates a new row.**
The portal's `TenantCostUpdater` reads that Cost Explorer line, sees `luke-5-28-26-dev` as the tenant name, and builds an id from it: `luke-5-28-26-dev` + `-dev` (because the account is SDLC) = `luke-5-28-26-dev-dev`. It looks that id up in the database, finds nothing, and creates a brand new tenant record. The real tenant's id is `luke-5-28-26-dev`. The computed phantom id is `luke-5-28-26-dev-dev`. The portal never notices the `-dev` was already there — there's no dedup, no "does this look doubled?" check, just a raw string concatenation and an exact-match lookup.

**The result:** A ghost tenant row in the portal that shows real daily cost ($2.16/day), has a real customer name and owner (added later by a human or enrichment sync), but was never provisioned and can't be deleted through the normal workflow.

**The common thread across all phantoms:**
All confirmed phantom tenants (cost-path ones) share the same shape: a real AWS footprint whose identity as AWS reports it doesn't round-trip cleanly to the stored tenant id, combined with an updater that responds to a DB miss by creating rather than reconciling.

**There are two places to fix this:**
1. Fix the AWS tag (find the resource tagged `luke-5-28-26-dev`, retag it to `luke-5-28-26`) — eliminates the trigger
2. Fix the portal's `TenantCostUpdater` create-on-miss — detect the doubled env suffix and log a warning instead of creating a new row

Either fix alone is sufficient. Both together is the complete solution.

---

## How to find the leaky resource

In AWS Cost Explorer: filter `Tenant = luke-5-28-26-dev`, group by Service. The service with ~$2.16/day spend is the resource type. Then use AWS Resource Explorer or Tag Editor to search for that tag value and find the specific ARN.

Alternatively, the orphaned-resources discovery job can find this: run with `TENANT_TARGETS='["luke-5-28-26-dev"]'` — it will find the resource via the tag.

---

## Fingerprints for finding other phantoms in the DB

The `TenantCostUpdater` always writes:
- `region: "unknown"`
- `accountStage: "pov"`
- `internalNotes` containing the string `"TenantCostUpdater"`

A read-only Mongo query to find all cost-path phantoms:

```js
db.tenantrepos.find({
  region: "unknown",
  internalNotes: /TenantCostUpdater/
}, {
  _id: 0, name: 1, id: 1, region: 1, accountStage: 1, internalNotes: 1, createdAt: 1
})
```

Any tenant whose id ends in `-dev-dev`, `-prod-prod`, etc. is a confirmed cost-path phantom.

---

## Original analysis (verbatim)

> Now I can pin the disconnect precisely. I checked the daily-cost path and the defender-pool provisioning, and here's
> the key fact that resolves your confusion:
>
> The portal does not invent the string luke-5-28-26-dev. AWS Cost Explorer hands it back. The create-phantom branch
> only fires in the monthly cost path (getAwsCostsPerTenant → updateTenantCost), and that path's tenant key is built
> directly from the Tenant cost-allocation tag that AWS returns (awsCostInfo.ts:163). So for a phantom named
> luke-5-28-26-dev to exist, some billed AWS resource must be tagged Tenant = luke-5-28-26-dev (the id) instead of
> Tenant = luke-5-28-26 (the bare name).
>
> That's the disconnect, and it lives across two systems:
>
> **Layer 1 — AWS (the origin)**
>
> One real tenant is emitting cost under two different Tenant tag spellings:
>
> - Most of luke-5-28-26's resources → tagged Tenant = luke-5-28-26 ✅ (I confirmed every IaC path I can see tags the
>   bare name: defender-pool CreateDefenderPoolEntryJob.ts:52, cluster config.ts:149, defender utils — all
>   getTenantName() strips the env).
> - At least one resource → tagged Tenant = luke-5-28-26-dev (the full id). This is the leak. It's a small, steady
>   ~$2.16/day line (flat, constant — smells like a single networking resource: NAT GW / ALB / EIP), which is why the
>   phantom shows live daily cost while ems-day1's now reads zero (its stray-tagged resource is gone).
>
> I cannot tell you from the portal repo which resource carries the bad tag — that tag isn't produced by any code in
> these checkouts, so it's either a resource type that derives its tag from the stack id, or something created
> out-of-band. (How to find it: in Cost Explorer filter Tenant = luke-5-28-26-dev and group by Service; or AWS
> Resource Explorer / Tag Editor search for that tag value. Your orphaned-resources collector can do this.)
>
> **Layer 2 — the portal (the amplifier)**
>
> Here's the exact divergence, step by step, for this tenant:
>
> ```
> Cost Explorer row:  Tenant tag = "luke-5-28-26-dev",  account = 545681961293
>         │
>         ├─ tenantName = "luke-5-28-26-dev"          (taken verbatim from the tag)
>         ├─ env        = "dev"                        (account is SDLC → dev)
>         └─ tenantId   = tenantName + "-" + env  =  "luke-5-28-26-dev-dev"
>                                   │
>                           getTenantById("luke-5-28-26-dev-dev")
>                                   │
>                                   └─ no such row  →  CREATE new tenant
>                                        name="luke-5-28-26-dev", region="unknown",
>                                        accountStage="pov", note="...TenantCostUpdater"
> ```
>
> The real tenant's stored id is luke-5-28-26-dev. The computed id is luke-5-28-26-dev-**dev**. The portal never
> compares those two and notices the env is doubled. Identity is a raw string built by concatenation, and matched by
> exact string equality. There is no:
> - canonical tenant key,
> - "does this look like an existing tenant with a doubled suffix?" check,
> - dedup pass,
> - or "skip-and-warn instead of create" guard.
>
> So even though the leaky tag is the trigger, the portal's create-on-miss is what converts a tagging glitch into a
> permanent duplicate row. Either side alone would be harmless: fix the tag and there's nothing to misparse; fix the
> create-on-miss and a bad tag just logs a warning instead of spawning a ghost.
>
> (The customer: "Cequence", accountStage: "internal", owner: luke.keller, additionalNotificationSlackChannel on this
> phantom were not written by TenantCostUpdater — it writes customer = name, accountStage = "pov", no owner. Those got
> filled in afterward, either by a later enrichment sync or a human trying to tidy it. The durable origin
> fingerprints remain region: "unknown" + the internalNotes string.)
>
> **The common thread (refined with this new data point)**
>
> You now have two clean cost-path phantoms — ems-day1 and luke-5-28-26 — plus the stack-path one (shub-test). What
> the cost-path ones share:
>
> 1. Real tenants (not garbage) that genuinely exist and bill.
> 2. Dev / internal tenants in the SDLC account 545681961293 — the ad-hoc, engineer-spun-up ones, where tagging
>    conventions drift (prod tenants go through the strict workflow and don't).
> 3. At least one resource tagged with the tenant id (name-env) instead of the bare name — that's the literal
>    precondition for the env-doubling. Both luke-5-28-26 and ems-day1 have ASG defenders enabled, which is my leading
>    suspect for the leaking resource, but that's a hypothesis I can only confirm by looking at the actual tags.
> 4. A name that does not already end in -dev, so the doubled phantom is visually obvious (-dev-dev).
>
> shub-test is the other flavor (stack-path, TenantPulumiDetailUpdater): there the DB row was missing while the Pulumi
> stack lingered — same root shape ("AWS footprint exists, DB lookup misses, updater creates"), different updater.
>
> So the single common denominator across all of them is: a real AWS footprint whose identity, as AWS reports it,
> doesn't round-trip to the stored tenant id — and an updater that responds to a lookup miss by creating rather than
> reconciling.
>
> Want me to write the read-only Mongo query to list every record matching the phantom fingerprints (region:"unknown"
> + the two internalNotes strings + doubled-env ids)? That gives you the confirmed blast radius before we decide on a
> fix. I won't run anything against the DB — just hand you the query.
