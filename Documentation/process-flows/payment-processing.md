# Payment Processing — Process Flow

> Source: PRA-KSHOP/Roadmap Specs/Spec_Payment Processing_Final v1_2 doc.pdf  
> Module: M7 — Payment Processing & Check Input

---

## End-to-End Payment Processing Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    PAYMENT PROCESSING LIFECYCLE                          │
│                                                                          │
│  [1] Owner Payable Data         [2] Validate                            │
│      (from M6 distribution)    ─────────────────────────────────────    │
│                                  • All owners have valid payment method  │
│                                  • EFT owners have banking info          │
│                                  • Vendor addresses established          │
│                                  • Cost centers valid for posting        │
│                                  • Footing at owner level (no cross-    │
│                                    decimal footing)                      │
│                                                                          │
│  [3] Extract                    [4] Analyze / Variance Review            │
│      Raw Transactions          ─────────────────────────────────────    │
│      from OPSL/distribution     • User-defined variance tolerances       │
│                                  • Drill-down to lowest owner detail     │
│                                  • Suspend individual lines              │
│                                  • Override variance                     │
│                                  • Hold check codes (No Print,          │
│                                    user-defined reason codes)            │
│                                                                          │
│  [5] Journalize                 [6] Finalize                             │
│      Journals                  ─────────────────────────────────────    │
│                                  • POST documents                        │
│                                  • CREATE check register                 │
│                                  • GENERATE outbound bank file           │
│                                    (positive pay / ACH)                  │
│                                  • FINALIZE output                       │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Detailed Step Descriptions

### Step 1 — Owner Payable Data

Input data flows from M6 Revenue Distribution. Each owner's gross value, tax deductions, marketing costs, and net payment amount are available at the lowest accounting level.

**Data elements carried through at the lowest level:**
- Marketing cost detail by type
- Tax adjustment detail by type (severance, ad valorem, withholding)
- Settlement type (Actual or Entitlement-based)
- BTU factor (for gas)
- DOI decimal and disbursement decimal

---

### Step 2 — Validation

Before any processing occurs, the system validates all of the following. Any failure blocks the payment run with a specific error message.

| Validation Check | Business Rule |
|-----------------|---------------|
| Payment method on Vendor master | Every owner must have a valid payment method (check, ACH, wire) configured |
| EFT banking information | Owners configured for electronic funds transfer must have bank account and routing number |
| Vendor address | Mailing address must be present (required for check printing) |
| Cost center validity | All OPSL booking cost centers must be open for posting in the current period |
| No duplicate checks | System must detect and reject duplicate check generation attempts |
| Withholding setup | State, federal, and non-resident alien withholding rates must be configured for applicable owners |

---

### Step 3 — Extract

The extract step reads from distribution records and master data to build raw payment transactions. Processing can be scoped by:

- Company code
- Owner (individual or group)
- Delivery Network
- Unit Venture
- Property

**Cash company grouping:** Multiple companies with shared treasury can be configured as a cash company group, allowing payment runs to be processed as a single outbound bank file across all associated entities.

---

### Step 4 — Variance Analysis (Pre-Finalization Hold)

This step is critical for operator control. No outbound bank file is generated until the operator explicitly releases payments after variance review.

**User controls available:**
- Suspend a specific payment line (e.g., disputed owner balance)
- Override variance tolerance for specific owners
- Assign hold codes (No Print, user-defined reason codes, multiple codes per owner)
- Drill down to the specific venture/DOI/product line comprising each check stub line
- View check detail for debit balances where applicable

**Variance criteria:** Configurable per company, with responsibility identification in the variance report to route exceptions to the correct team.

---

### Step 5 — Journalize

Creates journal entries for payment posting. Inter-process control between payment processing and all other journal entry posting processes prevents double-posting.

The proposed journalization preview report (from Check Input) shows:
- **Owner-level content:** Who will be paid, volume, value, tax, deductions
- **Account-level content:** GL accounts, property/venture, cost center, amount, UOM

---

### Step 6 — Finalize

Final step generates all output and posts documents:

1. Post payment documents to GL
2. Generate check register (date, check #, owner #, owner name, amount)
3. Create outbound bank file for ACH or positive pay
4. Update owner ledger with finalized payment amounts

**Checks are held** after variance analysis and prior to outbound file generation, allowing a final review before funds leave the operator's account.

---

## Remittance Advice Data Elements

### Check Header

| Field | Description |
|-------|-------------|
| Check date | Payment date |
| Check # | Sequential check number |
| Owner # | Business Associate / Vendor ID |
| Owner Name and Address | Payee information |
| Company Name and Address | Issuer information |

### Detail Lines (per Venture/DOI)

| Field | Description |
|-------|-------------|
| Venture # and Name | Producing venture identifier |
| DOI Name | Division of interest description |
| State and County | Geographic location |
| Interest Type | WI / RI / ORRI |
| Sales Month | Accounting period |
| Product Code | Oil / Gas / NGL / component |
| DOI Decimal | Owner's percentage interest |
| Disbursement Decimal | Effective payment percentage |
| BTU Factor | Heating value correction (gas) |
| Gross Volume — Owner and Property Level | Allocated volumes |
| Gross Value — Owner and Property Level | Pre-deduction value |
| Marketing Costs — Gross and Owner Level | Transportation, processing fees |
| Taxes by Type — Gross and Owner Level | Severance, ad valorem, withholding |
| Net Value — Owner and Property Level | Final payment amount |
| Price (calculated) | Effective price per unit |
| Settlement Type | Actual or Entitlement |
| UOM | Unit of measure for every volume field |

### Subtotals and Totals

- Subtotal per Venture/DOI
- Subtotal per Venture
- Owner Check Total: gross, deductions, net

---

## Withholding

### Types of Withholding

| Type | Applicability | Configuration |
|------|--------------|---------------|
| State income tax | State-specific; owner domicile | State tax rate table + owner attribute group |
| Federal backup withholding | Owners without valid W-9 on file | IRS-mandated rate (28%) |
| Non-resident alien (NRA) withholding | Foreign-domiciled owners | Treaty-based or statutory rate |

**Exemptions:** Individual owners can be flagged as exempt from specific withholding types on the Vendor master. Exemption reason codes are maintained and reportable.

---

## Minimum Payment Threshold

Payments below the configured minimum are suspended (not issued) and accumulated:

```
FUNCTION shouldIssuePayment(ownerId, periodAmount):
  thresholds = fetch from OwnerAttributeGroup for owner
  cumulativeBalance = fetch suspended balance for owner
  totalDue = cumulativeBalance + periodAmount
  
  IF totalDue >= thresholds.minimumPayAmount:
    ISSUE payment for totalDue
    CLEAR suspended balance
  ELSE:
    ADD periodAmount to suspended balance
    DO NOT issue check this period
```

**Threshold hierarchy:** Company default → State override → Owner-specific override at payment run level.

---

## Wyoming Escrow Process

Wyoming requires operators to escrow unclaimed owner payments for a defined dormancy period before escheat filing. The payment processing module creates escrow-flagged checks that are tracked separately from standard disbursements until the dormancy period expires.

---

## 1099 Processing

At year-end, the system generates IRS Form 1099-MISC / 1099-NEC for owner payments exceeding the reporting threshold:

| 1099 Type | Usage | Threshold |
|-----------|-------|-----------|
| 1099-MISC | Royalty payments | $10+ |
| 1099-NEC | Non-employee compensation | $600+ |

Stripper well credits and special production payment structures require manual override of standard 1099 logic. The `PP_Manual 1099 & Stripper Well` configuration allows compliance team to correct 1099 classifications before year-end filing.

---

## JIB Netting (Joint Interest Billing)

For joint venture operators, JIB (Joint Interest Billing) amounts owed by non-operating partners can be netted against revenue distributions before issuing checks. The JIB Netting Finalize workflow:

1. Calculate gross revenue distribution for NOP (non-operating partner)
2. Retrieve outstanding JIB receivable balance for same NOP
3. Net: paymentAmount = distributionAmount − jibBalance
4. If netAmount < 0: hold payment; carry forward to next period
5. Post JIB netting journal entry
