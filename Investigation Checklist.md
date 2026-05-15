# Investigation Checklist

Candidate causes of orphaned resources / good places to investigate, in rough order of "most likely to bite you."

---

## 1. Regional WAF WebACL associations

**Source:** `src/regionalWaf.ts` (exported as `regionalWafs`)

- [ ] Audit who calls `aws.wafv2.WebAclAssociation` referencing `regionalWafs[region]`.
- [ ] Check whether their destroy order removes the association **before** the protected resource.

AWS won't delete a WebACL while it has association objects pointing at it (ALBs, API Gateways, AppSync). If tenant destruction leaves an ALB/APIGW associated with one of these WAFs, the WAF itself is fine (it's shared, not being deleted) — but the tenant's ALB destruction may itself be blocked by other dependencies, and any audit of "what's still attached to my WAF?" will show orphans.

---

## 2. Route53 records in the tenant hosted zones

**Source:** `src/tenantRouting.ts` (exported as `zoneId` / `ultraApiZoneId`)

- [ ] Inventory `aws.route53.Record` resources that tenants create inside these zones via `StackReference`.
- [ ] Spot-check the zones for records belonging to already-destroyed tenants.

Tenants almost certainly create records inside these zones via `StackReference`. If tenant destruction fails partway, you get orphan records sitting in this stack's zones forever. Doesn't block this stack, but it's a classic "orphaned resource" — exactly the kind of thing this branch is about.

---

## 3. ECR pull-through cache repos

**Source:** `src/ecr/ecrPullThroughCache.ts`

- [ ] Confirm tenant destruction does not (and should not) clean these.
- [ ] Decide whether long-lived cache repos count as "orphans" for inventory purposes, or are intentionally shared and excluded.

Pull-through caches create real ECR repos on first pull. Those repos are owned by this stack's lifecycle, but they fill up with images pulled on behalf of tenants. Tenant destruction does not clean these up — they're shared cache. Not a blocker, but inventory drift.

---

## 4. Global secrets replicated per region

**Source:** `src/globalSecrets.ts`

- [ ] Check whether any tenant `aws.secretsmanager.Secret` has a replica relationship referencing these.
- [ ] Check for resource policies pointing back at the tenant secrets.

If a tenant Secrets Manager resource has a replica relationship or a resource policy referencing these, deletion order can get tangled.

---

## 5. Defender bucket

**Source:** `src/defenderConfiguration.ts` (exported as `defenderConfigurationBucketName`)

- [ ] Check whether tenants write objects into this bucket.
- [ ] If so, confirm tenant destruction clears those objects (the bucket itself is shared and stays).

---

## 6. Session Manager KMS key

**Source:** `src/awsSystemsManager.ts` (exported as `sessionManagerKeyArn`)

- [ ] Audit which tenant EC2/SSM resources reference this key in their config.
- [ ] Confirm key policy changes here don't lock tenants out.

Low risk for destruction specifically — but worth a sweep.

---

## 7. StackReference cycle risk

This stack pulls from `central-ops` and `workload-account`. If a tenant stack pulls from this stack, you have a three-layer chain.

- [ ] Confirm no destroy path for this stack runs while tenants still exist.
- [ ] If anyone ever does try to destroy this stack while tenants exist, the exported values disappear and tenant `pulumi refresh` / `pulumi up` calls start failing — flag this in the runbook.

Destroying tenants doesn't touch this stack, but the reverse is a real risk.
