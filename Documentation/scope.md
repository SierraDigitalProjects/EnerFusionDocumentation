# Scope

## EnerFusion Upstream Accounting — Product Scope Definition

This document defines what EnerFusion Upstream Accounting covers, what it explicitly excludes, and the boundary conditions where EnerFusion integrates with adjacent systems.

---

## In Scope

### Core Modules

| Module | Scope Summary |
|--------|---------------|
| **M1 — Production & Volume Allocation** | Daily production data capture, well completion management, measurement point volumes, delivery network allocation algorithm, downtime tracking, well tests |
| **M2 — Ownership & DOI** | Business partner master, lease identifiers, venture/DOI management, DOI check-in/check-out workflow, ownership transfer requests, chain-of-title maintenance |
| **M3 — Contractual Allocation & Product Control** | Sales contract allocation, entitlement calculations, gas plant NGL splits, percent return-to-lease, marketing group network imbalances, daily nominations |
| **M4 — Contracts, Pricing & Valuation** | Sales contract master, fixed pricing, API gravity scales, formula pricing (reserved word engine), severance/ad valorem tax calculation, settlement statement generation, run statements, prior period adjustments |
| **M5 — Balancing Workplace** | Product Balancing Agreement (PBA) master, owner imbalance tracking, manual balance adjustments, in-kind take-up / cashout balancing |
| **M6 — Revenue Distribution & Accounting** | Owner-level revenue distribution, late-payment interest calculation, Revenue Accounting Document (RAD) generation for GL posting |
| **M7 — Payment Processing & Check Input** | Incoming purchaser check processing (CDEX/EDI 820), outgoing owner payment disbursement, AR aging and write-offs, escheat/escrow processing |
| **M8 — Regulatory, Tax & Royalty Reporting** | Statutory report generation for 10 US states + ONRR federal; amended report handling; out-of-statute write-offs |
| **M9 — Administration & ILM** | Automation rulesets (bots), data archiving, personal data protection (blocking/anonymization), user access management, onboarding validation tools |

---

### RBAC Coverage

Every master data object and transactional entity supports role-based access control with at minimum the following permission levels:

| Permission | Description |
|------------|-------------|
| `read` | View records |
| `write` | Create and update records |
| `approve` | Authorize workflow state transitions |
| `execute` | Trigger processing runs (allocation, valuation, distribution) |
| `admin` | Configure master data, manage user access |
| `import` | Bulk upload via template |

---

### Regulatory Report Coverage (M8)

| Jurisdiction | Reports |
|-------------|---------|
| Texas | Form H-10, P1B, PR, GLO Royalty, UT Royalty |
| Arkansas | Form 7 |
| California | Report OG110 |
| Colorado | Form 7 |
| Louisiana | OGP/R5D, WR1 |
| New Mexico | Report C115 |
| North Dakota | ND Production Reports |
| Oklahoma | Form 1004 |
| Wyoming | Form 2, Gross Production Tax Report |
| Federal (ONRR) | MMS-PASR, MMS-OGOR, ONRR-2014 |

---

### High-Volume Data Support (NFR)

The platform is designed to support:

- Thousands of well completions, measurement points, and delivery networks in M1
- Tens of thousands of DOI owner interest records in M2
- Hundreds of thousands of daily volume transactions per month in M1/M3
- Mass check processing (thousands of checks per period) in M7
- Bulk owner distributions in M6

All list views use server-side pagination and virtual scrolling (AG Grid). API endpoints support cursor-based pagination for datasets exceeding 10,000 records.

---

## Out of Scope

The following functional areas are explicitly excluded from EnerFusion Upstream Accounting:

| Area | Reason / Alternative |
|------|---------------------|
| **General Ledger (GL) posting** | Handled by SAP FI/CO or equivalent ERP; EnerFusion posts RADs via API integration |
| **Accounts Payable (AP) — non-revenue** | Vendor invoice processing outside oil and gas payments is an ERP function |
| **Land Management (lease acquisition, AFE)** | Dedicated land management systems (e.g., OpenWells, WolfePak Land) |
| **Drilling & Completions Operations** | Rig scheduling, drilling programs, wellbore engineering |
| **Reservoir Engineering / Simulation** | Decline curve analysis, production forecasting models |
| **Environmental, Health & Safety (EHS) reporting** | OSHA, EPA non-production compliance |
| **International operations** | Non-US jurisdictions, VAT, foreign royalty regimes |
| **Gas marketing / trading** | Forward contracts, hedge accounting, mark-to-market |
| **Automated e-filing with state agencies** | Compliance officer submits generated reports via state portals manually |
| **FIRPTA withholding for foreign owners** | Foreign investor withholding beyond scope of initial release |
| **Non-standard fiscal periods** | Calendar-month reporting period only |
| **Cash balancing at PBA expiration** | In-kind balancing only in M5 initial release |

---

## Integration Boundaries

EnerFusion Upstream Accounting integrates with adjacent systems at well-defined API boundaries:

```
┌──────────────────────────────────────────────────────────────┐
│                     EXTERNAL SYSTEMS                         │
│                                                              │
│  ┌─────────────┐  ┌────────────┐  ┌──────────────────────┐  │
│  │  SCADA /    │  │  SAP FI/CO │  │  State Agency        │  │
│  │  Field Data │  │  (GL / AP) │  │  e-Filing Portals    │  │
│  └──────┬──────┘  └─────┬──────┘  └──────────────────────┘  │
│         │               │                    ▲               │
│         │ Volumes (API) │ RAD posting (API)  │ Manual export  │
└─────────┼───────────────┼────────────────────┼───────────────┘
          │               │                    │
          ▼               ▼                    │
┌─────────────────────────────────────────────────────────────┐
│                   ENERFUSION PRA                             │
│   M1 → M2 → M3 → M4 → M5 → M6 → M7 → M8 → M9             │
└─────────────────────────────────────────────────────────────┘
          │
          │ CDEX/EDI 820 (purchaser checks in)
          ▼
     ┌──────────┐
     │ Purchasers│
     └──────────┘
```

---

## Phasing (Suggested)

| Phase | Modules | Milestone |
|-------|---------|-----------|
| Phase 1 — Foundation | M1, M2, M9 (RBAC + Admin) | Production volumes and ownership records live |
| Phase 2 — Commercial | M3, M4, M5 | Contract allocation, valuation, balancing |
| Phase 3 — Revenue | M6, M7 | Owner distributions and payment processing |
| Phase 4 — Compliance | M8 | State and federal regulatory reporting |
