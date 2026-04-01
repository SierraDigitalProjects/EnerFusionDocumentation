# 01 — System Overview
## EnerFusion Upstream Accounting

---

## 1. Business Context

EnerFusion Upstream Accounting is a cloud-native upstream accounting platform for North American energy operators. It replaces the functional scope of SAP PRA (IS-Oil) with a modular, API-first architecture purpose-built for containerized cloud deployment.

The system manages the **complete revenue lifecycle** from physical production at the wellhead to statutory compliance reporting and owner payment disbursement.

### Primary Users

| User Type | Role |
|-----------|------|
| Production Engineers | Enter daily volumes, well tests, downtime |
| Land/Ownership Administrators | Manage DOIs, ownership transfers, JVA data |
| Commercial/Contract Managers | Set up contracts, pricing formulas, nominations |
| Revenue Accountants | Run valuation, distribution, check processing |
| Compliance Officers | Generate and file state/federal regulatory reports |
| System Administrators | Configure automation, manage ILM, user access |

---

## 2. Functional Scope

### Core Modules

#### M1 — Production & Volume Allocation
Captures physical production data from fields, platforms, wells, and measurement points. Allocates bulk network volumes back to individual well completions using theoretical calculations, downtime data, and well test results.

**Key outcomes:** Allocated well completion volumes, measurement point volumes, daily production summaries.

#### M2 — Ownership & Division of Interest (DOI)
Manages the legal ownership structure of producing assets. Defines which business partners hold working interest or royalty interest in each lease/venture. Maintains chain-of-title records and processes ownership transfers.

**Key outcomes:** Validated DOI records, ownership transfer approvals, Schedule A (Transfer Order) reports.

#### M3 — Contractual Allocation & Product Control
Bridges physical production and commercial agreements. Allocates production volumes to sales contracts per owner entitlements. Manages gas plant processing, NGL component allocation, network imbalances, and daily nominations.

**Key outcomes:** Contract-level allocated volumes, entitlement calculations, marketing group network imbalances.

#### M4 — Contracts, Pricing & Valuation
Applies commercial pricing rules (fixed price, API gravity scales, formula pricing) and calculates taxes, marketing deductions, and royalty allowances against allocated volumes. Generates settlement statements.

**Key outcomes:** Valued settlement statements, oil/gas run statements, prior period adjustments (PPNs).

#### M5 — Balancing Workplace
Manages scenarios where owners take more or less product than their entitled share. Maintains Product Balancing Agreements (PBAs), tracks cumulative imbalances, and processes manual adjustments.

**Key outcomes:** Balanced owner positions, balancing statements, imbalance exception reports.

#### M6 — Revenue Distribution & Accounting
Distributes the financially valued production to each owner according to their interest percentage. Calculates interest on late payments. Generates Revenue Accounting Documents (RADs) for general ledger posting.

**Key outcomes:** Owner-level distribution amounts, interest calculations, RADs for financial posting.

#### M7 — Payment Processing & Check Input
Handles both incoming purchaser checks (CDEX/EDI) and outgoing owner payment disbursements. Manages accounts receivable aging, write-offs, and escheat/escrow processing for abandoned funds.

**Key outcomes:** Processed check registers, owner payment stubs, AR aging reports, escheat filings.

#### M8 — Regulatory, Tax & Royalty Reporting
Compiles data from all upstream modules to satisfy state and federal compliance mandates. Generates statutory report formats for Texas, Oklahoma, Louisiana, New Mexico, Wyoming, and federal ONRR.

**Key outcomes:** Filed statutory reports, amended reports, out-of-statute write-offs.

#### M9 — Administration & Information Lifecycle Management (ILM)
Provides system-wide automation configuration (bots, rulesets), data archiving for completed transactions, personal data protection (GDPR-aligned blocking/deletion), and onboarding/regression validation tools.

**Key outcomes:** Archived records, anonymized personal data, automated run schedules.

#### M10 — Partner Onboarding
Manages the complete lifecycle of bringing a new joint venture partner into EnerFusion through a structured 7-step workflow: Company Registration → JOA & Working Interest → Financial Setup → COPAS Compliance → Document Collection → System Access Provisioning → Review & Submit. Serves as the entry gate before partners receive DOI records in M2 and distributions from M6.

**Key outcomes:** Activated partner records, verified compliance documents, provisioned system access, DOI interest records in M2.

#### M11 — Joint Venture Accounting (JVA)
Handles all financial interactions between the operating company and non-operating working interest partners. Manages Authorization for Expenditure (AFE) capital approval elections, monthly Joint Interest Billing (JIB) under COPAS standards, cash call advance funding collection, and COPAS audit finding management.

**Key outcomes:** Approved AFE elections, issued JIB billings, collected cash calls, resolved audit findings.

---

## 3. End-to-End Business Process Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  M10: PARTNER ONBOARDING                                        │
│  • Company registration, JOA & working interest setup          │
│  • Financial setup, COPAS compliance, document collection      │
│  • System access provisioning → activates DOI in M2           │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓ ua.onboarding.partner.completed
┌─────────────────────────────────────────────────────────────────┐
│  FIELD OPERATIONS                                               │
│  Daily: Well volumes, downtime, well tests, pressures          │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  M1: PRODUCTION & VOLUME ALLOCATION                             │
│  • Calculate theoreticals per well completion                   │
│  • Aggregate delivery network inputs                            │
│  • Compare against actual sales outputs                         │
│  • Adjust within tolerance → allocate back to wells             │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  M2: OWNERSHIP / DOI                                            │
│  • DOI→MP/WC cross-reference: who owns each well's volume       │
│  • Apply sliding scales, tax exemptions                         │
│  • Process any in-period ownership transfers                    │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  M3: CONTRACTUAL ALLOCATION & PRODUCT CONTROL                   │
│  • MP/WC → Transporter Contract cross-reference                 │
│  • Allocate volumes to sales contracts                          │
│  • Calculate entitlements, track network imbalances             │
│  • Gas plant: calc percent return-to-lease, NGL splits          │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  M4: VALUATION & PRICING                                        │
│  • Apply contract price (fixed / formula / API gravity)         │
│  • Deduct marketing costs, transportation                       │
│  • Calculate severance tax, ad valorem, royalty deductions      │
│  • Generate settlement statements & run statements              │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  M5: BALANCING WORKPLACE      M11: JOINT VENTURE ACCOUNTING     │
│  • Compare actual takes       • AFE elections (participate /    │
│    vs entitlements              non-consent)                    │
│  • Update cumulative          • JIB billing (COPAS monthly)    │
│    owner imbalance positions  • Cash call management            │
│  • Generate balancing         • COPAS audit findings           │
│    statements                                                   │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  M6: REVENUE DISTRIBUTION & ACCOUNTING                          │
│  • Distribute net revenue to each owner interest                │
│  • Calculate late-payment interest                              │
│  • Post Revenue Accounting Documents to GL                      │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  M7: PAYMENT PROCESSING & CHECK INPUT                           │
│  • Process incoming purchaser checks (CDEX/EDI)                 │
│  • Issue outgoing owner payment checks                          │
│  • Handle escheat / escrow for unclaimed funds                  │
│  • Manage AR aging and write-offs                               │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  M8: REGULATORY REPORTING                                       │
│  • Texas: H-10, P1B, PR, GLO/UT Royalty                        │
│  • Federal: MMS-PASR, MMS-OGOR, ONRR-2014                     │
│  • Multi-state: AR, CA, CO, LA, NM, ND, OK, WY                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  M9: ADMINISTRATION & ILM  (cross-cutting — all modules)        │
│  • RBAC role and user access management                         │
│  • Tamper-evident audit log of all data mutations               │
│  • Automation rulesets (bots) for scheduling and anomaly alerts │
│  • Data archiving and personal data protection (GDPR-aligned)   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Volume Allocation Mathematics (M1 Core Algorithm)

The 8-step allocation algorithm reconciles estimated production with actual measured sales:

| Step | Operation | Formula |
|------|-----------|---------|
| 1 | Determine Theoreticals | `theoretical = production × hoursUp / (testDuration / 60)` |
| 2 | Determine Network Inputs | `inputs = Σ(beginning theoreticals)` |
| 3 | Determine Outputs | `outputs = sales + (endingInventory − beginningInventory)` |
| 4 | Calculate % Difference | `pctDiff = ((outputs − inputs) / inputs) × 100` |
| 5 | Check Tolerance | Compare pctDiff against DN tolerance threshold |
| 6 | Adjust Inputs | Scale theoreticals so Σ inputs = outputs (if within tolerance) |
| 7 | Compute Final Production | `production = fuel + inventory + nonSales + sales − lease − receipts` |
| 8 | Post Allocated Volumes | Write per-well-completion allocated volumes to transaction store |

---

## 5. Regulatory Report Coverage

### State-Level Reports

| State | Report Form | Content |
|-------|-------------|---------|
| Texas | Form H-10, P1B, PR | Production volumes, well status |
| Texas | GLO Royalty, UT Royalty | State land royalty distribution |
| Arkansas | Form 7 | Production reporting |
| California | Report OG110 | Oil and gas production |
| Colorado | Form 7 | Production volumes |
| Louisiana | OGP/R5D, WR1 | Production and royalty |
| New Mexico | Report C115 | Monthly production |
| North Dakota | ND Reports | Production volumes |
| Oklahoma | Form 1004 | Gross production tax |
| Wyoming | Form 2, Gross Production Tax | Production and escheat |

### Federal Reports

| Agency | Form | Content |
|--------|------|---------|
| ONRR (formerly MMS) | MMS-PASR | Production accounting summary |
| ONRR | MMS-OGOR | Oil and gas operations report |
| ONRR | ONRR-2014 | Royalty report — federal leases |
