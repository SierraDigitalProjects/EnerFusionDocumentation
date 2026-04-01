# Project Structure

## Repository Layout

EnerFusion Upstream Accounting is organised as a **monorepo** with one directory per microservice, a shared packages directory, and infrastructure-as-code at the root.

```
enerfusion-pra/                          # Monorepo root
│
├── services/                            # One directory per microservice
│   ├── svc-production/                  # M1 — Production & Volume Allocation
│   ├── svc-ownership/                   # M2 — Ownership & DOI
│   ├── svc-allocation/                  # M3 — Contractual Allocation
│   ├── svc-valuation/                   # M4 — Contracts, Pricing & Valuation
│   ├── svc-balancing/                   # M5 — Balancing Workplace
│   ├── svc-revenue/                     # M6 — Revenue Distribution
│   ├── svc-payments/                    # M7 — Payment Processing
│   ├── svc-regulatory/                  # M8 — Regulatory Reporting
│   └── svc-admin/                       # M9 — Administration & ILM
│
├── shared/                              # Shared packages (npm workspaces)
│   ├── types/                           # Shared TypeScript interfaces
│   ├── errors/                          # RFC 7807 error classes
│   ├── auth/                            # JWT decode helpers, RBAC types
│   ├── events/                          # Service Bus event schemas
│   └── validation/                      # Common Zod schemas
│
├── frontend/                            # React 18 SPA
│   └── pra-ui/
│
├── bff/                                 # Backend for Frontend (Node.js)
│   └── pra-bff/
│
├── shared-services/                     # Cross-cutting backend services
│   ├── auth-svc/                        # Token exchange, session
│   ├── audit-svc/                       # Audit log aggregation
│   ├── notification-svc/                # Email / in-app notifications
│   └── file-svc/                        # Report file storage (Azure Blob)
│
├── infra/                               # Infrastructure as Code
│   ├── helm/                            # Helm charts
│   ├── terraform/                       # Azure resource provisioning
│   └── k8s/                             # Raw Kubernetes manifests (if needed)
│
├── .github/
│   └── workflows/                       # GitHub Actions CI/CD pipelines
│
├── Documentation/                       # MkDocs documentation
│   └── (this file tree)
│
├── mkdocs.yml                           # MkDocs site configuration
├── package.json                         # Monorepo root (npm workspaces)
└── turbo.json                           # Turborepo build pipeline config
```

---

## Service Structure (Backend — per microservice)

All backend services follow this internal structure:

```
services/svc-production/
│
├── src/
│   ├── api/
│   │   ├── routes/
│   │   │   ├── wells.routes.ts          # GET /api/v1/production/wells
│   │   │   ├── wellCompletions.routes.ts
│   │   │   ├── volumes.routes.ts
│   │   │   ├── allocation.routes.ts
│   │   │   └── measurementPoints.routes.ts
│   │   │
│   │   └── middleware/
│   │       ├── auth.middleware.ts       # JWT decode + claim extraction
│   │       ├── rbac.middleware.ts       # Module × action enforcement
│   │       ├── validate.middleware.ts   # Zod request schema validation
│   │       ├── rateLimit.middleware.ts  # Per-client rate limiting
│   │       └── error-handler.ts        # Global error handler (RFC 7807)
│   │
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── WellCompletion.ts
│   │   │   ├── AllocationRun.ts
│   │   │   └── MeasurementPoint.ts
│   │   │
│   │   ├── services/
│   │   │   ├── allocation.service.ts    # 8-step allocation algorithm
│   │   │   ├── wellVolume.service.ts
│   │   │   ├── downtime.service.ts
│   │   │   └── wellTest.service.ts
│   │   │
│   │   └── events/
│   │       └── volumesAllocated.event.ts
│   │
│   ├── infrastructure/
│   │   ├── db/
│   │   │   ├── migrations/              # Knex migration files
│   │   │   │   ├── 20240101_create_wells.ts
│   │   │   │   └── 20240102_create_well_completions.ts
│   │   │   ├── seeds/                   # Dev/test seed data
│   │   │   └── repositories/
│   │   │       ├── well.repo.ts
│   │   │       ├── wellCompletion.repo.ts
│   │   │       ├── allocationRun.repo.ts
│   │   │       └── mpVolume.repo.ts
│   │   │
│   │   ├── cache/
│   │   │   └── redis.client.ts
│   │   │
│   │   └── messaging/
│   │       └── serviceBus.publisher.ts
│   │
│   ├── config/
│   │   ├── env.ts                       # Key Vault + env var resolution
│   │   └── logger.ts                    # Winston JSON logger config
│   │
│   └── index.ts                         # Express app + server bootstrap
│
├── tests/
│   ├── unit/                            # Jest unit tests (services)
│   ├── integration/                     # Jest + Supertest (routes)
│   └── contract/                        # Pact consumer tests
│
├── Dockerfile                           # Multistage Node 20 Alpine
├── helm/                                # Helm chart for this service
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── hpa.yaml
│       ├── networkpolicy.yaml
│       ├── secretproviderclass.yaml
│       └── servicemonitor.yaml
│
├── jest.config.ts
├── tsconfig.json                        # strict: true
└── package.json
```

---

## Frontend Structure (React 18)

```
frontend/pra-ui/
│
├── src/
│   ├── app/
│   │   ├── App.tsx
│   │   ├── router.tsx                   # React Router v6, lazy-loaded modules
│   │   └── store.ts                     # Redux Toolkit store
│   │
│   ├── modules/                         # One folder per Upstream Accounting module
│   │   ├── production/                  # M1
│   │   │   ├── pages/
│   │   │   │   ├── WellListPage.tsx
│   │   │   │   ├── WellCompletionPage.tsx
│   │   │   │   └── AllocationRunPage.tsx
│   │   │   ├── components/
│   │   │   │   ├── WellCompletionForm/
│   │   │   │   │   ├── WellCompletionForm.tsx
│   │   │   │   │   ├── WellCompletionForm.module.css
│   │   │   │   │   └── WellCompletionForm.test.tsx
│   │   │   │   └── AllocationRunStatus/
│   │   │   ├── store/
│   │   │   │   ├── productionSlice.ts
│   │   │   │   └── productionThunks.ts
│   │   │   ├── api/
│   │   │   │   └── productionApi.ts     # RTK Query endpoints
│   │   │   └── types/
│   │   │       └── production.types.ts  # IWellCompletion, IAllocationRun
│   │   │
│   │   ├── ownership/                   # M2
│   │   ├── contractual-allocation/      # M3
│   │   ├── valuation/                   # M4
│   │   ├── balancing/                   # M5
│   │   ├── revenue/                     # M6
│   │   ├── payments/                    # M7
│   │   ├── regulatory/                  # M8
│   │   └── admin/                       # M9
│   │
│   ├── shared/
│   │   ├── components/
│   │   │   ├── DataGrid/               # AG Grid wrapper
│   │   │   ├── StatusBadge/
│   │   │   ├── PageHeader/
│   │   │   ├── PermissionGuard/        # RBAC-aware wrapper
│   │   │   └── ConfirmDialog/
│   │   ├── hooks/
│   │   │   ├── usePermissions.ts
│   │   │   ├── usePagination.ts
│   │   │   └── useNotification.ts
│   │   ├── api/
│   │   │   └── baseApi.ts              # RTK Query base with auth header
│   │   └── auth/
│   │       ├── msalConfig.ts
│   │       └── authProvider.tsx
│   │
│   └── styles/
│       ├── tokens.css                  # Design tokens (colors, spacing)
│       ├── global.css                  # Resets only
│       └── breakpoints.css             # Media query breakpoints
│
├── public/
├── vite.config.ts
├── tsconfig.json
└── package.json
```

---

## Shared Packages

```
shared/
│
├── types/                               # Shared TypeScript interfaces
│   ├── production.types.ts             # IWell, IWellCompletion, IAllocationRun
│   ├── ownership.types.ts              # IVenture, IDoiOwnerInterest, ITransfer
│   ├── valuation.types.ts              # IContract, ISettlementStatement
│   ├── payments.types.ts               # ICheck, IOwnerPayment
│   └── common.types.ts                 # IPaginatedResponse, IProblemDetail
│
├── errors/
│   ├── ApplicationError.ts             # Base error class
│   ├── ValidationError.ts              # HTTP 422
│   ├── NotFoundError.ts                # HTTP 404
│   ├── ForbiddenError.ts               # HTTP 403
│   └── ConflictError.ts                # HTTP 409 (DOI checkout conflict)
│
├── events/
│   ├── PraEvent.ts                     # Common event envelope interface
│   ├── volumesAllocated.event.ts
│   ├── doiChanged.event.ts
│   ├── settlementCompleted.event.ts
│   └── distributionCompleted.event.ts
│
└── validation/
    ├── pagination.schema.ts            # Common query param schemas
    ├── period.schema.ts                # YYYY-MM validation
    └── uuid.schema.ts
```

---

## Infrastructure Structure

```
infra/
│
├── terraform/
│   ├── modules/
│   │   ├── aks/                        # AKS cluster + node pools
│   │   ├── postgres/                   # Azure Database for PostgreSQL
│   │   ├── redis/                      # Azure Cache for Redis
│   │   ├── servicebus/                 # Azure Service Bus namespace + topics
│   │   ├── keyvault/                   # Azure Key Vault + access policies
│   │   ├── acr/                        # Azure Container Registry
│   │   └── apim/                       # Azure API Management
│   │
│   ├── environments/
│   │   ├── dev/
│   │   │   └── main.tf
│   │   ├── staging/
│   │   │   └── main.tf
│   │   └── prod/
│   │       └── main.tf
│   │
│   └── backend.tf                      # Azure Storage remote state
│
└── helm/
    ├── charts/                         # Per-service Helm charts
    │   └── (see Service Structure above)
    └── environments/
        ├── values-dev.yaml
        ├── values-staging.yaml
        └── values-prod.yaml
```

---

## GitHub Actions Workflows

```
.github/workflows/
│
├── ci.yml                              # PR gate: lint, typecheck, test, build, scan
├── deploy-staging.yml                  # Auto-deploy on merge to main
├── deploy-production.yml               # Manual approval gate → production deploy
├── migrate.yml                         # Database migration job (pre-deploy)
├── load-test.yml                       # Monthly k6 / Azure Load Testing
└── dependency-update.yml               # Weekly Dependabot + npm audit
```

---

## Module ↔ Service ↔ Schema Reference

| UI Module | Backend Service | PostgreSQL Schema | Azure Service Bus Topics (published) |
|-----------|----------------|-------------------|--------------------------------------|
| M1 Production | svc-production | `production` | `ua.production.volumes.allocated` |
| M2 Ownership | svc-ownership | `ownership` | `ua.ownership.doi.changed` |
| M3 Allocation | svc-allocation | `allocation` | `ua.allocation.completed` |
| M4 Valuation | svc-valuation | `valuation` | `ua.valuation.settlement.completed` |
| M5 Balancing | svc-balancing | `balancing` | — |
| M6 Revenue | svc-revenue | `revenue` | `ua.revenue.distribution.completed` |
| M7 Payments | svc-payments | `payments` | — |
| M8 Regulatory | svc-regulatory | `regulatory` | — |
| M9 Admin | svc-admin | `admin_ilm` | — |
| BFF | pra-bff | — | — |
| Shared | auth-svc, audit-svc | `shared` | — |

---

## Naming Conventions Quick Reference

| Context | Convention | Example |
|---------|-----------|---------|
| File names | kebab-case | `allocation-run.service.ts` |
| Classes | PascalCase | `AllocationRunService` |
| Interfaces | PascalCase + `I` prefix | `IAllocationRun` |
| REST routes | kebab-case, versioned | `/api/v1/well-completions` |
| gRPC methods | PascalCase | `GetWellCompletion` |
| Proto messages | PascalCase | `WellCompletionResponse` |
| Kubernetes resources | kebab-case | `svc-production-deployment` |
| Helm values | camelCase | `replicaCount`, `imageTag` |
| PostgreSQL tables | snake_case | `well_completions`, `doi_owner_interests` |
| CSS classes | kebab-case + BEM | `.well-card`, `.well-card__title--active` |
| Azure resources | kebab-case | `pra-prod-kv`, `pra-db-prod` |
