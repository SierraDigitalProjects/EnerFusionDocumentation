# Ad Valorem Tax — Year-End True-Up Process

> Source: PRA-KSHOP/Roadmap Specs/Ad Valorem Specs Workstream final v1.pdf  
> Module: M4 — Contracts, Pricing & Valuation

---

## Overview

Ad valorem tax is an **annual fee assessed by state governments** on the value of oil and gas production (essentially a property tax on producing minerals). Unlike severance tax (which is calculated and remitted monthly), ad valorem is typically invoiced by the county or state once per year.

**States currently using ad valorem true-up:** Utah, Colorado, Wyoming (and potentially others configured by the compliance team).

---

## The Two-Step Process

### Step 1 — Monthly Accrual (Automatic)

Each month, the valuation run (M4) automatically accrues an estimated ad valorem tax amount using the configured tax rates and processing allowances. This accrual appears on settlement statements as a deduction and flows to owner distributions.

```
monthly_accrual = grossValue × ad_valorem_rate × (1 - processingAllowance)
```

### Step 2 — Year-End True-Up (Manual Process)

At year-end, when the state/county issues the actual ad valorem invoice, the operator compares:

```
true_up_amount = actual_invoice_amount - sum(monthly_accruals_for_year)
```

The true-up may be positive (under-accrued — additional payment to owners) or negative (over-accrued — recoupment from owners).

---

## True-Up Requirements (from AD Valorem Spec)

| Req # | Requirement | Notes |
|--------|-------------|-------|
| ADV2.1 | Allow users to enter either **net or gross** true-up amount for disbursement or recoupment | Positive = distribute to owners; Negative = recoup from owners |
| ADV2.2 | Upload true-up amounts by **Venture/DOI or unit level**, by sales date (month/year), product, and marketing group | Bulk upload template; one row per venture/product/period |
| ADV2.3 | True-up processing must follow **same logic as VL/RD accrual** — including rounding, account determination, affiliates, and chain of title | Ensures consistency between accrual and true-up accounting |
| ADV2.4 | True-up must **not distribute** to tax-exempt, take-in-kind, or do-not-pay owners | Same exemption rules as monthly accrual |
| ADV2.5 | Generate **trial combined run** of accounts and totals before posting; ability to save trial runs | Allows accountant review before finalisation |
| ADV2.6 | **Document Level Workplace** — multiple users can review and process multiple true-up files simultaneously | Parallel processing across ventures |
| ADV2.7 | True-up should **not generate new PPNs** or force existing PPNs to process | True-up is a standalone adjustment; does not trigger prior-period re-valuation |
| ADV2.8 | Use contract number/marketing rep and sales date to determine **marketing arrangement** for disbursement to all eligible owners | Same contract routing as standard valuation |
| ADV2.9 | True-up is a **value-only entry** (no volume) | No volume impact; value only on settlement |
| ADV2.10 | Correct **cost center accounting** — including cost center on the created record | Required for GL posting accuracy |

---

## True-Up Processing Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                   AD VALOREM TRUE-UP PROCESSING                          │
│                                                                          │
│  [1] Receive State Invoice                                               │
│      • State/county issues annual ad valorem invoice                    │
│      • Compliance team compares to YTD accrual                          │
│                     │                                                    │
│  [2] Prepare True-Up Upload File                                        │
│      • By venture, product, sales year, marketing group                 │
│      • Gross or net amount                                               │
│      • Positive (under-accrued) or negative (over-accrued)             │
│                     │                                                    │
│  [3] Upload to Document Level Workplace                                 │
│      • System validates: venture active, contract exists,              │
│        owners flagged as exempt are excluded                             │
│                     │                                                    │
│  [4] Generate Trial Run                                                 │
│      • Preview: accounts, cost centers, amounts, owner distribution    │
│      • Save trial run for review                                         │
│                     │                                                    │
│  [5] Approve and Post                                                   │
│      • Follows same VL/RD accrual logic                                │
│      • No new PPNs generated                                             │
│      • RAD posted to GL                                                 │
│                     │                                                    │
│  [6] Payment Processing (M7)                                            │
│      • Positive true-up: additional owner distributions issued         │
│      • Negative true-up: recoupment against next payment run           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Upload File Format

| Column | Data Type | Description |
|--------|-----------|-------------|
| VentureId | String | Venture/DOI identifier |
| ProductType | Enum | Oil / Gas / NGL |
| SalesYear | YYYY | Year being trued up |
| SalesMonth | MM (optional) | If month-level granularity needed |
| MarketingGroup | String | Marketing group code |
| TrueUpAmount | Decimal | Positive = additional; Negative = recoup |
| AmountType | Enum (gross / net) | Whether amount is before or after deductions |
| CostCenter | String | GL cost center for posting |
| ContractNumber | String | Sales contract for marketing arrangement determination |

---

## Owner Exclusions

The following owner types are excluded from ad valorem true-up distributions (same as monthly accrual):

| Owner Flag | Description | Treatment |
|-----------|-------------|-----------|
| Tax-exempt | Owner holds a state or federal tax exemption | No deduction applied; no true-up distribution |
| Take-in-kind (TIK) | Owner takes production as physical product rather than cash | Excluded from cash distribution |
| Do-not-pay | Owner payment is blocked (legal hold, escheat process, etc.) | Accrual created but payment suspended |

---

## Account Determination

True-up account determination follows the same rules as the standard accrual:

1. Look up Valuation Cross-Reference (VCR) for the venture/product
2. Retrieve Settlement Statement Profile
3. Apply GL account determination logic (same account tree as VL/RD accrual)
4. Assign cost center from upload file (overrides any default cost center)
