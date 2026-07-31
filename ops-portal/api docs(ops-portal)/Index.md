# ops-portal — API Integration Docs

Reference documentation for all external API integrations in the ops-portal. One file per service — each file covers both the inbound routes (external systems calling into the portal) and the outbound client (the portal calling that service), where both exist.

Generated from source on 2026-06-22. Re-run agents to refresh if source changes significantly.

---

## Inbound Routes (external systems → portal)

| File | Routes covered |
|---|---|
| [[Slack]] | `POST /api/external/slack/commands`, `POST /api/external/slack/interactions` |
| [[Grafana]] | `POST /api/external/grafana/alerts` |
| [[Tenants]] | All `/api/external/tenants/*` routes (13 routes) |
| [[MCP]] | All `/api/external/mcp/*` routes (accounts, margins, audit, account-opportunities) |
| [[Defender Pools]] | All `/api/external/defender-pools/[tenant]/[poolId]/*` routes |
| [[Resource Versions]] | `/api/external/resource-versions`, `/api/external/resource-versions/[id]`, `/api/external/openapi.json` |

## Outbound Clients (portal → external services)

| File | Service | SDK / Client |
|---|---|---|
| [[Slack]] | Slack Web API | `@slack/web-api` |
| [[Grafana]] | Grafana Cloud (Prometheus, Alertmanager) | `axios` |
| [[GitLab]] | GitLab API | `@gitbeaker/rest`, `fetch` |
| [[AWS]] | Cost Explorer, EKS, EC2 | AWS SDK v3 |
| [[Salesforce]] | Salesforce REST API | `fetch` + Zod |
| [[ArgoCD]] | ArgoCD (dev + prod) | Generated OpenAPI client (Axios) |
| [[Kubernetes]] | Kubernetes API | `axios` (no K8s SDK) |
| [[Pulumi]] | Internal Pulumi sync sidecar | `fetch` |
| [[API Testing]] | Internal API testing service | `fetch` + Zod |
| [[Descope]] | Descope auth service | `@descope/node-sdk` |

---

## Notes

- **`sync/route.ts` and `ami-versions/route.ts`** under defender-pools do not exist in the current codebase — noted in [[Defender Pools]].
- **`GET /api/external/orphaned-resources/targets`** (S13 / PLAT-1730) is not yet built — will be added here once implemented.
- Auth across all inbound routes uses Descope Bearer token validation via `withDescopeAuth` or `authenticateRequest`. See [[Descope]] for the full auth logic.
