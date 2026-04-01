# Operations

## Overview

This document covers the operational model for EnerFusion Upstream Accounting running on Azure Kubernetes Service: deployment pipelines, observability, runbooks, and routine maintenance procedures.

---

## CI/CD Pipeline

### Pipeline Gates (Every PR)

All of the following must pass before a PR can merge to `main`:

```
PR Opened
  │
  ├── ESLint + Prettier check
  ├── TypeScript: tsc --noEmit (strict mode)
  ├── Unit tests: Jest (backend) / Vitest (frontend)
  │     └── Min 80% line coverage enforced
  ├── Contract tests: Pact consumer/provider verification
  ├── Docker build (multistage, no build failures)
  └── Trivy image scan (CRITICAL/HIGH = fail)
         │
         ▼
      PR approved + merged to main
```

### Deployment Pipeline (GitHub Actions)

```yaml
# Staging (automatic on merge to main)
on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    steps:
      - name: Build & tag image
        run: docker build -t $ACR/$SERVICE:$GITHUB_SHA .

      - name: Push to ACR
        run: docker push $ACR/$SERVICE:$GITHUB_SHA

      - name: Helm upgrade staging
        run: |
          helm upgrade --install $SERVICE ./helm/$SERVICE \
            --namespace pra-staging \
            -f helm/environments/values-staging.yaml \
            --set image.tag=$GITHUB_SHA \
            --wait --timeout 5m

      - name: Run smoke tests
        run: npm run test:smoke -- --env staging

# Production (manual approval gate)
  deploy-production:
    needs: deploy-staging
    environment: production   # GitHub environment with required reviewer
    steps:
      - name: Helm upgrade production
        run: |
          helm upgrade --install $SERVICE ./helm/$SERVICE \
            --namespace pra-prod \
            -f helm/environments/values-prod.yaml \
            --set image.tag=$GITHUB_SHA \
            --wait --timeout 10m
```

**Image tagging:** Git commit SHA is the only acceptable image tag in Kubernetes manifests. The `latest` tag is forbidden in all environment values files.

---

## Helm Chart Structure

```
helm/
├── charts/
│   ├── svc-production/
│   │   ├── Chart.yaml
│   │   ├── values.yaml           # Default values (no secrets, no env-specific config)
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── hpa.yaml
│   │       ├── networkpolicy.yaml
│   │       ├── secretproviderclass.yaml
│   │       └── servicemonitor.yaml   # Prometheus scrape config
│   ├── svc-ownership/
│   ├── svc-allocation/
│   ├── svc-valuation/
│   ├── svc-balancing/
│   ├── svc-revenue/
│   ├── svc-payments/
│   ├── svc-regulatory/
│   ├── svc-admin/
│   └── shared/
│       ├── auth-svc/
│       ├── audit-svc/
│       └── notification-svc/
└── environments/
    ├── values-dev.yaml
    ├── values-staging.yaml
    └── values-prod.yaml
```

**Key Helm Values Pattern:**

```yaml
# values-prod.yaml
image:
  repository: praacr.azurecr.io/svc-production
  tag: ""            # Always overridden by CI with git SHA
  pullPolicy: IfNotPresent

replicaCount: 3

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

keyVault:
  name: pra-prod-kv
  secretProviderClass: pra-production-secrets

# Secrets reference Key Vault — never plaintext
# database.url resolved from Key Vault at pod startup
```

---

## AKS Operations

### Node Pool Management

```bash
# Check node pool status
kubectl get nodes -l agentpool=services --show-labels

# Scale batch pool manually (e.g., for emergency month-end rerun)
az aks nodepool scale \
  --resource-group pra-rg \
  --cluster-name pra-prod \
  --name batch \
  --node-count 5

# Check HPA status for a service
kubectl get hpa svc-valuation -n pra-prod
```

### Rolling Deployments

All services deploy with `RollingUpdate` strategy:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0     # Zero-downtime requirement
```

Database migrations (`knex migrate:latest`) run as a Kubernetes Job before each deployment. Migrations must be backwards-compatible (additive only) to support rolling updates.

### Pod Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 2
```

The `/health` endpoint returns `200 OK` if the process is alive. The `/ready` endpoint returns `200 OK` only if the database connection and Redis connection are healthy.

---

## Observability

### Dashboards

| Dashboard | Tool | URL Pattern | Primary Audience |
|-----------|------|-------------|-----------------|
| API Latency / Error Rate | Grafana | `/d/pra-api-overview` | On-call SRE |
| Batch Job Status | Grafana | `/d/pra-batch-jobs` | Operations |
| Database Connections | Grafana | `/d/pra-postgres` | DBA / SRE |
| Service Bus Queue Depth | Azure Monitor | Portal | Operations |
| AKS Node Utilization | Grafana | `/d/pra-nodes` | SRE |

### Alert Runbooks

| Alert | Trigger | Immediate Action |
|-------|---------|-----------------|
| `APIErrorRateHigh` | Error rate > 1% for 5 min | Check pod logs: `kubectl logs -l app=svc-X -n pra-prod --tail=100` |
| `APILatencyHigh` | p99 > 2s for 5 min | Check DB connection pool: `kubectl exec -it pgbouncer -- psql -c 'SHOW POOLS'` |
| `PodCrashLooping` | Restarts > 3 in 5 min | `kubectl describe pod <name>` → check OOMKilled vs. startup failure |
| `BatchJobFailed` | Job status = Failed | Check job logs: `kubectl logs job/<job-name> -n pra-prod` → re-run via API |
| `ServiceBusDeadLetter` | DLQ depth > 0 | Inspect DLQ messages; fix consumer; re-submit events |
| `DBConnectionExhausted` | PgBouncer queue > 100 | Scale `services` node pool; investigate slow queries |

### Log Queries (Azure Monitor / Log Analytics)

```kusto
// Error rate by service (last 1 hour)
ContainerLog
| where TimeGenerated > ago(1h)
| where LogEntry contains '"level":"error"'
| extend service = tostring(parse_json(LogEntry).service)
| summarize ErrorCount = count() by service, bin(TimeGenerated, 5m)
| order by TimeGenerated desc

// Slow API requests (> 2 seconds)
ContainerLog
| where TimeGenerated > ago(1h)
| extend entry = parse_json(LogEntry)
| where toreal(entry.durationMs) > 2000
| project TimeGenerated, service=entry.service, path=entry.path,
           durationMs=entry.durationMs, correlationId=entry.correlationId
| order by durationMs desc
```

---

## Month-End Close Operations

### Recommended Close Sequence

| Step | Module | Action | Typical Duration |
|------|--------|--------|-----------------|
| 1 | M1 | Lock prior period (prevent late volume entries) | Instant |
| 2 | M1 | Run final allocation for all delivery networks | 30–90 min |
| 3 | M2 | Apply any pending ownership transfers effective in period | 5–15 min |
| 4 | M3 | Run contractual allocation | 30–60 min |
| 5 | M4 | Run valuation for all ventures | 60–240 min |
| 6 | M5 | Resolve any flagged balancing exceptions | Manual; time varies |
| 7 | M6 | Run revenue distribution | 30–60 min |
| 8 | M7 | Process outgoing owner payments | 30–60 min |
| 9 | M8 | Generate statutory reports | 30–60 min per state |
| 10 | M8 | Compliance review and file submission | Manual |

**Total automated processing window:** Target < 6 hours (batch node pool active).

### Triggering Batch Jobs via API

```bash
# Trigger allocation run for a delivery network
curl -X POST https://pra.company.com/api/v1/production/allocation/run \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"deliveryNetworkId": "dn-uuid", "period": "2024-01"}'

# Check run status
curl https://pra.company.com/api/v1/production/allocation/runs/{runId} \
  -H "Authorization: Bearer $TOKEN"

# Trigger valuation
curl -X POST https://pra.company.com/api/v1/valuation/run \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"ventureId": "all", "period": "2024-01"}'
```

---

## Database Operations

### Applying Migrations

```bash
# Applied automatically by the migration Job in the deployment pipeline
# Manual trigger (e.g., after hotfix):
kubectl run knex-migrate --image=praacr.azurecr.io/svc-production:$SHA \
  --restart=Never -n pra-prod \
  --env="DATABASE_URL=$(kubectl get secret db-secret -o jsonpath='{.data.url}' | base64 -d)" \
  -- npx knex migrate:latest
```

### Point-in-Time Recovery

```bash
# Restore PostgreSQL Flexible Server to a point in time
az postgres flexible-server restore \
  --resource-group pra-rg \
  --name pra-db-restored \
  --source-server pra-db-prod \
  --restore-time "2024-01-15T08:00:00Z"
```

---

## Routine Maintenance

| Task | Frequency | Owner | Tool |
|------|-----------|-------|------|
| Dependency updates (npm audit fix) | Weekly | DevOps | GitHub Dependabot |
| Trivy scan of running images | Daily | CI | Azure Container Registry scan policy |
| PostgreSQL VACUUM ANALYZE | Nightly | DBA | Automated (Azure Flexible Server) |
| Certificate renewal (TLS) | 90 days | cert-manager | Automatic (Let's Encrypt or Azure Key Vault) |
| Key Vault access review | Quarterly | Security | Azure RBAC review |
| Azure AD App Role review | Quarterly | Security | Entra ID review |
| ILM archive job | Monthly | M9 svc-admin | Automated ruleset |
| Partition creation (new month) | Monthly (prior month + 1) | DBA | Automated script in M9 |
| Load test (pre-month-end) | Monthly | DevOps | k6 / Azure Load Testing |
