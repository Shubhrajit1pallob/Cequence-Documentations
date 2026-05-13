**Investigation Risks:**

- Misidentifying active resources as orphaned
- Incomplete visibility due to insufficient access
- Cost data attribution errors

**Cleanup Risks:**

- Accidental deletion of shared/critical resources
- Data loss if resources are incorrectly classified
- Service disruption if dependencies aren't mapped

**Mitigation:**

- Dry-run validation before any deletions
- Peer review of cleanup scripts
- Backup/snapshot before deletion where applicable
- Staged rollout (start with lowest-cost resources)