**Yes**, it's about discovering orphaned resources automatically on a schedule.

**Where should it run?** I suggested GitLab CI as a scheduled pipeline, and that's a perfectly valid option. The tradeoffs:

- **GitLab Scheduled Pipeline**: Fits the existing CI/CD patterns. Easy to set up. Runs on your GitLab runners. Logs are in GitLab. The team is already familiar with it.
- **K8s CronJob**: Runs inside the cluster. Better if it needs direct cluster access to check K8s resources.
- **AWS Lambda**: Serverless, cheap, but another deployment target to manage.

GitLab scheduled pipeline is probably the right starting point given your stack. It can call AWS APIs via configured credentials, store results in DocumentDB, and the Ops Portal reads from there. Another good decision log entry.