# 02 — Solution Architecture
## EnerFusion Upstream Accounting — React + Node.js on Azure AKS

---

## 1. Architecture Principles

| Principle | Application |
|-----------|-------------|
| **Modular microservices** | Each of the 9 Upstream Accounting modules is an independent Node.js service with its own API, database schema, and deployment unit |
| **API-first** | Every module exposes versioned REST APIs consumed by the React frontend and by external integrators |
| **RBAC at every layer** | Authorization enforced in API Gateway, service middleware, and row-level PostgreSQL policies |
| **Scale-out by design** | Stateless services, horizontal pod autoscaling, connection pooling, read replicas |
| **Event-driven handoffs** | Inter-module data propagation via Azure Service Bus to avoid tight coupling |
| **Immutable containers** | All services containerized via Docker, deployed to AKS via Helm |

---

## 2. High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL CONSUMERS                             │
│   Field SCADA / ERP  │  State Agencies  │  ONRR  │  3rd-party BI       │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ HTTPS / REST
┌───────────────────────────────▼─────────────────────────────────────────┐
│                     AZURE API MANAGEMENT (APIM)                         │
│  Rate limiting │ OAuth2/OIDC token validation │ Module routing          │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────────┐
│                    AZURE KUBERNETES SERVICE (AKS)                       │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    INGRESS (NGINX / AGIC)                        │  │
│  └────────────────────────────────┬─────────────────────────────────┘  │
│                                   │                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │  React SPA   │  │  React SPA   │  │  Admin UI    │                  │
│  │  (Workbench) │  │  (Reports)   │  │  (ILM/RBAC)  │                  │
│  └──────────────┘  └──────────────┘  └──────────────┘                  │
│          │                  │                  │                        │
│  ┌───────▼──────────────────▼──────────────────▼──────────┐           │
│  │              BFF — Backend For Frontend (Node.js)       │           │
│  │   Session handling │ Token exchange │ Response shaping  │           │
│  └─────────────────────────────────┬───────────────────────┘           │
│                                    │                                    │
│  ┌─────────────────────────────────▼───────────────────────────────┐   │
│  │              INTERNAL SERVICE MESH (Linkerd / Istio)             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │ svc-prod │ │ svc-own  │ │ svc-alloc│ │ svc-val  │ │ svc-bal  │    │
│  │  M1      │ │  M2      │ │  M3      │ │  M4      │ │  M5      │    │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                  │
│  │ svc-rev  │ │ svc-pay  │ │ svc-reg  │ │ svc-adm  │                  │
│  │  M6      │ │  M7      │ │  M8      │ │  M9      │                  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘                  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                   SHARED SERVICES LAYER                          │  │
│  │  auth-svc  │  notification-svc  │  audit-svc  │  file-svc       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────────┐
│                        DATA TIER (AZURE)                                 │
│                                                                          │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  Azure Database for  │  │  Azure Cache for  │  │  Azure Blob      │   │
│  │  PostgreSQL Flex    │  │  Redis            │  │  Storage         │   │
│  │  (Primary + Replica)│  │  (Session/Cache)  │  │  (Reports/Docs)  │   │
│  └─────────────────────┘  └──────────────────┘  └──────────────────┘   │
│                                                                          │
│  ┌─────────────────────┐  ┌──────────────────┐                         │
│  │  Azure Service Bus  │  │  Azure Monitor /  │                         │
│  │  (Module Events)    │  │  App Insights     │                         │
│  └─────────────────────┘  └──────────────────┘                         │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Frontend Architecture (React 18)

### Application Shell

```
src/
├── app/                        # Root app, routing, theme
│   ├── App.tsx
│   ├── router.tsx              # React Router v6 with lazy loading
│   └── store.ts                # Redux Toolkit global store
│
├── modules/                    # One folder per Upstream Accounting module
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
│   ├── components/             # Reusable UI components
│   ├── hooks/                  # Custom React hooks
│   ├── api/                    # Axios clients per module
│   ├── auth/                   # MSAL (Azure AD) integration
│   └── rbac/                   # Permission-aware component wrappers
│
└── assets/
```

### Module Structure (per module)

```
modules/production/
├── pages/                      # Route-level page components
│   ├── WellCompletionPage.tsx
│   ├── MeasurementPointPage.tsx
│   └── DeliveryNetworkPage.tsx
├── components/                 # Module-specific components
├── store/                      # Redux slice for this module
│   ├── productionSlice.ts
│   └── productionThunks.ts
├── api/                        # API calls for this module
│   └── productionApi.ts
└── types/                      # TypeScript types
    └── production.types.ts
```

### RBAC in the Frontend

```typescript
// Declarative permission guard
<PermissionGuard module="production" action="write">
  <WellVolumeEntryForm />
</PermissionGuard>

// Hook-based check
const { can } = usePermissions();
if (can('valuation', 'approve')) { ... }
```

### Key Frontend Technology Choices

| Concern | Technology | Rationale |
|---------|------------|-----------|
| Framework | React 18 + TypeScript | Concurrent mode, strong typing for complex domain |
| State management | Redux Toolkit + RTK Query | Predictable state, auto-caching API calls |
| UI library | Ant Design or MUI | Data-dense tables, forms required by PRA |
| Auth | MSAL.js (Azure AD) | Enterprise SSO, token refresh |
| Charts | Recharts / Apache ECharts | Volume trends, allocation charts |
| Data grids | AG Grid Community | High-volume tabular data (100k+ rows virtual scroll) |
| Forms | React Hook Form + Zod | Complex multi-step forms with validation |

---

## 4. Backend Architecture (Node.js)

### Service Structure (per microservice)

```
svc-production/
├── src/
│   ├── api/
│   │   ├── routes/             # Express route definitions
│   │   │   ├── wells.routes.ts
│   │   │   ├── volumes.routes.ts
│   │   │   └── allocation.routes.ts
│   │   └── middleware/
│   │       ├── auth.middleware.ts       # JWT validation
│   │       ├── rbac.middleware.ts       # Permission enforcement
│   │       ├── rateLimit.middleware.ts
│   │       └── validate.middleware.ts   # Zod schema validation
│   │
│   ├── domain/
│   │   ├── entities/           # Domain objects
│   │   ├── services/           # Business logic
│   │   └── events/             # Domain events (Service Bus)
│   │
│   ├── infrastructure/
│   │   ├── db/                 # PostgreSQL via pg / Knex
│   │   │   ├── migrations/
│   │   │   ├── seeds/
│   │   │   └── repositories/
│   │   ├── cache/              # Redis client
│   │   └── messaging/          # Azure Service Bus publisher
│   │
│   └── config/
│       ├── env.ts
│       └── logger.ts           # Winston + structured JSON logs
│
├── Dockerfile
├── helm/                       # Helm chart for this service
└── package.json
```

### Key Backend Technology Choices

| Concern | Technology | Rationale |
|---------|------------|-----------|
| Runtime | Node.js 20 LTS | High I/O throughput, large npm ecosystem |
| Framework | Express.js | Lightweight, flexible middleware |
| ORM / Query | Knex.js + custom repositories | SQL control for complex PRA queries |
| Auth | jsonwebtoken + Azure AD JWKS | Stateless JWT, integrates with APIM |
| Validation | Zod | Runtime type safety at API boundary |
| Messaging | @azure/service-bus | Inter-module event propagation |
| Logging | Winston (JSON) → Azure Monitor | Structured logs for correlation |
| Testing | Jest + Supertest | Unit + integration tests per service |

---

## 5. Database Architecture (PostgreSQL)

### Schema-per-Module Pattern

Each module owns its database schema, enforcing domain isolation:

```sql
-- Separate schemas per module
CREATE SCHEMA production;    -- M1
CREATE SCHEMA ownership;     -- M2
CREATE SCHEMA allocation;    -- M3
CREATE SCHEMA valuation;     -- M4
CREATE SCHEMA balancing;     -- M5
CREATE SCHEMA revenue;       -- M6
CREATE SCHEMA payments;      -- M7
CREATE SCHEMA regulatory;    -- M8
CREATE SCHEMA admin_ilm;     -- M9
CREATE SCHEMA shared;        -- Cross-module reference data
```

### Row-Level Security for RBAC

```sql
-- Example: operators can only see their own company's wells
ALTER TABLE production.well_completions ENABLE ROW LEVEL SECURITY;

CREATE POLICY well_completion_tenant_isolation
ON production.well_completions
USING (company_code = current_setting('app.company_code'));
```

### Scaling Pattern

```
Azure Database for PostgreSQL — Flexible Server
├── Primary (reads + writes)
│   └── Connection pooling via PgBouncer (sidecar)
├── Read Replica 1 (reporting queries, M8)
├── Read Replica 2 (valuation batch jobs, M4)
└── Backup → Azure Blob Storage (point-in-time recovery)
```

---

## 6. Inter-Module Event Architecture

Modules communicate via **Azure Service Bus** topics to avoid synchronous coupling:

```
Topic: ua.production.volumes.allocated
  └── Subscriber: svc-allocation (M3)
  └── Subscriber: svc-regulatory (M8)

Topic: ua.ownership.doi.changed
  └── Subscriber: svc-allocation (M3)
  └── Subscriber: svc-valuation (M4)

Topic: ua.valuation.settlement.completed
  └── Subscriber: svc-revenue (M6)
  └── Subscriber: svc-payments (M7)

Topic: ua.revenue.distribution.completed
  └── Subscriber: svc-payments (M7)
  └── Subscriber: svc-regulatory (M8)
```

All events include: `eventId`, `moduleSource`, `companyCode`, `period`, `timestamp`, `correlationId`.

---

## 7. Authentication & Authorization Flow

```
User → Azure AD (Entra ID)
       ↓  OAuth2 Authorization Code + PKCE
Azure AD → Access Token (JWT)
       ↓
React SPA attaches Bearer token to all requests
       ↓
Azure APIM → validates token signature, extracts claims
       ↓
BFF / Service → rbac.middleware.ts:
  1. Decode JWT claims: roles[], companyCode, userId
  2. Load permission map from Redis cache (TTL 5 min)
  3. Enforce: module × action × companyCode
  4. Set pg session variable: app.company_code for RLS
```

---

## 8. AKS Cluster Topology

```
AKS Cluster: pra-prod
│
├── Node Pool: system (3 nodes, Standard_D4s_v3)
│   └── System pods, ingress, cert-manager
│
├── Node Pool: services (auto-scale 3–15, Standard_D8s_v3)
│   └── All PRA microservices (HPA: CPU 70% threshold)
│
├── Node Pool: batch (auto-scale 0–10, Standard_F16s_v2)
│   └── Valuation batch, regulatory report generation
│
└── Node Pool: frontend (2 nodes, Standard_D4s_v3)
    └── React SPA containers, BFF

Namespaces:
  pra-prod      → production workloads
  pra-shared    → auth, audit, notification services
  monitoring    → Prometheus, Grafana
  ingress-nginx → NGINX ingress controller
```

### Helm Release Structure

```
helm/
├── charts/
│   ├── svc-production/
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
