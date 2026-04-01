# Technical Architecture

## Stack Summary

| Layer | Technology | Version |
|-------|------------|---------|
| Frontend | React + TypeScript | React 18, TS 5.x |
| State Management | Redux Toolkit + RTK Query | Latest |
| UI Components | Ant Design / MUI | Latest stable |
| Data Grid | AG Grid Community | Latest |
| Auth (client) | MSAL.js (Azure AD) | v3 |
| Backend | Node.js + Express | Node 20 LTS |
| Language | TypeScript | 5.x |
| ORM | Knex.js + custom repositories | Latest |
| Validation | Zod | Latest |
| Messaging | Azure Service Bus | Standard tier |
| Database | PostgreSQL | 15+ (Azure Flexible Server) |
| Cache | Azure Cache for Redis | Latest |
| Container runtime | Docker | 20.x+ |
| Orchestration | Azure Kubernetes Service | K8s 1.28+ |
| Package manager | Helm | 3.x |
| Ingress | NGINX Ingress Controller | Latest |
| API Gateway | Azure API Management | Consumption / Standard |
| Logging | Winston (JSON) | Latest |
| Tracing | OpenTelemetry SDK | Latest |
| Metrics | prom-client | Latest |
| Testing (backend) | Jest + Supertest | 29.x |
| Testing (frontend) | Vitest + React Testing Library | 1.x |
| Contract testing | Pact | Latest |

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       EXTERNAL CONSUMERS                                 │
│  Field SCADA  │  SAP FI/CO (GL)  │  State Agencies  │  BI / Analytics   │
└──────────────────────────┬───────────────────────────────────────────────┘
                           │ HTTPS / REST
                           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                 AZURE API MANAGEMENT (APIM)                              │
│  OAuth2/OIDC token validation │ Rate limiting │ Module route mapping     │
└──────────────────────────┬───────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                  AZURE KUBERNETES SERVICE (AKS)                          │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  NGINX Ingress (pra-prod namespace)                                │  │
│  └─────────────────────┬──────────────────────────────────────────────┘  │
│                        │                                                 │
│  ┌─────────────────────▼──────────────────────────────────────────────┐  │
│  │  BFF — Backend for Frontend (Node.js)                              │  │
│  │  Session management │ Token exchange │ Response aggregation        │  │
│  └─────────────────────┬──────────────────────────────────────────────┘  │
│                        │                                                 │
│  ┌─────────────────────▼──────────────────────────────────────────────┐  │
│  │  Microservices (pra-prod namespace)                                 │  │
│  │                                                                    │  │
│  │  svc-production  svc-ownership  svc-allocation  svc-valuation      │  │
│  │  svc-balancing   svc-revenue    svc-payments    svc-regulatory     │  │
│  │  svc-admin                                                         │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Shared Services (pra-shared namespace)                             │  │
│  │  auth-svc │ audit-svc │ notification-svc │ file-svc               │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                        DATA TIER (AZURE)                                 │
│                                                                          │
│  Azure PostgreSQL Flexible Server     Azure Cache for Redis              │
│  (Primary + Read Replica)             (Session cache, permission cache)  │
│                                                                          │
│  Azure Service Bus (Standard)         Azure Blob Storage                 │
│  (Inter-module events / topics)       (Reports, archived records)        │
│                                                                          │
│  Azure Key Vault                      Azure Container Registry (ACR)     │
│  (Secrets, TLS certs, CMK)            (Service container images)         │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Frontend Architecture (React 18)

### Application Shell

```
src/
├── app/
│   ├── App.tsx                 # Root with React Router v6, lazy loading
│   ├── router.tsx              # Module-level route definitions
│   └── store.ts                # Redux Toolkit global store
│
├── modules/                    # One folder per Upstream Accounting module (M1–M9)
│   ├── production/             # M1
│   ├── ownership/              # M2
│   ├── contractual-allocation/ # M3
│   ├── valuation/              # M4
│   ├── balancing/              # M5
│   ├── revenue/                # M6
│   ├── payments/               # M7
│   ├── regulatory/             # M8
│   └── admin/                  # M9
│
├── shared/
│   ├── components/             # Reusable UI: DataGrid, StatusBadge, FormField
│   ├── hooks/                  # usePermissions, usePagination, useNotification
│   ├── api/                    # RTK Query base API + per-module API slices
│   ├── auth/                   # MSAL.js integration, token management
│   └── rbac/                   # PermissionGuard component, usePermissions hook
│
└── styles/
    ├── tokens.css              # Design tokens (colors, spacing, typography)
    ├── global.css              # Resets and body defaults only
    └── breakpoints.css         # Responsive breakpoint tokens
```

### Module Folder Structure

```
modules/production/
├── pages/
│   ├── WellListPage.tsx
│   ├── WellCompletionPage.tsx
│   └── AllocationRunPage.tsx
├── components/
│   ├── WellCompletionForm/
│   │   ├── WellCompletionForm.tsx
│   │   ├── WellCompletionForm.module.css
│   │   └── WellCompletionForm.test.tsx
│   └── AllocationRunStatus/
├── store/
│   ├── productionSlice.ts
│   └── productionThunks.ts
├── api/
│   └── productionApi.ts        # RTK Query endpoints
└── types/
    └── production.types.ts     # IWellCompletion, IAllocationRun, etc.
```

### RBAC in the Frontend

```typescript
// Declarative guard
<PermissionGuard module="production" action="write">
  <WellCompletionForm />
</PermissionGuard>

// Hook-based check
const { can } = usePermissions();
const canApproveTransfer = can('ownership', 'approve');

// RTK Query with permission-aware fetch
const { data: settlements } = useGetSettlementsQuery(
  { period: '2024-01' },
  { skip: !can('valuation', 'read') }
);
```

---

## Backend Architecture (Node.js)

### Service Structure

```
svc-production/
├── src/
│   ├── api/
│   │   ├── routes/
│   │   │   ├── wells.routes.ts
│   │   │   ├── wellCompletions.routes.ts
│   │   │   ├── volumes.routes.ts
│   │   │   └── allocation.routes.ts
│   │   └── middleware/
│   │       ├── auth.middleware.ts        # JWT decode + claim extraction
│   │       ├── rbac.middleware.ts        # Module × action enforcement
│   │       ├── rateLimit.middleware.ts
│   │       ├── validate.middleware.ts    # Zod schema validation
│   │       └── error-handler.ts         # Global error handler (RFC 7807)
│   │
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── WellCompletion.ts
│   │   │   └── AllocationRun.ts
│   │   ├── services/
│   │   │   ├── allocation.service.ts     # 8-step allocation algorithm
│   │   │   ├── wellVolume.service.ts
│   │   │   └── downtime.service.ts
│   │   └── events/
│   │       └── volumesAllocated.event.ts
│   │
│   ├── infrastructure/
│   │   ├── db/
│   │   │   ├── migrations/               # Knex migration files
│   │   │   ├── seeds/
│   │   │   └── repositories/
│   │   │       ├── wellCompletion.repo.ts
│   │   │       └── allocationRun.repo.ts
│   │   ├── cache/
│   │   │   └── redis.client.ts
│   │   └── messaging/
│   │       └── serviceBus.publisher.ts
│   │
│   └── config/
│       ├── env.ts                        # Azure Key Vault + env var config
│       └── logger.ts                     # Winston JSON logger
│
├── Dockerfile                            # Multistage Node 20 Alpine
├── helm/                                 # Helm chart for this service
├── jest.config.ts
└── package.json
```

### Global Error Handler (RFC 7807)

```typescript
// middleware/error-handler.ts
export const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  const problem: ProblemDetail = {
    type: err.type ?? 'about:blank',
    title: err.title ?? 'Internal Server Error',
    status: err.status ?? 500,
    detail: err.message,
    instance: req.path,
    traceId: req.headers['traceparent'] as string,
  };
  logger.error({ ...problem, stack: err.stack });
  res.status(problem.status).json(problem);
};
```

All controllers call `next(err)` — no inline try-catch in route handlers.

---

## Authentication & Authorization Flow

```
Browser
  │  1. Login → Azure AD (Entra ID)
  │     Authorization Code + PKCE
  ▼
Azure AD issues Access Token (JWT, 1hr TTL)
  │
  │  2. React SPA stores token (MSAL.js in-memory)
  │     Attaches: Authorization: Bearer <token>
  ▼
Azure APIM
  │  3. Validates JWT signature via Azure AD JWKS endpoint
  │     Checks: audience, issuer, expiry
  │     Forwards: decoded claims header to backend
  ▼
BFF / Microservice
  │  4. auth.middleware.ts: decode claims
  │     Extract: roles[], companyCode, userId, oid
  │
  │  5. rbac.middleware.ts: load permission map from Redis (TTL 5min)
  │     Enforce: module × action × companyCode match
  │
  │  6. Set PostgreSQL session variable:
  │     SET app.company_code = '<companyCode>'
  │     RLS policies use current_setting('app.company_code')
  ▼
Repository Layer → PostgreSQL (RLS active)
```

---

## Database Architecture

### Schema-Per-Module

```sql
CREATE SCHEMA production;    -- M1
CREATE SCHEMA ownership;     -- M2
CREATE SCHEMA allocation;    -- M3
CREATE SCHEMA valuation;     -- M4
CREATE SCHEMA balancing;     -- M5
CREATE SCHEMA revenue;       -- M6
CREATE SCHEMA payments;      -- M7
CREATE SCHEMA regulatory;    -- M8
CREATE SCHEMA admin_ilm;     -- M9
CREATE SCHEMA shared;        -- Business partners, GL accounts (replicated)
```

### Row-Level Security Example

```sql
ALTER TABLE ownership.doi_owner_interests ENABLE ROW LEVEL SECURITY;

CREATE POLICY doi_tenant_isolation
ON ownership.doi_owner_interests
USING (company_code = current_setting('app.company_code'));
```

### Connection Pooling

PgBouncer runs as a sidecar container alongside each service pod in transaction pooling mode:

```yaml
# Helm values
pgbouncer:
  maxClientConnections: 100
  defaultPoolSize: 20
  poolMode: transaction
```

### Read Replica Routing

```typescript
// infrastructure/db/knex.config.ts
export const writeDb = knex({ connection: process.env.DB_WRITE_URL });
export const readDb  = knex({ connection: process.env.DB_READ_URL });

// Repositories use readDb for SELECT, writeDb for INSERT/UPDATE/DELETE
```

---

## Inter-Module Messaging (Azure Service Bus)

```
Topics and Subscriptions:

ua.production.volumes.allocated
  └── subscription: svc-allocation   (M3)
  └── subscription: svc-regulatory   (M8)

ua.ownership.doi.changed
  └── subscription: svc-allocation   (M3)
  └── subscription: svc-valuation    (M4)

ua.allocation.completed
  └── subscription: svc-valuation    (M4)
  └── subscription: svc-regulatory   (M8)

ua.valuation.settlement.completed
  └── subscription: svc-revenue      (M6)
  └── subscription: svc-payments     (M7)

ua.revenue.distribution.completed
  └── subscription: svc-payments     (M7)
  └── subscription: svc-regulatory   (M8)
```

All events carry `eventId` for consumer idempotency. Dead-letter queues route failed messages to the audit service after 3 retries.

---

## AKS Cluster Topology

```
AKS Cluster: pra-prod (Single region; zone-redundant)
│
├── Node Pool: system (3 nodes, Standard_D4s_v3, zone-spread)
│   └── AKS system pods, cert-manager, CSI drivers
│
├── Node Pool: services (auto-scale 3–15, Standard_D8s_v3)
│   └── All 9 PRA microservices + shared services
│   └── HPA: CPU 70% → scale out; 30% → scale in
│
├── Node Pool: batch (auto-scale 0–10, Standard_F16s_v2)
│   └── Valuation batch runs (M4), regulatory report generation (M8)
│   └── Scales to 0 when no jobs queued
│
└── Node Pool: frontend (2 nodes fixed, Standard_D4s_v3)
    └── React SPA containers, BFF Node.js

Namespaces:
  pra-prod      → application workloads
  pra-shared    → auth, audit, notification, file services
  monitoring    → Prometheus, Grafana, OpenTelemetry Collector
  ingress-nginx → NGINX Ingress Controller
```

---

## Observability Stack

```
Application
  │  Winston JSON logs → stdout
  │  OpenTelemetry SDK → traces + metrics
  ▼
OpenTelemetry Collector (DaemonSet)
  │
  ├── Logs → Azure Monitor Log Analytics
  ├── Traces → Azure Application Insights
  └── Metrics → Prometheus (scraped) → Azure Monitor / Grafana
```

Every log entry format:

```json
{
  "level": "info",
  "service": "svc-production",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "correlationId": "req-abc123",
  "userId": "user-uuid-here",
  "companyCode": "OPERATOR1",
  "message": "Allocation run completed",
  "deliveryNetworkId": "dn-uuid",
  "period": "2024-01",
  "runStatus": "completed"
}
```
