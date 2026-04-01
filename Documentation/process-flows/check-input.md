# Check Input — Process Flow

> Source: PRA-KSHOP/Roadmap Specs/Specification_Check Input_1.1.pdf  
> Module: M7 — Payment Processing & Check Input

---

## Overview

Check Input is the process by which the operator receives and applies purchaser payment checks (incoming revenue) to settlement statements. It is distinct from outgoing owner payment processing — Check Input handles money *received from* purchasers (buyers), while Payment Processing handles money *sent to* owners (royalty/working interest holders).

---

## Check Input Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         CHECK INPUT LIFECYCLE                            │
│                                                                          │
│  [1] Purchaser Submits Check                                             │
│      • Physical check with remittance detail                             │
│      • CDEX/EDI 820 electronic remittance                                │
│      • Manual entry for non-EDI purchasers                               │
│                     │                                                    │
│                     ▼                                                    │
│  [2] Remitter/Property Cross-Reference Lookup                            │
│      • Match purchaser (remitter) to venture/property                   │
│      • Apply property routing rules                                      │
│      • Support historical cross-reference with effective dating          │
│                     │                                                    │
│                     ▼                                                    │
│  [3] Validate & Calculate                                                │
│      • Check for blocked cost centers (pre-posting report)              │
│      • Calculate severance taxes per DOI accounting rules               │
│      • Apply special disbursement rules (PDX logic)                     │
│                     │                                                    │
│                     ▼                                                    │
│  [4] Preview — Proposed Journalization Report                            │
│      • Owner-level: who receives funds, volume, value, tax, deducts     │
│      • Account-level: GL accounts, cost centers, amounts                │
│                     │                                                    │
│                     ▼                                                    │
│  [5] Post (Journalize)                                                   │
│      • Post to owner ledger                                              │
│      • Post to GL accounts                                               │
│      • Update AR balance for remitter                                    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Master Data: Remitter/Property Cross-Reference

The **Remitter/Property Cross-Reference** (PDX) is the core configuration that maps a purchaser (remitter) to one or more producing properties (ventures/DOIs). This is the most important master data record in the Check Input module.

| Attribute | Description |
|-----------|-------------|
| Remitter ID | Purchaser's business partner ID |
| Property Number | Venture/DOI or regulatory property identifier |
| Product Code | Oil / Gas / NGL component |
| DOI Decimal | Owner's decimal interest for distribution |
| Effective From / To | Date range for this cross-reference |
| CDEX Company Code | EDI identifier (expanded to allow > 2 characters) |

**Historical tracking:** Every change to the cross-reference record is logged with the effective date of the change, allowing operators to see what configuration was in place for any historical period.

---

## Mass Upload

For operators with hundreds or thousands of remitter/property cross-reference records, mass upload via a structured template is required:

| Feature | Description |
|---------|-------------|
| Template format | Excel-based upload template |
| Scope | Create and update cross-reference records in bulk |
| Validation | Pre-upload validation report flags missing/invalid values |
| Processing | Async background job with progress tracking |
| Change log | All uploads recorded with upload user and timestamp |

---

## Copy / Reverse Feature

The Copy/Reverse feature is used for corrections and adjustments to previously processed checks:

| Enhancement | Description |
|-------------|-------------|
| Multi-select | Select multiple lines from a processed check for reversal or copy |
| Date range search | Search by sales date range to find related check records |
| Batch reverse | Reverse selected lines in a single operation |
| Batch copy | Copy selected lines to create an adjustment entry |

**Limitation removed:** Previous version restricted selection to one remitter/property/product combination at a time. Enhanced version supports multiple simultaneous selections.

---

## Severance Tax Calculation in Check Input

For properties where the operator uses DOI accounting rules (purchaser pays royalty), the system calculates severance tax independently within Check Input:

```
IF checkInputMasterData.calculateSeveranceTaxFlag = TRUE:
  severanceTax = checkGrossValue × stateTaxRate × (1 - processingAllowance)
  netAmount = checkGrossValue - severanceTax - otherDeductions
  
  // Tax is deducted from owner distributions, not from purchaser's check
  // Ensures operator correctly accounts for tax liability on check receipts
```

---

## Special Disbursement Rules (PDX Logic)

When a purchaser remits a net amount (after marketing deductions) rather than a gross amount:

```
FUNCTION distributeNetAmount(checkAmount, remitterPropertyId):
  // PDX net-amount distribution
  owners = fetch owners from DOI cross-reference
  totalDecimal = SUM(owners.decimal)
  
  FOR each owner:
    ownerShare = checkAmount × (owner.decimal / totalDecimal)
    // No discrepancy between amount received and amount disbursed
    ENSURE SUM(ownerShares) = checkAmount (within rounding tolerance)
```

This logic was modified to allow a net amount on a check to be distributed to selected owners only (not all DOI owners), while preserving reconciliation integrity.

---

## AR Point Distribution (Common Receivable Point)

When a check covers multiple properties that share a common receivable point, the system distributes the check amount using tracking participation factors:

```
commonARPoint = fetch from CheckInputMasterData
IF commonARPoint configured:
  distributionKey = 'participation_factor'
  FOR each property under common AR point:
    propertyAllocation = checkAmount × property.participationFactor
    // Sum of participationFactors must = 1.0
```

---

## Reports

### Blocked Cost Center Report

Generated before posting to alert the operator of any cost centers that would block the journalization:

| Output | Description |
|--------|-------------|
| Blocked cost center codes | List of centers with posting block active |
| Impacted check lines | Which check lines would fail posting |
| Recommended action | Unblock cost center or redirect to open cost center |

### Proposed Journalization Report

A preview of what will post before final journalization. Includes two sections:

**Owner Section:**
- Owner ID and name
- Volume by product
- Gross value
- Tax deductions
- Marketing deductions
- Net payment amount

**Account Section:**
- GL account number
- Property/venture
- Cost center
- Amount
- Volume
- Other attributes

### DOI Decimal Comparison Report

Validates that the decimal interest configured in the PDX cross-reference matches the aggregated owner interests on the related DOI:

```
comparisonResult = ABS(pdx.decimalInterest - SUM(doi.ownerInterests)) > tolerance
IF comparisonResult:
  FLAG discrepancy → compliance review required
```

---

## CDEX/EDI 820 Integration

**CDEX** (Chemical Data Exchange — adapted for oil and gas) provides the EDI 820 Payment Order/Remittance Advice format used by purchasers to submit check detail electronically.

| Integration Point | Description |
|------------------|-------------|
| Inbound file receipt | EDI 820 file received via Azure Service Bus or file upload |
| Remitter matching | CDEX company code matched to business partner via cross-reference |
| Auto-distribution | If cross-reference exists, system auto-distributes without manual entry |
| Exception queue | Unmatched CDEX records routed to manual review queue |
| Search capability | Look up CDEX records by remitter, date range, property, or check amount |

**Company code expansion:** CDEX company code field expanded to accommodate more than two characters, supporting larger operator environments with complex purchaser hierarchies.

---

## Regulatory Reporting Support

Properties designated in Check Input master data can feed state tax and royalty reports via the Tax 2.0 reporting framework. This allows check-input-derived distributions to appear in:

- Texas GLO and UT Royalty Reports
- ONRR-2014 (where check input covers federal lease royalties)
- State production tax returns
