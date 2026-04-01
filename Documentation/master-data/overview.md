# Master Data Overview

## Data Ownership Model

EnerFusion Upstream Accounting coexists with enterprise systems (primarily SAP ERP) in most operator environments. Master data objects fall into three ownership categories that define where records originate, who maintains them, and how they flow between systems.

---

## Ownership Categories

```
┌─────────────────────────────────────────────────────────────────┐
│                     MASTER DATA OWNERSHIP                        │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  CENTRALLY       │  │  LOCALLY         │  │  NOT IN      │  │
│  │  MANAGED         │  │  MANAGED         │  │  SAP         │  │
│  │  IN SAP          │  │  IN ENERFUSION   │  │              │  │
│  │                  │  │                  │  │              │  │
│  │  SAP is system   │  │  EnerFusion is   │  │  EnerFusion- │  │
│  │  of record →     │  │  system of       │  │  only data;  │  │
│  │  replicated to   │  │  record;         │  │  no SAP      │  │
│  │  EnerFusion      │  │  SAP may receive │  │  equivalent  │  │
│  │                  │  │  read-only copy  │  │  entity      │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Master Data Entity Map

### Production (M1)

| Entity | Ownership | SAP Equivalent | Notes |
|--------|-----------|----------------|-------|
| Field | Locally Managed | Functional Location | EnerFusion is system of record |
| Reservoir | Locally Managed | None | No SAP equivalent |
| Platform | Locally Managed | Functional Location | Offshore-specific |
| Well | Locally Managed | Equipment (EQUI) | API number is canonical |
| Well Completion | Locally Managed | None | platform-specific sub-object |
| Measurement Point | Not in SAP | None | EnerFusion-only |
| Delivery Network | Not in SAP | None | EnerFusion-only |

### Ownership (M2)

| Entity | Ownership | SAP Equivalent | Notes |
|--------|-----------|----------------|-------|
| Business Partner | Centrally in SAP | Business Partner (BP) | Replicated from SAP via API |
| Owner Attribute Group | Locally Managed | None | Payment/withholding config |
| Upstream Lease Identifier | Locally Managed | RE Lease (RE-FX) | EnerFusion is primary |
| Venture / Base DOI | Locally Managed | Venture (IS-Oil) | Migrated from SAP PRA |
| DOI Owner Interest | Locally Managed | DOI (IS-Oil) | |
| Unit / Tract Record | Locally Managed | None | Regulatory pooling unit |

### Contracts & Pricing (M4)

| Entity | Ownership | SAP Equivalent | Notes |
|--------|-----------|----------------|-------|
| Sales Contract | Locally Managed | SD Contract | EnerFusion maintains full upstream accounting contract |
| Fixed Price Condition | Locally Managed | Condition Record (VK11) | Simplified; no SAP pricing procedure |
| Formula Pricing Rule | Not in SAP | None | EnerFusion formula engine |
| API Gravity Scale | Not in SAP | None | EnerFusion-only |
| State Tax Rate | Not in SAP | Tax Code (FTXP) | Maintained by compliance team in EnerFusion |
| Tax Classification | Not in SAP | None | Well-level tax category |
| Valuation Cross-Reference (VCR) | Not in SAP | None | platform-specific mapping record |

### Valuation & Revenue (M4, M6)

| Entity | Ownership | SAP Equivalent | Notes |
|--------|-----------|----------------|-------|
| Chart of Accounts (Upstream Accounting GL) | Centrally in SAP | GL Account (FS00) | EnerFusion maps to SAP GL via account code |
| Purchaser-Remitter Xref | Locally Managed | Customer (KNA1) | Purchaser side of check input |
| Check Layout | Not in SAP | SAPscript/Smartform | EnerFusion check template config |
| Product Balancing Agreement | Not in SAP | None | EnerFusion-only |

### Administration (M9)

| Entity | Ownership | SAP Equivalent | Notes |
|--------|-----------|----------------|-------|
| User / Role | Centrally in SAP | User (SU01) + Role | Provisioned via Azure AD; synced if SAP SSO enabled |
| Automation Ruleset | Not in SAP | Workflow (SWDD) | EnerFusion-native bots |
| Archive Policy | Not in SAP | ILM (SARA) | EnerFusion ILM configuration |

---

## Replication Patterns

### SAP → EnerFusion (Centrally Managed Objects)

```
SAP System
  │
  │  Change Document / IDoc / API event
  ▼
Azure Integration Layer (Logic Apps / API Management)
  │
  │  REST POST /api/v1/shared/business-partners
  ▼
EnerFusion shared.business_partners table
  │
  │  Read-only in EnerFusion UI
  ▼
M2 Ownership, M4 Contracts, M7 Payments
```

**Frequency:** Near-real-time via event trigger (preferred) or nightly batch sync.

**Conflict resolution:** SAP is authoritative. EnerFusion rejects local edits to centrally managed fields. Changes must be made in SAP and re-synced.

---

### EnerFusion → SAP (Revenue Accounting Documents)

```
EnerFusion M6: Revenue Distribution
  │
  │  Revenue Accounting Document (RAD) generated
  ▼
EnerFusion M6 API: GET /api/v1/revenue/documents/{id}/gl-posting
  │
  │  GL posting payload (account, cost center, amount, period)
  ▼
SAP FI — Journal Entry (FB01 / BAPI_ACC_DOCUMENT_POST)
```

**Frequency:** Triggered post-distribution run, before owner payment cutoff.

---

## Volume Summary by Module

| Module | Master Data Objects | Typical Record Counts |
|--------|--------------------|-----------------------|
| M1 Production | 7 entities | Wells: 1,000–50,000; MPs: 500–10,000 |
| M2 Ownership | 6 entities | DOI Interests: 10,000–500,000 |
| M3 Allocation | 4 entities | Contracts: 100–5,000 |
| M4 Valuation | 8 entities | Tax rates: hundreds; VCRs: thousands |
| M5 Balancing | 2 entities | PBAs: tens to hundreds |
| M6 Revenue | 2 entities | GL accounts: SAP-driven |
| M7 Payments | 3 entities | Purchasers: dozens to hundreds |
| M8 Regulatory | 2 entities | Report configs: per-state |
| M9 Admin | 3 entities | Rulesets: tens to hundreds |

---

## Related Pages

- [Centrally Managed in SAP](centrally-managed.md)
- [Locally Managed in EnerFusion](locally-managed.md)
- [Not Managed in SAP](not-managed.md)
