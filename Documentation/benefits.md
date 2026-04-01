# Benefits

## EnerFusion Upstream Accounting Value Proposition

EnerFusion Upstream Accounting replaces the functional scope of SAP IS-Oil PRA with a cloud-native, API-first platform purpose-built for Azure Kubernetes Service. The following benefits apply to upstream energy operators in North America.

---

## Business Benefits

### 1. Complete Revenue Lifecycle in a Single Platform

EnerFusion Upstream Accounting covers the full upstream accounting chain — wellhead allocation through regulatory filing — in one integrated product. Operators no longer need to bridge multiple point solutions (allocation tool + spreadsheet + payment system) to close a production month.

| Without EnerFusion | With EnerFusion |
|-------------------|-----------------|
| Manual volume transfer between systems | Event-driven automation (Azure Service Bus) |
| Spreadsheet-based royalty calculations | Engine-validated settlement statements |
| Siloed state compliance tools | Built-in statutory report generation for 10+ states + ONRR |
| Disconnected owner payment systems | Integrated check input/output, escheat, AR aging |

---

### 2. North American Regulatory Compliance — Built In

Statutory report formats for Texas, Oklahoma, Louisiana, New Mexico, Wyoming, Colorado, Arkansas, California, North Dakota, and federal ONRR are embedded in M8. Operators file directly from the platform without external reporting tools.

**Covered Reports:**

- **Texas:** Form H-10, P1B, PR, GLO Royalty, UT Royalty
- **Federal ONRR:** MMS-PASR, MMS-OGOR, ONRR-2014
- **Multi-state:** AR Form 7, CA OG110, CO Form 7, LA OGP/R5D + WR1, NM C115, ND Reports, OK Form 1004, WY Form 2 + Gross Production Tax

---

### 3. Ownership Accuracy and Audit Trail

The DOI check-in / check-out workflow (M2) enforces a single-write lock on venture ownership records. Every transfer generates an immutable chain-of-title record. Decimal interest integrity is validated to eight decimal places before any distribution is calculated.

---

### 4. Flexible Commercial Pricing

M4 supports three pricing paradigms used across North American upstream:

- **Fixed price** with time-effective conditions per product type
- **API gravity scales** for crude oil quality differentials
- **Formula pricing** with configurable reserved words (NYMEX, WTI, BTU factor, transport deduction) evaluated per reporting period

---

### 5. Imbalance and Balancing Transparency

M5 (Balancing Workplace) gives commercial teams a real-time view of over- and under-takes against owner entitlements. Product Balancing Agreements (PBAs) and manual adjustments resolve cumulative positions without requiring offline spreadsheet reconciliation.

---

## Technical Benefits

### 6. Cloud-Native Scalability on Azure AKS

The platform is deployed on Azure Kubernetes Service with horizontal pod autoscaling. Month-end valuation batch runs and regulatory report generation use a dedicated `batch` node pool that scales to zero outside processing windows, reducing idle compute costs.

| Workload Type | AKS Node Pool | Scale Range |
|--------------|---------------|-------------|
| API services | `services` pool (Standard_D8s_v3) | 3 – 15 nodes |
| Batch / valuation | `batch` pool (Standard_F16s_v2) | 0 – 10 nodes |
| Frontend / BFF | `frontend` pool (Standard_D4s_v3) | 2 nodes (fixed) |

---

### 7. Modular Architecture — Isolate Change, Contain Risk

Nine independent microservices, each with its own PostgreSQL schema, Helm chart, and deployment pipeline. A schema change to `M3: Contractual Allocation` cannot break `M7: Payment Processing`. Each module is independently deployable.

---

### 8. RBAC at Every Layer

Role-based access control is enforced at three levels:

1. **Azure APIM** — token validation at the API gateway
2. **Service middleware** — module × action permission checks per request
3. **PostgreSQL Row-Level Security** — tenancy isolation at the database

This gives compliance teams auditable proof that user A could not have accessed owner data outside their company scope.

---

### 9. API-First Integration Surface

All modules expose versioned REST APIs (`/api/v1/…`). External systems — SCADA, ERP financial modules, state agency portals, BI tools — integrate via well-documented contracts. CDEX/EDI for purchaser check import is supported in M7.

---

### 10. Lower Total Cost of Ownership vs. SAP IS-Oil

| Cost Driver | SAP IS-Oil | EnerFusion Upstream Accounting |
|-------------|-----------|----------------|
| Licensing model | Per-user SAP perpetual / subscription | Platform-based SaaS or self-hosted on AKS |
| Upgrade cadence | Major upgrades every 3–5 years | Continuous delivery via GitHub Actions CI/CD |
| Customization | ABAP development, transport requests | TypeScript + React, standard PR-based workflow |
| Infrastructure | On-premise or SAP RISE | Azure AKS — pay for what you use |
| Reporting changes | Custom SAP reports (costly) | Direct SQL + configurable report templates |

---

## Summary

| Benefit Area | Key Outcome |
|-------------|-------------|
| Revenue accuracy | Automated allocation, valuation, and distribution reduce manual errors |
| Compliance | 10+ state + federal reports generated within platform |
| Scalability | AKS auto-scaling handles month-end peaks without over-provisioning |
| Auditability | Immutable chain-of-title, full RBAC audit log, OpenTelemetry tracing |
| Integration | API-first design connects to ERP, SCADA, and BI systems without custom adapters |
| Cost | Modular deployment and cloud-native architecture reduce ongoing TCO |
