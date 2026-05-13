1. Where the Pulumi State Backend Is Configured

  The backend is selected at runtime in ops-portal/app/_lib/Environment.tsx:29-44 based on the pulumiOrg:

  - cequence org → Pulumi Cloud (PULUMI_BACKEND_URL=https://api.pulumi.com, auth via PULUMI_ACCESS_TOKEN)
  - Any other org → S3 backend (s3://${pulumiS3BackendBucket}?region=${pulumiS3BackendRegion}&awssdk=v2)

  State is encrypted using PULUMI_CONFIG_PASSPHRASE (Environment.tsx:46-47). The backend type can be
  pulumi-cloud, s3, or both (Environment.tsx:144).

  GitLab CI mirrors this: cluster-services/.gitlab-ci.yml:9-11 sets PULUMI_BACKEND_S3 and the stack path
  organization/$PULUMI_PROJECT/$TENANT.

  2. How Ops-Portal Triggers Tenant Onboarding

  Entry point — ops-portal/app/_lib/tenant/TenantService.ts:291-394 createTenant():
  3. Saves tenant doc to MongoDB
  The backend is selected at runtime in ops-portal/app/_lib/Environment.tsx:29-44 based on the pulumiOrg:

  - cequence org → Pulumi Cloud (PULUMI_BACKEND_URL=https://api.pulumi.com, auth via PULUMI_ACCESS_TOKEN)
  - cequence org → Pulumi Cloud (PULUMI_BACKEND_URL=https://api.pulumi.com, auth via PULUMI_ACCESS_TOKEN)
  - Any other org → S3 backend (s3://${pulumiS3BackendBucket}?region=${pulumiS3BackendRegion}&awssdk=v2)

  State is encrypted using PULUMI_CONFIG_PASSPHRASE (Environment.tsx:46-47). The backend type can be pulumi-cloud, s3, or both (Environment.tsx:144).

  GitLab CI mirrors this: cluster-services/.gitlab-ci.yml:9-11 sets PULUMI_BACKEND_S3 and the stack path organization/$PULUMI_PROJECT/$TENANT.

  2. How Ops-Portal Triggers Tenant Onboarding

  Entry point — ops-portal/app/_lib/tenant/TenantService.ts:291-394 createTenant():
  3. Saves tenant doc to MongoDB
  4. If startProvisioning=true, calls triggerProvisioning() → startWorkflow(tenantId, PROVISION_TENANT_WORKFLOW)

  Workflow definition — ops-portal/app/_lib/tenant/provisioning/TenantProvisionCoordinator.ts:29-81, a 23-steppipeline. The Pulumi-relevant steps:
  - CreateCICDEntryJob — registers tenant in .tenant-stacks-{env} list in cicd-templates repo

  - cequence org → Pulumi Cloud (PULUMI_BACKEND_URL=https://api.pulumi.com, auth via PULUMI_ACCESS_TOKEN)
  The backend is selected at runtime in ops-portal/app/_lib/Environment.tsx:29-44 based on the pulumiOrg:


  - cequence org → Pulumi Cloud (PULUMI_BACKEND_URL=https://api.pulumi.com, auth via PULUMI_ACCESS_TOKEN)
  - Any other org → S3 backend (s3://${pulumiS3BackendBucket}?region=${pulumiS3BackendRegion}&awssdk=v2)

  State is encrypted using PULUMI_CONFIG_PASSPHRASE (Environment.tsx:46-47). The backend type can be
  - Any other org → S3 backend (s3://${pulumiS3BackendBucket}?region=${pulumiS3BackendRegion}&awssdk=v2)

  State is encrypted using PULUMI_CONFIG_PASSPHRASE (Environment.tsx:46-47). The backend type can be
  pulumi-cloud, s3, or both (Environment.tsx:144).

  State is encrypted using PULUMI_CONFIG_PASSPHRASE (Environment.tsx:46-47). The backend type can be
  pulumi-cloud, s3, or both (Environment.tsx:144).
  pulumi-cloud, s3, or both (Environment.tsx:144).

  GitLab CI mirrors this: cluster-services/.gitlab-ci.yml:9-11 sets PULUMI_BACKEND_S3 and the stack path
  organization/$PULUMI_PROJECT/$TENANT.

  2. How Ops-Portal Triggers Tenant Onboarding
  3. How Ops-Portal Triggers Tenant Onboarding

  Entry point — ops-portal/app/\_lib/tenant/TenantService.ts:291-394 createTenant():
  4. Saves tenant doc to MongoDB
  5. If startProvisioning=true, calls triggerProvisioning() → startWorkflow(tenantId,
  PROVISION_TENANT_WORKFLOW)

  Workflow definition — ops-portal/app/\_lib/tenant/provisioning/TenantProvisionCoordinator.ts:29-81, a 23-step
  pipeline. The Pulumi-relevant steps:
  - CreateCICDEntryJob — registers tenant in .tenant-stacks-{env} list in cicd-templates repo
  - CreateClusterEntryJob — writes Pulumi.{cluster}-{env}.yaml to cluster repo
  - CreateClusterServicesEntryJob — same, for cluster-services
  - CreateTenantEntryJob (.../create/CreateTenantEntryJob.ts:14-102) — writes the tenant's
  Pulumi.{tenantName}-{environment}.yaml, commits, pushes
  - CreateDefenderPoolEntryJob — writes Pulumi.{tenant}-{poolId}-{env}.yaml

  Each commit kicks off a GitLab CI pipeline (GitlabPipelineJobBase.ts:52-95) that runs pulumi up. Ops-portal
  also uses the Pulumi Automation API directly via @pulumi/pulumi/automation in ops-portal/app/\_lib/pulumi/pulumiProject.ts to inspect/manage stacks (automation.LocalWorkspace.selectStack(...) at lines 193, 273, 319, 345).

  3. Where the Tenant's Stack State Is Stored

  Two storage layers:

  Stack config files (in Git): one YAML per tenant per project, e.g.
  - tenant/Pulumi.amadeus-prod.yaml
  - cluster/Pulumi.{cluster}-{env}.yaml
  - cluster-services/Pulumi.{tenant}-{env}.yaml
  - defender-pool/Pulumi.{tenant}-{poolId}-{env}.yaml

  Actual state (where Pulumi stores resource graph + outputs):
  - Pulumi Cloud at api.pulumi.com (for cequence org), or
  - S3 at s3://{bucket}/organization/{project}/{stack}/ (e.g. s3://.../organization/saas-tenant/amadeus-prod/)

  There's also a pulumi-s3-bridge sidecar service (ops-portal/app/\_lib/pulumi/pulumiSyncServiceClient.ts) that
  reads S3 and mirrors stack metadata into MongoDB so the ops-portal UI can list stacks without hitting
  S3/Pulumi Cloud every time.

  4. Stack Naming Convention

  Generated in pulumiProject.ts:20-23:
  `${env.pulumiOrg}/${name}-${environment}`   // e.g. cequence/amadeus-prod

  Per-project identifiers:
  - Tenant / cluster / cluster-services: {name}-{env}
  - Defender pool: {tenant}-{poolId}-{env}

  YAML file path generation is at pulumiProject.ts:266-268; stack discovery (regex ^Pulumi\.(.+)\.yaml$) at
  pulumiProject.ts:140-154.

  ---
  TL;DR: Ops-portal doesn't directly run pulumi up for onboarding — it commits a Pulumi.{tenant}-{env}.yaml
  file into each Pulumi project's Git repo, and GitLab CI runs the actual pulumi up. State lives either in
  Pulumi Cloud or in an S3 bucket at organization/{project}/{stack}/, depending on the
  pulumiOrg/pulumiBackendType env config. Ops-portal reads state back either via the Pulumi Automation API or
  via the pulumi-s3-bridge → MongoDB sync.