# M4 — Contracts, Pricing & Valuation

---

## 1. Module Purpose

Applies commercial pricing, tax calculations, and deductions to the allocated volumes from M3 to determine the net financial value owed to each owner. Generates settlement statements and manages prior period adjustments. This is the **financial computation engine** of PRA.

---

## 2. Master Data

### 2.1 Sales Contract

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| contractId | UUID | Yes | PK |
| contractNumber | String(30) | Yes | |
| contractType | Enum(gas_purchase, oil_purchase, ngl_purchase, transport) | Yes | |
| buyerPartnerId | UUID | Yes | FK → BusinessPartner (purchaser) |
| sellerPartnerId | UUID | Yes | FK → BusinessPartner (operator) |
| effectiveDate | Date | Yes | |
| expirationDate | Date | No | |
| pricingMethod | Enum(fixed, formula, index, api_gravity) | Yes | |
| contractStatus | Enum(draft, approved, active, expired) | Yes | |
| marketingCostType | Enum(none, flat_rate, percent_of_value) | No | |
| co2RemovalFee | Decimal(10,4) | No | $/MCF CO2 removal cost |
| royaltyReimbursementFlag | Boolean | No | Purchaser reimburses royalty |

### 2.2 Fixed Price Conditions

| Attribute | Type | Description |
|-----------|------|-------------|
| fixedPriceId | UUID | PK |
| contractId | UUID | FK → Contract |
| productType | Enum(oil, gas, ngl, ethane, propane, ...) | |
| fixedPrice | Decimal(10,4) | $/unit |
| priceUnit | Enum(bbl, mcf, mmbtu, gallon) | |
| effectiveDate | Date | |
| expirationDate | Date | |

### 2.3 Formula Pricing

Complex formula-based pricing using reserved words and calculation steps:

| Attribute | Type | Description |
|-----------|------|-------------|
| formulaId | UUID | PK |
| contractId | UUID | FK |
| formulaName | String(100) | |
| formulaExpression | Text | Formula using reserved words |
| calculationSteps | JSONB | Step-by-step breakdown |
| globalTextRefs | JSONB | Reserved word definitions |
| effectiveDate | Date | |

**Reserved Word Examples:**

| Reserved Word | Definition |
|---------------|------------|
| `NYMEX_GAS` | Henry Hub monthly average |
| `WTI_OIL` | WTI Cushing monthly average |
| `API_GRAVITY_ADJ` | API gravity adjustment lookup |
| `BTU_FACTOR` | Heating value correction |
| `TRANSPORT_DEDUCT` | Pipeline transportation deduction |

### 2.4 API Gravity Scale

Oil pricing varies by API gravity (quality):

| Attribute | Type | Description |
|-----------|------|-------------|
| gravityScaleId | UUID | PK |
| scaleName | String(100) | |
| baseGravity | Decimal(4,1) | Reference API gravity |
| basePrice | Decimal(10,4) | $/bbl at base gravity |
| adjustmentPerDegree | Decimal(8,4) | $/bbl per degree above/below |

**API Gravity Adjustment Scale:**

| Gravity Range | Adjustment |
|--------------|------------|
| < 25° | Discount (heavy crude) |
| 25–40° | Base price |
| > 40° | Premium (light crude) |

### 2.5 Tax Master Data

#### State Tax Rates

| Attribute | Type | Description |
|-----------|------|-------------|
| stateTaxRateId | UUID | PK |
| state | String(2) | State code |
| taxType | Enum(severance, ad_valorem, conservation) | |
| productType | Enum(oil, gas, ngl) | |
| rateType | Enum(flat_pct, tiered, volume_based) | |
| taxRate | Decimal(6,5) | Applicable rate |
| effectiveDate | Date | |

#### Tax Classification (Well Level)

| Attribute | Type | Description |
|-----------|------|-------------|
| taxClassId | UUID | PK |
| wellCompletionId | UUID | FK |
| taxCategory | Enum(standard, stripper, enhanced_recovery, marginal) | |
| exemptions | String[] | Array of applicable exemption codes |

#### Tier Taxes (Production Volume Thresholds)

| Attribute | Type | Description |
|-----------|------|-------------|
| tierTaxId | UUID | PK |
| taxType | String | |
| volumeThreshold | Decimal(12,3) | Upper bound of this tier |
| rateForTier | Decimal(6,5) | Tax rate for volumes in tier |

#### Processing Allowances

| Entity | Attributes | Description |
|--------|-----------|-------------|
| TaxProcessingAllowance | state, productType, allowancePct, maxAmount | Deduction allowed against taxable value |
| RoyaltyProcessingAllowance | leaseId, allowancePct, maxMonthlyAmount | Royalty deduction for processing costs |
| MarketingCostAllowance | contractId, state, allowancePct | Allowable marketing cost deduction |

### 2.6 Valuation Cross-Reference (VCR)

Maps physical production to the correct pricing + tax treatment:

| Attribute | Type | Description |
|-----------|------|-------------|
| vcrId | UUID | PK |
| ventureId | UUID | FK → Venture |
| wellCompletionId | UUID | FK |
| contractId | UUID | FK → Contract |
| taxClassId | UUID | FK → TaxClassification |
| statementProfileId | UUID | FK → OilGasStatementProfile |
| effectiveDate | Date | |

### 2.7 Settlement Statement / DOI Cross-Reference

| Attribute | Type | Description |
|-----------|------|-------------|
| ssDoiXrefId | UUID | PK |
| settlementStatementId | UUID | FK |
| doiOwnerId | UUID | FK → DOI Owner Interest |
| effectiveDate | Date | |

---

## 3. Transactional Entities

### 3.1 Settlement Statement

| Attribute | Type | Description |
|-----------|------|-------------|
| settlementId | UUID | PK |
| contractId | UUID | FK |
| ventureId | UUID | FK |
| period | YearMonth | |
| productType | Enum | |
| allocatedVolume | Decimal(12,3) | From M3 |
| grossValue | Decimal(14,2) | volume × price |
| severanceTax | Decimal(12,2) | State production tax |
| adValoremTax | Decimal(12,2) | Property tax deduction |
| transportationDeduct | Decimal(12,2) | Pipeline charges |
| marketingDeduct | Decimal(12,2) | Marketing fee |
| processingDeduct | Decimal(12,2) | Gas plant fee |
| netValue | Decimal(14,2) | grossValue - all deductions - taxes |
| statementStatus | Enum(draft, posted, adjusted, closed) | |
| postedDate | Date | |

### 3.2 Oil and Gas Run Statement

Separate document for each physical "run" of oil or gas delivery:

| Attribute | Type | Description |
|-----------|------|-------------|
| runStatementId | UUID | PK |
| measurementPointId | UUID | FK |
| runDate | Date | |
| grossVolume | Decimal(12,3) | |
| temperatureCorrection | Decimal(6,5) | |
| gravityCorrection | Decimal(6,5) | |
| netVolume | Decimal(12,3) | Corrected volume |
| apiGravity | Decimal(4,1) | |
| bsw | Decimal(5,4) | Basic sediment and water % |
| purchaserPartnerId | UUID | FK |

### 3.3 Prior Period Adjustment (PPN Event)

| Attribute | Type | Description |
|-----------|------|-------------|
| ppnId | UUID | PK |
| originalSettlementId | UUID | FK → Settlement |
| adjustmentPeriod | YearMonth | Period being corrected |
| adjustmentReason | Text | |
| volumeAdjustment | Decimal(12,3) | |
| valueAdjustment | Decimal(14,2) | |
| ppnType | Enum(manual, system, notification) | |
| createdBy | UUID | FK → User |
| approvalStatus | Enum(pending, approved, rejected) | |

---

## 4. Business Processing Rules

### 4.1 Valuation Processing Flow

```
FUNCTION runValuation(ventureId, period):
  1. Fetch ContractualAllocation records (from M3) for period
  2. FOR each allocation record:
     a. Lookup VCR → get contractId, taxClassId, statementProfileId
     b. Fetch pricing rule from contract:
        IF pricingMethod = FIXED: price = FixedPriceCondition
        IF pricingMethod = FORMULA: price = evaluateFormula(formulaId)
        IF pricingMethod = API_GRAVITY: price = lookupGravityScale(apiGravity)
     c. grossValue = allocatedVolume × price
     d. Calculate deductions:
        severanceTax  = grossValue × StateTaxRate.rate × TaxProcessingAllowance
        adValorem     = grossValue × adValoremRate
        transport     = allocatedVolume × contract.transportRate
        marketing     = grossValue × contract.marketingCostPct
        IF owner.marketingFreeStatus: marketing = 0
        IF owner.royaltyType = RI: severanceTax per state rules
     e. netValue = grossValue - severanceTax - adValorem - transport - marketing
     f. PERSIST SettlementStatement
  3. POST event: ua.valuation.settlement.completed
```

### 4.2 Formula Evaluation Engine

```
FUNCTION evaluateFormula(formulaId, evaluationDate, volume):
  formula = fetch Formula + steps
  context = {}
  FOR step IN formula.calculationSteps (ordered):
    SWITCH step.type:
      CASE PRICE_INDEX: context[step.var] = fetchIndexPrice(step.indexCode, evaluationDate)
      CASE ARITHMETIC: context[step.var] = evaluate(step.expression, context)
      CASE LOOKUP: context[step.var] = lookupTable(step.tableId, context[step.key])
  price = context[formula.outputVar]
  RETURN price
```

### 4.3 Tiered Tax Calculation

```
FUNCTION calculateTieredTax(grossValue, volume, taxType, state, productType):
  tiers = fetch TierTaxes WHERE taxType AND state AND productType
  totalTax = 0
  remainingVolume = volume
  FOR tier IN tiers SORTED BY threshold:
    tierVolume = MIN(remainingVolume, tier.threshold - previousThreshold)
    tierValue = (tierVolume / volume) × grossValue
    totalTax += tierValue × tier.rateForTier
    remainingVolume -= tierVolume
  RETURN totalTax
```

### 4.4 API Gravity Pricing

```
FUNCTION calculateApiGravityPrice(basePrice, actualGravity, gravityScaleId):
  scale = fetch ApiGravityScale
  adjustment = (actualGravity - scale.baseGravity) × scale.adjustmentPerDegree
  RETURN basePrice + adjustment
```

---

## 5. API Endpoints

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| GET | `/api/v1/valuation/contracts` | List contracts | `valuation:read` |
| POST | `/api/v1/valuation/contracts` | Create contract | `valuation:write` |
| PUT | `/api/v1/valuation/contracts/{id}/approve` | Approve contract | `valuation:approve` |
| GET | `/api/v1/valuation/pricing/fixed` | List fixed prices | `valuation:read` |
| POST | `/api/v1/valuation/pricing/fixed` | Create fixed price | `valuation:write` |
| GET | `/api/v1/valuation/pricing/formulas` | List formulas | `valuation:read` |
| POST | `/api/v1/valuation/pricing/formulas` | Create formula | `valuation:write` |
| GET | `/api/v1/valuation/pricing/api-gravity` | List gravity scales | `valuation:read` |
| POST | `/api/v1/valuation/pricing/api-gravity` | Create gravity scale | `valuation:write` |
| POST | `/api/v1/valuation/run` | Execute valuation for period | `valuation:execute` |
| GET | `/api/v1/valuation/settlements` | Query settlement statements | `valuation:read` |
| GET | `/api/v1/valuation/settlements/{id}` | Settlement detail | `valuation:read` |
| POST | `/api/v1/valuation/settlements/{id}/adjust` | Create PPN adjustment | `valuation:write` |
| GET | `/api/v1/valuation/run-statements` | Oil/gas run statements | `valuation:read` |
| GET | `/api/v1/valuation/tax-rates` | List state tax rates | `valuation:read` |
| POST | `/api/v1/valuation/tax-rates` | Create tax rate | `valuation:admin` |
| GET | `/api/v1/valuation/vcr` | List VCR records | `valuation:read` |
| POST | `/api/v1/valuation/vcr` | Create VCR | `valuation:write` |

---

## 6. Reports

| Report | Description |
|--------|-------------|
| Settlement Statement | Owner-level net value by contract/period |
| Oil and Gas Run Statement | Per-delivery-run quality and volume detail |
| Valuation Summary | Gross value, deductions, net value by venture/period |
| Tax Calculation Detail | Breakdown of all tax components |
| Prior Period Adjustment Report | All PPNs by period with original vs adjusted values |
| Formula Pricing Audit | Reserved word values used in formula evaluations |
