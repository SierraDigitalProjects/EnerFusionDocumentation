# Data Architecture

## Entity Relationship Overview

The following summarises the principal master data entities across modules and how they relate to each other. Cross-module relationships are navigated via API, not direct foreign keys.

---

## Cross-Module Entity Flow

```
shared.business_partners (replicated from SAP)
  │
  ├── ownership.pra_leases
  │     └── ownership.ventures
  │           └── ownership.doi_owner_interests
  │                 └── ownership.doi_mp_xref
  │                       │
  │                       ▼
  │              production.measurement_points
  │                       │
  │                       ▼
  │              production.well_completions
  │                       │
  │                       ▼
  │              production.wc_volumes (daily — allocation output)
  │                       │
  │                       ▼
  │              allocation.contractual_allocations
  │                       │
  │                       ▼
  │              valuation.settlement_statements
  │                       │
  │           ┌───────────┴────────────────────┐
  │           ▼                                ▼
  │    revenue.distributions          balancing.owner_positions
  │           │
  │           ▼
  │    payments.owner_payments
  │           │
  │           ▼
  │    regulatory.report_data (period snapshot)
```

---

## Schema Inventory by Module

### `shared` Schema

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `business_partners` | Replicated BP master (SAP source) | `id`, `legal_name`, `tax_id` (encrypted), `cdex_id` |
| `gl_accounts` | Replicated GL account master | `account_number`, `description`, `account_type` |
| `cost_centers` | Replicated cost center master | `cost_center_id`, `company_code`, `valid_from/to` |
| `integration_sync_log` | SAP → EnerFusion replication audit | `source_system`, `record_id`, `changed_fields`, `synced_at` |

---

### `production` Schema (M1)

| Table | Type | Est. Row Count |
|-------|------|----------------|
| `fields` | Master | 100s |
| `reservoirs` | Master | 100s |
| `platforms` | Master | 100s |
| `wells` | Master | 1k – 50k |
| `well_completions` | Master (date-effective) | 1k – 500k |
| `measurement_points` | Master | 500 – 10k |
| `tank_strapping_records` | Master (versioned) | 1k – 50k |
| `delivery_networks` | Master | 10s – 1k |
| `wc_volumes_daily` | Transaction | 100M+ (partitioned by month) |
| `mp_volumes_daily` | Transaction | 10M+ (partitioned by month) |
| `downtime_summaries` | Transaction | 10M+ |
| `well_test_records` | Transaction | 1M+ |
| `allocation_runs` | Transaction | 10k+ |

**Partitioning:** `wc_volumes_daily` and `mp_volumes_daily` are partitioned by `production_date` (monthly range partitions) to keep query plans efficient for period-based reporting.

```sql
CREATE TABLE production.wc_volumes_daily (
  wc_volume_id UUID NOT NULL,
  well_completion_id UUID NOT NULL,
  production_date DATE NOT NULL,
  reporting_period CHAR(7) NOT NULL,   -- 'YYYY-MM'
  allocated_oil_bbl NUMERIC(12,3),
  allocated_gas_mcf NUMERIC(12,3),
  allocated_water_bbl NUMERIC(12,3),
  allocation_status VARCHAR(20) NOT NULL,
  allocation_run_id UUID NOT NULL,
  company_code VARCHAR(10) NOT NULL
) PARTITION BY RANGE (production_date);

-- Monthly partitions created by automation
CREATE TABLE production.wc_volumes_2024_01
  PARTITION OF production.wc_volumes_daily
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

---

### `ownership` Schema (M2)

| Table | Type | Est. Row Count |
|-------|------|----------------|
| `pra_leases` | Master | 100s – 10k |
| `owner_attribute_groups` | Master | 10s – 100s |
| `ventures` | Master (date-effective) | 1k – 50k |
| `doi_owner_interests` | Master (date-effective) | 10k – 2M |
| `unit_tracts` | Master | 100s – 10k |
| `doi_mp_xref` | Master | 10k – 500k |
| `doi_checkout_records` | Transaction (workflow) | 1k – 100k |
| `ownership_transfers` | Transaction | 1k – 100k |
| `chain_of_title` | Immutable log | Unbounded (append-only) |

**Constraint Example:**

```sql
-- Decimal interest integrity enforced at DB level
CREATE OR REPLACE FUNCTION ownership.check_doi_sum()
RETURNS TRIGGER AS $$
DECLARE
  total NUMERIC;
BEGIN
  SELECT SUM(decimal_interest) INTO total
  FROM ownership.doi_owner_interests
  WHERE venture_id = NEW.venture_id
    AND interest_type = NEW.interest_type
    AND effective_date = NEW.effective_date
    AND (expiration_date IS NULL OR expiration_date > CURRENT_DATE);

  IF ABS(total - 1.0) > 0.00000001 THEN
    RAISE EXCEPTION 'DOI interests do not sum to 1.0 (sum: %)', total;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

### `valuation` Schema (M4)

| Table | Type | Est. Row Count |
|-------|------|----------------|
| `contracts` | Master | 100 – 5k |
| `fixed_price_conditions` | Master | 1k – 50k |
| `formula_pricing_rules` | Master | 10s – 100s |
| `formula_steps` | Master | 100s – 1k |
| `api_gravity_scales` | Master | 10s – 100s |
| `state_tax_rates` | Master | 100s |
| `tax_classifications` | Master | 1k – 50k |
| `tier_taxes` | Master | 10s – 100s |
| `tax_processing_allowances` | Master | 100s |
| `royalty_processing_allowances` | Master | 1k – 50k |
| `vcr` (valuation cross-reference) | Master | 1k – 100k |
| `settlement_statements` | Transaction | 10k – 1M |
| `run_statements` | Transaction | 10k – 1M |
| `ppn_events` | Transaction | 1k – 100k |

---

## Encryption

| Data Element | Schema.Table | Encryption Method |
|-------------|-------------|-------------------|
| Business partner tax ID (EIN/SSN) | `shared.business_partners.tax_id` | AES-256, application-layer (envelope encryption via Azure Key Vault) |
| Business partner bank account | `shared.business_partners.bank_account` | AES-256, application-layer |
| Owner payment address | `ownership.doi_owner_interests.payment_address` | AES-256, application-layer |
| All data at rest | All tables | Azure PostgreSQL Flexible Server CMK (Azure Key Vault) |

---

## Data Retention and Archiving

| Module | Transaction Type | Hot Retention | Archive After | Delete After |
|--------|-----------------|---------------|--------------|-------------|
| M1 | Daily volumes | 2 years | 2 years (Blob Cool) | 7 years |
| M2 | Ownership transfers | 7 years (active) | After 7 years | Never (chain of title) |
| M4 | Settlement statements | 2 years | 2 years (Blob Cool) | 7 years |
| M7 | Check records | 3 years | 3 years (Blob Cool) | 7 years |
| M8 | Regulatory reports | 7 years | 7 years (Blob Archive) | Per state |
| All | Audit logs | 3 years | 3 years (Blob Cool) | 7 years |

Archive jobs run in M9 (svc-admin) on a configurable schedule, typically monthly. Archived records remain queryable via Azure Blob Storage with a pre-signed URL download available from the Admin UI.

---

## Index Strategy

High-volume tables have composite indexes on the most common query patterns:

```sql
-- M1: Volume queries by well + period (most common access pattern)
CREATE INDEX idx_wc_volumes_wc_period
  ON production.wc_volumes_daily (well_completion_id, reporting_period);

-- M2: DOI lookup by venture + effective date
CREATE INDEX idx_doi_venture_effective
  ON ownership.doi_owner_interests (venture_id, effective_date, interest_type);

-- M4: Settlement lookup by contract + period
CREATE INDEX idx_settlements_contract_period
  ON valuation.settlement_statements (contract_id, period, statement_status);

-- M7: AR aging by purchaser + posted date
CREATE INDEX idx_ar_purchaser_date
  ON payments.ar_records (purchaser_id, posted_date, ar_status);
```

---

## Backup and Recovery

| Tier | Configuration | RPO | RTO |
|------|--------------|-----|-----|
| Azure PostgreSQL automated backup | Daily full + continuous WAL | 5 minutes | < 4 hours |
| Point-in-time restore | Available for 35 days | — | < 2 hours |
| Geo-redundant backup | Cross-region backup copy | — | < 8 hours (DR scenario) |
| Azure Blob Storage (archived records) | LRS + geo-redundant copy | — | Minutes (restore from URI) |
