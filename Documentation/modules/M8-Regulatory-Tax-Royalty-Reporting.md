# M8 — Regulatory, Tax & Royalty Reporting

---

## 1. Module Purpose

Generates all statutory, tax, and royalty reports required by federal and state regulatory bodies. This includes ONRR-2014 royalty reporting for federal and Indian leases, state severance and production tax returns, state production reports, and ad valorem declarations. M8 provides the template management system that allows report formats to be updated without a full application release.

This module is the **compliance and reporting engine** — it takes settled, distributed production data and formats it into the specific layouts and submission structures mandated by each regulatory authority.

---

## 2. Core Concepts

| Concept | Definition |
|---------|------------|
| **ONRR-2014** | The federal form used to report and pay royalties on oil, gas, and NGL produced from federal and Indian leases; submitted to the Office of Natural Resources Revenue |
| **Transaction Code (TC)** | A two-digit code on the ONRR-2014 that classifies the nature of each reported line (e.g., TC01 = normal production, TC13 = processing allowance) |
| **Payor** | The entity (operator or non-operator) legally responsible for submitting ONRR royalty payments and reports |
| **PPN (Production Payment Number)** | ONRR-assigned identifier for each lease-level royalty line; used to group TC lines within the ONRR-2014 |
| **Section 6 Provision** | An ONRR regulatory provision allowing operators to reduce royalty rates under defined conditions; requires additional rate lookup and TC 37/38 generation |
| **Statutory Report Template** | A versioned, configurable report layout stored in the system that can be updated by compliance staff without code changes |
| **Amended Report** | A corrective filing submitted to a regulatory body to replace a previously filed report; triggers out-of-period adjustment processing |
| **Out-of-Statute Write-Off** | When an amended filing would correct an error beyond the regulatory statute of limitations, the difference is written off to a designated GL account rather than filed |
| **Indian Recoupment** | ONRR-specific calculation for Indian mineral leases; allows operators to recover prior overpayments by offsetting against current royalties owed |
| **State Severance Tax Return** | State-mandated periodic filing reporting production volumes and values, with corresponding tax payments |

---

## 3. Master Data

### 3.1 Statutory Report Template

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| templateId | UUID | Yes | PK |
| reportCode | String(20) | Yes | Short code (e.g., ONRR-2014, TX-PROD-MONTHLY, OK-SEV) |
| reportName | String(100) | Yes | Full descriptive name |
| regulatoryAuthority | Enum(ONRR, TX_RRC, OK_CC, NM_OCD, WY_OSC, CO_COGCC, LA_DNR, ND_IC, KS_CC, MT_BOGC) | Yes | Issuing agency |
| filingFrequency | Enum(monthly, quarterly, annual) | Yes | |
| submissionMethod | Enum(electronic_portal, edi, email, paper) | Yes | |
| templateVersion | String(10) | Yes | Version of the format (e.g., "2024-01") |
| effectiveDate | Date | Yes | When this template version takes effect |
| expirationDate | Date | No | When superseded by next version |
| templateDefinition | JSONB | Yes | Field layout, validations, and formatting rules |
| isActive | Boolean | Yes | |

### 3.2 ONRR Payor Cross-Reference

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| payorXRefId | UUID | Yes | PK |
| companyCode | String(10) | Yes | EnerFusion company code |
| onrrPayorCode | String(10) | Yes | ONRR-assigned payor code |
| reportingEntityName | String(100) | Yes | Name as registered with ONRR |
| effectiveDate | Date | Yes | |
| expirationDate | Date | No | |

### 3.3 ONRR Lease Hierarchy

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| onrrLeaseId | UUID | Yes | PK |
| ventureId | UUID | Yes | FK → Venture (M2) |
| leaseName | String(100) | Yes | |
| onrrLeaseNumber | String(20) | Yes | ONRR-assigned lease number |
| leaseType | Enum(federal_onshore, federal_offshore, indian_allotted, indian_tribal) | Yes | |
| state | String(2) | Yes | |
| mineralType | Enum(oil, gas, ngl, geothermal) | Yes | |
| royaltyRate | Decimal(6,4) | Yes | Default royalty rate (e.g., 0.1250 = 12.5%) |
| section6Applicable | Boolean | Yes | Whether Section 6 provisions may apply |
| indianRecoupmentApplicable | Boolean | Yes | Indian mineral lease flag |
| payorCode | String(10) | Yes | FK → ONRRPayorCrossReference |

### 3.4 PPN (Production Payment Number) Configuration

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| ppnId | UUID | Yes | PK |
| onrrLeaseId | UUID | Yes | FK → ONRRLease |
| salesTypeCode | String(3) | Yes | ONRR sales type (e.g., OIL, GAS, NGP) |
| productCode | String(4) | Yes | ONRR product code |
| ppnType | Enum(auto_generated, manually_assigned, inherited) | Yes | How the PPN was assigned |
| ppnValue | String(20) | Yes | The actual PPN identifier |
| effectiveDate | Date | Yes | |
| expirationDate | Date | No | |

**PPN Type Precedence:**

| Priority | PPN Type | Description |
|----------|----------|-------------|
| 1 | Manually Assigned | Set by compliance staff; overrides all others |
| 2 | Inherited | Carried forward from a prior period's filing |
| 3 | Auto-Generated | System-derived from lease + sales type + product |

### 3.5 Section 6 Rate Configuration

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| section6RateId | UUID | Yes | PK |
| onrrLeaseId | UUID | Yes | FK — lease-level override takes priority |
| state | String(2) | No | If null, federal default applies |
| productType | Enum | Yes | |
| section6RoyaltyRate | Decimal(6,4) | Yes | Reduced royalty rate under Section 6 |
| qualificationCriteria | String(500) | Yes | Legal basis for the reduced rate |
| effectiveDate | Date | Yes | |
| expirationDate | Date | No | |

**Section 6 Rate Resolution Hierarchy:**

```
1. Lease-level override (onrrLeaseId match)
2. State + product default
3. Federal default rate
4. If none found → error; filing blocked until rate is configured
```

### 3.6 State Tax Rate Configuration

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| stateTaxRateId | UUID | Yes | PK |
| state | String(2) | Yes | State abbreviation |
| productType | Enum | Yes | |
| taxType | Enum(severance, conservation, ad_valorem, franchise) | Yes | |
| rateType | Enum(flat_percent, tiered, volume_based, value_based) | Yes | |
| rate | Decimal(8,6) | Yes | Rate value |
| tierDefinition | JSONB | No | Tier breakpoints for tiered rates |
| exemptionCodes | String[] | No | Codes for qualifying exemptions |
| effectiveDate | Date | Yes | |
| expirationDate | Date | No | |

---

## 4. Transactional Entities

### 4.1 Regulatory Report Run

| Attribute | Type | Description |
|-----------|------|-------------|
| reportRunId | UUID | PK |
| templateId | UUID | FK → StatutoryReportTemplate |
| period | YearMonth | Reporting period |
| companyCode | String(10) | |
| runType | Enum(original, amended, test) | |
| runStatus | Enum(pending, generating, review, approved, submitted, accepted, rejected) | |
| startedAt | Timestamp | |
| completedAt | Timestamp | |
| triggeredBy | UUID | FK → User |
| submittedAt | Timestamp | When sent to regulatory body |
| submissionReference | String(50) | Confirmation number from regulatory portal |
| priorRunId | UUID | FK → self; populated for amended runs |

### 4.2 ONRR-2014 Report Record

| Attribute | Type | Description |
|-----------|------|-------------|
| onrrRecordId | UUID | PK |
| reportRunId | UUID | FK → RegulatoryReportRun |
| onrrLeaseId | UUID | FK → ONRRLease |
| ppnId | UUID | FK → PPN |
| period | YearMonth | |
| tcCode | String(2) | TC01, TC11, TC13, TC15, TC37, TC38 |
| salesTypeCode | String(3) | |
| productCode | String(4) | |
| royaltyVolume | Decimal(12,3) | Royalty-bearing volume |
| salesValue | Decimal(14,2) | Gross sales value |
| deductions | Decimal(12,2) | Allowable deductions (transport, processing) |
| royaltyValue | Decimal(14,2) | Taxable value after deductions |
| royaltyRate | Decimal(6,4) | Applied royalty rate |
| royaltyDue | Decimal(14,2) | Calculated royalty amount |
| section6Applied | Boolean | Whether Section 6 reduced rate was applied |
| indianRecoupmentAmount | Decimal(12,2) | Indian lease overpayment recouped this period |
| lineStatus | Enum(draft, validated, included, excluded, amended) | |
| validationMessages | JSONB | Array of validation warnings/errors |

**TC Code Generation Conditions:**

| TC Code | Name | Generation Condition |
|---------|------|---------------------|
| TC 01 | Normal Production | Standard royalty-bearing production; always generated |
| TC 11 | Transportation Allowance | Approved arm's-length transportation deduction exists |
| TC 13 | Processing Allowance | Approved processing allowance or quality differential (ERQADIFD) |
| TC 15 | No Production | Lease is active but no production occurred in period |
| TC 37 | Section 6 — Reduced Rate | Section 6 qualification confirmed; rate below standard |
| TC 38 | Section 6 — Standard Rate | Section 6 qualification confirmed; standard rate portion |

!!! warning "TC 13 Restriction"
    Quality differential deductions (ERQADIFD) map to TC 13 — NOT to the standard processing deduction bucket. Mapping ERQADIFD to processing deductions is a filing error.

### 4.3 State Tax Report Record

| Attribute | Type | Description |
|-----------|------|-------------|
| stateTaxRecordId | UUID | PK |
| reportRunId | UUID | FK → RegulatoryReportRun |
| ventureId | UUID | FK |
| period | YearMonth | |
| productType | Enum | |
| taxType | Enum | Severance / conservation / ad valorem |
| grossVolume | Decimal(12,3) | |
| grossValue | Decimal(14,2) | |
| exemptVolume | Decimal(12,3) | Volumes qualifying for exemption |
| taxableVolume | Decimal(12,3) | grossVolume - exemptVolume |
| taxableValue | Decimal(14,2) | |
| taxRate | Decimal(8,6) | Applied rate |
| taxDue | Decimal(14,2) | |
| lineStatus | Enum(draft, validated, included) | |

### 4.4 Amended Report Adjustment

| Attribute | Type | Description |
|-----------|------|-------------|
| amendmentId | UUID | PK |
| originalRunId | UUID | FK → original RegulatoryReportRun |
| amendedRunId | UUID | FK → amended RegulatoryReportRun |
| ventureId | UUID | FK |
| period | YearMonth | The period being corrected |
| originalAmount | Decimal(14,2) | Originally reported royalty/tax |
| correctedAmount | Decimal(14,2) | Corrected amount |
| adjustmentAmount | Decimal(14,2) | correctedAmount - originalAmount |
| withinStatute | Boolean | Whether the correction period is within the statute of limitations |
| writeOffAmount | Decimal(14,2) | Amount written off if beyond statute |
| writeOffGLAccount | String(20) | GL account for out-of-statute write-off |
| amendmentReason | String(500) | |

### 4.5 Indian Recoupment Record

| Attribute | Type | Description |
|-----------|------|-------------|
| recoupmentId | UUID | PK |
| onrrLeaseId | UUID | FK → ONRRLease (Indian type only) |
| period | YearMonth | Period in which recoupment is taken |
| overpaymentPeriod | YearMonth | Period in which original overpayment occurred |
| overpaymentAmount | Decimal(14,2) | Original excess royalty paid |
| recoupmentAmountThisPeriod | Decimal(14,2) | Portion recovered in current period |
| remainingBalance | Decimal(14,2) | Unrecovered balance |
| recoupmentStatus | Enum(pending, partial, completed, waived) | |

---

## 5. Business Processing Rules

### 5.1 ONRR-2014 Processing Workflow (7 Steps)

```
STEP 1 — POPULATE
  Source settled production (M1) and distribution records (M6)
  Cross-reference ventures to ONRR lease hierarchy
  Assign PPNs per precedence rules
  Group records by: leaseName + salesTypeCode + productCode

STEP 2 — VALIDATE
  Check: royaltyVolume > 0 for all TC01 lines
  Check: no duplicate PPNs for same period + lease + sales type
  Check: TC 13 not mapped to processing bucket (ERQADIFD check)
  Check: payor code registered and active
  Check: Section 6 rate exists if section6Applicable = TRUE
  Flag: any Indian leases without recoupment config

STEP 3 — GENERATE TC LINES
  FOR each PPN group:
    Generate TC 01 (always)
    IF approved transport allowance exists → TC 11
    IF approved processing allowance OR ERQADIFD exists → TC 13
    IF no production → TC 15 (suppress TC 01)
    IF section6Applicable:
      Calculate threshold volume (if applicable)
      Generate TC 37 for Section 6 volume at reduced rate
      Generate TC 38 for remaining volume at standard rate
      Suppress TC 01
    IF Indian lease and overpayment balance exists:
      Calculate recoupment amount for period
      Apply as offset to TC 01 royaltyDue

STEP 4 — EXTRACT
  Produce report-level summary: totalRoyaltyDue by payor
  Validate: sum of TC lines balances per PPN
  Produce ASCII flat file (per ONRR layout specification)

STEP 5 — SUMMARIZE
  Aggregate across leases for remittance summary
  Generate payment advice for ACH/wire to ONRR

STEP 6 — JOURNALIZE
  Generate GL entries:
    DR: Royalty Payable (per lease)
    CR: Cash / AP (royalty payment)
  Post RAD to M6 GL posting queue

STEP 7 — FINALIZE
  Lock report run (runStatus = 'approved')
  Mark period as filed in regulatory calendar
  Archive ASCII file and PDF attestation
  PUBLISH event: ua.regulatory.onrr.filed
```

### 5.2 Section 6 Processing

```
FUNCTION applySection6(ppnGroup, period):
  lease = fetch ONRRLease for ppnGroup
  IF NOT lease.section6Applicable → skip
  
  rate = resolveSection6Rate(lease, period)
  // Rate resolution hierarchy (see §3.5)
  
  IF rate IS NULL:
    THROW validation error "Section 6 rate not configured"
  
  // Split volume between Section 6 and standard portions
  // (qualification criteria determine split — configured per lease)
  qualifyingVolume = calculateSection6Volume(lease, ppnGroup)
  standardVolume = ppnGroup.royaltyVolume - qualifyingVolume
  
  // Generate TC 37 — Section 6 reduced rate
  IF qualifyingVolume > 0:
    tc37.royaltyDue = qualifyingVolume × salePrice × rate.section6RoyaltyRate
    PERSIST ONRR2014Record (tcCode = 'TC37')
  
  // Generate TC 38 — standard rate remainder
  IF standardVolume > 0:
    tc38.royaltyDue = standardVolume × salePrice × standardRate
    PERSIST ONRR2014Record (tcCode = 'TC38')
  
  // Suppress TC 01 — it is replaced by TC 37 + TC 38
  MARK ppnGroup.tc01 = 'excluded'
```

### 5.3 Amended Report Processing

```
FUNCTION createAmendedReport(originalRunId, period, correctionReason):
  original = fetch RegulatoryReportRun
  
  // Create amended run linked to original
  amendedRun = NEW RegulatoryReportRun (
    runType = 'amended',
    priorRunId = originalRunId
  )
  
  // Re-run data extraction with corrected source data
  regenerateLines(amendedRun, period)
  
  FOR each line in amendedRun:
    originalLine = fetch matching line from originalRun
    adjustment = line.royaltyDue - originalLine.royaltyDue
    
    // Check statute of limitations
    statuteYears = getStatuteLimit(reportCode, state)
    withinStatute = period > (currentPeriod - statuteYears years)
    
    IF NOT withinStatute:
      // Write off the correction — do not file
      writeOff(adjustment, writeOffGLAccount)
      PERSIST AmendedReportAdjustment (withinStatute = FALSE)
    ELSE:
      PERSIST AmendedReportAdjustment (withinStatute = TRUE)
      // Include in submission
  
  RETURN amendedRun
```

### 5.4 State Severance Tax Report Generation

```
FUNCTION generateStateSeveranceReport(state, period, companyCode):
  template = fetch StatutoryReportTemplate for state + period
  rates = fetch StateTaxRate for state + period
  
  settlements = fetch SettlementStatements (M4) filtered by state + period
  
  FOR each settlement:
    // Apply exemptions
    exemptVolume = evaluateExemptions(settlement, rates.exemptionCodes)
    taxableVolume = settlement.allocatedVolume - exemptVolume
    taxableValue = taxableVolume × settlement.unitPrice
    
    IF rates.rateType = 'tiered':
      taxDue = applyTieredRate(taxableValue, rates.tierDefinition)
    ELSE:
      taxDue = taxableValue × rates.rate
    
    PERSIST StateTaxReportRecord
  
  // Format per template definition
  output = renderTemplate(template, reportRecords)
  RETURN output
```

### 5.5 Out-of-Statute Write-Off Rules

| Scenario | Action |
|----------|--------|
| Amendment within statute of limitations | File corrected amount; adjust royalty payable |
| Amendment beyond statute — additional royalty owed | Write off to `Royalty Expense — Out of Statute` GL account |
| Amendment beyond statute — royalty overpaid | No refund claim; write off credit to `Royalty Overpayment Waived` GL account |
| Indian lease — beyond ONRR statute (7 years) | Consult Tax/Legal before write-off; flag for manual approval |

---

## 6. State Report Coverage

| State | Report Types | Regulatory Body | Filing Frequency |
|-------|-------------|----------------|-----------------|
| Texas | Severance Tax (Oil/Gas/Condensate), Production Report | RRC, Texas Comptroller | Monthly |
| Oklahoma | Gross Production Tax, Conservation Excise Tax | Oklahoma Tax Commission | Monthly |
| New Mexico | Oil and Gas Severance Tax, Conservation Tax | NM Taxation & Revenue, OCD | Monthly |
| Wyoming | Severance Tax, Ad Valorem Tax Declaration | WY Dept. of Revenue, OSC | Monthly / Annual |
| Louisiana | Severance Tax (Oil/Gas) | LA Dept. of Revenue, DNR | Monthly |
| Colorado | Severance Tax, COGCC Production Report | CO Dept. of Revenue, COGCC | Monthly |
| North Dakota | Oil Extraction Tax, Gas Tax | ND Industrial Commission | Monthly |
| Kansas | Severance Tax (separate framework) | KS Dept. of Revenue | Monthly |
| Montana | Resource Indemnity and Gross Proceeds Tax | MT Dept. of Revenue, BOGC | Quarterly |
| Federal (ONRR) | ONRR-2014 Royalty Report | ONRR | Monthly |

---

## 7. API Endpoints

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| GET | `/api/v1/regulatory/templates` | List report templates | `regulatory:read` |
| GET | `/api/v1/regulatory/templates/{id}` | Template detail with field layout | `regulatory:read` |
| POST | `/api/v1/regulatory/templates` | Create new template version | `regulatory:admin` |
| PUT | `/api/v1/regulatory/templates/{id}` | Update template (new version) | `regulatory:admin` |
| POST | `/api/v1/regulatory/reports/run` | Trigger report generation run | `regulatory:execute` |
| GET | `/api/v1/regulatory/reports/runs/{id}` | Run status and summary | `regulatory:read` |
| GET | `/api/v1/regulatory/reports/runs/{id}/lines` | All report lines for a run | `regulatory:read` |
| PUT | `/api/v1/regulatory/reports/runs/{id}/approve` | Approve run for submission | `regulatory:approve` |
| POST | `/api/v1/regulatory/reports/runs/{id}/submit` | Submit to regulatory portal | `regulatory:execute` |
| POST | `/api/v1/regulatory/reports/runs/{id}/amend` | Create amended run | `regulatory:execute` |
| GET | `/api/v1/regulatory/onrr/leases` | List ONRR lease configurations | `regulatory:read` |
| POST | `/api/v1/regulatory/onrr/leases` | Create ONRR lease | `regulatory:admin` |
| GET | `/api/v1/regulatory/onrr/ppns` | List PPNs | `regulatory:read` |
| POST | `/api/v1/regulatory/onrr/ppns` | Create/assign PPN | `regulatory:admin` |
| GET | `/api/v1/regulatory/onrr/section6-rates` | List Section 6 rates | `regulatory:read` |
| POST | `/api/v1/regulatory/onrr/section6-rates` | Create Section 6 rate | `regulatory:admin` |
| GET | `/api/v1/regulatory/amendments` | List amended reports | `regulatory:read` |
| GET | `/api/v1/regulatory/amendments/{id}` | Amendment adjustment detail | `regulatory:read` |
| GET | `/api/v1/regulatory/indian-recoupment` | Indian recoupment balances | `regulatory:read` |

---

## 8. Reports

| Report | Description | Trigger |
|--------|-------------|---------|
| ONRR-2014 Detail Report | All TC lines by PPN, with royalty due, allowances, and Section 6 breakdown | Report run completion |
| ONRR Royalty Summary | Totals by payor code and lease type; matches payment advice | Monthly close |
| Section 6 Application Report | Leases where Section 6 reduced rates were applied, volumes, rate differential | Monthly close |
| Indian Recoupment Schedule | Open balances, period recoupments, remaining amounts by lease | Monthly / on demand |
| State Severance Tax Summary | Tax due by state, product, and tax type | Monthly close |
| Amended Report Comparison | Side-by-side original vs. corrected amounts; adjustment detail | On demand |
| Out-of-Statute Write-Off Report | Write-off amounts, periods, GL accounts; requires Tax/Legal sign-off | On demand |
| Regulatory Calendar | Filed vs. due dates; upcoming filing deadlines with status | Weekly / on demand |
| TC Line Validation Report | Lines with validation warnings or errors before submission | Report run completion |
| State Production Report | Volumes by state, product, lease, and well; formatted per state agency spec | Monthly close |
| PPN Activity Report | PPN assignments, changes, and new auto-generated PPNs | Monthly close |
| Royalty Reconciliation | Compares ONRR filing totals to M6 distribution royalty deductions | Monthly close |
