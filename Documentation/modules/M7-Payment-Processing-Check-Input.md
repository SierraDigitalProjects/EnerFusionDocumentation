# M7 — Payment Processing & Check Input

---

## 1. Module Purpose

Handles both sides of the cash flow in upstream accounting:

- **Inbound:** Processes purchaser check receipts (CDEX/EDI 820 or manual) and applies them to settlement statements — *Check Input*
- **Outbound:** Generates owner payment disbursements (ACH, check, wire) from M6 revenue distributions — *Payment Processing*

Also manages accounts receivable aging, manual write-offs, and escheat/escrow processing for unclaimed owner funds.

---

## 2. Core Concepts

| Concept | Definition |
|---------|------------|
| **Remitter** | The purchaser who submits a check to the operator for oil/gas purchased |
| **PDX (Producer Distribution Cross-Reference)** | Remitter/property cross-reference that maps a purchaser check to the correct ventures/DOIs for distribution |
| **CDEX** | Chemical Data Exchange — the industry-standard EDI format (820 Payment Order) used by purchasers to submit remittance detail |
| **Check Input** | The process of receiving purchaser payments and distributing the proceeds to owner interests |
| **Payment Run** | A batch job that generates outgoing owner payment disbursements from M6 distribution records |
| **Escheat** | The legal process of remitting unclaimed owner funds to the owner's state of domicile after the dormancy period expires |
| **Hold Code** | A code applied to a payment that prevents it from being issued until the hold is cleared |
| **JIB Netting** | Offsetting a non-operating partner's outstanding Joint Interest Billing receivable against their revenue distribution before issuing a check |
| **Cash Company** | A company code or group of company codes sharing a single treasury account for consolidated disbursement |

---

## 3. Master Data

### 3.1 Remitter / Property Cross-Reference (PDX)

The most critical master data record in M7. Maps a purchaser (remitter) to one or more producing ventures/DOIs for check distribution.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| pdxId | UUID | Yes | PK |
| remitterId | UUID | Yes | FK → BusinessPartner (purchaser) |
| propertyIdentifier | String(20) | Yes | Venture/DOI or regulatory property number |
| productTypeCode | Enum | Yes | Oil / Gas / NGL component |
| decimalInterest | Decimal(10,8) | Yes | Distribution decimal for this remitter/property |
| cdexCompanyCode | String(10) | No | EDI identifier (up to 10 chars) |
| calculateSeveranceTaxFlag | Boolean | No | Whether to calculate severance independently in CI |
| effectiveFrom | Date | Yes | |
| effectiveTo | Date | No | Open-ended until superseded |

**Historical versioning:** Every change is logged with the effective date and change user. Historical cross-reference records are retained for period-level reporting.

### 3.2 Owner Attribute Group

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| ownerAttributeGroupId | UUID | Yes | PK |
| groupName | String(100) | Yes | |
| paymentMethod | Enum(check, ach, wire) | Yes | Default disbursement method |
| paymentThreshold | Decimal(10,2) | No | Minimum payment; accumulate if below |
| annualMinimum | Decimal(10,2) | No | Annual minimum — issue even if below monthly threshold |
| stateTaxWithholdingPct | Decimal(5,4) | No | |
| federalWithholdingPct | Decimal(5,4) | No | Backup withholding for owners without W-9 |
| nraWithholdingPct | Decimal(5,4) | No | Non-resident alien withholding |
| nraWithholdingExempt | Boolean | No | Treaty-exempt flag |
| payFrequency | Enum(monthly, quarterly, annual) | No | |
| paymentBlock | Boolean | No | Hard block on all payments |

### 3.3 Check Layout Template

| Attribute | Type | Description |
|-----------|------|-------------|
| checkLayoutId | UUID | PK |
| layoutName | String(100) | |
| companyCode | String(10) | |
| logoPath | String | Azure Blob Storage path for company logo |
| templateFormat | JSONB | Field positions, font sizes, stub layout |
| stubbLines | Integer | Number of detail lines per stub |
| overflowBehavior | Enum(second_page, summary_only) | |

### 3.4 Escheat Configuration

| Attribute | Type | Description |
|-----------|------|-------------|
| escheatConfigId | UUID | PK |
| escheatState | String(2) | State to which unclaimed property is remitted |
| dormancyPeriodYears | Integer | Years before funds become reportable |
| reportingThreshold | Decimal(10,2) | Minimum balance subject to escheat |
| filingDeadlineMonth | Integer | Month of year when filing is due |
| filingFormat | String | State-specific report format |

### 3.5 Cash Company Configuration

| Attribute | Type | Description |
|-----------|------|-------------|
| cashCompanyId | UUID | PK |
| primaryCompanyCode | String(10) | The company code that holds the bank account |
| associatedCompanyCodes | String[] | Other companies in the group |
| bankAccountId | String | Bank reference for ACH/wire |

---

## 4. Transactional Entities

### 4.1 Incoming Check Record

| Attribute | Type | Description |
|-----------|------|-------------|
| checkRecordId | UUID | PK |
| remitterId | UUID | FK → BusinessPartner |
| checkNumber | String(20) | Purchaser's check number |
| checkDate | Date | |
| checkAmount | Decimal(14,2) | Total amount remitted |
| receiptDate | Date | Date operator received the check |
| cdexFileId | UUID | FK → CdexImportFile (if EDI) |
| processingStatus | Enum(unprocessed, matched, distributed, posted, reversed) | |
| pdxId | UUID | FK → PDX cross-reference |
| salesDateFrom / To | Date | Sales date range covered by this check |

### 4.2 Check Distribution Line

| Attribute | Type | Description |
|-----------|------|-------------|
| distributionLineId | UUID | PK |
| checkRecordId | UUID | FK |
| ventureId | UUID | FK |
| businessPartnerId | UUID | FK → Owner |
| productType | Enum | |
| salesMonth | YearMonth | |
| ownerVolume | Decimal(12,3) | |
| ownerGrossValue | Decimal(14,2) | |
| severanceTaxDeduct | Decimal(12,2) | |
| marketingDeduct | Decimal(12,2) | |
| ownerNetValue | Decimal(14,2) | |

### 4.3 Owner Payment Record (Outgoing)

| Attribute | Type | Description |
|-----------|------|-------------|
| paymentId | UUID | PK |
| paymentRunId | UUID | FK → PaymentRun |
| businessPartnerId | UUID | FK → Owner |
| paymentMethod | Enum(check, ach, wire) | |
| checkNumber | String(20) | Assigned check number (if check) |
| paymentDate | Date | |
| grossAmount | Decimal(14,2) | Before withholding |
| withholdingAmount | Decimal(12,2) | State + federal + NRA |
| netAmount | Decimal(14,2) | Actual disbursement |
| holdCodes | String[] | Active hold codes |
| paymentStatus | Enum(pending, approved, issued, voided, stale) | |
| escheatFlag | Boolean | Flagged for escheat processing |
| escheatDate | Date | Date funds transferred to state |

### 4.4 Payment Run

| Attribute | Type | Description |
|-----------|------|-------------|
| paymentRunId | UUID | PK |
| runDate | Date | |
| companyCode | String(10) | |
| cashCompanyId | UUID | FK → CashCompany |
| runScope | Enum(all_owners, single_owner, delivery_network, venture) | |
| runStatus | Enum(validated, extracted, analyzed, journalized, finalized) | |
| totalChecks | Integer | |
| totalAmount | Decimal(16,2) | |
| outboundFileId | UUID | FK → generated bank file |

### 4.5 AR Record

| Attribute | Type | Description |
|-----------|------|-------------|
| arId | UUID | PK |
| arKey | String(100) | Composite key: remitter + venture + product + salesMonth |
| remitterId | UUID | FK |
| ventureId | UUID | FK |
| productType | Enum | |
| salesMonth | YearMonth | |
| openingBalance | Decimal(14,2) | |
| currentBalance | Decimal(14,2) | |
| arStatus | Enum(open, reconciled, deferred, categorized, write_off_pending, written_off) | |
| categoryCode | String(10) | User-defined category |
| agingDays | Integer | Days since original posting |
| responsibilityId | UUID | FK → User (accountant assigned) |

### 4.6 AR Write-Off Record

| Attribute | Type | Description |
|-----------|------|-------------|
| writeOffId | UUID | PK |
| arId | UUID | FK → ARRecord |
| writeOffAmount | Decimal(14,2) | |
| reasonCode | String(10) | |
| submittedBy | UUID | FK → User |
| approvedBy | UUID | FK → User |
| approvalStatus | Enum(pending, approved, rejected, deleted) | |
| comment | Text | |

### 4.7 Escheat Record

| Attribute | Type | Description |
|-----------|------|-------------|
| escheatId | UUID | PK |
| businessPartnerId | UUID | FK → Owner |
| accumulatedAmount | Decimal(14,2) | Total unclaimed balance |
| dormancySince | Date | Date funds first became unclaimed |
| escheatState | String(2) | State to remit to |
| filingStatus | Enum(accumulating, reportable, filed, remitted) | |
| filingYear | Integer | Year of state filing |
| stateVoucherNumber | String(50) | State's acknowledgment reference |

---

## 5. Business Processing Rules

### 5.1 Check Input — CDEX/EDI 820 Processing

```
FUNCTION processCdexFile(fileContent):
  1. Parse EDI 820 segments: BPR (payment), TRN (trace), ENT (entity), RMR (remittance)
  2. Match BPR.remitterCode to BusinessPartner via CDEX company code
  3. FOR each RMR detail line:
     a. Lookup PDX cross-reference: remitterId + propertyNumber + productCode
     b. IF match found: auto-route to distribution
     c. IF no match: route to manual exception queue
  4. Validate: SUM(distributionLines) = checkAmount (within $0.01 tolerance)
  5. PERSIST CheckRecord + CheckDistributionLines
  6. UPDATE AR balance for remitter
```

### 5.2 Payment Run — Six-Step Lifecycle

```
STEP 1: VALIDATE
  - Every owner has valid payment method
  - EFT owners have banking information
  - All cost centers open for posting
  - No duplicate check generation

STEP 2: EXTRACT
  - Read M6 OwnerDistributionRecords for run scope
  - Apply minimum payment threshold check
  - Apply JIB netting if configured

STEP 3: VARIANCE ANALYSIS (user review)
  - Present variance report by owner
  - Allow: suspend line, override variance, assign hold code
  - Require explicit user approval before proceeding

STEP 4: JOURNALIZE
  - Create payment journal entries
  - Preview via Proposed Journalization Report
  - Inter-process lock prevents duplicate GL entries

STEP 5: FINALIZE
  - Post GL documents
  - Generate check register
  - Generate outbound ACH/positive pay bank file
  - Update owner payment status

STEP 6: OUTPUT
  - Print or transmit checks
  - Export remittance advice for owner stubs
  - Archive finalized run
```

### 5.3 Minimum Payment Threshold

```
FUNCTION evaluatePaymentThreshold(ownerId, periodNetAmount):
  group = fetch OwnerAttributeGroup for owner
  suspendedBalance = fetch prior suspended balance for owner
  totalDue = suspendedBalance + periodNetAmount
  
  IF totalDue >= group.paymentThreshold:
    RELEASE totalDue for payment
    CLEAR suspendedBalance
  ELSE IF isDecemberRun AND totalDue >= group.annualMinimum:
    RELEASE totalDue (annual sweep)
  ELSE:
    ACCUMULATE periodNetAmount to suspendedBalance
    SKIP this owner in payment run
```

### 5.4 Withholding Calculation

```
FUNCTION calculateWithholding(ownerId, grossPayment, state):
  group = fetch OwnerAttributeGroup
  
  stateWithholding  = grossPayment × group.stateTaxWithholdingPct
  
  IF owner.w9OnFile = FALSE:
    federalWithholding = grossPayment × 0.28  // IRS backup withholding
  ELSE:
    federalWithholding = grossPayment × group.federalWithholdingPct
  
  IF owner.isNonResidentAlien AND NOT group.nraWithholdingExempt:
    nraWithholding = grossPayment × group.nraWithholdingPct
  
  totalWithholding = stateWithholding + federalWithholding + nraWithholding
  netPayment = grossPayment - totalWithholding
  RETURN { netPayment, totalWithholding }
```

### 5.5 JIB Netting

```
FUNCTION applyJibNetting(ownerId, distributionAmount):
  jibBalance = fetch outstanding JIB receivable for owner
  IF jibBalance > 0:
    netAmount = distributionAmount - jibBalance
    IF netAmount >= 0:
      POST JIB netting journal entry for jibBalance
      CLEAR JIB receivable
      RETURN netAmount
    ELSE:
      // Distribution insufficient to cover JIB
      POST partial JIB journal entry for distributionAmount
      REDUCE JIB balance by distributionAmount
      RETURN 0  // No cash issued; carry remaining JIB
```

### 5.6 Escheat Processing

```
FUNCTION evaluateEscheat(businessPartnerId):
  payments = fetch unclaimed payments for owner
  FOR each payment:
    dormancyDays = TODAY - payment.issueDate
    config = fetch EscheatConfig for owner.escheatState
    
    IF dormancyDays >= config.dormancyPeriodYears × 365:
      IF SUM(unclaimedBalance) >= config.reportingThreshold:
        CREATE EscheatRecord
        FLAG for state filing in config.filingDeadlineMonth
```

---

## 6. API Endpoints

### Check Input

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| POST | `/api/v1/payments/check-input/cdex` | Import CDEX/EDI 820 file | `payments:write` |
| POST | `/api/v1/payments/check-input/manual` | Enter manual check | `payments:write` |
| POST | `/api/v1/payments/check-input/mass-upload` | Bulk check upload | `payments:write` |
| GET | `/api/v1/payments/check-input` | Query check records | `payments:read` |
| GET | `/api/v1/payments/check-input/{id}` | Check detail | `payments:read` |
| POST | `/api/v1/payments/check-input/{id}/reverse` | Reverse a check | `payments:approve` |
| GET | `/api/v1/payments/pdx` | List PDX cross-references | `payments:read` |
| POST | `/api/v1/payments/pdx` | Create PDX record | `payments:write` |
| PUT | `/api/v1/payments/pdx/{id}` | Update PDX record | `payments:write` |
| POST | `/api/v1/payments/pdx/mass-upload` | Bulk PDX upload | `payments:write` |

### Payment Processing

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| POST | `/api/v1/payments/runs` | Initiate payment run | `payments:execute` |
| GET | `/api/v1/payments/runs/{id}` | Run status | `payments:read` |
| POST | `/api/v1/payments/runs/{id}/approve-variance` | Approve variance and proceed | `payments:approve` |
| POST | `/api/v1/payments/runs/{id}/finalize` | Finalize and generate bank file | `payments:approve` |
| GET | `/api/v1/payments/runs/{id}/register` | Check register | `payments:read` |
| GET | `/api/v1/payments/runs/{id}/bank-file` | Download outbound bank file | `payments:admin` |
| GET | `/api/v1/payments/owner-payments` | Query owner payments | `payments:read` |
| POST | `/api/v1/payments/owner-payments/{id}/void` | Void a payment | `payments:approve` |

### AR Workplace

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| GET | `/api/v1/payments/ar` | Query AR balances | `payments:read` |
| PUT | `/api/v1/payments/ar/{id}/reconcile` | Mark reconciled | `payments:write` |
| PUT | `/api/v1/payments/ar/{id}/defer` | Mark deferred | `payments:write` |
| PUT | `/api/v1/payments/ar/{id}/categorize` | Assign category code | `payments:write` |
| POST | `/api/v1/payments/ar/{id}/write-off` | Submit write-off | `payments:write` |
| PUT | `/api/v1/payments/ar/write-offs/approve` | Bulk approve write-offs | `payments:approve` |
| POST | `/api/v1/payments/ar/{id}/transfer` | Transfer balance to another AR key | `payments:write` |
| POST | `/api/v1/payments/ar/auto-write-off` | Trigger scheduled auto write-off | `payments:execute` |

### Escheat

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| GET | `/api/v1/payments/escheat` | Query escheat records | `payments:read` |
| POST | `/api/v1/payments/escheat/evaluate` | Run dormancy evaluation | `payments:execute` |
| GET | `/api/v1/payments/escheat/report/{state}` | Generate state escheat report | `payments:read` |
| PUT | `/api/v1/payments/escheat/{id}/file` | Mark as filed with state | `payments:approve` |

---

## 7. Reports

| Report | Description | Trigger |
|--------|-------------|---------|
| Check Register | All payments issued in a payment run: date, number, owner, amount | Payment run finalization |
| Owner Payment Stubs | Detailed remittance advice per owner (venture/product/period breakdown) | Per payment run |
| AR Aging Report | Outstanding purchaser receivables by age bucket (0-30, 31-60, 61-90, 90+ days) | Monthly / on demand |
| AR Control Report | Period summary: opening balance, new checks, write-offs, transfers, closing balance | Period close |
| Proposed Journalization Report | Preview of owner and account-level distributions before final posting | Pre-journalization |
| Blocked Cost Center Report | Cost centers that would block journalization | Pre-journalization |
| Payment Run Variance Report | Variance exceptions with responsibility identification | Variance analysis step |
| DOI Decimal Comparison Report | PDX decimal vs. DOI decimal discrepancy flags | On demand / pre-journalization |
| Escheat Report | Unclaimed owner balances by state, dormancy, and amount | Annual / state filing deadline |
| 1099 Report | Year-end 1099-MISC/NEC data by owner | Year-end |
| Withholding Summary | Total state, federal, and NRA withholding by owner and period | Monthly / year-end |
| Check Input Write-Off Report | Write-offs applied to AR balances from check input discrepancies | Period close |
