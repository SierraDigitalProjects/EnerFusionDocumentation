# M5 — Balancing Workplace

---

## 1. Module Purpose

Manages product imbalances that arise when owners physically take more or less product than their entitled share. Tracks cumulative imbalance positions per owner, enforces Product Balancing Agreement (PBA) limits, and provides tools to resolve imbalances through cash settlements or make-up volumes.

---

## 2. Core Concepts

| Concept | Definition |
|---------|------------|
| **Imbalance** | Difference between an owner's entitled volume and their actual take |
| **Over-take** | Owner took more than entitled; they "owe" volumes/cash |
| **Under-take** | Owner took less than entitled; they are "owed" volumes/cash |
| **Cash-out** | Converting volume imbalance to a cash obligation |
| **Make-up** | Resolving imbalance by adjusting future production takes |
| **PBA** | Product Balancing Agreement — contract governing imbalance resolution terms |
| **Balancing Statement** | Formal document showing each owner's imbalance position |

---

## 3. Master Data

### 3.1 Product Balancing Agreement (PBA) — see also M3

Full attributes in M3; key fields driving M5 logic:

| Attribute | Type | Description |
|-----------|------|-------------|
| pbaId | UUID | PK |
| imbalanceLimit | Decimal(12,3) | Trigger for mandatory resolution |
| cashOutTrigger | Decimal(12,3) | Auto cash-out threshold |
| cashOutPrice | Decimal(10,4) | Settlement price for cash-out |
| cashOutPriceMethod | Enum(fixed, spot, contract) | How cash-out price is determined |
| makeUpPeriodMonths | Integer | Months allowed for make-up |
| interestRatePct | Decimal(5,4) | Interest on unpaid cash balances |
| imbalanceRetentionOnTransfer | Boolean | Keep imbalance with old owner on transfer |

### 3.2 Owner Balance Record

Snapshot of cumulative imbalance per owner per agreement:

| Attribute | Type | Description |
|-----------|------|-------------|
| balanceRecordId | UUID | PK |
| pbaId | UUID | FK → ProductBalancingAgreement |
| businessPartnerId | UUID | FK → BusinessPartner |
| reportMonth | YearMonth | |
| openingBalanceVolume | Decimal(12,3) | Prior month close |
| periodEntitlement | Decimal(12,3) | This month's entitlement |
| periodActualTake | Decimal(12,3) | This month's actual take |
| periodVariance | Decimal(12,3) | entitlement - actualTake |
| closingBalanceVolume | Decimal(12,3) | openingBalance + periodVariance |
| cashBalanceAmount | Decimal(14,2) | For cash-out balances |
| balanceStatus | Enum(open, cash_out_pending, settled) | |

---

## 4. Transactional Entities

### 4.1 Manual Balance Adjustment

| Attribute | Type | Description |
|-----------|------|-------------|
| adjustmentId | UUID | PK |
| pbaId | UUID | FK |
| businessPartnerId | UUID | FK |
| adjustmentDate | Date | |
| adjustmentVolume | Decimal(12,3) | Positive = add volume; negative = reduce |
| adjustmentReason | Text | |
| adjustmentType | Enum(manual_correction, transfer_adjustment, cash_out, make_up) | |
| supportingDocRef | String(100) | Reference to authorization document |
| createdBy | UUID | FK → User |
| approvedBy | UUID | FK → User |

### 4.2 Balancing Statement

| Attribute | Type | Description |
|-----------|------|-------------|
| statementId | UUID | PK |
| pbaId | UUID | FK |
| statementPeriod | YearMonth | |
| generatedDate | Date | |
| statementStatus | Enum(draft, issued, acknowledged, disputed) | |
| totalOverTakeVolume | Decimal(12,3) | Sum of positive imbalances |
| totalUnderTakeVolume | Decimal(12,3) | Sum of negative imbalances |
| reportFileRef | String | Blob storage path |

### 4.3 Cash-Out Settlement

| Attribute | Type | Description |
|-----------|------|-------------|
| cashOutId | UUID | PK |
| pbaId | UUID | FK |
| businessPartnerId | UUID | FK |
| settlementPeriod | YearMonth | |
| imbalanceVolume | Decimal(12,3) | Volume being cashed out |
| cashOutPrice | Decimal(10,4) | Price applied |
| settlementAmount | Decimal(14,2) | imbalanceVolume × cashOutPrice |
| interestAmount | Decimal(12,2) | Accrued interest |
| totalDue | Decimal(14,2) | settlementAmount + interestAmount |
| paymentStatus | Enum(pending, paid, written_off) | |
| paymentRef | UUID | FK → CheckPayment (M7) |

---

## 5. Business Processing Rules

### 5.1 Period-End Imbalance Calculation

```
FUNCTION calculatePeriodImbalance(pbaId, period):
  FOR each owner IN pba.owners:
    entitlement = fetch from M3 ContractualAllocationRecord
    actualTake = fetch confirmed nomination volumes for owner/period
    variance = entitlement - actualTake

    ownerBalance.openingBalance = prior period closingBalance
    ownerBalance.closingBalance = openingBalance + variance

    // Check thresholds
    IF ABS(ownerBalance.closingBalance) > PBA.cashOutTrigger:
      initiateCashOut(owner, ownerBalance)
    ELSE IF ABS(ownerBalance.closingBalance) > PBA.imbalanceLimit:
      ALERT revenue_accountant
```

### 5.2 Cash-Out Processing

```
FUNCTION initiateCashOut(owner, balanceRecord):
  SWITCH PBA.cashOutPriceMethod:
    CASE FIXED: price = PBA.cashOutPrice
    CASE SPOT: price = fetchSpotPrice(productType, period)
    CASE CONTRACT: price = fetchContractPrice(owner.contractId, period)

  settlement = CashOutSettlement(
    imbalanceVolume = balanceRecord.closingBalance,
    cashOutPrice = price,
    settlementAmount = imbalanceVolume × price,
    interestAmount = calculateInterest(imbalanceVolume, PBA.interestRatePct, periodsOutstanding)
  )

  // Zero out volume balance, carry forward cash balance
  balanceRecord.closingBalanceVolume = 0
  balanceRecord.cashBalanceAmount += settlement.totalDue
  PERSIST CashOutSettlement
  PUBLISH event to M7 for payment processing
```

### 5.3 Ownership Transfer — Balance Handling

```
FUNCTION handleTransferBalance(transferRequest):
  IF PBA.imbalanceRetentionOnTransfer = TRUE OR transferRequest.retainBalanceFlag = TRUE:
    // Imbalance stays with from-owner
    fromOwnerBalance unchanged
  ELSE:
    // Transfer imbalance to new owner
    toOwnerBalance.closingBalance += fromOwnerBalance.closingBalance
    fromOwnerBalance.closingBalance = 0
    CREATE ManualBalanceAdjustment(
      type = TRANSFER_ADJUSTMENT,
      reason = "Transfer ref: " + transferRequest.transferId
    )
```

### 5.4 Imbalance Settings Retained During Transfer

Per PBA configuration, the system can preserve:
- Cumulative volume imbalance position
- Cash balance position
- Make-up period rights

---

## 6. Balancing Workplace UI Features

| Feature | Description |
|---------|-------------|
| Owner Balance Dashboard | Tabular view of all owners' current imbalance positions |
| Balance History | Month-by-month imbalance trend per owner |
| Manual Adjustment Entry | Form to enter balance corrections with approval workflow |
| Cash-Out Initiation | Trigger and review pending cash-out settlements |
| Balancing Statement Generation | Generate, regenerate, and reprint formal statements |
| PBA Configuration | Manage agreement terms, thresholds, and pricing |

---

## 7. API Endpoints

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| GET | `/api/v1/balancing/pba` | List PBAs | `balancing:read` |
| POST | `/api/v1/balancing/pba` | Create PBA | `balancing:admin` |
| GET | `/api/v1/balancing/owner-balances` | Query owner balances | `balancing:read` |
| GET | `/api/v1/balancing/owner-balances/{ownerId}` | Owner balance history | `balancing:read` |
| POST | `/api/v1/balancing/adjustments` | Create manual adjustment | `balancing:write` |
| PUT | `/api/v1/balancing/adjustments/{id}/approve` | Approve adjustment | `balancing:approve` |
| POST | `/api/v1/balancing/statements/generate` | Generate balancing statement | `balancing:execute` |
| GET | `/api/v1/balancing/statements` | List statements | `balancing:read` |
| GET | `/api/v1/balancing/statements/{id}` | Statement detail | `balancing:read` |
| POST | `/api/v1/balancing/cash-outs` | Initiate cash-out | `balancing:execute` |
| GET | `/api/v1/balancing/cash-outs` | List cash-out settlements | `balancing:read` |

---

## 8. Reports

| Report | Description |
|--------|-------------|
| Owner Balances by Sales Month | Current imbalance position per owner per reporting month |
| Owner Balances by Report Month | Same data organized by report period |
| Balancing Exception Report | Owners exceeding PBA limits flagged for action |
| Balancing Statement | Formal per-owner document sent to interest holders |
| Historical Ownership Report | Ownership history for cross-reference with balance history |
| Zero Interest Owners Report | Owners with no current interest who may have residual balances |
