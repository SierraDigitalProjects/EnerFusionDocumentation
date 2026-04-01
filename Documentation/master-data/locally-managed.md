# Locally Managed in EnerFusion

## Definition

**Locally Managed** objects are master data records where **EnerFusion Upstream Accounting is the authoritative system of record**. These records are created, updated, and approved within EnerFusion. SAP or other adjacent systems may receive read-only copies for reference, but EnerFusion governs the lifecycle of these objects.

This category covers the platform-specific domain that SAP ERP does not natively support at the required depth, or where the operator has chosen EnerFusion as the replacement for SAP IS-Oil functionality.

---

## M1 — Production Master Data

### Well & Well Completion

**EnerFusion Schema:** `production.wells`, `production.well_completions`

Wells are the fundamental production asset. Well Completions (WC) represent a producing zone within a well and carry the attributes needed for volume allocation.

| Object | Managed By | Key Operations |
|--------|-----------|----------------|
| Field | Production Engineer / Admin | Create, update geographic grouping |
| Reservoir | Geoscience / Admin | Attach to field; geological classification |
| Platform | Operations / Admin | Offshore structure; attach to field |
| Well | Production Engineer | Register via API number; assign to platform |
| Well Completion | Production Engineer | Define producing zone, method, product type |

**CRUD Scale:** Up to 50,000 well completions per operator in large basins. List views require server-side pagination; bulk import via Excel template supported.

**Key Governance Rules:**
- API number uniqueness is enforced system-wide
- A well completion cannot be deleted if it has associated volume transactions
- `effectiveDate` is required — completions are date-effective records
- `producingMethod` and `productTypeCode` drive theoretical calculation logic

---

### Measurement Point (MP)

**EnerFusion Schema:** `production.measurement_points`

Measurement points are physical metering locations (meters, tank batteries, flare points, fuel meters) within a delivery network. Each MP captures gross and net volumes that feed the allocation algorithm.

| Attribute Category | Who Manages | Notes |
|-------------------|-------------|-------|
| MP type, DN assignment | Operations / Admin | Setup only; infrequent changes |
| Meter specifications | Metering Engineer | JSONB; calibration data |
| Tank strapping table | Metering Engineer | Gauge-to-volume lookup; versioned by calibration date |
| Heating value (BTU) | Gas Measurement Team | Used in gas volume corrections |

**CRUD Scale:** 500–10,000 MPs per operator. Tank strapping tables are versioned — each calibration creates a new record.

---

### Delivery Network (DN)

**EnerFusion Schema:** `production.delivery_networks`

Delivery networks define the topology of how wells connect to sales points. The DN record controls allocation tolerance thresholds and the allocation method applied to all wells in the network.

| Attribute | Managed By | Impact |
|-----------|-----------|--------|
| `allocationTolerance` | Operations Manager | Determines when allocation is auto-approved vs. escalated |
| `allocationMethod` | Operations Manager | Theoretical / historical / flat rate |
| `downstreamNodeId` | Admin | Chained network configuration |
| `measurementGroupId` | Measurement Team | Batch grouping of MPs |

---

## M2 — Ownership Master Data

### Upstream Lease Identifier

**EnerFusion Schema:** `ownership.pra_leases`

The upstream lease record is EnerFusion's representation of the legal lease instrument. It links regulatory lease numbers to the operator, royalty rate, and geographic description. Data originates from the land department.

| Attribute | Source | Notes |
|-----------|--------|-------|
| Lease number | State regulatory agency | Must match statutory filing identifier |
| Legal description | Land Department | Metes and bounds or section/township/range |
| Royalty rate | Lease agreement | Overridden at DOI level for non-standard arrangements |
| Operator | Land Department | FK to Business Partner (centrally managed) |

---

### Venture / Base Division of Interest (DOI)

**EnerFusion Schema:** `ownership.ventures`, `ownership.doi_owner_interests`

The venture is the pooling unit that links leases to owners. DOI owner interest records within a venture define the decimal interest for each owner.

**Lifecycle:**
1. Land Admin creates venture and attaches lease
2. Land Admin creates DOI owner interest records (WI, RI, ORRI)
3. DOI validation confirms interests sum to 1.0
4. Venture is checked out → modified → submitted for approval → checked in
5. Effective date activates new ownership split

**CRUD Scale:** 10,000–500,000 DOI owner interest records in large operators. Bulk DOI upload via Excel template is critical for initial data load and large-scale interest changes.

**Governance:**
- Only one checkout per venture at any time (conflict lock)
- Decimal interests must sum to exactly 1.0 (±0.00000001) per interest type per period
- Transfers create immutable chain-of-title records — soft delete only

---

### Owner Attribute Group

**EnerFusion Schema:** `ownership.owner_attribute_groups`

Groups of owners sharing common payment attributes. Assigned to individual DOI owner interests. Controls payment method, minimum payment threshold, and tax withholding rates.

| Attribute | Description | Impact |
|-----------|-------------|--------|
| Payment method | Check / ACH / Wire | Routes payment through appropriate disbursement path |
| Payment threshold | Minimum disbursement amount | Amounts below threshold are suspended until cumulative minimum is reached |
| State withholding % | State income tax rate | Applied during M6 distribution calculation |
| Federal withholding % | Backup withholding | Applied for owners without valid W-9 on file |

---

## M3 — Contractual Allocation Master Data

### Contract ↔ Measurement Point Cross-Reference

**EnerFusion Schema:** `allocation.contract_mp_xref`

Maps physical measurement points to sales contracts. Determines which contract receives production from each MP for allocation purposes.

---

### Nomination Records

**EnerFusion Schema:** `allocation.nominations`

Daily or monthly volume nominations submitted by owners or purchasers. Drive entitlement calculations in gas allocation scenarios.

| Attribute | Description |
|-----------|-------------|
| Owner / purchaser | Who nominated |
| Nominated volume | MCF or BBL per day |
| Effective date | Nomination period |
| Contract reference | Which contract the nomination applies to |

---

## M4 — Sales Contracts and Pricing

### Sales Contract

**EnerFusion Schema:** `valuation.contracts`

The primary commercial agreement between operator (seller) and purchaser (buyer). EnerFusion manages the complete PRA contract lifecycle — from draft through approval and expiration — independently of SAP SD contracts.

| Status | Description |
|--------|-------------|
| Draft | Under review; not yet effective |
| Approved | Authorized for use in valuation |
| Active | Currently pricing transactions |
| Expired | Past expiration date; read-only |

**CRUD Operations:**
- Contract Manager creates and edits draft contracts
- Valuation Approver approves contracts before the effective period
- Contracts cannot be deleted once they have associated settlement statements

---

### Fixed Price Conditions

**EnerFusion Schema:** `valuation.fixed_price_conditions`

Time-effective fixed prices per product type (oil, gas, NGL, ethane, propane, etc.). Multiple conditions can be active for a single contract across different date ranges.

---

### API Gravity Scale

**EnerFusion Schema:** `valuation.api_gravity_scales`

Crude oil pricing differential tables. Configured once and referenced by multiple contracts that price based on crude quality. Adjustment is applied per degree API gravity above or below the base gravity point.

---

### Purchaser-Remitter Cross-Reference

**EnerFusion Schema:** `payments.purchaser_remitter_xref`

Maps purchaser Business Partner IDs to the check remitter information used in CDEX/EDI 820 check matching. Required for automated check input in M7.

---

## M5 — Product Balancing Agreement (PBA)

**EnerFusion Schema:** `balancing.product_balancing_agreements`

PBAs define the terms under which owners with cumulative imbalances can take make-up volumes or settle balances. EnerFusion is the sole system of record for PBAs — there is no SAP equivalent.

| Attribute | Description |
|-----------|-------------|
| PBA parties | Owner interests bound by the agreement |
| Product type | Oil / gas / NGL |
| Make-up volume cap | Maximum monthly make-up allowed |
| Tolerance band | Allowable imbalance before PBA triggers |
| Settlement terms | In-kind or cash at termination |

---

## Governance Principles for Locally Managed Data

1. **Effective dating** — All locally managed master records use `effectiveDate` / `expirationDate` pairs. No hard deletes for records with associated transactions.
2. **Approval workflows** — High-impact records (DOI changes, contract approvals, ownership transfers) require a two-step create → approve workflow.
3. **Audit logging** — All mutations are written to `admin_ilm.audit_log` (who, what, when, before/after values).
4. **Bulk operations** — Objects with CRUD scale exceeding 1,000 records must support Excel template import and async background processing with progress tracking.
5. **RBAC enforcement** — Every CRUD operation is gated by a module-level permission check in the API middleware layer.
