# M1 — Production & Volume Allocation

---

## 1. Module Purpose

Captures physical production data from all upstream assets and allocates bulk delivery network volumes back to individual well completions. This is the **origin of all volume data** in the lifecycle — every downstream module depends on M1 outputs.

---

## 2. Master Data

### 2.1 Producing Entities

| Entity | Key Attributes | Notes |
|--------|---------------|-------|
| `Field` | fieldId (PK), fieldName, state, county, operatorId | Top-level geographic grouping |
| `Reservoir` | reservoirId (PK), fieldId (FK), formationName | Geological classification |
| `Platform` | platformId (PK), fieldId (FK), platformType | Offshore/onshore structure |
| `Well` | wellId (PK, API#), platformId (FK), wellName, drillingStatus | API number is unique national identifier |

### 2.2 Well Completions (WC)

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| wellCompletionId | UUID | Yes | Primary key |
| wellId | UUID | Yes | FK → Well |
| effectiveDate | Date | Yes | Date-effective record |
| producingMethod | Enum(flowing, pump, gas_lift, plunger) | Yes | |
| productTypeCode | Enum(oil, gas, ngl, water) | Yes | |
| contaminationOverridePct | Decimal(5,4) | No | Override contamination % |
| theoreticalOverride | Boolean | No | Use manual theoretical |
| leaseWideGOR | Decimal(10,2) | No | Gas/Oil Ratio override |
| downtimeReasonCode | String(4) | No | Standard downtime codes |
| hierarchyParentWcId | UUID | No | For commingled completions |

### 2.3 Measurement Points (MP)

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| measurementPointId | UUID | Yes | Primary key |
| mpTypeCode | Enum(meter, tank_battery, flare, fuel) | Yes | |
| deliveryNetworkId | UUID | Yes | FK → DeliveryNetwork |
| heatingValueBtu | Decimal(8,3) | No | Default BTU/MCF |
| pressureBase | Decimal(6,2) | No | Base pressure (psia) |
| temperatureBase | Decimal(5,2) | No | Base temperature (°F) |
| meterSpecification | JSONB | No | Meter calibration data |

**Tank Strapping Sub-entity:** Each tank-type MP has calibration data:

| Attribute | Type | Description |
|-----------|------|-------------|
| tankId | UUID | PK |
| measurementPointId | UUID | FK → MeasurementPoint |
| calibrationDate | Date | Last calibration |
| strappingTable | JSONB | Gauge → volume conversion table |
| linearConversionConstant | Decimal(10,6) | Calculated volumetric constant |

### 2.4 Delivery Networks (DN)

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| deliveryNetworkId | UUID | Yes | PK |
| networkType | Enum(network, fuel_system, gas_plant) | Yes | |
| allocationTolerance | Decimal(5,2) | Yes | % variance allowed |
| allocationMethod | Enum(theoretical, historical, flat_rate) | Yes | |
| measurementGroupId | UUID | No | Grouping for batched MPs |
| downstreamNodeId | UUID | No | Next node in chain |

---

## 3. Transactional Entities

### 3.1 Well Completion Volumes (Daily)

| Attribute | Type | Description |
|-----------|------|-------------|
| wcVolumeId | UUID | PK |
| wellCompletionId | UUID | FK |
| productionDate | Date | |
| reportingPeriod | YearMonth | |
| allocatedOilBbl | Decimal(12,3) | Final allocated barrels |
| allocatedGasMcf | Decimal(12,3) | Final allocated MCF |
| allocatedWaterBbl | Decimal(12,3) | |
| allocationStatus | Enum(estimated, allocated, finalized) | |
| allocationRunId | UUID | FK → AllocationRun |

### 3.2 Downtime Summary

| Attribute | Type | Description |
|-----------|------|-------------|
| downtimeId | UUID | PK |
| wellCompletionId | UUID | FK |
| downtimeDate | Date | |
| hoursDown | Decimal(4,2) | 0–24 |
| reasonCode | String(4) | Standard code table |
| approvedBy | UUID | FK → User |

### 3.3 Well Test Record

| Attribute | Type | Description |
|-----------|------|-------------|
| wellTestId | UUID | PK |
| wellCompletionId | UUID | FK |
| testDate | Date | |
| testType | Enum(potential, adjusted, regulatory) | |
| multiStreamType | String(2) | |
| oilRateBblDay | Decimal(10,3) | Tested oil rate |
| gasRateMcfDay | Decimal(10,3) | Tested gas rate |
| waterRateBblDay | Decimal(10,3) | |
| gOrRatio | Decimal(8,3) | Gas/oil ratio |
| heatingValueBtu | Decimal(8,3) | |
| flowLinePressure | Decimal(8,2) | psi |
| testDurationHours | Decimal(4,2) | |

### 3.4 Measurement Point Volumes (Daily)

| Attribute | Type | Description |
|-----------|------|-------------|
| mpVolumeId | UUID | PK |
| measurementPointId | UUID | FK |
| measurementDate | Date | |
| grossVolume | Decimal(12,3) | Raw measurement |
| netVolume | Decimal(12,3) | After corrections |
| volumeType | Enum(sales, fuel, flare, inventory) | |
| correctionFactor | Decimal(6,5) | Pressure/temp correction |

### 3.5 Allocation Run

| Attribute | Type | Description |
|-----------|------|-------------|
| allocationRunId | UUID | PK |
| deliveryNetworkId | UUID | FK |
| runDate | Timestamp | |
| reportingPeriod | YearMonth | |
| totalInputs | Decimal(15,3) | Σ theoreticals |
| totalOutputs | Decimal(15,3) | Actual sales + inventory |
| percentDifference | Decimal(5,2) | |
| withinTolerance | Boolean | |
| runStatus | Enum(pending, running, completed, failed) | |
| triggeredBy | Enum(manual, scheduled, auto) | |

---

## 4. Business Processing Rules

### 4.1 Volume Allocation Algorithm

```
FUNCTION runAllocation(deliveryNetworkId, period):
  1. Fetch all WC downtime summaries for period
  2. Fetch latest well test records per WC
  3. FOR each WC in DN:
       theoretical = productionRate × hoursUp / (testDuration / 60)
  4. inputs = SUM(theoreticals)
  5. outputs = totalSales + (endInventory - beginInventory)
  6. pctDiff = ((outputs - inputs) / inputs) × 100
  7. IF ABS(pctDiff) <= DN.allocationTolerance:
       scaleFactor = outputs / inputs
       FOR each WC: allocatedVolume = theoretical × scaleFactor
  8. ELSE:
       FLAG AllocationRun as OUT_OF_TOLERANCE → notify supervisor
  9. PERSIST WcVolume records with allocationStatus = ALLOCATED
 10. PUBLISH event: ua.production.volumes.allocated
```

### 4.2 Tank Strapping Volume Calculation

```
FUNCTION calculateTankVolume(gaugeReading, tankId):
  strappingTable = fetch calibration table for tankId
  volume = interpolate(strappingTable, gaugeReading)
  netVolume = volume × linearConversionConstant
  RETURN netVolume
```

### 4.3 Downtime Handling

- If `hoursDown = 24`: well produces zero theoretical volume for that day
- Partial downtime: `hoursUp = 24 - hoursDown`
- Downtime adjustments retroactively recalculate allocations for the affected period

### 4.4 Automation (Delivery Network Auto-Run)

Configurable rulesets drive automated allocation:

| Ruleset Type | Trigger | Action |
|-------------|---------|--------|
| Run Schedule | Daily cron (configurable) | Execute allocation for all DNs in scope |
| Observation Bot | Volume reading received | Validate reading, flag anomalies |
| Decision Ruleset | Tolerance check | Auto-approve if within tolerance, else escalate |

---

## 5. API Endpoints

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| GET | `/api/v1/production/wells` | List wells | `production:read` |
| GET | `/api/v1/production/wells/{wellId}` | Well detail | `production:read` |
| GET | `/api/v1/production/well-completions` | List WCs | `production:read` |
| POST | `/api/v1/production/well-completions` | Create WC | `production:write` |
| PUT | `/api/v1/production/well-completions/{id}` | Update WC | `production:write` |
| POST | `/api/v1/production/volumes/daily` | Submit daily volumes | `production:write` |
| GET | `/api/v1/production/volumes/daily` | Query daily volumes | `production:read` |
| POST | `/api/v1/production/downtime` | Record downtime | `production:write` |
| POST | `/api/v1/production/well-tests` | Submit well test | `production:write` |
| GET | `/api/v1/production/measurement-points` | List MPs | `production:read` |
| POST | `/api/v1/production/allocation/run` | Trigger allocation run | `production:execute` |
| GET | `/api/v1/production/allocation/runs/{id}` | Run status | `production:read` |
| GET | `/api/v1/production/allocation/runs/{id}/results` | Allocated volumes | `production:read` |
| GET | `/api/v1/production/delivery-networks` | List DNs | `production:read` |
| POST | `/api/v1/production/delivery-networks` | Create DN | `production:admin` |

---

## 6. Reports

| Report | Description | Trigger |
|--------|-------------|---------|
| Measurement Document | Daily volume summary per MP | On demand / scheduled |
| Daily Pressures Report | Pressure readings per well | On demand |
| Allocation Variance Report | pctDiff by DN for period | Monthly close |
| WC Volume Summary | Allocated volumes by well completion | Monthly |
| Downtime Analysis | Hours down by reason code | Monthly |
