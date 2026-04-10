# M3 — Contractual Allocation & Product Control

---

## 1. Module Purpose

Bridges physical production (M1) and commercial agreements to determine how allocated volumes are distributed among sales contracts and individual owners. Also manages daily product nominations and gas plant processing, including NGL component splits and percent return-to-lease calculations.

---

## 2. Core Concepts

| Concept | Definition |
|---------|------------|
| **Contractual Allocation** | Distribution of physical production volumes to specific sales contracts |
| **Entitlement** | An owner's fair share of production based on their DOI percentage |
| **Network Imbalance** | Cumulative difference between actual takes and entitled volumes per owner |
| **Nomination** | Daily declaration of expected product volumes to be delivered |
| **Keep-Whole Allocation** | Gas plant processing where producer retains equivalent BTU value |
| **Percent Return to Lease (PRTL)** | % of gas volumes returned to lease after processing |
| **NGL Component Allocation** | Distribution of liquid components by chemical analysis |

---

## 3. Master Data

### 3.1 MP/WC → Transporter Contract Cross-Reference

The **critical link** between physical meters/wells and sales contracts:

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| xrefId | UUID | Yes | PK |
| measurementPointId | UUID | No | FK → MeasurementPoint |
| wellCompletionId | UUID | No | FK → WellCompletion |
| contractId | UUID | Yes | FK → Contract |
| transporterId | UUID | Yes | FK → BusinessPartner |
| effectiveDate | Date | Yes | |
| expirationDate | Date | No | |
| allocationPriority | Integer | No | For multiple contract coverage |
| keepWholeFlag | Boolean | No | Gas plant processing type |

### 3.2 Capacity & Availability

| Entity | Attributes | Description |
|--------|-----------|-------------|
| CapacityRoll_MP | mpId, period, capacityVolume | Max delivery capacity per MP |
| CapacityRoll_WC | wcId, period, capacityVolume | Max delivery per well completion |
| AvailabilityGrouping | groupId, groupName, mpIds[], wcIds[] | Pools sources for nomination |

### 3.3 NGL Component Allocation Basis

| Attribute | Type | Description |
|-----------|------|-------------|
| componentBasisId | UUID | PK |
| deliveryNetworkId | UUID | FK |
| componentCode | Enum(ethane, propane, butane, pentane, hexane_plus) | |
| allocationMethod | Enum(mole_percent, btu_equivalent, volume_percent) | |
| analysisDate | Date | Gas analysis effective date |

### 3.4 Product Balancing Agreement (PBA) Master

| Attribute | Type | Description |
|-----------|------|-------------|
| pbaId | UUID | PK |
| pbaName | String(100) | |
| productType | Enum(gas, oil, ngl) | |
| imbalanceLimit | Decimal(12,3) | Max allowed imbalance before cash-out |
| cashOutTrigger | Decimal(12,3) | Auto cash-out threshold |
| cashOutPrice | Decimal(10,4) | Price for cash-out settlements |
| interestRate | Decimal(5,4) | Interest on cash imbalances |

### 3.5 PBA → WC/MP Cross-Reference

| Attribute | Type | Description |
|-----------|------|-------------|
| pbaXrefId | UUID | PK |
| pbaId | UUID | FK → ProductBalancingAgreement |
| wellCompletionId | UUID | FK (nullable) |
| measurementPointId | UUID | FK (nullable) |
| effectiveDate | Date | |

---

## 4. Transactional Entities

### 4.1 Contractual Allocation Record

| Attribute | Type | Description |
|-----------|------|-------------|
| allocationRecordId | UUID | PK |
| contractId | UUID | FK |
| ventureId | UUID | FK |
| businessPartnerId | UUID | FK |
| period | YearMonth | |
| productType | Enum(oil, gas, ngl) | |
| entitledVolume | Decimal(12,3) | Calculated entitlement |
| allocatedVolume | Decimal(12,3) | Actual contract allocation |
| varianceVolume | Decimal(12,3) | entitledVolume - allocatedVolume |
| allocationMethod | Enum(sales_point_orig, division_order, owner_level) | |

### 4.2 Marketing Group Network Imbalance

| Attribute | Type | Description |
|-----------|------|-------------|
| imbalanceId | UUID | PK |
| marketingGroupId | UUID | FK |
| period | YearMonth | |
| productType | String | |
| cumulativeImbalance | Decimal(12,3) | + = over-takes; - = under-takes |
| lastUpdated | Timestamp | |

### 4.3 Nomination

| Attribute | Type | Description |
|-----------|------|-------------|
| nominationId | UUID | PK |
| availabilityGroupingId | UUID | FK |
| nominationDate | Date | |
| nominatedVolume | Decimal(12,3) | |
| confirmedVolume | Decimal(12,3) | Transporter confirmation |
| nominationStatus | Enum(submitted, confirmed, adjusted, cancelled) | |
| submittedBy | UUID | FK → User |
| submittedAt | Timestamp | |

### 4.4 Percent Return to Lease (PRTL)

| Attribute | Type | Description |
|-----------|------|-------------|
| prtlId | UUID | PK |
| deliveryNetworkId | UUID | FK (gas plant DN) |
| period | YearMonth | |
| inletVolumeMcf | Decimal(12,3) | Gas entering plant |
| returnedVolumeMcf | Decimal(12,3) | Residue returned to lease |
| percentReturnPct | Decimal(5,4) | returnedVolume / inletVolume |
| extractedNglBbl | Decimal(12,3) | Total NGLs extracted |

---

## 5. Business Processing Rules

### 5.1 Allocation Methods

**Method A — Sales Point Originated Allocation (Keep-Whole)**

```
FUNCTION salesPointAllocation(mpId, period):
  totalSales = fetch total sales at MP for period
  FOR each owner in DOI:
    entitledVolume = totalSales × owner.decimalInterest
    contractVolume = assign to applicable sales contract
  PUBLISH ContractualAllocationRecord
```

**Method B — Division Order Sales Allocation**

```
FUNCTION divisionOrderAllocation(contractId, period):
  contractVolumes = fetch contract delivery volumes
  FOR each owner in Contract DOI:
    allocatedVolume = contractVolumes × owner.contractInterest
  RECONCILE against production allocation from M1
```

**Method C — Owner Level Allocation**

Each owner's volume is tracked individually against their contracted amount, allowing owner-level nominations and imbalances.

### 5.2 Entitlement Calculation

```
FUNCTION calculateEntitlement(ventureId, period, productType):
  totalProduction = fetch from M1 allocated volumes
  FOR each owner in venture.DOI:
    IF productType = 'GAS' AND gasPlantFlag:
      netProduction = totalProduction × PRTL.percentReturnPct
    ELSE:
      netProduction = totalProduction
    entitlement = netProduction × owner.decimalInterest
  CHECK: SUM(entitlements) = totalProduction (within tolerance)
```

### 5.3 Gas Plant Processing

```
FUNCTION gasPlantProcessing(plantNetworkId, period):
  inletGas = fetch MP volumes at plant inlet

  // Calculate residue gas
  gasAnalysis = fetch latest ComponentAnalysis for plant
  extractedBtu = inletGas × (1 - gasAnalysis.residuePct)
  returnedGas = inletGas × PRTL.percentReturnPct

  // Calculate NGL yields
  FOR each component IN [ethane, propane, butane, pentane, hexaneplus]:
    yieldBbl = inletGas × gasAnalysis.componentMolePct × component.gallonsPerMole / 42
    allocateNGL(component, yieldBbl)

  // Apply sliding scale to plant owners if configured
  IF plantOwner.slidingScaleFlag:
    processingFee = lookupSlidingScale(inletGas, plantOwner.scalId)
```

### 5.4 Network Imbalance Tracking

```
ON allocation period close:
  FOR each owner in marketing group:
    periodVariance = entitledVolume - allocatedVolume
    imbalance.cumulativeImbalance += periodVariance

    IF ABS(imbalance.cumulativeImbalance) > PBA.cashOutTrigger:
      triggerCashOut(owner, imbalance)

    IF ABS(imbalance.cumulativeImbalance) > PBA.imbalanceLimit:
      ALERT operations supervisor
```

---

## 6. API Endpoints

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| GET | `/api/v1/allocation/contracts` | List contracts | `allocation:read` |
| POST | `/api/v1/allocation/xref/mp-contract` | Create MP→Contract xref | `allocation:write` |
| GET | `/api/v1/allocation/entitlements` | Query entitlements by period | `allocation:read` |
| POST | `/api/v1/allocation/run` | Execute allocation for period | `allocation:execute` |
| GET | `/api/v1/allocation/records` | Query allocation records | `allocation:read` |
| GET | `/api/v1/allocation/imbalances` | List network imbalances | `allocation:read` |
| POST | `/api/v1/allocation/nominations` | Submit nomination | `allocation:write` |
| GET | `/api/v1/allocation/nominations` | Query nominations | `allocation:read` |
| PUT | `/api/v1/allocation/nominations/{id}` | Update nomination | `allocation:write` |
| POST | `/api/v1/allocation/gas-plant/run` | Run gas plant processing | `allocation:execute` |
| GET | `/api/v1/allocation/prtl` | Query PRTL records | `allocation:read` |
| GET | `/api/v1/allocation/pba` | List PBA master records | `allocation:read` |
| POST | `/api/v1/allocation/pba` | Create PBA | `allocation:admin` |
| GET | `/api/v1/allocation/validation/volume-report` | CA Volume Validation Report | `allocation:read` |

---

## 7. Reports

| Report | Description |
|--------|-------------|
| CA Volume Validation Report | Reconciles allocated vs entitled volumes; highlights variances |
| Marketing Group Network Imbalance Report | Cumulative imbalances per owner per marketing group |
| Nomination vs Actual Report | Compares nominations against actual deliveries |
| Gas Plant Processing Summary | Inlet volumes, PRTL %, NGL yield by component |
| Balancing Exception Report | Owners exceeding PBA imbalance limits |
