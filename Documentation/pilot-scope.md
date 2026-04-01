# Pilot Scope — Minimum Viable Solution

---

## 1. Purpose

This document defines the minimum viable solution (MVS) required to execute the [Build-A-Biz end-to-end test scenario](test-cases/e2e-scenario.md). It is the development target for the EnerFusion pilot phase — the smallest complete vertical slice through all nine modules that proves the end-to-end revenue lifecycle works before full-scale feature build-out begins.

The pilot is **not** a prototype. All code written must meet the architectural standards defined in CLAUDE.md (TypeScript strict mode, Prisma migrations, RBAC middleware, structured logging, OpenTelemetry tracing). It becomes the foundation for subsequent feature sprints.

---

## 2. Pilot Objectives

| # | Objective |
|---|-----------|
| 1 | Execute all P1 — Blocker test cases from E2E Phase 1 without manual workarounds |
| 2 | Execute all P1 — Blocker test cases from E2E Phase 2 |
| 3 | Demonstrate data flowing unbroken from wellhead measurement (M1) through owner payment record (M7) |
| 4 | Demonstrate a prior-period adjustment (PPN) propagating correctly through CA and valuation |
| 5 | Demonstrate SAP FI GL posting via a stubbed integration that can be upgraded to live BAPI later |
| 6 | All data isolated per `company_code` via PostgreSQL RLS from day one |

---

## 3. Module-by-Module Pilot Scope

### M1 — Production & Volume Allocation

**In Pilot Scope:**

| Feature | Notes |
|---------|-------|
| Field, Reservoir, Well, Well Completion master data | Full CRUD + dated characteristics |
| Measurement Point (MP) creation and dated config | Tank Lease, Tank Sales, Gas Meter MPs |
| Tank strapping table (basic linear interpolation) | Sufficient for test data volumes |
| Well test entry and theoretical production derivation | Used as allocation basis |
| Downtime recording and uptime fraction calculation | Influences fuel consumption calc |
| Tank gauge readings (open/close) → gross volume | With temperature/pressure correction to standard conditions |
| Gas meter volume entry with seal identifier | Standard condition conversion |
| Production Volume Allocation run (Oil DN, Gas DN) | Both DNs must balance to measurements |
| Allocated WC volume results (by volume type) | Viewable via API |
| Non-allocated DN (manual MP volume entry) | Required for outside-operated test case |
| `ua.production.volumes.allocated` Service Bus event | Triggers M3 |

**Deferred (Post-Pilot):**

| Feature | Reason |
|---------|--------|
| Production regulatory reports (state production filings) | M8 covers regulatory; M1 state reports deferred to Phase 2 |
| Automated well test scheduling | Manual entry sufficient for pilot |
| CO₂ / inert gas OGOR deduction | Not exercised in Build-A-Biz scenario |
| Multiple allocation method variants beyond well-test | Only well-test method needed |

---

### M2 — Ownership & DOI

**In Pilot Scope:**

| Feature | Notes |
|---------|-------|
| Business Partner create/read (customer and vendor types) | Affiliate flag, intercompany code, owner type (WI/RI/ORRI), payment method |
| Venture create and JV configuration | Equity percentage, cost center link |
| Base DOI create with decimal interests | Interest types: WI, RI, ORRI |
| DOI balance validation (must sum to 1.0) | Enforced pre-approval |
| DOI approval and check-in workflow | Two-step: approve → check-in |
| Division Order (DO) with effective dating | Dated characteristic support |
| Bearer Group (WI burdened royalty shares) | Required for royalty burden allocation |
| DOI Accounting configuration | Payment method, minimum threshold, withholding % |
| Owner transfer request (with funds) | Farmer Ted → Son 1 + Son 2 |
| Owner transfer request (correction / no funds) | DOI correction triggering PPN |
| Transfer release → approve → execute workflow | Full lifecycle |
| Chain of title record | Immutable history of ownership changes |
| Owner suspense flag and reason code | Son 2 (no address) |
| WC ↔ DOI cross-reference | Links production entities to ownership |
| `ua.ownership.doi.changed` Service Bus event | Triggers PPN generation |

**Deferred (Post-Pilot):**

| Feature | Reason |
|---------|--------|
| Unit ventures and tract participation factors | Not in Build-A-Biz scenario |
| Bulk DOI import / migration tooling | Phase 1 data migration sprint |
| JIB (Joint Interest Billing) partner setup | Post-pilot finance feature |
| FIRPTA withholding for foreign owners | Tax/Legal dependency (OQ-B-002) |

---

### M3 — Contractual Allocation

**In Pilot Scope:**

| Feature | Notes |
|---------|-------|
| Marketing group configuration | One Oil group (DO Sales), two Gas groups (WI-owner marketed) |
| Contract sales template generation | Per DN + period |
| Pipeline actuals entry against template | Gas actuals from operator statement |
| Contractual Allocation run (Oil and Gas DNs) | Both allocation methods: `division_order_sales`, `sales_point_originated` |
| Take-In-Kind (TIK) contract handling | JV Partner gas → `netValue: 0` |
| CA results viewable per owner and contract | Via API |
| `ua.allocation.completed` Service Bus event | Triggers M4 |

**Deferred (Post-Pilot):**

| Feature | Reason |
|---------|--------|
| Keepwhole processing | Gas plant contract; not in Build-A-Biz |
| Settlement diversity groups | Advanced feature; OQ-CA-002 |
| NGL fractionated component allocation | Depends on OQ-B-009 resolution |
| PVR / PTR reports | Post-pilot reporting sprint |

---

### M4 — Contracts, Pricing & Valuation

**In Pilot Scope:**

| Feature | Notes |
|---------|-------|
| Contract master data (oil sales, gas sales, TIK) | Full CRUD with effective dating |
| Valuation formula create (posted price, fixed spot) | Oil gravity differential + gas compression deduction |
| Valuation Cross-Reference (VCR) linking | Contract + MP → formula |
| Marketing deduct rate configuration | Compression rate for gas |
| Posted oil price entry (monthly) | Manual entry sufficient for pilot |
| Fixed gas spot price entry | Manual entry |
| Gravity scale / API differential pricing | Required for oil valuation formula |
| Valuation run — draft mode (no posting) | Review before commit |
| Valuation run — post | Settlement statements committed; GL entries generated |
| Settlement statement detail (gross value, deductions, net) | Per contract and owner type |
| Severance tax calculation (single state) | One state is sufficient for pilot |
| PPN generation on closed-period data change | Triggered by price change or DOI correction |
| PPN Workplace — list, review impact, set disposition | Required for Phase 2 test cases |
| PPA reprocessing of CA and VL via PPN | Re-run with prior-period mode flag |
| JE Summary report (delta view) | Before/after comparison for PPA review |
| `ua.valuation.settlement.completed` Service Bus event | Triggers M6 |

**Deferred (Post-Pilot):**

| Feature | Reason |
|---------|--------|
| NYMEX / index formula pricing (live feed) | Requires external data contract (OQ-T-004) |
| Ad valorem true-up | Year-end feature; not in Month 1/2 cycle |
| Multi-state severance tax | Single state covers Build-A-Biz; expand per operating states |
| Index differential (Platts/Argus) pricing | Post-pilot |
| Formula-based NGL pricing per component | Depends on OQ-B-009 |

---

### M5 — Balancing Workplace

**In Pilot Scope:**

| Feature | Notes |
|---------|-------|
| Product Balancing Agreement (PBA) create | Linked to WCs |
| PBA entity assignment | WC1 and WC2 |
| Balancing statement generation for a period | Shows over/under per owner |
| Statement detail view (cumulative positions) | Per owner and product |
| TIK owner balance position modelling | JV Partner position reflected correctly |

**Deferred (Post-Pilot):**

| Feature | Reason |
|---------|--------|
| Cash balancing at PBA expiration | OQ-B-001 unresolved |
| In-kind make-up scheduling | Advanced balancing feature |
| Balancing imbalance transfer / monetisation | Post-pilot finance feature |

---

### M6 — Revenue Distribution & Accounting

**In Pilot Scope:**

| Feature | Notes |
|---------|-------|
| Distribution run — all owners for a period | Full per-owner calculation |
| Per-owner deduction allocation (severance, transport, marketing) | Applied from settlement statement |
| TIK owner exclusion from cash distribution | `netValue: 0` |
| Royalty interest owner-specific tax handling | RI severance tax calculation path |
| Owner suspense flagging (no payment method, "Do Not Pay") | Son 2 in suspense |
| Revenue Accounting Document (RAD) generation | One RAD per distribution run |
| RAD GL posting — **stubbed SAP FI integration** | Returns mock `externalDocumentNumber`; real BAPI added post-pilot |
| `ua.revenue.distribution.completed` Service Bus event | Triggers M7 |

**Deferred (Post-Pilot):**

| Feature | Reason |
|---------|--------|
| Chain-of-title daily proration | Needed when ownership changes mid-period; Phase 2 enhancement |
| Late-payment interest calculation | No overdue payments in pilot cycle |
| Affiliate netting flag | Post-pilot intercompany feature |
| Live SAP FI BAPI integration | Stub sufficient for pilot; real integration Phase 1 close |

---

### M7 — Payment Processing & Check Input

**In Pilot Scope:**

| Feature | Notes |
|---------|-------|
| Owner Attribute Group configuration | Minimum threshold, withholding %, payment method |
| Payment run — 6-step lifecycle (validate → extract → variance → journalize → finalize → output) | Full lifecycle required for P1 test case |
| Minimum payment threshold evaluation and suspense | Son 2 accumulates in suspense |
| Check output file generation (flat file) | Simple format sufficient; positive pay deferred |
| EFT output file generation | For owners with EFT payment method |
| Check layout template (read-only for pilot) | Pre-configured; admin UI deferred |
| PDX cross-reference (remitter ↔ venture/DOI) | Required for check input test case |
| Remitter ↔ layout cross-reference | JV Partner mapped to default layout |
| Inbound check entry (PPD — Payment Posting Desktop) | Manual keyed entry |
| Check validation (PDX resolution, DOI match) | HTTP 422 on missing PDX |
| Check valuation (special distribution — net basis) | Balancing owner pay code handling |
| Check posting to GL | Debit cash / credit revenue entries |
| AR record creation | For unreconciled amounts |
| Suspended balance accumulation and release | Son 2 balance grows; release when address added |

**Deferred (Post-Pilot):**

| Feature | Reason |
|---------|--------|
| CDEX / EDI 820 electronic check input | Manual entry sufficient for pilot; EDI middleware selection pending (OQ-B-010) |
| JIB netting | OQ-PP-002 unresolved |
| 1099 processing | Tax review dependency (OQ-PP-004) |
| NRA / FIRPTA withholding | OQ-B-002 / OQ-PP-003 unresolved |
| Escheat processing | Requires dormancy period > pilot timeline |
| Positive pay bank file | Post-pilot banking integration |
| Check mass upload / copy-reverse | Operational efficiency feature |
| AR write-off | No aged AR in Month 1/2 pilot |

---

### M8 — Regulatory, Tax & Royalty Reporting

**In Pilot Scope:**

| Feature | Notes |
|---------|-------|
| Statutory report template store (read + apply) | Pre-loaded templates for ONRR-2014 and one state |
| ONRR payor cross-reference | Company code ↔ ONRR payor code |
| ONRR lease hierarchy | Venture → Lease → Sales Type |
| PPN configuration (auto-generated type only) | Manual PPN assignment deferred |
| ONRR-2014 report generation (7-step workplace) | Steps 1–4 (Populate, Validate, Generate TC lines, Extract) fully functional |
| TC 01 generation (normal production) | Always generated |
| TC 11 (transport allowance) | If transport deduction present |
| TC 13 (processing / ERQADIFD) | With TC 13 restriction enforced |
| TC 15 (no production) | Zero-production periods |
| Report approve and submit (mock portal submission) | Returns mock `submissionReference`; real portal integration deferred |
| State severance tax report generation | One state matching pilot simulation state |
| TC line validation report | Errors and warnings before submission |

**Deferred (Post-Pilot):**

| Feature | Reason |
|---------|--------|
| Section 6 reduced royalty rate (TC 37 / TC 38) | OQ-ONRR-002 unresolved |
| Indian recoupment | OQ-ONRR-003 unresolved |
| Amended report processing and out-of-statute write-off | No amendments in Month 1/2 cycle |
| Live ONRR portal submission (PAAS) | OQ-B-004 unresolved; mock sufficient for pilot |
| Multi-state severance tax reports | Expand after single-state validated |
| North Dakota, Kansas, Montana special frameworks | OQ-TAX-002/003/004 pending |
| Statutory template update UI | Admin feature; pre-loaded templates sufficient |

---

### M9 — Administration & ILM

**In Pilot Scope:**

| Feature | Notes |
|---------|-------|
| Role definition CRUD | Minimum roles: Production Engineer, Revenue Accountant, Compliance Officer, System Admin |
| User role assignment (permanent assignments only) | Temporary grants deferred |
| Accounting period calendar (open / soft-close / hard-close) | Full state machine required for close test case |
| Period close prerequisite validation | All five close-date checks enforced |
| Onboarding validation checklist | All 10 standard check codes implemented |
| Audit log — write path | Every create/update/delete/approve action logged |
| Audit log — read/search | Basic filter by module, entity, date range |
| Integration sync log (write) | Records each SAP stub and mock portal call |

**Deferred (Post-Pilot):**

| Feature | Reason |
|---------|--------|
| Automation rulesets engine | Useful but not required to pass E2E test cases |
| ILM archive policy and archive runs | No data aged enough during pilot |
| Personal data blocking / anonymization | GDPR workflow; no DSARs in pilot |
| Temporary role assignment with expiry | Access governance enhancement |
| Stale access report | Operational tooling; post-pilot |

---

## 4. Integration Approach for Pilot

All external integrations use **lightweight stubs** during the pilot. The stub contract matches the real interface so that replacing it requires no changes to consuming services.

| Integration | Pilot Approach | Real Integration Target |
|-------------|---------------|------------------------|
| SAP FI GL Posting (BAPI_ACC_DOCUMENT_POST) | Stub returns hardcoded `externalDocumentNumber: "PILOT-DOC-{uuid}"` and `postingStatus: posted` | Phase 1 close sprint |
| SAP Business Partner Replication | Pilot: BPs created directly in EnerFusion; no SAP sync | Phase 1 integration sprint (OQ-T-010) |
| ONRR PAAS Portal Submission | Stub returns `submissionReference: "PILOT-ONRR-{uuid}"` | Phase 2 (OQ-B-004) |
| ACH / Bank EFT File Transmission | File generated and written to blob storage; no bank transmission | Phase 1 payment sprint |
| Azure Service Bus | **Real** — all inter-module events use actual Service Bus topics | In scope from day one |
| Azure Key Vault | **Real** — all secrets referenced via Key Vault | In scope from day one |

---

## 5. Infrastructure for Pilot

The pilot runs on a simplified but production-equivalent AKS configuration:

| Component | Pilot Config | Production Target |
|-----------|-------------|------------------|
| AKS Node Pool — System | 1 node (B4ms) | 3 nodes fixed |
| AKS Node Pool — Services | 2 nodes (D4s_v3), no HPA | 3–15 nodes with HPA |
| AKS Node Pool — Batch | 0–2 nodes (scale-to-zero) | 0–10 nodes |
| PostgreSQL Flexible Server | Burstable B2s (2 vCore) | Production SKU pending OQ-T-006 |
| Azure Service Bus | Standard tier (required for topics) | Standard or Premium |
| Azure Blob Storage | LRS, single region | GRS + DR region (OQ-T-009) |
| NGINX Ingress | Single replica | HA pair |
| PgBouncer | Shared deployment (1 replica) | Sidecar or shared per OQ-T-005 |

All Helm charts and Kubernetes manifests must be written for production equivalence — the pilot infra is a scale-down of production, not a throwaway environment.

---

## 6. Data Constraints for Pilot

| Dimension | Pilot Limit | Production Target |
|-----------|------------|------------------|
| Company codes | 1 | Unlimited |
| Ventures / DOIs | 2 (WC1, WC2) + 1 OO | 500k+ WCs |
| Business partners | ~15 | 2M+ DOI owners |
| Accounting periods active | 2 (Month 1 + Month 2) | Rolling 7-year active window |
| Concurrent test users | 5 | 500 |
| Delivery networks | 3 (Oil, Gas, OO) | Unlimited |

---

## 7. API Endpoints Required for Pilot

The following is the minimum set of API endpoints that must be implemented and passing for all P1 test cases to execute. Endpoints marked with a stub indicator use in-memory or fixed responses.

### Shared / Business Partners
| Method | Path |
|--------|------|
| POST | `/api/v1/business-partners` |
| GET | `/api/v1/business-partners` |
| GET | `/api/v1/business-partners/{id}` |
| PUT | `/api/v1/business-partners/{id}` |

### M1 — Production
| Method | Path |
|--------|------|
| POST / GET | `/api/v1/production/fields` |
| POST / GET | `/api/v1/production/reservoirs` |
| POST / GET | `/api/v1/production/wells` |
| POST / GET / PUT | `/api/v1/production/well-completions` |
| POST / GET | `/api/v1/production/measurement-points` |
| POST / GET | `/api/v1/production/delivery-networks` |
| POST / GET | `/api/v1/production/well-tests` |
| POST / GET | `/api/v1/production/downtime` |
| POST / GET | `/api/v1/production/tank-gauges` |
| POST / GET | `/api/v1/production/meter-volumes` |
| POST | `/api/v1/production/allocation-runs` |
| GET | `/api/v1/production/allocation-runs/{id}` |
| GET | `/api/v1/production/wc-volumes` |
| POST / GET | `/api/v1/production/mp-volumes` |

### M2 — Ownership & DOI
| Method | Path |
|--------|------|
| POST / GET | `/api/v1/ventures` |
| POST / GET | `/api/v1/ownership/doi` |
| GET | `/api/v1/ownership/doi/{id}/validation` |
| POST | `/api/v1/ownership/doi/{id}/approve` |
| POST | `/api/v1/ownership/doi/{id}/check-in` |
| GET | `/api/v1/ownership/doi/{id}/owners` |
| POST / GET | `/api/v1/ownership/transfer-requests` |
| POST | `/api/v1/ownership/transfer-requests/{id}/release` |
| POST | `/api/v1/ownership/transfer-requests/{id}/approve` |
| POST | `/api/v1/ownership/transfer-requests/{id}/execute` |
| GET | `/api/v1/ownership/chain-of-title/{doiId}` |
| GET | `/api/v1/ownership/ppns` |
| GET | `/api/v1/ownership/ppns/{id}/impact` |
| PUT | `/api/v1/ownership/ppns/{id}` |

### M3 — Contractual Allocation
| Method | Path |
|--------|------|
| POST | `/api/v1/allocation/templates/generate` |
| PUT | `/api/v1/allocation/templates/{id}/actuals` |
| POST | `/api/v1/allocation/runs` |
| GET | `/api/v1/allocation/runs/{id}` |
| GET | `/api/v1/allocation/results` |

### M4 — Valuation
| Method | Path |
|--------|------|
| POST / GET / PUT | `/api/v1/contracts` |
| POST / GET | `/api/v1/valuation/formulas` |
| POST / GET | `/api/v1/valuation/vcr` |
| POST / GET | `/api/v1/valuation/posted-prices` |
| POST / GET | `/api/v1/valuation/formula-prices` |
| POST | `/api/v1/valuation/runs` |
| GET | `/api/v1/valuation/runs/{id}` |
| GET | `/api/v1/valuation/runs/{id}/settlement-statements` |
| POST | `/api/v1/valuation/runs/{id}/post` |
| GET | `/api/v1/valuation/runs/{id}/accounting-detail` |
| GET | `/api/v1/valuation/runs/{id}/je-summary` |

### M5 — Balancing
| Method | Path |
|--------|------|
| POST / GET | `/api/v1/balancing/pbas` |
| POST | `/api/v1/balancing/pbas/{id}/entities` |
| POST | `/api/v1/balancing/pbas/{id}/statements/generate` |
| GET | `/api/v1/balancing/pbas/{id}/statements/{id}` |

### M6 — Revenue Distribution
| Method | Path |
|--------|------|
| POST | `/api/v1/revenue/distribution/run` |
| GET | `/api/v1/revenue/distribution/runs/{id}` |
| GET | `/api/v1/revenue/distribution/runs/{id}/owners` |
| GET | `/api/v1/revenue/documents` |
| POST | `/api/v1/revenue/documents/{id}/gl-posting` *(stub)* |
| GET | `/api/v1/revenue/suspense` |

### M7 — Payment Processing & Check Input
| Method | Path |
|--------|------|
| POST / GET | `/api/v1/payments/runs` |
| GET | `/api/v1/payments/runs/{id}` |
| GET | `/api/v1/payments/runs/{id}/variance` |
| GET | `/api/v1/payments/runs/{id}/owner-payments` |
| POST / GET | `/api/v1/payments/remitter-layout-xref` |
| POST / GET | `/api/v1/payments/pdx` |
| POST / GET | `/api/v1/payments/incoming-checks` |
| POST | `/api/v1/payments/incoming-checks/{id}/validate` |
| POST | `/api/v1/payments/incoming-checks/{id}/value` |
| POST | `/api/v1/payments/incoming-checks/{id}/post` |
| GET | `/api/v1/payments/incoming-checks/{id}/accounting-detail` |

### M8 — Regulatory Reporting
| Method | Path |
|--------|------|
| GET | `/api/v1/regulatory/templates` |
| POST | `/api/v1/regulatory/reports/run` |
| GET | `/api/v1/regulatory/reports/runs/{id}` |
| GET | `/api/v1/regulatory/reports/runs/{id}/lines` |
| PUT | `/api/v1/regulatory/reports/runs/{id}/approve` |
| POST | `/api/v1/regulatory/reports/runs/{id}/submit` *(stub)* |
| POST / GET | `/api/v1/regulatory/onrr/leases` |
| POST / GET | `/api/v1/regulatory/onrr/ppns` |

### M9 — Administration
| Method | Path |
|--------|------|
| POST / GET / PUT | `/api/v1/admin/roles` |
| POST / GET / DELETE | `/api/v1/admin/users/{id}/assignments` |
| GET / PUT | `/api/v1/admin/calendar/{companyCode}/{period}/open` |
| PUT | `/api/v1/admin/calendar/{companyCode}/{period}/soft-close` |
| PUT | `/api/v1/admin/calendar/{companyCode}/{period}/hard-close` |
| POST | `/api/v1/admin/onboarding/checklists` |
| POST | `/api/v1/admin/onboarding/checklists/{id}/run` |
| GET | `/api/v1/admin/audit-log` |

---

## 8. Pilot Acceptance Criteria

The pilot is considered complete when all of the following are satisfied:

| # | Criterion | Verified By |
|---|-----------|------------|
| AC-01 | All P1 — Blocker test cases in Phase 1 pass without manual data workarounds | Test run report |
| AC-02 | All P1 — Blocker test cases in Phase 2 pass | Test run report |
| AC-03 | Data flows end-to-end: a WC volume entered in M1 is traceable to an owner payment record in M7 via API queries alone | Manual trace walk-through |
| AC-04 | A prior-period price correction in M4 generates a PPN, the PPN propagates through CA and VL re-runs, and the delta posts correctly to the GL (stub) | TC-E2E-P2-M4-002 |
| AC-05 | Company code isolation: a user scoped to Company A cannot retrieve Company B records (RLS enforced) | Security test |
| AC-06 | All audit log entries written for every create / approve / post action in the E2E run | Audit log query |
| AC-07 | All inter-module Service Bus events published and consumed without dead-lettering | Azure Service Bus dead-letter queue = 0 |
| AC-08 | `tsc --noEmit` passes with zero errors across all services | CI pipeline |
| AC-09 | All Jest/Vitest unit tests pass; coverage ≥ 80% on Service layer | CI pipeline |
| AC-10 | Docker build completes for all services; Trivy scan finds zero critical CVEs | CI pipeline |

---

## 9. What Pilot Scope Explicitly Excludes

| Item | Justification |
|------|--------------|
| React UI / frontend | API-first; UI built in a parallel sprint after APIs are stable |
| Live SAP FI BAPI integration | Stub is contractually equivalent; replaced in Phase 1 close sprint |
| Live ONRR portal submission | OQ-B-004 unresolved |
| CDEX / EDI 820 | OQ-B-010 unresolved; manual check entry covers test case |
| Multi-tenant / multi-company isolation beyond RLS | OQ-B-007 unresolved |
| Data migration from SAP PRA | Separate migration workstream |
| Disaster recovery / geo-replication | OQ-T-009 unresolved |
| Production-scale load testing | Post-pilot performance sprint |
| Service mesh (Linkerd / Istio) | OQ-T-003 deferred to Phase 1 |
| Escheat, 1099, NRA withholding | Tax/Legal dependencies unresolved |
