# AWS SSO Setup

How to configure AWS SSO profiles and wire them into the local environment. Required for the orphaned-resources fetch and deletion scripts.

---

## 1. Configure SSO Profiles

Run `aws configure sso` and answer the prompts:

```
SSO session name (Recommended): cequence
SSO start URL [None]: https://identitycenter.amazonaws.com/ssoins-79072483d0ef7f0e
SSO region [None]: us-west-2
SSO registration scopes [sso:account:access]:        ← press Enter
```

A browser opens — approve the login. The CLI then shows the accounts you have access to:

```
There are N AWS accounts available to you.
> Cequence ... (123456789012)        ← arrow-pick the account for THIS profile
Using the role name "AWSPowerUserAccess"
CLI default client Region [None]: us-east-1
CLI default output format [None]: json
CLI profile name [AWSPowerUserAccess-123456789012]: cdn-administrator    ← name it
```

The last line — **CLI profile name** — is the profile. Name it something clear (e.g. `cdn-administrator`). That exact string is what goes into `.env` as `AWS_PROFILE`.

### Second account

Run `aws configure sso` again. When the account list appears, pick the other account and give it a different profile name (e.g. `saas-tenant-workloads-prod`). Since the SSO session `cequence` already exists, it will reuse it — may not re-open the browser.

---

## 2. Log In

One login covers all profiles on the same session:

```bash
aws sso login --sso-session cequence
```

---

## 3. Wire the `.env` Files

Update each `envs/*.env`:

```
AWS_PROFILE=cdn-administrator
RESOURCE_EXPLORER_VIEW_ARN=<your existing value, unchanged>
```

---

## 4. Verify a Profile

```bash
aws sts get-caller-identity --profile cdn-administrator
```

If that prints an account ID and ARN, the profile is working and the scripts will accept it.

---

## Mental Model

| Concept | What it is | Created |
|---|---|---|
| SSO session (`cequence`) | The login — tied to the start URL + region | Once, shared across all profiles |
| Profile (`cdn-administrator`) | Per-account name you assign | Once per AWS account; goes in `.env` as `AWS_PROFILE` |
