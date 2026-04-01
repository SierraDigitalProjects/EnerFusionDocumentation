# Units of Measure & Product Definitions

> Source: PRA-KSHOP/UOM.rtf, PRA-KSHOP/Configuration Guides/IS_PRA_Configuration_guide_UOM.docx

---

## Oil

| Measure Type | Unit | Description |
|-------------|------|-------------|
| Volume | BBL (barrel) | Standard US measurement for crude oil; 1 BBL = 42 US gallons |
| Density / Quality | API Gravity (°API) | Inverse measure of crude density; higher API = lighter crude = typically higher value |
| Density / Quality | Specific Gravity | Weight of oil relative to water; inversely related to API gravity |

### API Gravity Ranges and Quality Classification

| API Range | Classification | Commercial Impact |
|-----------|---------------|-------------------|
| < 25° API | Heavy crude | Discount to WTI benchmark; refinery processing penalty |
| 25° – 40° API | Medium crude | Near-benchmark pricing |
| > 40° API | Light crude | Premium to WTI benchmark; preferred by most refineries |

**API Gravity Pricing Formula:**

```
price = basePrice + (actualGravity - baseGravity) × adjustmentPerDegree
```

Where `adjustmentPerDegree` is configured on the API Gravity Scale master record in M4 Valuation.

---

## Natural Gas

| Measure Type | Unit | Description |
|-------------|------|-------------|
| Volume | MCF | Thousand cubic feet (standard condition: 14.73 psia, 60°F) |
| Volume | MMCF | Million cubic feet |
| Volume | BCF | Billion cubic feet |
| Energy | MMBTU | Million British Thermal Units (primary energy billing unit) |
| Energy | MBtu | Thousand BTUs |
| Quality | BTU/MCF (heating value) | Energy content per unit volume; higher BTU = more valuable gas |

### Heating Value (BTU Factor)

The **heating value** is the BTU content per cubic foot of gas. It varies by gas composition (methane %, heavier hydrocarbon content):

```
Energy (MMBTU) = Volume (MCF) × Heating Value (BTU/MCF) / 1,000,000
```

| Heating Value Range | Description |
|--------------------|-------------|
| < 1,000 BTU/MCF | Lean gas (mostly methane); lower processing value |
| 1,000 – 1,100 BTU/MCF | Typical pipeline quality |
| > 1,100 BTU/MCF | Rich gas; significant NGL content; high processing value |

**Keepwhole provision:** When the sum of residue gas + NGL return values (measured in MMBTU equivalent) is less than the owner's raw gas entitlement heating value, the keepwhole provision triggers a compensating payment. See M3 Contractual Allocation for calculation detail.

### Pressure and Temperature Corrections

Gas volumes are measured at field conditions (varying pressure and temperature) and must be corrected to standard conditions before reporting:

```
correctedVolume = measuredVolume × (measuredPressure / standardPressure) × (standardTemp / measuredTemp)
correctionFactor = (measuredPressure / 14.73) × (520 / (measuredTemp + 460))
```

Standard conditions (used in EnerFusion):
- Pressure base: 14.73 psia (configurable per measurement point)
- Temperature base: 60°F (configurable per measurement point)

---

## Natural Gas Liquids (NGL)

| Component | Typical Unit | Notes |
|-----------|-------------|-------|
| Mixed NGLs | GAL (gallons) | Most liquid volumes reported in gallons |
| Ethane (C2) | GAL | Lightest NGL; petrochemical feedstock |
| Propane (C3) | GAL | Consumer heating fuel; common transport |
| iso-Butane (iC4) | GAL | Refinery alkylation feed |
| normal-Butane (nC4) | GAL | Blended into gasoline |
| Natural Gasoline (C5+) | BBL or GAL | Heavier NGL; can be measured in barrels at standard conditions (40°C and 40°F) |

**NGL Unit Conversion:**
- 1 BBL NGL = 42 US gallons
- Heavier NGL components (natural gasoline, C5+) are typically measured in barrels (at 40°C and 40°F reference temperature)

---

## Measurement Point Volume Types

| Volume Type | Description | Reporting Treatment |
|------------|-------------|-------------------|
| Sales | Volume sold to purchaser under contract | Primary billing volume |
| Fuel | Gas consumed on-lease for operational equipment | Deducted before ONRR royalty calculation |
| Flare | Gas vented/flared due to operational constraints | Non-revenue volume; reported for environmental compliance |
| Inventory | Oil in tank at period end minus beginning | Adjusts sales to total production |
| Lease Use | Gas used for lease operations (compression, lift) | Royalty-exempt in most states |
| Plant Shrinkage | Volume loss in gas plant processing | Reduces gross gas entitlement before NGL recovery |

---

## Unit of Measure Configuration in EnerFusion

Each measurement point and well completion is configured with its applicable UOM:

| Configuration Field | Options | Impact |
|--------------------|---------|--------|
| Volume UOM | BBL, MCF, GAL | Determines storage precision and display |
| Pressure base | 14.73 psia (default), or state-specific | Corrects measured volumes to standard |
| Temperature base | 60°F (default) | Corrects measured volumes to standard |
| Heating value (default) | BTU/MCF | Used when no meter-level heating value is available |
| Specific gravity | Decimal | Used in theoretical calculation for oil |

---

## BTU Factor in Revenue Calculations

In gas sales contracts, the price may be quoted in $/MMBTU rather than $/MCF. The BTU factor converts volume to energy:

```
grossValue = volume_MCF × heatingValue_BTU_per_MCF / 1,000,000 × price_per_MMBTU
```

The BTU factor used in settlement statements is sourced from:
1. Measurement point meter specification (measured value)
2. Well completion override (if measurement data unavailable)
3. Delivery network default (fallback)

In ONRR-2014 reporting, the BTU factor appears on each TC line and affects the royalty volume computation.

---

## CO₂ Considerations (OGOR Reporting)

For Outer Continental Shelf (Gulf of Mexico) federal leases, CO₂ content in produced gas must be reported on the Oil and Gas Operations Report (OGOR). CO₂ removal fees are configured as a separate pricing deduction:

```
co2RemovalFee ($/MCF) × allocatedVolume_MCF × co2ContentPct = co2Deduction
```

This deduction appears as a separate line item on settlement statements for affected federal leases and is reported as a processing allowance on OGOR submissions.
