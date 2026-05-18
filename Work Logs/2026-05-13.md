**Focus:** Researching the  workflow of Tenant Creation Provisioning.

**Time:** 9:30 AM - 6:00 PM

**Lunch:** 3:00 PM - 3:30 PM

**What I did:**
- Investigated the Repositories and follow up on the references mentioned in the Orphaned documentation.
-  Checked into how the creation of the tenant cluster and the following services works. Currently I am on the ops-portal repo where the provisioning happens. This will be a complete walkthrough of the workflow.
	  Checkout: [[Tenant provisioning workflow]]
- Completed the walkthrough of how the tenant's are created through the GitLabs pipeline through different Jobs.
- Completed understanding and documenting how the provisioning of jobs takes place in the code.
  (_Note: This is a basic fundamental design of the how the entire tenant creation logic works.)
- Started to research the account-services repo. 

**Blockers:**
- **Descope ops-portal-admin role**: Not available in my dropdown. Blocks cq-pulumi setup which blocks Pulumi state inspection.
- **AWS account access**: Pending team lead approval. Blocks cost validation and resource inventory work.
- **DocumentDB read access**: Pending. Blocks tenant data model investigation.