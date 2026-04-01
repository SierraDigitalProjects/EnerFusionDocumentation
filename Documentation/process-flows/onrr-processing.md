# ONRR-2014 Processing — Process Flow

> Source: PRA-KSHOP/ONRR/ONRR concepts.docx, How to ONRR steps execution - workplace.docx, How to Finalization.docx, How to Section 6 ONRR logic details.docx, How to PPN works in ONRR.docx  
> Module: M8 — Regulatory, Tax & Royalty Reporting

---

## Overview

The ONRR-2014 (formerly MMS-2014) is the federal royalty report submitted to the Office of Natural Resources Revenue (ONRR) for production from federal and Indian leases. It is the most complex statutory report in the system due to its multi-level venture → lease → sales type hierarchy, Section 6 special royalty provisions, and the finalization/journalization process that posts General Ledger documents.

---

## ONRR Reporting Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│  PAYOR (issued by ONRR agency)                                  │
│    mapped to: Company Code                                       │
│    table: OIUREP_PAYOR                                           │
│                                                                  │
│  VENTURE (V1)                                                    │
│    participates in ONRR reporting                                │
│                                                                  │
│    ├── LEASE L1 (60% of Venture)                                 │
│    │    ├── SALES TYPE S1 (40% of L1 = 24% of V1)               │
│    │    └── SALES TYPE S2 (60% of L1 = 36% of V1)               │
│    │                                                             │
│    └── LEASE L2 (40% of Venture)                                 │
│         └── SALES TYPE S1 (100% of L2 = 40% of V1)             │
│                                                                  │
│  Volume and value divided: Venture → Leases → Sales Types       │
└─────────────────────────────────────────────────────────────────┘
```

**Master Data Types:**
- `Venture/DOI with multiple owners / multiple leases` — most common; table: `OIUREP_PROWNER`
- `Well/WC with multiple owners / multiple leases` — more granular; used when WC-level allocation is required
- `MMS Leases` (types: federal, Indian, or both) — table: `OIUREP_FEDINDIAN`
- `Payor/Company Code cross-reference` — table: `OIUREP_PAYOR`

> **Configuration note:** Most operators configure ONRR master data at the Venture/DOI level rather than Well/WC level because a single venture may contain many well completions, making WC-level setup operationally burdensome.

---

## ONRR-2014 Processing Steps

The workplace executes steps sequentially. A parallel processing architecture runs steps by company code and sales date simultaneously; if any parallel process fails, subsequent steps do not execute.

```
┌─────────────────────────────────────────────────────────────────┐
│  ONRR-2014 WORKPLACE EXECUTION SEQUENCE                         │
│                                                                  │
│  [1] POPULATE                                                    │
│      • Read from GL and OPSL tables                              │
│      • Match entries that don't net to zero                      │
│      • Populate OIUREP_RTPEND (raw transaction pending) table   │
│                                                                  │
│  [2] VALIDATE                                                    │
│      • Apply ONRR business rules                                 │
│      • Check lease/payor/sales type master data exists           │
│      • Section 6 royalty rate validation                         │
│      • Results in OIUREP_MMS_REJS (reject table)               │
│        - extract_run_type blank = Extract step reject            │
│                                                                  │
│  [3] GENERATE (Post Processing)                                  │
│      • Process TC 01 entries with Section 6 enabled leases      │
│      • Create TC 37 (severance + additional royalty)            │
│        or TC 38 (additional royalty only) entries               │
│      • Build PRDT/RDPT tables                                   │
│                                                                  │
│  [4] EXTRACT                                                     │
│      • Read from GL and OIUH_TEMP_GL                            │
│      • Prepare ASCII output file                                 │
│      • Records stored in OIUREP_MMS_2014_LOAD_EDITS            │
│                                                                  │
│  [5] SUMMARIZE (FP_PAY_SUM population)                          │
│      • Push distribution data to /PRA/FP_PAY_SUM table          │
│      • Queue /UA/FP/MAIN must be unlocked                       │
│      • /UA/FP_JE_LOCK entries cleaned on successful completion  │
│                                                                  │
│  [6] JOURNALIZE                                                  │
│      • Create journal entries from finalized ONRR data          │
│      • Post to GL (RAD or JEINT documents)                      │
│                                                                  │
│  [7] FINALIZE                                                    │
│      • Update Out-of-Statute (OOS) flags                        │
│      • Write PRDT/RDPT history (LT2 and LT8 entries)           │
│      • Unbrand run ID from FP_PAY_SUM entries                  │
│      • Clean up FP_PAY_SUM (subtract JEINT/RAD negatives)      │
│      • Delete migrated report entry if applicable               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Transaction Codes (TC)

ONRR-2014 uses specific transaction codes to classify royalty and adjustment line items:

| TC | Description | When Generated |
|----|-------------|----------------|
| TC 01 | Royalty (base production royalty) | Standard royalty on all production |
| TC 11 | Transportation allowance | When RVLA on TC 01 is nonzero; reduces royalty base |
| TC 13 | Marketing reimbursement / Quality differential | When result words `ERQADIFR` (reimbursement) or `ERQADIFD` (deduction) are used in VL formula |
| TC 15 | Processing allowance | When RVLA on TC 01 is nonzero; reduces royalty base |
| TC 37 | Section 6 — Severance + Additional Royalty | Section 6 lease with additional royalty provision |
| TC 38 | Section 6 — Additional Royalty only | Section 6 lease without severance rate applied |

> **Rule:** If TC 01 RVLA (Royalty Value Less Allowances) is nonzero, TC 11 (Transport) and TC 15 (Processing) entries may be added to TC 01 to reduce the reported royalty basis.

---

## Section 6 Processing

Section 6 applies to specific federal leases that carry additional royalty obligations beyond the standard rate.

### Configuration Hierarchy

| Step | Configuration Level | Transaction / Table | Priority |
|------|--------------------|--------------------|---------|
| 1 | Default severance rate | Config: Default rates by product + company code | Base |
| 2 | Lease royalty rate | Master data: Lease record with SECT6_FL flag | Overrides step 1 |
| 3 | Override royalty rate | Master data: FP_MASTER record (SECTION_6_LSE_FL) | Overrides step 2 |
| 4 | Ad hoc override | Admin workplace: Override Sev Tax Assessment Rate + Additional Royalty Rate by lease + product + validity range | Overrides steps 1–3 |

### Rate Resolution Logic

```
FUNCTION resolveSection6Rates(leaseId, productCode, salesDate):
  
  // Severance rate: Step 4 overrides Step 1
  severanceRate = config.defaultSeveranceRate(productCode, companyCode)
  IF adHocOverride exists for (leaseId, productCode, salesDate):
    severanceRate = adHocOverride.overrideSevTaxRate
  
  // Royalty rate: Step 3 overrides Step 2
  royaltyRate = lease.royaltyRate  // from FP_LEASE (step 2)
  IF fp_master.royaltyRateOverride exists:
    royaltyRate = fp_master.royaltyRateOverride  // step 3
  
  // Additional royalty rate: Only set in Step 4
  addRoyaltyRate = adHocOverride.additionalRoyaltyRate ?? 0
  
  RETURN { severanceRate, royaltyRate, addRoyaltyRate }
```

### TC Generation

```
IF lease.SECT6_FL = TRUE AND TC_01_exists:
  
  IF processing TC 37:
    tc37Amount = tc01Value × (severanceRate + addRoyaltyRate)
    APPEND TC 37 to RDPT (alongside existing TC 01)
  
  IF processing TC 38:
    tc38Amount = tc01Value × addRoyaltyRate
    APPEND TC 38 to RDPT
  
  // Final state: RDPT contains both TC 01 and TC 37 (or TC 38)
```

---

## Prior Period Notifications (PPNs) in ONRR

ONRR uses two types of master data PPNs that behave differently:

| PPN Type | Trigger | Change User in `OIUREPP_PRPAYXRF` | Scope |
|----------|---------|-----------------------------------|-------|
| Type 1 | Master data change at **venture/sales type** level | `MMSVNAME` | Applies only to that venture/sales type |
| Type 2 | Master data change at **venture/lease or venture/payor** level | Any other user | Applies to the whole venture and all related leases/sales types |

**Precedence rules:**

- If a Type 2 (venture-level) PPN exists and a subsequent Type 1 (sales-type) change occurs → system retains the Type 2 PPN
- If a Type 1 (sales-type) PPN exists and a subsequent Type 2 (venture-level) change occurs → system upgrades to Type 2 (removes `MMSVNAME` change user)

> **Result:** The most comprehensive PPN type always wins. A venture-level PPN supersedes a sales-type PPN.

---

## Finalization Detail

The Finalization step is the point of no return for an ONRR-2014 run. Key operations:

1. **OOS flag update** — marks out-of-statute entries so they are excluded from future processing
2. **History creation** — overwrites LT 2 and LT 8 entries in PRDT/RDPT tables to create permanent history records
3. **Migrated report cleanup** — deletes `FP_HST_REV` entry if the prior run was a migrated report that had a reversal
4. **Run ID unbranding** — removes the run ID from `FP_PAY_SUM` entries, completing the payment link
5. **FP_PAY_SUM cleanup** — adds JEINT or RAD entries (negative values) to corresponding FP_PAY_SUM entries (positive VL data); net result clears the payable balance
6. **Account document posting** — actual posting occurs in `P2_ACC_DOC_POST`; P2 tables are updated with run ID and document details are cleared

**Parallel process failure handling:** If any parallel process fails during Finalize, some documents may post while others fail. Check the journalize output table for remaining entries and logs to identify the failed parallel processes by sales date and company code.

---

## Queue Management

The ONRR queue (`/UA/FP/MAIN`) must be unlocked before summarization can proceed. Common issue:

**Symptom:** "Duplicate key" error during summarization step.  
**Cause:** `FP_JE_LOCK` entries not deleted on a failed prior summarization run — queue remains locked.  
**Resolution:**
1. Remove lock via ONRR-2014 Admin Workplace
2. Run report without debug mode
3. If `FP_JE_LOCK` entries persist, delete via admin script and re-trigger `R_TRIGGER_FP_QUE_BATCH`

---

## ONRR Master Data Setup

### Payor / Company Code Cross-Reference

Every ONRR run requires a payor code — an identifier issued by the ONRR agency to the reporting entity. The payor is mapped to the system's company code.

```
Payor XYZ123 → Company Code 1001
Payor ABC789 → Company Code 1002
```

Multiple company codes can map to the same payor if the operator reports consolidated.

### Venture → Lease Cross-Reference

For each venture that has federal or Indian lease production:

1. Set lease participation percentages (L1 = 60%, L2 = 40%)
2. Assign sales types per lease with their volume split percentages
3. Flag lease type (federal / Indian / both)
4. Configure Section 6 if applicable

---

## ASCII File Generation

The ONRR-2014 ASCII output file is generated after the Extract step:

```
Function: UA_FP_GENERATE_ASCII
Output path: /usr/sap/<SID>/DVEBMGS<instance>/work/ONRR_2014_<SID>_<runId>.TXT
Format: Fixed-width ASCII per ONRR PAAS electronic filing specification
```

The generated file is made available for download from the Regulatory Reporting UI and submitted to ONRR via the PAAS electronic filing portal.

---

## Quality Differential Deduction (TC 13)

When a valuation formula uses the reserved words `ERQADIFR` or `ERQADIFD`, the system generates a TC 13 line on the ONRR-2014:

| Result Word | TC Code | Description |
|------------|---------|-------------|
| `ERQADIFR` | TC 13 | Quality differential **reimbursement** (adds to royalty basis) |
| `ERQADIFD` | TC 13 | Quality differential **deduction** (reduces royalty basis) |

> **Important:** The quality differential must generate as a separate TC 13 line — it must not be collapsed into the Processing Deduct bucket (marketing bucket 16 in standard configuration). If `ERQADIFD` is mapped to the Processing bucket, ONRR rejects the line because processing allowance is not permitted on major product 1 (oil) in some lease types.

---

## Reports

| Report | Description |
|--------|-------------|
| ONRR-2014 Preview | Pre-finalization view of all TC lines by lease/sales type |
| ONRR Reject Report | List of validation rejects with rejection reason codes |
| ONRR Reconciliation Report | Compares ONRR report values to Revenue Distribution records; flags exceptions |
| Revenue Accounting Document / Recon Exception Report | GL document postings vs ONRR-reported amounts |
| Section 6 Override Audit | Shows all ad hoc rate overrides applied in the run |
| PPN Status Report | Open PPNs by venture, type, and age |
