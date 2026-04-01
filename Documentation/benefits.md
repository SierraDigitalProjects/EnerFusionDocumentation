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

### 2. Automated Volume Allocation — Eliminating Manual Spreadsheets (M1)

The 8-step allocation algorithm in M1 automatically reconciles estimated theoretical production with actual measured sales at each delivery network. The system flags allocations that exceed the configured tolerance and routes them for supervisor review — removing the manual daily spreadsheet that most operators maintain outside their ERP.

| Without EnerFusion | With EnerFusion M1 |
|-------------------|-------------------|
| Manual daily volume entry in spreadsheets | Structured daily volume capture per well completion |
| Tolerance checking done by accountant | Automated tolerance gate with supervisor escalation |
| No audit trail for allocation adjustments | Full allocation run history with before/after volumes |

---

### 3. Ownership Accuracy and Audit Trail (M2)

The DOI check-in / check-out workflow (M2) enforces a single-write lock on venture ownership records. Every transfer generates an immutable chain-of-title record. Decimal interest integrity is validated to eight decimal places before any distribution is calculated.

---

### 4. Contractual Allocation Transparency (M3)

M3 connects physical production volumes to commercial sales contracts, calculating per-owner entitlements and tracking network imbalances at the marketing group level. Gas plant NGL splits and percent-return-to-lease calculations are configured and executed within the platform — replacing manual gas balancing workbooks.

---

### 5. Flexible Commercial Pricing (M4)

M4 supports three pricing paradigms used across North American upstream:

- **Fixed price** with time-effective conditions per product type
- **API gravity scales** for crude oil quality differentials
- **Formula pricing** with configurable reserved words (NYMEX, WTI, BTU factor, transport deduction) evaluated per reporting period

---

### 6. Imbalance and Balancing Transparency (M5)

M5 (Balancing Workplace) gives commercial teams a real-time view of over- and under-takes against owner entitlements. Product Balancing Agreements (PBAs) and manual adjustments resolve cumulative positions without requiring offline spreadsheet reconciliation.

---

### 7. Automated Revenue Distribution (M6)

M6 distributes net settlement values to each owner interest in proportion to their DOI decimal interest, calculates late-payment interest on overdue distributions, and generates Revenue Accounting Documents (RADs) for posting directly to the SAP general ledger. Operators eliminate manual distribution spreadsheets and the associated reconciliation effort.

---

### 8. Integrated Payment Processing and AR Management (M7)

M7 handles both incoming purchaser checks (via CDEX/EDI 820) and outgoing owner payment disbursements (ACH or check) within the same platform. Accounts receivable aging, write-offs, minimum payment threshold management, and escheat/escrow tracking are all managed natively — eliminating separate AR tools and manual disbursement files.

---

### 9. North American Regulatory Compliance — Built In (M8)

Statutory report formats for Texas, Oklahoma, Louisiana, New Mexico, Wyoming, Colorado, Arkansas, California, North Dakota, and federal ONRR are embedded in M8. Operators file directly from the platform without external reporting tools.

**Covered Reports:**

- **Texas:** Form H-10, P1B, PR, GLO Royalty, UT Royalty
- **Federal ONRR:** MMS-PASR, MMS-OGOR, ONRR-2014
- **Multi-state:** AR Form 7, CA OG110, CO Form 7, LA OGP/R5D + WR1, NM C115, ND Reports, OK Form 1004, WY Form 2 + Gross Production Tax

---

### 10. Governance, Data Lifecycle, and Automation (M9)

M9 provides centralized RBAC management, a tamper-evident audit log of all data mutations across every module, configurable automation rulesets (bots) for scheduling allocation runs and detecting anomalies, and ILM data archiving with GDPR/CCPA-aligned personal data protection. Administrators manage the entire platform lifecycle — user access, data retention, and compliance controls — from a single administration interface.

---

## Technical Benefits

### 11. Cloud-Native Scalability on Azure AKS

The platform is deployed on Azure Kubernetes Service with horizontal pod autoscaling. Month-end valuation batch runs and regulatory report generation use a dedicated `batch` node pool that scales to zero outside processing windows, reducing idle compute costs.

| Workload Type | AKS Node Pool | Scale Range |
|--------------|---------------|-------------|
| API services | `services` pool (Standard_D8s_v3) | 3 – 15 nodes |
| Batch / valuation | `batch` pool (Standard_F16s_v2) | 0 – 10 nodes |
| Frontend / BFF | `frontend` pool (Standard_D4s_v3) | 2 nodes (fixed) |

---

### 12. Modular Architecture — Isolate Change, Contain Risk

Nine independent microservices, each with its own PostgreSQL schema, Helm chart, and deployment pipeline. A schema change to `M3: Contractual Allocation` cannot break `M7: Payment Processing`. Each module is independently deployable.

---

### 13. RBAC at Every Layer

Role-based access control is enforced at three levels:

1. **Azure APIM** — token validation at the API gateway
2. **Service middleware** — module × action permission checks per request
3. **PostgreSQL Row-Level Security** — tenancy isolation at the database

This gives compliance teams auditable proof that user A could not have accessed owner data outside their company scope.

---

### 14. API-First Integration Surface

All modules expose versioned REST APIs (`/api/v1/…`). External systems — SCADA, ERP financial modules, state agency portals, BI tools — integrate via well-documented contracts. CDEX/EDI for purchaser check import is supported in M7.

---

### 15. Lower Total Cost of Ownership vs. SAP IS-Oil

| Cost Driver | SAP IS-Oil | EnerFusion Upstream Accounting |
|-------------|-----------|----------------|
| Licensing model | Per-user SAP perpetual / subscription | Platform-based SaaS or self-hosted on AKS |
| Upgrade cadence | Major upgrades every 3–5 years | Continuous delivery via GitHub Actions CI/CD |
| Customization | ABAP development, transport requests | TypeScript + React, standard PR-based workflow |
| Infrastructure | On-premise or SAP RISE | Azure AKS — pay for what you use |
| Reporting changes | Custom SAP reports (costly) | Direct SQL + configurable report templates |

---

## Summary

| Module | Benefit |
|--------|---------|
| M1 Production | Automated 8-step allocation replaces manual daily spreadsheets |
| M2 Ownership | Immutable chain-of-title and single-lock DOI workflow eliminate ownership disputes |
| M3 Contractual Allocation | Contract-level entitlement and gas plant NGL splits managed natively |
| M4 Valuation | Formula pricing engine covers fixed, gravity-scale, and index-linked contracts |
| M5 Balancing | Real-time imbalance tracking against PBA terms without offline reconciliation |
| M6 Revenue | Automated owner distributions and RAD posting to ERP GL |
| M7 Payments | Integrated purchaser check input, owner disbursement, AR aging, and escheat |
| M8 Regulatory | 10+ state + federal statutory reports generated within the platform |
| M9 Administration | Centralized RBAC, audit logging, automation, and ILM data lifecycle management |
| M10 Partner Onboarding | Structured 7-step workflow replaces email-based partner onboarding; document verification and system provisioning in one place |
| M11 Joint Venture Accounting | COPAS-compliant AFE elections, JIB billing, and cash call management replaces spreadsheet-based JV accounting |
