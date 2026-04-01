# M11 — Joint Venture Accounting (JVA)

## Module Overview

**Service:** `svc-jva`  
**Schema:** `jva`  
**Primary Users:** JV Accountants, Revenue Analysts, Finance Controllers  
**Domain:** Joint Venture Finance

Joint Venture Accounting manages all financial interactions between EnerFusion (as operating partner) and non-operating working interest partners. It enforces COPAS (Council of Petroleum Accountants Societies) accounting standards across Authorization for Expenditure (AFE) approvals, Joint Interest Billing (JIB) preparation, cash call management, and audit tracking.

---

## Business Context

When a company operates a producing asset on behalf of co-owners (non-operating working interest partners), it must:

- Seek partner approval for capital expenditures via AFE elections
- Bill each partner monthly for their proportionate share of operating costs via JIB
- Collect advance funding for upcoming capital programs via Cash Calls
- Support COPAS-standard audit processes for any disputed billings

JVA replaces spreadsheet-based billing and email-based AFE election workflows with a fully integrated, auditable system connected to partner records (M10), ownership data (M2), and payment processing (M7).

---

## JVA Business Process Flow

```
AFE                 Partner             JIB
Authorization  →    Elections      →    Preparation
(project budget)    (participate/       (actual billing)
                    non-consent)              │
                                              ▼
Audit               Cash Call           Receipt
Management     ←    Funding        ←    Application
(findings)          (calls/overdue)     (payments)
```

---

## Sub-Module: AFE Management (Authorization For Expenditure)

### AFE Creation

Capital project approval requests issued to all non-operating partners before expenditure begins.

| Field | Description |
|-------|-------------|
| AFE number | Unique identifier |
| AFE type | Drilling / Workover / Facility / Plug & Abandon |
| Description | Project description and location |
| Estimated total cost | Gross authorized amount |
| COPAS cost breakdown | By cost code: 100-Labor, 200-Materials, 300-Transport, 400-Services, 600-Overhead, 700-Taxes |
| Effective date | Authorization start date |
| Expiration date | When elections must be submitted |

### NOP Elections (Non-Operating Partner)

Non-operating partners respond to each AFE with a participation decision.

| Election Status | Description |
|----------------|-------------|
| Participating | Partner consents to their WI% share of costs |
| Non-consent | Partner elects not to participate; non-consent penalty rate applies |
| Pending | Deadline not yet reached; awaiting response |
| Defaulted | Deadline passed with no response; treated as non-consent |

**Non-consent penalty:** Configurable penalty multiplier on the operator's AFE costs recovered from non-consenting partners' production before revenue distributions begin.

### Budget Tracking

Real-time comparison of authorized amount vs. actual incurred costs.

| Metric | Description |
|--------|-------------|
| Authorized amount | Original AFE approval amount |
| Actual to date | Sum of approved JIB charges linked to AFE |
| Remaining budget | Authorized minus actual |
| Variance % | Triggers alert when actual exceeds threshold (default 10%) |

### Cost Tracker

Per-COPAS-category breakdown of real-time budget vs. actual:

| COPAS Code | Category |
|-----------|----------|
| 100 | Labor |
| 200 | Materials |
| 300 | Transportation |
| 400 | Services |
| 600 | Overhead |
| 700 | Taxes |

---

## Sub-Module: JIB Preparation (Joint Interest Billing)

Monthly billing of non-operating partners for their proportionate share of operating costs.

### Billing Calculation

| Component | Calculation |
|-----------|-------------|
| Direct costs | Actual operating costs × partner WI% |
| COPAS overhead | Fixed Rate per Well per COPAS schedule |
| Escalation adjustment | COPAS Index applied to overhead base |
| Total JIB amount | Direct costs + overhead |

### JIB Line Items

Each JIB is broken down by COPAS cost code with:
- Cost category description
- Gross amount
- Partner WI%
- Net billed amount

### Delivery

| Delivery Method | Detail |
|----------------|--------|
| EDI 810 | Electronic billing to partner ERP systems |
| Email | PDF attachment per partner preference |
| Portal | Partner portal self-service view |

### Payment Status Tracking

| Status | Description |
|--------|-------------|
| Issued | JIB sent; payment due |
| Partial | Partial payment received |
| Paid | Payment applied in full |
| Overdue | Past due date; interest accrual begins |

---

## Sub-Module: Cash Call Management

Advance funding requests for upcoming capital or operating expenditures.

### Cash Call Record

| Field | Description |
|-------|-------------|
| Cash call ID | Unique identifier |
| Partner | Non-operating partner being billed |
| Amount due | Partner's WI% share of upcoming expenditure |
| Due date | Required payment date |
| Purpose | Linked AFE or operating program description |
| Status | Issued / Partial / Paid / Overdue |

### Overdue Detection and Interest Accrual

- Overdue flag set when due date passes without full payment
- Daily interest accrues on overdue amount at configurable rate
- Interest ledger maintained per cash call

### Receipt Application

Incoming payments are matched against open cash calls:
1. Match payment to partner record
2. Apply to oldest outstanding cash call first (configurable)
3. If overpayment: credit to partner suspense account
4. If partial: update remaining balance; maintain overdue flag if applicable

### Forecasting

- Projects upcoming cash requirements by summing open AFE-linked expenditures
- Shows partner-level funding shortfall
- Integrates with M7 (Payment Processing) for partner payment receipts

---

## Sub-Module: Audit Management

Documents and tracks COPAS audit findings raised by non-operating partners.

### Audit Finding Record

| Field | Description |
|-------|-------------|
| Finding ID | Unique identifier |
| Audit period | Billing period under dispute |
| COPAS category | Cost code disputed (100–700) |
| Dollar amount | Amount under audit |
| Partner | Non-operating partner raising the finding |
| Status | Open / Under Review / Resolved / Rejected |
| Resolution notes | Operator response and supporting documentation |
| Resolution date | Date finding closed |

### Audit Timeline

- Audit window: configurable (default 24 months per COPAS)
- Overdue detection: findings not resolved within 90 days flagged
- Summary report by period, partner, and COPAS category

---

## Master Data Entities

### JVA Partner Interest

**Schema:** `jva.partner_interests`

Links JVA billing configuration to M2 ownership records.

| Attribute | Type | Description |
|-----------|------|-------------|
| `interest_id` | UUID | Primary key |
| `partner_id` | UUID | FK → onboarding.partners |
| `venture_id` | UUID | FK → ownership.ventures |
| `wi_percentage` | DECIMAL(10,8) | Working Interest for cost billing |
| `nri_percentage` | DECIMAL(10,8) | Net Revenue Interest for revenue |
| `copas_version` | VARCHAR(20) | COPAS standard version |
| `overhead_method` | ENUM | fixed_rate_per_well, flat_rate |
| `effective_date` | DATE | Interest effective date |

### AFE

**Schema:** `jva.afes`

| Attribute | Type | Description |
|-----------|------|-------------|
| `afe_id` | UUID | Primary key |
| `afe_number` | VARCHAR(20) | Unique AFE reference |
| `afe_type` | ENUM | drilling, workover, facility, plug_abandon |
| `authorized_amount` | DECIMAL(15,2) | Gross authorized amount |
| `election_deadline` | DATE | NOP response deadline |
| `status` | ENUM | draft, issued, approved, closed |

### JIB (Joint Interest Bill)

**Schema:** `jva.jibs`

| Attribute | Type | Description |
|-----------|------|-------------|
| `jib_id` | UUID | Primary key |
| `partner_id` | UUID | FK → onboarding.partners |
| `billing_period` | CHAR(7) | YYYY-MM |
| `direct_cost_amount` | DECIMAL(15,2) | Direct operating costs billed |
| `overhead_amount` | DECIMAL(15,2) | COPAS overhead charged |
| `total_amount` | DECIMAL(15,2) | Total JIB amount |
| `status` | ENUM | issued, partial, paid, overdue |
| `due_date` | DATE | Payment due date |

### Cash Call

**Schema:** `jva.cash_calls`

| Attribute | Type | Description |
|-----------|------|-------------|
| `call_id` | UUID | Primary key |
| `partner_id` | UUID | FK → onboarding.partners |
| `afe_id` | UUID | FK → jva.afes (optional) |
| `amount_due` | DECIMAL(15,2) | Amount requested |
| `due_date` | DATE | Required payment date |
| `amount_received` | DECIMAL(15,2) | Payments applied |
| `status` | ENUM | issued, partial, paid, overdue |
| `interest_accrued` | DECIMAL(15,2) | Overdue interest to date |

### Audit Finding

**Schema:** `jva.audit_findings`

| Attribute | Type | Description |
|-----------|------|-------------|
| `finding_id` | UUID | Primary key |
| `partner_id` | UUID | FK → onboarding.partners |
| `audit_period` | CHAR(7) | YYYY-MM billing period disputed |
| `copas_category` | ENUM | labor, materials, transport, services, overhead, taxes |
| `amount_disputed` | DECIMAL(15,2) | Dollar amount under audit |
| `status` | ENUM | open, under_review, resolved, rejected |
| `resolution_date` | DATE | Date finding closed |

---

## Transactional Events

| Event | Published When | Consumed By |
|-------|---------------|-------------|
| `ua.jva.afe.approved` | All partner elections received and AFE authorized | M1 (production tracking), M9 (audit log) |
| `ua.jva.jib.issued` | JIB generated and delivered to partner | M7 (receivable tracking), M9 (audit log) |
| `ua.jva.cashcall.overdue` | Cash call past due date | M9 (notifications, audit log) |
| `ua.jva.jib.paid` | JIB payment fully applied | M7 (payment reconciliation) |

---

## Integration Points

| Module | Interaction |
|--------|------------|
| M2 — Ownership & DOI | JVA partner interests mirror DOI WI% records; ownership changes trigger JVA interest update |
| M7 — Payment Processing | JIB receivables feed M7 AR aging; JIB netting applied against owner revenue distributions |
| M10 — Partner Onboarding | Completed onboarding activates JVA partner interest records; COPAS config from onboarding Step 4 |
| M9 — Administration & ILM | All JVA mutations logged to audit trail; archive policies apply to closed AFEs and resolved findings |

---

## RBAC

| Scope | Permission |
|-------|-----------|
| `jva:read` | View AFEs, JIBs, cash calls, audit findings |
| `jva:write` | Create AFEs, prepare JIBs, issue cash calls |
| `jva:approve` | Authorize AFEs, approve JIB billing runs |
| `jva:execute` | Run JIB billing, apply cash call receipts |
| `jva:admin` | Configure COPAS rates, overhead methods, audit windows |

---

## Reports

| Report | Description |
|--------|-------------|
| AFE Election Status Report | All open AFEs with partner election summary |
| JIB Billing Summary | Monthly JIB amounts by partner and COPAS category |
| JIB Aging Report | Outstanding JIB balances by age bucket |
| Cash Call Forecast | Upcoming funding requirements by partner |
| Overdue Cash Call Report | Calls past due with accrued interest |
| COPAS Audit Finding Summary | Open audit findings by partner, period, and category |
| Budget vs. Actual by AFE | COPAS cost code variance tracking per project |
