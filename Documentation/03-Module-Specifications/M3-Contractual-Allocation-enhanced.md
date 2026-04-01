# M3 — Contractual Allocation & Product Control (Enhanced)

> Source references: PRA-KSHOP/CA/, PRA-KSHOP/Roadmap Specs/Spec_Rounding and Alloc_V11.pdf, PRA-KSHOP/Roadmap Documents/CA-SPO-*.doc

---

## 1. Module Purpose

Bridges physical production volumes from M1 and ownership structures from M2 to commercial sales contracts. Allocates production to sales contracts per owner entitlements, manages gas plant processing, tracks network imbalances, and handles daily product nominations.

---

## 2. Core Concepts

| Concept | Definition |
|---------|------------|
| **Marketing Group** | A grouping of owners whose production is pooled for allocation and imbalance tracking |
| **Sales Point Forecast (SPF)** | Projected future production volumes per owner for nomination and entitlement planning |
| **Sales Point Actual (SPO)** | Actual measured volumes at the sales point for the period; basis for final allocation |
| **Entitlement** | The volume an owner is entitled to take based on their DOI decimal interest × total production |
| **Network Imbalance** | Cumulative difference between an owner's actual takes and their entitlement |
| **Keepwhole** | Contract provision ensuring an owner receives the heating value equivalent of their raw gas regardless of plant processing efficiency |
| **Settlement Diversity** | A configuration where selected owners are grouped together and valued/settled at the group level rather than individually |
| **PVR** | Production Volume Reconciliation — period-end reconciliation of allocated vs actual volumes |
| **PTR** | Production Transfer Report — summarises volume movements between accounting entities |
| **Percent Return to Lease (PRTL)** | Fraction of processed NGL/residue gas returned to the lease; remainder is retained by the gas plant as a processing fee |

---

## 3. Master Data

### 3.1 Marketing Group

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| marketingGroupId | UUID | Yes | PK |
| groupName | String(100) | Yes | |
| productType | Enum(gas, oil, ngl) | Yes | |
| allocationMethod | Enum(entitlement, equal_split, volume_nominated) | Yes | |
| settlementDiversity | Boolean | No | Group-level valuation enabled |
| keepwholeFlag | Boolean | No | Plant keepwhole provision active |
| roundingLevel | Enum(owner, marketing_group, venture) | Yes | Level at which rounding differences are absorbed |

### 3.2 Sales Contract ↔ Marketing Group Cross-Reference

| Attribute | Type | Description |
|-----------|------|-------------|
| xrefId | UUID | PK |
| contractId | UUID | FK → Contract (M4) |
| marketingGroupId | UUID | FK → MarketingGroup |
| measurementPointId | UUID | FK → MeasurementPoint |
| effectiveDate | Date | |
| productTypeCode | String | Oil / Gas / NGL component |

### 3.3 Gas Plant Processing Configuration

| Attribute | Type | Description |
|-----------|------|-------------|
| plantConfigId | UUID | PK |
| inletMeasurementPointId | UUID | Raw gas inlet MP |
| residueGasPctReturn | Decimal(6,5) | Fraction of residue gas returned to lease |
| ethanePctReturn | Decimal(6,5) | |
| propanePctReturn | Decimal(6,5) | |
| butanePctReturn | Decimal(6,5) | |
| naturalGasolinePctReturn | Decimal(6,5) | |
| fuelUsagePct | Decimal(6,5) | Plant fuel deducted before PRTL |
| shrinkagePct | Decimal(6,5) | Volume loss in processing |
| effectiveDate | Date | |

### 3.4 Nomination Record

| Attribute | Type | Description |
|-----------|------|-------------|
| nominationId | UUID | PK |
| ventureId | UUID | FK → Venture |
| contractId | UUID | FK → Contract |
| nominatedVolume | Decimal(12,3) | MCF or BBL per day |
| nominationDate | Date | Effective date |
| nominationType | Enum(daily, monthly) | |
| submittedBy | UUID | FK → BusinessPartner |

### 3.5 Settlement Diversity Group

| Attribute | Type | Description |
|-----------|------|-------------|
| diversityGroupId | UUID | PK |
| groupName | String(100) | |
| marketingGroupId | UUID | FK → MarketingGroup |
| valuationLevel | Enum(group, owner) | How valuation is applied |

---

## 4. Transactional Entities

### 4.1 Contractual Allocation Record

| Attribute | Type | Description |
|-----------|------|-------------|
| allocationId | UUID | PK |
| ventureId | UUID | FK → Venture |
| contractId | UUID | FK → Contract |
| period | YearMonth | |
| productTypeCode | String | |
| allocatedVolume | Decimal(12,3) | Final owner volume |
| entitledVolume | Decimal(12,3) | Calculated entitlement |
| actualTakeVolume | Decimal(12,3) | What owner actually took |
| imbalanceVolume | Decimal(12,3) | Take minus entitlement (cumulative delta) |
| allocationBasis | Enum(entitlement, equal_split) | |
| roundingAdjustment | Decimal(8,5) | Volume absorbed to balance |
| allocationRunId | UUID | FK → AllocationRun |

### 4.2 Marketing Group Network Imbalance

| Attribute | Type | Description |
|-----------|------|-------------|
| imbalanceId | UUID | PK |
| marketingGroupId | UUID | FK |
| ventureId | UUID | FK |
| period | YearMonth | |
| openingImbalance | Decimal(12,3) | Balance forward from prior period |
| currentPeriodTake | Decimal(12,3) | Actual volume taken |
| currentPeriodEntitlement | Decimal(12,3) | Entitled volume |
| closingImbalance | Decimal(12,3) | = opening + take − entitlement |
| imbalanceType | Enum(over_take, under_take, balanced) | |

### 4.3 Sales Point Adjustment (Manual Entry)

| Attribute | Type | Description |
|-----------|------|-------------|
| adjustmentId | UUID | PK |
| contractId | UUID | FK |
| period | YearMonth | |
| adjustmentVolume | Decimal(12,3) | Positive = add, Negative = subtract |
| adjustmentReason | Text | |
| approvedBy | UUID | FK → User |

---

## 5. Business Processing Rules

### 5.1 Entitlement Calculation

```
FUNCTION calculateEntitlement(ventureId, period, productType):
  doi = fetch DOI owner interests for venture (active in period)
  totalProduction = fetch allocated volume from M1 for period
  
  FOR each owner IN doi:
    entitledVolume = owner.decimalInterest × totalProduction
    IF owner.slidingScaleFlag:
      decimalInterest = calculateSlidingRoyalty(totalProduction, owner)
      entitledVolume = decimalInterest × totalProduction
    
    PERSIST ContractualAllocation.entitledVolume
```

### 5.2 Allocation to Owners Within Marketing Group

Allocation follows a priority ordering to ensure equitable treatment of owners with small decimal interests:

```
STEP 1: Calculate primary product entitlements at Venture/DOI level
  entitlement_i = decimalInterest_i × marketingGroupVolume

STEP 2: Calculate adjusted entitlements
  IF owner has adjustment/nomination override:
    adjustedEntitlement = entitlement_i + nominationAdjustment_i
  
STEP 3: Allocate actual sales volumes to owners
  FOR each owner (sorted by decimal interest DESC to avoid small-interest exclusion):
    allocatedVolume_i = actualSales × (adjustedEntitlement_i / sumAdjustedEntitlements)

STEP 4: Handle rounding differences
  SWITCH marketingGroup.roundingLevel:
    CASE 'owner':      assign rounding to last owner processed
    CASE 'marketing_group': accumulate to group-level rounding bucket
    CASE 'venture':    carry to venture rounding account

STEP 5: Validate: SUM(allocatedVolumes) = actualSalesVolume (within floating-point tolerance)
```

### 5.3 Keepwhole Processing

When a gas plant processes rich gas and the NGL + residue value is less than the owner's raw gas entitlement value, the keepwhole provision compensates the owner for the shortfall:

```
FUNCTION calculateKeepwhole(ownerId, period, marketingGroupId):
  rawGasEntitlement = entitlement × rawGasBtuFactor
  processedReturnValue = residueGasValue + nglComponentsValue
  
  IF processedReturnValue < rawGasEntitlement:
    keepwholePayment = rawGasEntitlement - processedReturnValue
    CREATE ManualCAEntry type=KEEPWHOLE amount=keepwholePayment
  
  RETURN keepwholePayment
```

### 5.4 Gas Plant NGL Component Allocation (PRTL)

```
STEP 1: Receive wet gas at inlet MP
STEP 2: Apply shrinkage factor → residue gas volume
STEP 3: Deduct plant fuel usage → net inlet
STEP 4: Apply PRTL per component:
         residueGasToLease = netInlet × plantConfig.residueGasPctReturn
         ethaneToLease     = ngls × plantConfig.ethanePctReturn
         propaneToLease    = ngls × plantConfig.propanePctReturn
         ...
STEP 5: Allocate returned volumes to lease owners per DOI entitlement
STEP 6: Plant retains remainder as processing fee
```

### 5.5 Network Imbalance Monitoring

```
EVERY period close:
  FOR each marketingGroup:
    closingImbalance = openingImbalance + actualTake - entitlement
    
    IF ABS(closingImbalance) > marketingGroup.imbalanceTolerance:
      PUBLISH imbalanceAlert event
      FLAG for Balancing Workplace (M5) review
    
    PERSIST MarketingGroupImbalance record
```

### 5.6 Rounding Rules (from Spec_Rounding and Alloc)

The system must eliminate the "subtraction method" (where one owner receives everything left over after allocating to others) and replace it with a fair rounding approach:

1. **No subtraction method** — all owners with equal decimal interests must receive equal allocated volumes
2. **Owner-level rounding configuration** — operators may choose a different rounding precision at owner level vs. marketing group level
3. **Floating quantity fields** — all volume and energy fields float regardless of aggregation level
4. **Settlement diversity groups** — group-level volume/value transactions are allocated to individual owner level using group participation factors

---

## 6. ONRR Venture → Lease → Sales Type Hierarchy

For federal leases, contractual allocation must maintain the ONRR reporting hierarchy:

```
Venture (V1)
  ├── Lease L1 (60% participation)
  │     ├── Sales Type S1 (40% of L1 = 24% of V1)
  │     └── Sales Type S2 (60% of L1 = 36% of V1)
  └── Lease L2 (40% participation)
        └── Sales Type S1 (100% of L2 = 40% of V1)
```

Volumes and values are divided from venture level → leases → sales types, maintaining proportional splits at each level. This hierarchy feeds M8 ONRR-2014 reporting.

---

## 7. API Endpoints

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| GET | `/api/v1/allocation/marketing-groups` | List marketing groups | `allocation:read` |
| POST | `/api/v1/allocation/marketing-groups` | Create marketing group | `allocation:write` |
| GET | `/api/v1/allocation/entitlements` | Query entitlements by venture/period | `allocation:read` |
| POST | `/api/v1/allocation/run` | Trigger contractual allocation run | `allocation:execute` |
| GET | `/api/v1/allocation/runs/{id}` | Allocation run status | `allocation:read` |
| GET | `/api/v1/allocation/imbalances` | Query network imbalances | `allocation:read` |
| POST | `/api/v1/allocation/adjustments` | Create manual sales point adjustment | `allocation:write` |
| GET | `/api/v1/allocation/nominations` | List nominations | `allocation:read` |
| POST | `/api/v1/allocation/nominations` | Create nomination | `allocation:write` |
| GET | `/api/v1/allocation/gas-plant-configs` | List plant configs | `allocation:read` |
| POST | `/api/v1/allocation/gas-plant-configs` | Create plant config | `allocation:admin` |
| GET | `/api/v1/allocation/pvr` | Production Volume Reconciliation report | `allocation:read` |

---

## 8. Reports

| Report | Description | Trigger |
|--------|-------------|---------|
| CA Volume Validation Report | Reconciles allocated volumes to MP-level production; flags out-of-balance conditions | Monthly close |
| Network Imbalance Report | Cumulative and period imbalances by marketing group and owner | Monthly / on demand |
| Production Volume Reconciliation (PVR) | Compares nominated vs actual vs allocated volumes | Period end |
| Production Transfer Report (PTR) | Volume movements between accounting entities | On demand |
| Owner Entitlement Summary | Entitlement vs actual take vs imbalance per owner | Monthly |
| Gas Plant PRTL Summary | Returns to lease by component per plant | Monthly |
| Keepwhole Calculation Detail | Shortfall calculation and compensation per marketing group | When keepwhole triggered |
