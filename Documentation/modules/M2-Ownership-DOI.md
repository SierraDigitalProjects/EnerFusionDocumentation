# M2 — Ownership & Division of Interest (DOI)

---

## 1. Module Purpose

Defines and maintains the legal ownership structure of all upstream producing assets. Establishes which business partners hold working interest (WI) or royalty interest (RI) in each lease, venture, or unit — and in what percentage. This is the **financial identity layer** that every downstream revenue calculation depends on.

---

## 2. Core Concepts

| Concept | Definition |
|---------|------------|
| **Division of Interest (DOI)** | A record that maps each owner to their decimal interest share in a specific lease/venture for a defined time period |
| **Working Interest (WI)** | Owner bears proportional share of costs and revenues |
| **Royalty Interest (RI)** | Owner receives gross revenue share with no cost burden |
| **Overriding Royalty Interest (ORRI)** | Carved out of the WI; no cost burden |
| **Venture** | A pooling of multiple leases/tracts for production purposes |
| **Unit Venture** | Regulatory pooling unit across multiple owners/operators |
| **Joint Venture (JVA)** | Multiple working interest owners sharing a single wellbore |
| **Chain of Title** | Historical record of all ownership changes |

---

## 3. Master Data

### 3.1 Business Partner

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| businessPartnerId | UUID | Yes | PK |
| partnerType | Enum(individual, company, trust, government) | Yes | |
| legalName | String(200) | Yes | |
| taxId | String(20) | Yes | EIN or SSN (encrypted) |
| address | JSONB | Yes | Physical address |
| paymentAddress | JSONB | No | Override payment address |
| isActive | Boolean | Yes | |
| escheatState | String(2) | No | State for unclaimed property |
| cdexId | String(20) | No | CDEX/EDI identifier |

### 3.2 Owner Attribute Group

Groups of owners sharing common payment/tax attributes:

| Attribute | Type | Description |
|-----------|------|-------------|
| ownerAttributeGroupId | UUID | PK |
| groupName | String(100) | |
| paymentMethod | Enum(check, ach, wire) | |
| paymentThreshold | Decimal(10,2) | Minimum payment amount |
| stateTaxWithholdingPct | Decimal(5,4) | |
| federalWithholdingPct | Decimal(5,4) | |

### 3.3 Upstream Lease Identifier

| Attribute | Type | Description |
|-----------|------|-------------|
| leaseId | UUID | PK |
| leaseNumber | String(20) | Regulatory lease number |
| stateLease | String(2) | State abbreviation |
| county | String(50) | |
| legalDescription | Text | Metes and bounds or section description |
| leaseType | Enum(oil, gas, combo) | |
| operatorId | UUID | FK → BusinessPartner |
| royaltyRate | Decimal(5,4) | Landowner royalty fraction |

### 3.4 Venture / Base DOI

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| ventureId | UUID | Yes | PK |
| ventureName | String(100) | Yes | |
| leaseId | UUID | Yes | FK → PraLeaseIdentifier |
| ventureType | Enum(base, unit) | Yes | |
| productType | Enum(oil, gas, ngl) | Yes | |
| effectiveDate | Date | Yes | |
| expirationDate | Date | No | |
| checkoutStatus | Enum(active, checked_out, pending_approval) | Yes | |

**DOI Owner Interest Records** (child of Venture):

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| doiOwnerId | UUID | Yes | PK |
| ventureId | UUID | Yes | FK → Venture |
| businessPartnerId | UUID | Yes | FK → BusinessPartner |
| interestType | Enum(WI, RI, ORRI, NPRI, PRRI) | Yes | |
| decimalInterest | Decimal(10,8) | Yes | Must sum to 1.0 across same type |
| slidingScaleFlag | Boolean | No | Volume-sensitive royalty |
| marketingFreeStatus | Boolean | No | Exempt from marketing deduction |
| taxExemptionCode | String(4) | No | State/federal exemption |
| productionPaymentFlag | Boolean | No | Volume-based payment obligation |
| effectiveDate | Date | Yes | |
| expirationDate | Date | No | |

### 3.5 Unit / Tract Records (Unit Ventures)

| Attribute | Type | Description |
|-----------|------|-------------|
| unitId | UUID | PK |
| ventureId | UUID | FK → Venture (type=unit) |
| tractId | UUID | PK |
| tractName | String(100) | |
| tractParticipationFactor | Decimal(10,8) | Allocation % within unit |
| regulatoryUnitNumber | String(20) | State-assigned unit number |

---

## 4. Transactional Entities

### 4.1 DOI Check-in / Check-out

Workflow to protect data integrity during ownership changes:

| Attribute | Type | Description |
|-----------|------|-------------|
| checkoutId | UUID | PK |
| ventureId | UUID | FK |
| checkedOutBy | UUID | FK → User |
| checkoutTimestamp | Timestamp | |
| checkoutReason | Text | |
| checkInTimestamp | Timestamp | Null until checked back in |
| approvalStatus | Enum(pending, approved, rejected) | |
| approvedBy | UUID | FK → User |

### 4.2 Ownership Transfer Request

| Attribute | Type | Description |
|-----------|------|-------------|
| transferId | UUID | PK |
| ventureId | UUID | FK |
| fromOwnerId | UUID | FK → BusinessPartner |
| toOwnerId | UUID | FK → BusinessPartner |
| transferDate | Date | Effective date of transfer |
| interestType | Enum(WI, RI, ORRI) | |
| transferDecimalInterest | Decimal(10,8) | |
| transferFundsFlag | Boolean | Transfer suspended funds |
| retainBalanceFlag | Boolean | Retain imbalance with old owner |
| approvalStatus | Enum(pending, approved, rejected) | |

### 4.3 DOI → MP/WC Cross-Reference

The **critical link** between ownership and physical production:

| Attribute | Type | Description |
|-----------|------|-------------|
| xrefId | UUID | PK |
| ventureId | UUID | FK → Venture |
| measurementPointId | UUID | FK → MeasurementPoint (nullable) |
| wellCompletionId | UUID | FK → WellCompletion (nullable) |
| effectiveDate | Date | |
| allocationBasis | Enum(decimal_interest, sliding_scale) | |

---

## 5. Business Processing Rules

### 5.1 DOI Validation Rules

```
RULE: DecimalInterestIntegrity
  SUM(decimalInterest) WHERE interestType = 'WI' AND period = X MUST = 1.0
  Tolerance: ±0.00000001 (floating point)

RULE: CheckoutExclusivity
  Only ONE checkout record may be active per venture at a time
  Attempting a second checkout MUST return CONFLICT error

RULE: TransferEffectiveDateValidation
  transferDate MUST be >= first day of an open accounting period
  transferDate MUST NOT be in a locked/closed period

RULE: BalanceRetention
  IF retainBalanceFlag = FALSE:
    Transfer cumulative imbalance to toOwner
  ELSE:
    Imbalance remains attributed to fromOwner
```

### 5.2 Sliding Scale Royalty

Volume-sensitive royalties tier based on monthly production:

```
FUNCTION calculateSlidingRoyalty(baseRoyalty, monthlyVolume, slidingScaleId):
  tiers = fetch SlidingScaleTiers WHERE scalId = slidingScaleId
  FOR tier IN tiers (sorted by volume threshold ASC):
    IF monthlyVolume <= tier.threshold:
      RETURN tier.royaltyRate
  RETURN tiers.last().royaltyRate  // max tier
```

### 5.3 Marketing Free Processing

Owners flagged `marketingFreeStatus = TRUE` receive gross value with no marketing cost deductions. Valuation module (M4) checks this flag via cross-reference before applying deductions.

### 5.4 Chain of Title Maintenance

Every ownership transfer creates an immutable audit record:

```
ON transfer APPROVED:
  1. Set expirationDate on current DOI owner record = transferDate - 1 day
  2. Create new DOI owner record for toOwner effective transferDate
  3. Append to chain_of_title table (immutable, no deletes)
  4. Generate Schedule A (Transfer Order) report
  5. PUBLISH event: ua.ownership.doi.changed
```

---

## 6. Collective DOI Workplace

The primary UI for bulk DOI management:

| Action | Description | Required Permission |
|--------|-------------|-------------------|
| Upload DOI Template | Bulk create/update via Excel template | `ownership:import` |
| Create Base DOI | Single venture + owner interest setup | `ownership:write` |
| Check Out | Lock venture for editing | `ownership:write` |
| Modify Owner Interests | Edit decimal interests while checked out | `ownership:write` |
| Submit for Approval | Route to approver | `ownership:write` |
| Approve / Reject | Final approval of DOI change | `ownership:approve` |
| Check In | Release lock and activate changes | `ownership:approve` |

---

## 7. API Endpoints

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| GET | `/api/v1/ownership/ventures` | List ventures | `ownership:read` |
| POST | `/api/v1/ownership/ventures` | Create venture | `ownership:write` |
| GET | `/api/v1/ownership/ventures/{id}` | Venture + DOI detail | `ownership:read` |
| PUT | `/api/v1/ownership/ventures/{id}` | Update venture | `ownership:write` |
| GET | `/api/v1/ownership/ventures/{id}/owners` | List owner interests | `ownership:read` |
| POST | `/api/v1/ownership/ventures/{id}/owners` | Add owner interest | `ownership:write` |
| PUT | `/api/v1/ownership/ventures/{id}/owners/{ownerId}` | Update interest | `ownership:write` |
| DELETE | `/api/v1/ownership/ventures/{id}/owners/{ownerId}` | Remove interest | `ownership:write` |
| POST | `/api/v1/ownership/ventures/{id}/checkout` | Check out venture | `ownership:write` |
| POST | `/api/v1/ownership/ventures/{id}/checkin` | Check in / approve | `ownership:approve` |
| GET | `/api/v1/ownership/transfers` | List transfer requests | `ownership:read` |
| POST | `/api/v1/ownership/transfers` | Create transfer | `ownership:write` |
| PUT | `/api/v1/ownership/transfers/{id}/approve` | Approve transfer | `ownership:approve` |
| GET | `/api/v1/ownership/business-partners` | List partners | `ownership:read` |
| POST | `/api/v1/ownership/business-partners` | Create partner | `ownership:write` |
| GET | `/api/v1/ownership/leases` | List leases | `ownership:read` |
| POST | `/api/v1/ownership/xref/doi-mp` | Create DOI↔MP xref | `ownership:write` |

---

## 8. Reports

| Report | Description |
|--------|-------------|
| Schedule A (Transfer Order) | Formal transfer documentation per completed transfer |
| Historical Ownership Report | All owner interests for a venture across all time periods |
| Zero Interest Owners Report | Owners with 0.0% interest flagged for cleanup |
| DOI Validation Report | Decimal interest integrity check across all ventures |
| Chain of Title Report | Full ownership history per lease |
