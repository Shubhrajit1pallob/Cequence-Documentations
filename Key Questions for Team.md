
- **Process Understanding:**
    - "What's the current tenant deletion procedure? Is it documented?"
    - "Do we have centralized Pulumi destroy logs or GitLab pipeline history for deleted tenants?"
    - "How are resources tagged? Is there a tenant identifier we can query?" ( Either check the ==architecture== repo or the ==account_services==)


- **Scope & Priority:**
    - "Are there specific deleted tenants you want me to investigate first?" (The costliest ones first I assume)
    - "What's the priority: quick cleanup or full process overhaul?" (This has been answered in the =="What 'Done' looks like "== section)


- **Historical Context:**
    - "Has anyone investigated orphaned resources before? What did they find?"


- **Approval workflow preference**: Slack, Portal, GitLab MR-based, or a combination?
- **Who are the authorized approvers** for resource cleanup?
- **Discovery job cadence**: Daily? Weekly?
- **DocumentDB access**: Can I get read/write for storing orphan inventory?
- **Grafana access**: Can I get write access for building monitoring dashboards?
- **Pulumi state backend**: Where is it hosted (S3, Pulumi Cloud)? Can I get read access?