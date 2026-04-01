# Module Architecture

## Overview

EnerFusion Upstream Accounting is structured as nine independent microservices, each corresponding to a PRA business domain. Modules communicate via Azure Service Bus events (async) and expose versioned REST APIs (sync). No module accesses another module's database schema directly.

---

## Module Map

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          ENERFUSION UPSTREAM ACCOUNTING MODULES                              │
│                                                                              │
│   ┌────────────┐     ┌────────────┐     ┌──────────────────────────────┐    │
│   │  M1        │────▶│  M2        │────▶│  M3                          │    │
│   │ Production │     │ Ownership  │     │ Contractual Allocation        │    │
│   │ & Volume   │     │ & DOI      │     │ & Product Control            │    │
│   │ Allocation │     │            │     │                              │    │
│   └─────┬──────┘     └─────┬──────┘     └──────────────┬───────────────┘    │
│         │ volumes.allocated│ doi.changed                │ allocation.done   │
│         │                  │                            │                   │
│         ▼                  ▼                            ▼                   │
│   ┌──────────────────────────────────────────────────────────────────────┐   │
│   │  M4: Contracts, Pricing & Valuation                                  │   │
│   │  Settlement statements | Formula pricing | Tax calculation           │   │
│   └──────────────────────────────┬───────────────────────────────────────┘   │
│                                  │ settlement.completed                      │
│                    ┌─────────────┼─────────────┐                            │
│                    ▼             ▼             ▼                            │
│             ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│             │  M5      │  │  M6      │  │  M8      │                       │
│             │ Balancing│  │ Revenue  │  │ Regulatory│                      │
│             │ Workplace│  │ Distrib. │  │ Reporting │                      │
│             └──────────┘  └────┬─────┘  └──────────┘                       │
│                                │ distribution.completed                     │
│                                ▼                                            │
│                         ┌──────────┐                                        │
│                         │  M7      │                                        │
│                         │ Payment  │                                        │
│                         │ Processing│                                       │
│                         └──────────┘                                        │
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐   │
│   │  M9: Administration & ILM  (cross-cutting — all modules)             │   │
│   └──────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Module Details

### M1 — Production & Volume Allocation

**Service:** `svc-production`  
**Schema:** `production`  
**Primary Users:** Production Engineers, Operations

**Responsibilities:**
- Master data for fields, reservoirs, platforms, wells, and well completions
- Measurement point and tank strapping management
- Delivery network configuration and tolerance management
- Daily well completion volume capture and downtime recording
- Well test data management
- 8-step theoretical volume allocation algorithm execution
- Automated, manual, and scheduled allocation run management

**Outbound Events:**
- `ua.production.volumes.allocated` → consumed by M3, M8

**Inbound Events:** None — M1 is the origin of volume data.

**Key API Scopes:** `production:read`, `production:write`, `production:execute`, `production:admin`

---

### M2 — Ownership & Division of Interest

**Service:** `svc-ownership`  
**Schema:** `ownership`  
**Primary Users:** Land / Ownership Administrators

**Responsibilities:**
- Business partner management (replicated from SAP; locally extended)
- upstream lease identifiers and legal descriptions
- Venture and DOI owner interest record management (WI, RI, ORRI, NPRI, PRRI)
- DOI check-in / check-out exclusivity workflow
- Ownership transfer request lifecycle and approval
- Immutable chain-of-title maintenance
- DOI ↔ Measurement Point / Well Completion cross-reference
- Unit venture and tract participation factor management
- Owner attribute group configuration

**Outbound Events:**
- `ua.ownership.doi.changed` → consumed by M3, M4

**Inbound Events:** None — M2 changes are driven by human workflows.

**Key API Scopes:** `ownership:read`, `ownership:write`, `ownership:approve`, `ownership:import`

---

### M3 — Contractual Allocation & Product Control

**Service:** `svc-allocation`  
**Schema:** `allocation`  
**Primary Users:** Commercial / Contract Managers, Operations

**Responsibilities:**
- Allocation of M1 production volumes to sales contracts per owner entitlements
- Gas plant processing configuration (NGL splits, percent-return-to-lease)
- Network imbalance tracking at the marketing group level
- Daily nomination management per owner / contract
- Sales point manual adjustment entries
- Entitlement calculation using DOI interest from M2

**Inbound Events:**
- `ua.production.volumes.allocated` (M1) — triggers contractual allocation run
- `ua.ownership.doi.changed` (M2) — recalculates entitlements

**Outbound Events:**
- `ua.allocation.completed` → consumed by M4, M8

**Key API Scopes:** `allocation:read`, `allocation:write`, `allocation:execute`

---

### M4 — Contracts, Pricing & Valuation

**Service:** `svc-valuation`  
**Schema:** `valuation`  
**Primary Users:** Revenue Accountants, Contract Managers

**Responsibilities:**
- Sales contract master data lifecycle (draft → approved → active → expired)
- Fixed price condition management
- Formula pricing rule engine (reserved words, multi-step evaluation)
- API gravity scale configuration and crude oil quality pricing
- State tax rate and tax classification maintenance
- Processing allowance configuration (tax and royalty)
- Valuation Cross-Reference (VCR) mapping
- Settlement statement generation (gross value, deductions, net value)
- Oil and gas run statement generation
- Prior period adjustment (PPN) processing and approval

**Inbound Events:**
- `ua.allocation.completed` (M3) — triggers valuation run
- `ua.ownership.doi.changed` (M2) — re-evaluates marketing-free and royalty-type flags

**Outbound Events:**
- `ua.valuation.settlement.completed` → consumed by M6, M7

**Key API Scopes:** `valuation:read`, `valuation:write`, `valuation:approve`, `valuation:execute`, `valuation:admin`

---

### M5 — Balancing Workplace

**Service:** `svc-balancing`  
**Schema:** `balancing`  
**Primary Users:** Revenue Accountants, Commercial Managers

**Responsibilities:**
- Cumulative owner imbalance position tracking (over/under-take vs. entitlement)
- Product Balancing Agreement (PBA) master data management
- Manual balance adjustment processing (with reason code and approver)
- In-kind volume make-up tracking against PBA terms
- Balancing exception reporting

**Inbound Events:**
- `ua.allocation.completed` (M3) — updates imbalance positions for the period

**Outbound Events:** None — M5 is a reporting and adjustment module.

**Key API Scopes:** `balancing:read`, `balancing:write`, `balancing:approve`

---

### M6 — Revenue Distribution & Accounting

**Service:** `svc-revenue`  
**Schema:** `revenue`  
**Primary Users:** Revenue Accountants

**Responsibilities:**
- Owner-level revenue distribution from M4 settlement statements
- Late-payment interest calculation on overdue distributions
- Revenue Accounting Document (RAD) generation for GL posting
- Integration with SAP FI for RAD posting (via `BAPI_ACC_DOCUMENT_POST`)
- Owner interest summary and interest payment reporting

**Inbound Events:**
- `ua.valuation.settlement.completed` (M4) — triggers distribution run

**Outbound Events:**
- `ua.revenue.distribution.completed` → consumed by M7, M8

**Key API Scopes:** `revenue:read`, `revenue:write`, `revenue:execute`

---

### M7 — Payment Processing & Check Input

**Service:** `svc-payments`  
**Schema:** `payments`  
**Primary Users:** Revenue Accountants, Finance

**Responsibilities:**
- Incoming purchaser check processing (CDEX/EDI 820 format)
- Manual and mass check input for non-EDI purchasers
- Outgoing owner payment generation (ACH, check) using Owner Attribute Group config
- Minimum payment threshold management (suspend and accumulate)
- Accounts receivable aging management and manual write-offs
- Escheat / escrow tracking and state-specific unclaimed property reporting

**Inbound Events:**
- `ua.revenue.distribution.completed` (M6) — triggers outgoing payment generation

**Outbound Events:** None.

**Key API Scopes:** `payments:read`, `payments:write`, `payments:approve`, `payments:admin`

---

### M8 — Regulatory, Tax & Royalty Reporting

**Service:** `svc-regulatory`  
**Schema:** `regulatory`  
**Primary Users:** Compliance Officers

**Responsibilities:**
- Statutory report generation for 10 US states and federal ONRR
- Amended report handling
- Out-of-statute write-off processing
- Report template version management (state regulation changes)

**Inbound Events:**
- `ua.production.volumes.allocated` (M1) — production volumetric data
- `ua.revenue.distribution.completed` (M6) — royalty distribution values

**Outbound Events:** None.

**Key API Scopes:** `regulatory:read`, `regulatory:submit`, `regulatory:admin`

---

### M9 — Administration & ILM

**Service:** `svc-admin`  
**Schema:** `admin_ilm`  
**Primary Users:** System Administrators

**Responsibilities:**
- RBAC role and permission management UI
- Integration sync logging (SAP → EnerFusion replication events)
- Automation ruleset configuration (bots for M1 allocation scheduling)
- Data archiving policy management and execution
- Personal data identification, blocking, and anonymization
- Audit log access and reporting
- Onboarding validation tool

**Inbound Events:** Cross-cutting — subscribes to audit events from all modules.

**Outbound Events:** None (drives other services via API, not events).

**Key API Scopes:** `admin:read`, `admin:write`, `admin:ilm`, `admin:audit`

---

## Inter-Module Event Schema

All Service Bus events share a common envelope:

```typescript
interface PraEvent<T> {
  eventId: string;         // UUID — idempotency key for consumers
  eventType: string;       // e.g., "ua.production.volumes.allocated"
  moduleSource: string;    // e.g., "svc-production"
  companyCode: string;     // Tenant identifier
  period: string;          // "YYYY-MM" reporting period
  timestamp: string;       // ISO 8601 UTC
  correlationId: string;   // Propagated from originating HTTP request
  payload: T;              // Module-specific event data
}
```

---

## Module-Level RBAC Matrix

| Module | Reader | Writer | Approver | Executor | Admin |
|--------|--------|--------|----------|----------|-------|
| M1 Production | View volumes, wells | Enter volumes, downtime | — | Run allocation | Configure DNs |
| M2 Ownership | View DOI, transfers | Create/edit DOI, initiate transfer | Approve DOI changes, transfers | — | Import DOI bulk |
| M3 Allocation | View allocations | Manual adjustments, nominations | — | Trigger allocation | Configure gas plant |
| M4 Valuation | View settlements | Create contracts, pricing | Approve contracts, PPNs | Run valuation | Tax rate admin |
| M5 Balancing | View imbalances | Manual adjustments | Approve adjustments | — | PBA admin |
| M6 Revenue | View distributions, RADs | — | — | Run distribution | GL account config |
| M7 Payments | View checks, AR | Manual checks, write-offs | Approve write-offs | — | Escheat config |
| M8 Regulatory | View reports | — | — | Generate reports | Template admin |
| M9 Admin | View audit, sync logs | — | — | Run archive jobs | Manage roles, policies |
