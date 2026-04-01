# Accounts Receivable Workplace — Process Flow

> Source: PRA-KSHOP/Roadmap Specs/AR workplace Specification_v1.1.pdf  
> Module: M7 — Payment Processing & Check Input

---

## Overview

The AR Workplace allows revenue accountants to review, categorize, write off, transfer, and report on outstanding accounts receivable balances. These balances arise when purchaser remittances do not exactly match settled amounts — common in complex multi-property, multi-product settlement environments.

---

## AR Balance Lifecycle

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      AR BALANCE LIFECYCLE                                │
│                                                                          │
│  Check Input posts → AR key created                                     │
│         │                                                                │
│         ▼                                                                │
│  AR Workplace — User reviews outstanding balance                        │
│         │                                                                │
│         ├──── Flag as RECONCILED (variance cleared)                     │
│         │                                                                │
│         ├──── Flag as DEFERRED (pending resolution, aging continues)    │
│         │                                                                │
│         ├──── CATEGORIZE all or part of balance                        │
│         │         └── Assign category code (not just "X")              │
│         │                                                                │
│         ├──── Flag for WRITE-OFF                                        │
│         │         └── Requires APPROVE or can be DELETED               │
│         │                                                                │
│         ├──── TRANSFER all or part to another AR key                   │
│         │         (move balance to correct remitter/property)           │
│         │                                                                │
│         └──── Add COMMENT (tied to write-off, transfer, or general)    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## AR Key Structure

An AR key uniquely identifies an outstanding receivable balance and is the primary search and grouping dimension:

| Component | Description |
|-----------|-------------|
| Remitter | Purchaser business partner ID (with name/description display) |
| Venture/DOI | Producing property (with name/description display) |
| Product | Oil / Gas / NGL |
| Sales Month | Accounting period |
| Company Code | Operator's company |

---

## User Actions

### Reconcile

Mark a balance as resolved — no further action required. Used when the operator confirms the variance is within acceptable tolerance or has been explained by an offsetting transaction.

### Defer

Mark a balance for future resolution without stopping the aging clock. Useful when the operator is in discussion with the purchaser but wants to track the balance as active.

### Categorize

Assign a category code to all or part of a balance. Category codes are user-defined and allow management reporting by category:

| Example Category | Use Case |
|-----------------|---------|
| `DISPUTE` | Balance under active dispute with purchaser |
| `PENDING-CDEX` | Waiting for CDEX file match |
| `AUDIT` | Under external audit review |
| `ESCROW` | Portion held in escrow |

> **Enhancement:** The UI displays the actual category code, not a generic "X" marker, allowing reviewers to immediately understand the balance status.

### Flag for Write-Off

Submit a balance or partial balance for write-off:
- Minimum one-character write-off reason code required
- Write-off requires approval (separate approver role)
- Approver can approve or delete the write-off request
- **Multi-row approval:** Approvers can select multiple write-off rows and approve all simultaneously, with a single approval-level comment applied to all selected rows

### Transfer

Move all or part of a balance to a different AR key:
- Used when a check was applied to the wrong remitter or property
- Source balance reduced; target balance increased
- Transfer is logged with reason code and approver

### Comment

Attach a free-text comment to any AR balance action (write-off, transfer, or general note). Comments are:
- Uploaded in bulk via comment upload template (for batch reconciliation workflows)
- Retained when Excel downloads are re-sorted (comments stay linked to their AR key)

---

## Auto Write-Off

The system supports scheduled automatic write-offs for small balances that meet configured criteria:

| Configuration | Description |
|--------------|-------------|
| Dollar threshold | Balances below this amount are eligible for auto write-off |
| Percentage threshold | Balances below this % of the original check are eligible |
| Maximum dollar amount | Cap on auto write-off regardless of percentage threshold |
| Age minimum | Balance must be outstanding for at least N days before auto write-off |
| Frequency | Can be scheduled more frequently than monthly (e.g., weekly) — configurable as part of Accounting Period Close |
| Exclusions | Category codes that exempt a balance from auto write-off |

---

## AR Reporting

### Summary Report

Shows total outstanding AR balances aggregated by company and remitter (or product), with optional subtotals by responsibility ID (the accountant assigned to each remitter):

| Column | Description |
|--------|-------------|
| Company | Operator company code |
| Remitter | Purchaser name and ID |
| Product | Oil / Gas / NGL |
| Sales Month | Period |
| Balance Amount | Open AR |
| Aging Bucket | 0–30, 31–60, 61–90, 90+ days |
| Responsibility ID | Accountant assigned |

### Detail Report

Shows individual AR balance lines meeting the user's selection criteria. Useful for period-end close review:

- All open balances above a dollar threshold
- Balances deferred more than N days
- Balances flagged for write-off pending approval
- Balances by category code

### Accounting Month View

Toggle between:
- **Sales Month** — the production month the balance relates to
- **Accounting Month** — the period in which the check was posted

This distinction matters when reconciling AR to the GL, since posting month and sales month may differ.

---

## Search and Selection Enhancements

### Multi-Value Selection

All search criteria on the AR Workplace support multi-value input with F4 help (SAP-style value help):
- Select multiple remitters in one search
- Select multiple ventures/DOIs
- Select a range of sales months

### Category Code Filter

Category code is available as a search/filter criterion with multi-select support. Allows compliance teams to quickly pull all balances in a specific category for bulk review.

### Expanded UI

The AR Workplace UI is rendered full-screen to maximise the number of balance rows visible simultaneously, reducing scrolling for accountants managing large remitter portfolios.

---

## Integration with Accounting Period Close

AR auto write-off can be triggered as a step within the Accounting Period Close sequence:

```
Accounting Period Close Sequence (M7 steps):
  1. Lock prior period for new check input
  2. Run AR aging report — identify stale balances
  3. Execute auto write-off (configurable threshold)
  4. Generate AR Control Report (summary of all write-offs)
  5. Approve remaining write-off queue
  6. Post final AR journal entries
  7. Reconcile AR to GL
```

---

## AR Control Report

A period-end management report summarising all AR activity:

| Section | Content |
|---------|---------|
| Opening balance by remitter | AR balance at start of period |
| New balances posted | Checks received and posted |
| Write-offs approved | Balances written off this period |
| Transfers | Balances moved between AR keys |
| Closing balance | Outstanding AR at period end |
| Aging summary | Days outstanding distribution |

---

## Technical Remediation

For AR balances that arise from system errors (e.g., misposted check input, incorrect allocation logic), the Technical Remediation workflow allows authorized users to:
1. Identify the source transaction that created the erroneous AR balance
2. Create a correcting entry that reverses the error
3. Re-post with correct values
4. Document the remediation reason for audit purposes

This workflow requires elevated permissions (`payments:admin`) and generates an audit log entry.
