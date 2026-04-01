# Non-Functional Requirements

Requirements use the format: `NFR-[Category]-[###]`

---

## Performance

| ID | Requirement | Target | Rationale |
|----|-------------|--------|-----------|
| NFR-PERF-001 | API p99 response time for read operations on master data lists | < 500 ms | Accountants browse large DOI and contract lists frequently |
| NFR-PERF-002 | API p99 response time for single-record retrieval | < 200 ms | |
| NFR-PERF-003 | Allocation run execution for a delivery network with 500 wells | < 60 seconds | Month-end networks must complete within batch window |
| NFR-PERF-004 | Valuation run execution for 10,000 settlement statements | < 4 hours | Must complete within the nightly batch window |
| NFR-PERF-005 | Revenue distribution run for 50,000 owner interest records | < 2 hours | |
| NFR-PERF-006 | Regulatory report generation (e.g., Form H-10 — 5,000 wells) | < 30 minutes | Generated asynchronously; user notified on completion |
| NFR-PERF-007 | Mass check input — 10,000 purchaser check records | < 15 minutes | Batch job with progress tracking |
| NFR-PERF-008 | UI list views for datasets > 10,000 records | Virtual scroll; server-side pagination (page size 100) | AG Grid virtual rendering |
| NFR-PERF-009 | Azure Monitor alert threshold — API error rate | > 1% → PagerDuty alert | Per CLAUDE.md observability standard |
| NFR-PERF-010 | Azure Monitor alert threshold — p99 latency | > 2 seconds → PagerDuty alert | Per CLAUDE.md observability standard |

---

## Scalability

| ID | Requirement | Target | Notes |
|----|-------------|--------|-------|
| NFR-SCALE-001 | Well Completion master records | Up to 500,000 per tenant | Large operator in multiple basins |
| NFR-SCALE-002 | DOI Owner Interest records | Up to 2,000,000 per tenant | Complex royalty structures in Permian/Midcontinent |
| NFR-SCALE-003 | Daily volume transactions per month | Up to 15,000,000 rows | 500,000 WCs × 30 days |
| NFR-SCALE-004 | Concurrent API users during month-end | Up to 500 simultaneous sessions | |
| NFR-SCALE-005 | Horizontal pod autoscaling trigger | CPU utilization > 70% → scale out | HPA on `services` node pool |
| NFR-SCALE-006 | Batch node pool scale-to-zero | Pool scales to 0 nodes when no batch jobs are queued | Cost optimisation |
| NFR-SCALE-007 | Database connection pooling | PgBouncer sidecar per service; max 20 connections per pod | Prevents connection exhaustion on PostgreSQL Flexible Server |
| NFR-SCALE-008 | Redis caching — permission maps | TTL 5 minutes; hit rate > 90% under normal load | RBAC permission cache |
| NFR-SCALE-009 | Read replica offload | Reporting queries (M8) and batch reads (M4 valuation) routed to read replica | Primary handles writes only during batch |

---

## Availability

| ID | Requirement | Target | Notes |
|----|-------------|--------|-------|
| NFR-AVAIL-001 | API services availability (SLA) | 99.9% monthly uptime | ~44 minutes allowable downtime per month |
| NFR-AVAIL-002 | Batch job availability (off-peak window) | 99.5% within scheduled window | Non-critical compared to API SLA |
| NFR-AVAIL-003 | Database availability | Azure PostgreSQL Flexible Server HA (zone-redundant) | Automatic failover < 30 seconds |
| NFR-AVAIL-004 | Health endpoints | Every service must expose `/health` (liveness) and `/ready` (readiness) | AKS probe required |
| NFR-AVAIL-005 | Graceful shutdown | Pods must drain in-flight requests within 30 seconds on SIGTERM | `terminationGracePeriodSeconds: 30` |
| NFR-AVAIL-006 | Circuit breaker | All outgoing calls to external services (SAP GL posting, state filing APIs, price feeds) wrapped in `opossum` circuit breaker | Per CLAUDE.md dependency |

---

## Security

| ID | Requirement | Target / Rule | Notes |
|----|-------------|---------------|-------|
| NFR-SEC-001 | JWT validation | Every request to every service must validate the Azure AD JWT using JWKS endpoint before processing | |
| NFR-SEC-002 | Token expiry | Access tokens must not exceed 1-hour lifetime; refresh token rotation enabled | Azure AD app configuration |
| NFR-SEC-003 | Secrets management | All secrets (DB passwords, Service Bus keys, API keys) stored in Azure Key Vault; mounted via CSI secret store driver | No plaintext secrets in Helm values or env vars |
| NFR-SEC-004 | Row-Level Security | PostgreSQL RLS enabled on all tenant-scoped tables; `app.company_code` session variable set per request | |
| NFR-SEC-005 | Network policies | Every microservice namespace must have a Kubernetes NetworkPolicy restricting ingress to known services | Default-deny; explicit allow-list |
| NFR-SEC-006 | Container image scanning | All images scanned with Trivy before push to ACR; critical/high CVE blocks deployment | CI gate |
| NFR-SEC-007 | TLS | All inter-service communication within the mesh encrypted via mTLS (Linkerd / Istio) | |
| NFR-SEC-008 | Data encryption at rest | Azure PostgreSQL and Blob Storage encrypted with customer-managed keys (Azure Key Vault) | |
| NFR-SEC-009 | Personal data protection | Business partner tax IDs encrypted at the application layer (AES-256) before storage | Column-level encryption |
| NFR-SEC-010 | Audit logging | All data mutations logged to `admin_ilm.audit_log` with user ID, timestamp, table, record ID, before/after JSON | Tamper-evident; append-only |
| NFR-SEC-011 | OWASP Top 10 | All API endpoints must pass OWASP ZAP scan in CI before production deployment | |
| NFR-SEC-012 | Input validation | All external inputs validated with Zod schemas at the API boundary before processing | Per CLAUDE.md coding standard |

---

## Reliability & Data Integrity

| ID | Requirement | Rule |
|----|-------------|------|
| NFR-REL-001 | Database migrations | Prisma Migrate for all schema changes; migrations must be backwards-compatible (additive-only) for zero-downtime rolling deploys |
| NFR-REL-002 | Idempotent batch operations | Allocation runs, valuation runs, and distribution runs must be idempotent — re-running a completed run for the same period must not create duplicate records |
| NFR-REL-003 | Event at-least-once delivery | Azure Service Bus inter-module events delivered at-least-once; consumers must be idempotent |
| NFR-REL-004 | Transactional consistency | Multi-table writes within a single module use PostgreSQL transactions; cross-module writes use the outbox pattern via Service Bus |
| NFR-REL-005 | Data retention | Transaction records retained for 7 years (minimum US tax record retention); ILM archiving triggers after 7 years |
| NFR-REL-006 | Backup and recovery | Azure PostgreSQL point-in-time recovery; RPO < 1 hour; RTO < 4 hours |
| NFR-REL-007 | DOI decimal integrity | Decimal interest validation enforced at API and database constraint level; failing validation returns HTTP 422 |

---

## Observability

| ID | Requirement | Implementation |
|----|-------------|----------------|
| NFR-OBS-001 | Structured logging | Winston JSON logger; every log entry includes `service`, `timestamp`, `traceId`, `correlationId`, `userId` |
| NFR-OBS-002 | Distributed tracing | OpenTelemetry SDK; `traceparent` header propagated across all HTTP and gRPC calls |
| NFR-OBS-003 | Metrics | Prometheus metrics exposed at `/metrics` via `prom-client`; scraped by Azure Monitor |
| NFR-OBS-004 | Dashboards | Grafana dashboard per module: request rate, error rate, p99 latency, active pods |
| NFR-OBS-005 | Alerting | Azure Monitor alerts for: error rate > 1%, p99 > 2s, pod restarts > 3 in 5 min, batch job failure |
| NFR-OBS-006 | Correlation IDs | Every HTTP request assigned a `correlationId`; propagated in all downstream service calls and log entries |

---

## Maintainability

| ID | Requirement | Standard |
|----|-------------|----------|
| NFR-MAINT-001 | TypeScript strict mode | `strict: true` in all `tsconfig.json` files; no `any` types |
| NFR-MAINT-002 | Test coverage | Unit tests for all service-layer business logic; minimum 80% line coverage |
| NFR-MAINT-003 | Contract tests | Pact consumer-driven contract tests for all inter-module API integrations |
| NFR-MAINT-004 | CI gates | Every PR must pass: ESLint, `tsc --noEmit`, unit tests, contract tests, Docker build, Trivy scan |
| NFR-MAINT-005 | Image tags | Git commit SHA used as image tag in all Kubernetes manifests; `latest` tag forbidden in production |
| NFR-MAINT-006 | Module independence | No direct database cross-joins between module schemas; cross-module data access via API or event |
| NFR-MAINT-007 | API versioning | All routes versioned at `/api/v1/`; breaking changes require `/api/v2/` — no in-place breaking changes |

---

## Compliance

| ID | Requirement | Standard |
|----|-------------|----------|
| NFR-COMP-001 | Data residency | All data stored in US Azure regions (e.g., East US 2 primary, West US 3 DR) | 
| NFR-COMP-002 | SOC 2 Type II alignment | Audit logging, access controls, and change management controls designed to support SOC 2 audit evidence |
| NFR-COMP-003 | 1099 / tax reporting | Business partner tax IDs retained and accessible for IRS 1099-MISC / 1099-NEC generation |
| NFR-COMP-004 | Regulatory report accuracy | Statutory report values must reconcile to source production and revenue data to within rounding tolerance defined per state |
