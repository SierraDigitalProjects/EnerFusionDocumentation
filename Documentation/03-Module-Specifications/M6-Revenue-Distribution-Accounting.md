# M6 — Revenue Distribution & Accounting

---

## 1. Module Purpose

Distributes the financially valued production (from M4 settlement statements) to each owner according to their Division of Interest (DOI) decimal interest. Calculates late-payment interest on overdue distributions, and generates Revenue Accounting Documents (RADs) for posting to the external General Ledger system.

This module is the **financial disbursement engine** — it converts settlement values into owner-level entitlements that drive check issuance in M7.

---

## 2. Core Concepts

| Concept | Definition |
|---------|------------|
| **Revenue Accounting Document (RAD)** | The accounting document generated per owner distribution that maps to GL account, cost center, and amount for ERP posting |
| **Distribution Run** | A batch process that calculates and records owner-level revenue shares for a given period and venture scope |
| **Interest on Late Payment** | Statutory or contractual interest charged when the operator fails to distribute revenue by the applicable state-mandated payment deadline |
| **Revenue Distribution** | The allocation of net settlement value to individual DOI owner interests per their decimal percentage |
| **Chain of Title Allocation** | Distribution of revenue through transferred ownership interests, ensuring value follows the correct chain of ownership for the period |
| **Owner Suspense** | Funds held on behalf of an owner when the owner's payment details are incomplete or disputed, pending resolution |

---

## 3. Master Data

### 3.1 Revenue Distribution Configuration

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| distributionConfigId | UUID | Yes | PK |
| companyCode | String(10) | Yes | Operator company |
| productType | Enum(oil, gas, ngl) | Yes | |
| distributionMethod | Enum(decimal_interest, entitlement) | Yes | How to split net value |
| interestRateSource | Enum(state_statutory, contractual, federal) | Yes | Source for late-payment interest rate |
| paymentDeadlineDays | Integer | Yes | Days after period close; triggers interest if missed |
| glPostingMethod | Enum(rad, journal_entry) | Yes | How documents are posted to GL |
| affiliateNettingFlag | Boolean | No | Net intercompany distributions before issuing |

### 3.2 GL Account Mapping (PRA Chart of Accounts)

| Attribute | Type | Description |
|-----------|------|-------------|
| glMappingId | UUID | PK |
| productType | Enum | Oil / Gas / NGL |
| transactionType | Enum(revenue, royalty, severance_tax, ad_valorem, transport, marketing, interest) | |
| interestType | Enum(WI, RI, ORRI) | |
| glAccountNumber | String(20) | FK → shared.gl_accounts (SAP-replicated) |
| costCenterDefault | String(20) | Default cost center if not overridden |
| effectiveDate | Date | |

### 3.3 Interest Rate Table

| Attribute | Type | Description |
|-----------|------|-------------|
| interestRateId | UUID | PK |
| state | String(2) | State abbreviation (or 'FED' for federal) |
| productType | Enum | |
| annualRate | Decimal(6,4) | Annual interest rate (e.g., 0.1200 = 12%) |
| compoundingMethod | Enum(simple, compound_monthly) | |
| effectiveDate | Date | |
| expirationDate | Date | |

**State-mandated payment deadlines and interest rates vary:**

| State | Payment Deadline | Interest Rate (typical) |
|-------|-----------------|------------------------|
| Texas | 60 days after end of production month | Variable; set by state |
| Oklahoma | 6 months after production month | 12% per annum |
| Louisiana | 60 days after production month | Prime + 1% |
| New Mexico | 6 months after production month | 15% per annum |
| Wyoming | 6 months after production month | 18% per annum |

---

## 4. Transactional Entities

### 4.1 Distribution Run

| Attribute | Type | Description |
|-----------|------|-------------|
| distributionRunId | UUID | PK |
| period | YearMonth | Accounting period |
| companyCode | String(10) | |
| runScope | Enum(all_ventures, single_venture, settlement_id) | |
| runStatus | Enum(pending, running, completed, failed, reversed) | |
| startedAt | Timestamp | |
| completedAt | Timestamp | |
| triggeredBy | UUID | FK → User |
| totalDistributed | Decimal(16,2) | Sum of all owner net distributions |
| ownerCount | Integer | Number of distinct owners with distributions |

### 4.2 Owner Distribution Record

| Attribute | Type | Description |
|-----------|------|-------------|
| distributionId | UUID | PK |
| distributionRunId | UUID | FK → DistributionRun |
| ventureId | UUID | FK → Venture |
| settlementId | UUID | FK → SettlementStatement (M4) |
| businessPartnerId | UUID | FK → BusinessPartner |
| interestType | Enum(WI, RI, ORRI, NPRI, PRRI) | |
| decimalInterest | Decimal(10,8) | Owner's share |
| period | YearMonth | |
| productType | Enum | |
| grossVolume | Decimal(12,3) | Owner's share of gross production |
| grossValue | Decimal(14,2) | Owner's share before deductions |
| severanceTaxShare | Decimal(12,2) | Owner's portion of severance tax |
| adValoremShare | Decimal(12,2) | |
| transportDeductShare | Decimal(12,2) | |
| marketingDeductShare | Decimal(12,2) | |
| processingDeductShare | Decimal(12,2) | |
| netValue | Decimal(14,2) | Owner's net payment amount |
| suspenseFlag | Boolean | Held pending owner details |
| suspenseReason | String(100) | Reason if suspended |

### 4.3 Revenue Accounting Document (RAD)

| Attribute | Type | Description |
|-----------|------|-------------|
| radId | UUID | PK |
| distributionRunId | UUID | FK |
| documentDate | Date | Posting date |
| postingPeriod | YearMonth | GL posting period |
| companyCode | String(10) | |
| glLines | JSONB | Array of GL line items |
| postingStatus | Enum(draft, posted, failed, reversed) | |
| externalDocumentNumber | String(20) | SAP FI document number (returned after posting) |
| postedAt | Timestamp | |

**GL Line Item Structure (within `glLines` JSONB):**

```json
{
  "lineNumber": 1,
  "glAccount": "130000",
  "costCenter": "1001-OIL",
  "debitCredit": "C",
  "amount": 145230.50,
  "currency": "USD",
  "productType": "oil",
  "transactionType": "revenue",
  "ventureId": "uuid",
  "period": "2024-01"
}
```

### 4.4 Interest Calculation Record

| Attribute | Type | Description |
|-----------|------|-------------|
| interestId | UUID | PK |
| businessPartnerId | UUID | FK → BusinessPartner |
| ventureId | UUID | FK |
| period | YearMonth | Period the original distribution relates to |
| originalDueDate | Date | State-mandated payment deadline |
| actualPaymentDate | Date | Date payment was actually issued |
| overdueDays | Integer | actualPaymentDate - originalDueDate |
| principalAmount | Decimal(14,2) | Original undistributed amount |
| interestRate | Decimal(6,4) | Annual rate applied |
| interestAmount | Decimal(12,2) | Calculated interest owed to owner |
| interestStatus | Enum(calculated, approved, paid, waived) | |

---

## 5. Business Processing Rules

### 5.1 Distribution Run Algorithm

```
FUNCTION runDistribution(period, companyCode, scope):
  1. Fetch all SettlementStatements (M4) for period + scope
     WHERE statementStatus = 'posted'
  
  2. FOR each settlement:
     a. Fetch DOI owner interests via SS/DOI cross-reference (M4)
     b. FOR each owner interest:
          grossVolume = settlement.allocatedVolume × owner.decimalInterest
          grossValue  = settlement.grossValue      × owner.decimalInterest
          
          // Apply per-owner adjustments
          severanceTax = settlement.severanceTax × owner.decimalInterest
          IF owner.interestType = 'RI' OR owner.taxExemptionCode IS NOT NULL:
            severanceTax = calculateOwnerSpecificTax(owner, grossValue, state)
          
          IF owner.marketingFreeStatus = TRUE:
            marketingDeduct = 0
          ELSE:
            marketingDeduct = settlement.marketingDeduct × owner.decimalInterest
          
          netValue = grossValue
                   - severanceTax - adValorem
                   - transportDeduct - marketingDeduct - processingDeduct
          
          IF owner.paymentMethod = 'TAKE_IN_KIND':
            netValue = 0  // Volume take; no cash distribution
          
          PERSIST OwnerDistributionRecord
  
  3. Generate Revenue Accounting Documents (RADs)
  4. Submit RADs to GL posting queue
  5. PUBLISH event: ua.revenue.distribution.completed
```

### 5.2 Chain of Title Distribution

When ownership changed mid-period (via an approved transfer in M2):

```
FUNCTION distributeWithChainOfTitle(ventureId, period):
  transfers = fetch transfers effective within period for venture
  
  FOR each day in period:
    activeOwners = DOI interests active on that day
    dailyValue = settlement.netValue / daysInPeriod
    
    FOR each activeOwner:
      ownerShare = dailyValue × owner.decimalInterest
      accumulateToOwner(owner, ownerShare)
  
  // Final distribution is proportional to days-held within period
```

### 5.3 Late-Payment Interest Calculation

```
FUNCTION calculateLatePaymentInterest(distributionId):
  dist = fetch OwnerDistributionRecord
  state = fetch venture state from lease
  deadline = period.lastDay + interestConfig.paymentDeadlineDays
  
  IF actualPaymentDate > deadline:
    overdueDays = (actualPaymentDate - deadline).days
    rate = fetch InterestRate for state + productType (active on deadline)
    
    IF rate.compoundingMethod = 'simple':
      interest = dist.netValue × rate.annualRate × (overdueDays / 365)
    ELSE:
      months = CEIL(overdueDays / 30)
      interest = dist.netValue × ((1 + rate.annualRate/12)^months - 1)
    
    PERSIST InterestCalculationRecord
    ADD interest to owner's next payment run
```

### 5.4 Owner Suspense

Distributions are suspended (not paid) when:

- Owner has no valid payment method configured
- Owner has no mailing address
- Owner is flagged "Do Not Pay" (legal hold)
- Owner payment is below minimum threshold (threshold check happens in M7)

Suspended distributions accumulate in the owner's balance and are released when the blocking condition is resolved.

---

## 6. RAD Posting to SAP FI

Revenue Accounting Documents are submitted to SAP FI via the BAPI integration:

```
EnerFusion RAD → POST /api/v1/revenue/documents/{id}/gl-posting
    │
    ▼
Integration Layer → BAPI_ACC_DOCUMENT_POST
    │
    ▼
SAP FI Journal Entry (document type: RD)
    │
    ▼
SAP returns: companyCode + fiscalYear + documentNumber
    │
    ▼
EnerFusion: UPDATE rad SET externalDocumentNumber = SAP_docNum,
                           postingStatus = 'posted'
```

If posting fails, the RAD is marked `postingStatus = 'failed'` and routed to the Revenue Accountant's error queue for review and resubmission.

---

## 7. API Endpoints

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| POST | `/api/v1/revenue/distribution/run` | Trigger distribution run | `revenue:execute` |
| GET | `/api/v1/revenue/distribution/runs/{id}` | Run status and summary | `revenue:read` |
| GET | `/api/v1/revenue/distribution/runs/{id}/owners` | Owner distribution details | `revenue:read` |
| GET | `/api/v1/revenue/distributions` | Query distributions by venture/period/owner | `revenue:read` |
| GET | `/api/v1/revenue/distributions/{id}` | Single distribution record | `revenue:read` |
| GET | `/api/v1/revenue/documents` | List RADs | `revenue:read` |
| GET | `/api/v1/revenue/documents/{id}` | RAD detail with GL lines | `revenue:read` |
| POST | `/api/v1/revenue/documents/{id}/gl-posting` | Submit RAD to GL | `revenue:execute` |
| GET | `/api/v1/revenue/interest` | Query interest calculations | `revenue:read` |
| POST | `/api/v1/revenue/interest/calculate` | Trigger interest calculation run | `revenue:execute` |
| GET | `/api/v1/revenue/interest/rates` | List state interest rates | `revenue:read` |
| POST | `/api/v1/revenue/interest/rates` | Create interest rate | `revenue:admin` |
| GET | `/api/v1/revenue/suspense` | List suspended distributions | `revenue:read` |
| PUT | `/api/v1/revenue/suspense/{id}/release` | Release suspended distribution | `revenue:write` |

---

## 8. Reports

| Report | Description | Trigger |
|--------|-------------|---------|
| Owner Interest Summary | Net distribution by owner × product × period | Monthly close |
| Interest Calculation Detail | Overdue days, principal, rate, interest amount per owner | Monthly / on demand |
| Summary Interest Paid | Aggregate interest paid by state / company / period | Period end |
| Revenue Distribution Register | Full detail of all owner distributions for a run | Distribution run completion |
| RAD Posting Status Report | RADs posted vs. failed vs. pending, with SAP document references | Daily / on demand |
| Suspense Balance Report | Owners with suspended distributions, reason, and aging | Monthly / on demand |
| GL Reconciliation Report | Compares EnerFusion distribution totals to SAP FI posted amounts | Monthly close |
| Validations and Exceptions Report | Distributions that failed validation rules before posting | Distribution run completion |
