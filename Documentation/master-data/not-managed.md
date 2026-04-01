# Not Managed in SAP

## Definition

**Not Managed in SAP** objects are master data entities that have no equivalent in SAP ERP or SAP IS-Oil. They are created and maintained exclusively within EnerFusion Upstream Accounting. There is no synchronization to SAP for these records — they represent capabilities unique to the EnerFusion domain model.

---

## M1 — Production

### Tank Strapping Table

**EnerFusion Schema:** `production.tank_strapping_records`

Calibration data for oil tank batteries. Converts a raw gauge reading (inches) to a volume (barrels) using an interpolated lookup table derived from the physical tank calibration. The `linearConversionConstant` captures the volumetric constant for the overall tank geometry.

| Why Not in SAP | Detail |
|----------------|--------|
| SAP stores no tank calibration data | IS-Oil references external measurement but does not maintain strapping tables |
| EnerFusion interpolates in real time | Each volume submission triggers a lookup against the active strapping table version |

**Versioning:** A new strapping record is created for each calibration event. The allocation algorithm always uses the strapping table active on the measurement date. Historical calibrations are retained for audit.

---

### Delivery Network Automation Ruleset

**EnerFusion Schema:** `admin_ilm.automation_rulesets`

Configurable rules that drive automated allocation runs, anomaly detection, and tolerance decisions without human intervention. This is an EnerFusion-native automation capability.

| Ruleset Type | Description | Trigger |
|-------------|-------------|---------|
| Run Schedule | Cron-based daily allocation trigger per DN | Configurable schedule |
| Observation Bot | Validates incoming volume readings against expected ranges | Volume receipt event |
| Decision Ruleset | Auto-approves allocation if within tolerance; escalates if not | Allocation completion |

---

## M2 — Ownership

### Unit / Tract Record

**EnerFusion Schema:** `ownership.unit_tracts`

Regulatory pooling unit data. Tract participation factors determine how a unit's production is allocated across the contributing tracts. Required for state-mandated unit operations but has no SAP master data equivalent.

| Attribute | Description |
|-----------|-------------|
| Tract participation factor | Decimal allocation % for this tract within the unit |
| Regulatory unit number | State-assigned pooling unit identifier |
| Effective dating | Factor changes when state-approved unit amendments occur |

---

## M3 — Contractual Allocation

### Gas Plant Processing Configuration

**EnerFusion Schema:** `allocation.gas_plant_configs`

Configures how rich gas volumes entering a processing plant are split into residue gas and NGL components (ethane, propane, butane, natural gasoline). Percent-return-to-lease (PRTL) records define the fraction of processed product returned to each lease.

| Configuration Element | Description |
|----------------------|-------------|
| Plant inlet MP | Measurement point receiving raw gas |
| Component splits (%) | Shrinkage and NGL recovery fractions per product |
| PRTL record | Per-lease portion of processed product returned |
| Fuel usage % | Plant fuel consumption deducted before allocation |

SAP IS-Oil handled gas plant allocation through IS-Oil-specific configuration tables that are not available in standard SAP ERP. This is entirely EnerFusion territory.

---

### Marketing Group Network Imbalance

**EnerFusion Schema:** `allocation.marketing_group_imbalances`

Tracks cumulative volume imbalances at the marketing group level — where multiple owners nominate into a shared pipeline but actual deliveries deviate from nominations. No SAP equivalent exists.

---

## M4 — Valuation

### Formula Pricing Rule

**EnerFusion Schema:** `valuation.formula_pricing_rules`, `valuation.formula_steps`

The formula pricing engine evaluates multi-step expressions using reserved words (index prices, BTU factors, transport deductions) to compute a period-specific price. This is a PRA-domain capability not present in SAP SD condition management.

| Reserved Word | Data Source | Description |
|---------------|------------|-------------|
| `NYMEX_GAS` | External price feed | Henry Hub monthly average |
| `WTI_OIL` | External price feed | WTI Cushing monthly settlement |
| `API_GRAVITY_ADJ` | API Gravity Scale record | Quality-based crude adjustment |
| `BTU_FACTOR` | Measurement Point | Heating value correction |
| `TRANSPORT_DEDUCT` | Contract terms | Pipeline charge deduction |
| `CO2_REMOVAL_FEE` | Contract | CO₂ removal cost per MCF |

**Price index feeds** are loaded into EnerFusion from external market data providers (e.g., Platts, Argus) via a nightly file import or API feed. The formula engine references these as inputs but does not connect to market data APIs directly.

---

### Valuation Cross-Reference (VCR)

**EnerFusion Schema:** `valuation.vcr`

The VCR is the central mapping record that connects a producing venture and well completion to the correct contract, tax classification, and settlement statement profile. This three-way join drives the valuation run. No SAP equivalent.

| VCR maps… | …to… | Purpose |
|-----------|------|---------|
| Venture + Well Completion | Contract | Which pricing contract applies |
| Venture + Well Completion | Tax Classification | Which tax category and exemptions apply |
| Contract | Settlement Statement Profile | How to lay out the statement |

---

### Tax Processing Allowance / Royalty Processing Allowance

**EnerFusion Schema:** `valuation.tax_processing_allowances`, `valuation.royalty_processing_allowances`

State-specific and lease-specific allowances that reduce the taxable or royalty base for processing costs. Configured in EnerFusion by the tax / compliance team based on state rules and lease instrument terms.

| Record Type | What It Reduces | Example |
|------------|----------------|---------|
| Tax Processing Allowance | Severance tax base | Wyoming processing cost deduction |
| Royalty Processing Allowance | Royalty calculation base | Lease allows processing cost deduction before royalty |
| Marketing Cost Allowance | Reportable taxable value | State-specific allowable marketing cost limit |

---

### Settlement Statement / DOI Cross-Reference

**EnerFusion Schema:** `valuation.ss_doi_xref`

Links a settlement statement to the specific DOI owner interests that receive a distribution share from it. This cross-reference drives M6 revenue distribution. Entirely EnerFusion-native.

---

## M5 — Balancing

### Product Balancing Agreement (PBA)

**EnerFusion Schema:** `balancing.product_balancing_agreements`

*(Also referenced in Locally Managed — PBAs are locally managed because EnerFusion is the only system managing them.)*

| Attribute | Description |
|-----------|-------------|
| Tolerance band | Allowed imbalance before PBA triggers |
| Make-up cap | Maximum monthly make-up volume per party |
| Cash-out rate | Price at which balances are settled in cash at termination |

---

## M7 — Payments

### Check Layout Template

**EnerFusion Schema:** `payments.check_layouts`

Configures the physical layout of owner payment checks — field positions, font, logo placement, stub format. Generated as PDF via a templating engine. No SAP equivalent for PRA check output.

---

### Escheat / Escrow Configuration

**EnerFusion Schema:** `payments.escheat_configs`

Per-state configuration for unclaimed property reporting thresholds, dormancy periods, and report format requirements. EnerFusion tracks the aging of unclaimed owner payments and generates state-specific escheat filings when dormancy thresholds are exceeded.

| Attribute | Description |
|-----------|-------------|
| Escheat state | The owner's state of domicile for reporting purposes |
| Dormancy period | Number of years before funds are reportable (varies by state) |
| Reporting threshold | Minimum balance subject to escheat |
| Filing format | State-specific file format (varies) |

---

## M8 — Regulatory

### Statutory Report Template

**EnerFusion Schema:** `regulatory.report_templates`

EnerFusion maintains the field mapping, format rules, and validation logic for each state/federal statutory report. These templates are maintained by the EnerFusion product team and updated when regulatory requirements change.

| Template | Jurisdiction | Format |
|----------|-------------|--------|
| Form H-10 | Texas RRC | Fixed-width text |
| ONRR-2014 | Federal ONRR | PAAS electronic format |
| Form 1004 | Oklahoma OTC | CSV / XML |
| Report C115 | New Mexico OCD | Fixed-width |
| OGP/R5D | Louisiana DNR | XML |

---

## M9 — Administration

### Data Archive Policy

**EnerFusion Schema:** `admin_ilm.archive_policies`

Configures which transaction types are eligible for archiving, the minimum age before archiving, and the target archive tier (Azure Blob Storage — Cool / Archive). ILM archiving is an EnerFusion-native capability with no SAP equivalent in this context.

---

### Personal Data Protection Record

**EnerFusion Schema:** `admin_ilm.personal_data_records`

Tracks personal data objects (owner mailing addresses, tax IDs, bank details) across EnerFusion schemas. Used to support right-to-be-forgotten (GDPR/CCPA-aligned) workflows:

1. Identify all records containing the business partner's personal data
2. Mark as blocked (prevents new transactions)
3. Anonymize non-transaction-critical fields after applicable retention period
4. Log anonymization event in the audit trail

---

## Summary

| Module | EnerFusion-Only Objects |
|--------|------------------------|
| M1 | Tank strapping tables, delivery network automation rulesets |
| M2 | Unit/tract participation records |
| M3 | Gas plant processing configs, marketing group imbalances |
| M4 | Formula pricing rules, VCR, tax/royalty processing allowances, SS/DOI xref |
| M5 | Product Balancing Agreements |
| M7 | Check layout templates, escheat/escrow configs |
| M8 | Statutory report templates (10+ state + federal formats) |
| M9 | Data archive policies, personal data protection records |
