# End-to-End Test Cases — "Build-A-Biz" Scenario

---

## 1. Scenario Overview

The **Build-A-Biz** scenario is the primary end-to-end acceptance test for EnerFusion. It traces a fictitious oil and gas company from initial formation through two full months of operated production, ownership changes, prior-period adjustments, inbound royalty income (check input), and outside-operated property setup.

The scenario is divided into two phases that mirror the EnerFusion delivery phases:

| Phase | Coverage | Months Simulated |
|-------|----------|-----------------|
| **Phase 1** | Company formation, master data, first month operated cycle (M1 → M7) | Month 1 |
| **Phase 2** | Balancing, ownership transfers, prior-period adjustments, check input, outside-operated | Month 2 + retrospective |

### Business Narrative — "The Biz"

A new oil and gas company is formed with a 50/50 Joint Venture partner. Two wells are drilled:

- **Well 1** — Oil and gas producer; 50/50 working interest split with JV partner; 1/8 royalty interest owed to a landowner (Farmer Ted); burden shared pro-rata between the two WI owners
- **Well 2** — Oil and gas producer; same 50/50 WI split; additionally carries a 2% override royalty (ORRI) owed to a neighboring landowner

Both wells produce into separate Oil and Gas Delivery Networks. Gas is sold under individual WI-owner contracts; oil is marketed under a single Division Order sales arrangement. In Month 2, Farmer Ted transfers his royalty interest to his two sons; one son has no valid address and must be suspended. A prior-period price correction triggers Prior Period Adjustments (PPAs). The company also begins receiving royalty income checks from the JV partner for a non-operated interest.

---

## 2. Test Case Conventions

| Field | Convention |
|-------|-----------|
| **TC-ID** | `E2E-Pn-Mxx-nnn` — Phase, Module, sequence |
| **Module** | EnerFusion module code (M1–M9) |
| **Type** | `Happy Path` \| `Negative` \| `Edge Case` |
| **Priority** | `P1 — Blocker` \| `P2 — High` \| `P3 — Medium` |
| **Auth Scope** | API scope required to execute |

---

## 3. Phase 1 Test Cases — Company Formation & First Month Operations

### 3.1 Master Data & Company Setup

#### TC-E2E-P1-M2-001 — Create Business Partners (Customers / Vendors)

| Field | Value |
|-------|-------|
| Module | M2 — Ownership & DOI |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `ownership:write` |

**Preconditions:**
- No existing business partners with test company names
- Azure AD user with `ownership:write` scope

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/business-partners` with `partnerType: customer`, `accountGroup: EP01`, `companyCode: <test>`, `intercompanyCode: <affiliate code>` for the Oil & Gas Affiliate | HTTP 201; `businessPartnerId` returned |
| 2 | Repeat for: Oil Transportation Affiliate, Oil Marketing Affiliate (with sales org), Gas Marketing Affiliate (with sales org), Gas Pipeline Affiliate | HTTP 201 each; all `isAffiliate: true` |
| 3 | `POST /api/v1/business-partners` for JV Partner — `intercompanyCode: null`, `accountGroup: EP01` | HTTP 201; `isAffiliate: false` |
| 4 | Create vendors: Oil & Gas Affiliate, JV Partner, Royalty Owner 1 (Farmer Ted), Override Royalty Owner 1 | HTTP 201 each |
| 5 | For Royalty Owner 1: set `paymentMethod: CHECK`, `ownerType: RI` | `paymentMethod` and `ownerType` persisted |
| 6 | For Override Royalty Owner 1: set `ownerType: ORRI` | `ownerType: ORRI` persisted |
| 7 | `GET /api/v1/business-partners?accountGroup=EP01` | All 10 partners returned in list |

**Expected Final State:** 10 business partners active; affiliates have intercompany codes; royalty owners have correct owner types and payment methods.

---

#### TC-E2E-P1-M2-002 — Create JV Partner and JOA Structure

| Field | Value |
|-------|-------|
| Module | M2 — Ownership & DOI |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `ownership:write` |

**Preconditions:** JV Partner business partner created (TC-E2E-P1-M2-001)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/ventures` with `ventureType: JV`, `equityPercentage: 50.0` linking to JV Partner BP | HTTP 201; `ventureId` returned |
| 2 | `POST /api/v1/ventures/{id}/cost-centers` to assign a cost center | Cost center linked to venture |
| 3 | `GET /api/v1/ventures/{id}` | Venture shows correct JV structure and cost center |

---

#### TC-E2E-P1-M1-001 — Create Well and Production Hierarchy

| Field | Value |
|-------|-------|
| Module | M1 — Production & Volume Allocation |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `production:write` |

**Preconditions:** Company code and cost center configured

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/production/fields` — create Field | HTTP 201; `fieldId` returned |
| 2 | `POST /api/v1/production/reservoirs` — create Reservoir linked to Field | HTTP 201 |
| 3 | `POST /api/v1/production/wells` — create Well 1 with unique API number | HTTP 201; `wellId` returned |
| 4 | Repeat for Well 2 with different API number | HTTP 201 |
| 5 | `POST /api/v1/production/well-completions` — create WC1 linked to Well 1, Field, Reservoir, UOM Group | HTTP 201; `wcId` returned |
| 6 | Repeat for WC2 linked to Well 2 | HTTP 201 |
| 7 | `GET /api/v1/production/well-completions` | Both WCs returned with correct field/reservoir links |

**Negative Test (TC-E2E-P1-M1-001N):** Attempt to create a WC with a duplicate API number → HTTP 409 Conflict.

---

#### TC-E2E-P1-M2-003 — Create Venture, Base DOIs, and Division Orders

| Field | Value |
|-------|-------|
| Module | M2 — Ownership & DOI |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `ownership:write` |

**Preconditions:** Business partners and WCs created (TC-E2E-P1-M2-001, TC-E2E-P1-M1-001)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/ownership/ventures/{id}/doi` — create Base DOI for WC1 | HTTP 201; `doiId` returned |
| 2 | Add owner interests for WC1 DOI: `WI: 0.50000000` (Our Company), `WI: 0.50000000` (JV Partner), `RI: 0.12500000` (Farmer Ted) | All interests created |
| 3 | Verify DOI balance: `GET /api/v1/ownership/doi/{id}/validation` | `balanced: true`; NRI sum = 0.875 after RI burden |
| 4 | Create Base DOI for WC2; add same WI split plus `ORRI: 0.02000000` (Neighbor) | HTTP 201 |
| 5 | Verify WC2 DOI balance | `balanced: true` |
| 6 | `POST /api/v1/ownership/doi/{id}/approve` for both DOIs | `doiStatus: approved` |
| 7 | `POST /api/v1/ownership/doi/{id}/check-in` | `doiStatus: checked_in`; effective date set |
| 8 | Configure DOI Accounting rules: payment method, minimum threshold, withholding | Configuration persisted |
| 9 | `GET /api/v1/ownership/doi/{id}/owners` | Returns 3 owners for WC1, 4 owners for WC2 |

**Negative Test (TC-E2E-P1-M2-003N):** Attempt to add a WI interest that causes DOI to exceed 1.0 → HTTP 422 with validation message "Decimal interests would exceed 1.0".

---

#### TC-E2E-P1-M1-002 — Create Delivery Networks (Oil and Gas)

| Field | Value |
|-------|-------|
| Module | M1 — Production & Volume Allocation |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `production:write` |

**Preconditions:** WCs created (TC-E2E-P1-M1-001)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Create Measurement Points: Tank Lease, Tank Sales, Gas Sales Meter | Three MPs created |
| 2 | Define tank strapping characteristics for Tank Lease MP | Strapping table linked |
| 3 | Create Oil Delivery Network; assign allocation method `division_order_sales`; link WCs to Tank MPs | Oil DN created |
| 4 | Create Gas Delivery Network; assign allocation method `sales_point_originated`; link WCs to Gas Meter MP | Gas DN created |
| 5 | Define Volume Types per DN: Oil → `inventory`, `sales`; Gas → `fuel_use`, `sales` | VTs configured |
| 6 | Associate transporter/purchaser customers to sales MPs | Customer links persisted |
| 7 | `GET /api/v1/production/delivery-networks` | Both DNs returned with correct WC links and MPs |

---

#### TC-E2E-P1-M4-001 — Create Contracts, Pricing Formulas, and VCRs

| Field | Value |
|-------|-------|
| Module | M4 — Contracts, Pricing & Valuation |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `valuation:write` |

**Preconditions:** Business partners, DNs, and DOIs created

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/contracts` — Oil Sales contract linked to oil purchaser customer and Tank Sales MP | HTTP 201 |
| 2 | Create Gas Sales contract for Our Company WI (marketed gas) | HTTP 201 |
| 3 | Create Gas contract for JV Partner with `paymentType: TAKE_IN_KIND` | `paymentType: TIK` persisted |
| 4 | Create Valuation Formula for Oil: formula type `posted_price`, gravity scale reference | Formula created |
| 5 | Create Valuation Formula for Gas: includes compression marketing deduction | Formula created |
| 6 | Create VCR entries linking each contract+MP to the appropriate formula | VCRs created |
| 7 | `POST /api/v1/contracts/{id}/marketing-deduct-rates` — compression rate for gas | Rate persisted |
| 8 | `GET /api/v1/contracts` | All contracts returned |

---

### 3.2 First Month Operations — Production

#### TC-E2E-P1-M1-003 — Record Production Measurements

| Field | Value |
|-------|-------|
| Module | M1 — Production & Volume Allocation |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `production:write` |

**Preconditions:** DNs and volume types configured (TC-E2E-P1-M1-002)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/production/well-tests` for WC1 and WC2 — enter oil rate, GOR, water cut | Well tests recorded |
| 2 | `POST /api/v1/production/downtime` for any periods of downtime | Downtime records persisted; uptime fraction calculated |
| 3 | `POST /api/v1/production/tank-gauges` — opening and closing gauge readings for Tank Lease | Gauge readings recorded; gross volume calculated |
| 4 | `POST /api/v1/production/tank-gauges` — Tank Sales readings | Gauge readings recorded |
| 5 | `POST /api/v1/production/meter-volumes` — gas sales meter readings (with seal identifier) | Gas volumes recorded; transformed to standard conditions |
| 6 | `GET /api/v1/production/measurements?period=<month1>` | All measurements returned for the period |

---

#### TC-E2E-P1-M1-004 — Run Production Volume Allocation

| Field | Value |
|-------|-------|
| Module | M1 — Production & Volume Allocation |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `production:execute` |

**Preconditions:** All measurements recorded for Month 1 (TC-E2E-P1-M1-003)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/production/allocation-runs` — trigger allocation for Oil DN, period Month 1 | `allocationRunId` returned; status `running` |
| 2 | Poll `GET /api/v1/production/allocation-runs/{id}` | Status transitions to `completed` |
| 3 | Repeat for Gas DN | `completed` |
| 4 | `GET /api/v1/production/wc-volumes?wcId={wc1}&period=<month1>` | WC1 shows allocated oil and gas volumes by VT |
| 5 | Verify: oil volumes balance to tank measurements; gas volumes balance to meter | `allocationBalanced: true` |
| 6 | `GET /api/v1/production/allocation-runs/{id}/discrepancy` | Discrepancy report: 0% variance (within tolerance) |

**Negative Test (TC-E2E-P1-M1-004N):** Trigger allocation when measurements are incomplete → HTTP 422 "Allocation blocked: missing meter readings for Gas DN".

---

### 3.3 First Month Operations — Contractual Allocation

#### TC-E2E-P1-M3-001 — Generate and Enter Contract Sales Template

| Field | Value |
|-------|-------|
| Module | M3 — Contractual Allocation |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `allocation:write` |

**Preconditions:** Production allocation completed (TC-E2E-P1-M1-004)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/allocation/templates/generate` — generate contract sales template for both DNs, Month 1 | Template rows created for each contract × sales point |
| 2 | `PUT /api/v1/allocation/templates/{id}/actuals` — enter gas pipeline actuals for each contract | Actuals recorded |
| 3 | `POST /api/v1/allocation/runs` — run Contractual Allocation for Gas DN | `allocationRunId` returned |
| 4 | Poll until `completed` | Status `completed` |
| 5 | Verify: JV Partner's gas volume shows `paymentType: TIK`; no cash value generated | `netValue: 0` for TIK contract |
| 6 | Repeat CA run for Oil DN | `completed` |
| 7 | `GET /api/v1/allocation/results?ventureId={id}&period=<month1>` | Owner-level allocated volumes returned for all contracts |

---

### 3.4 First Month Operations — Valuation

#### TC-E2E-P1-M4-002 — Enter Prices and Run Valuation

| Field | Value |
|-------|-------|
| Module | M4 — Contracts, Pricing & Valuation |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `valuation:execute` |

**Preconditions:** CA completed (TC-E2E-P1-M3-001)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/valuation/posted-prices` — enter oil posted price for Month 1 ($/BBL) | Price recorded |
| 2 | `POST /api/v1/valuation/formula-prices` — enter gas spot price (fixed) | Price recorded |
| 3 | `POST /api/v1/valuation/runs` — trigger valuation for Oil DN, Month 1, `draftMode: true` | `valuationRunId` returned; status `running` |
| 4 | Poll until `completed` | Status `completed` |
| 5 | `GET /api/v1/valuation/runs/{id}/settlement-statements` | Draft settlement statements returned with gross value, deductions, net value |
| 6 | Verify: oil gravity differential applied; compression deduction calculated for gas | Deductions match formula configurations |
| 7 | Verify: severance tax calculated per state rate | Tax amounts populated |
| 8 | `POST /api/v1/valuation/runs/{id}/post` — post the Oil DN valuation | `statementStatus: posted`; GL entries generated |
| 9 | Repeat steps 3–8 for Gas DN | Gas statements posted |
| 10 | `GET /api/v1/valuation/runs/{id}/accounting-detail` | GL debit/credit entries visible |

**Negative Test (TC-E2E-P1-M4-002N):** Post valuation when oil price is not entered → HTTP 422 "Valuation blocked: no price found for product OIL, period Month 1".

---

### 3.5 First Month Operations — Revenue Distribution

#### TC-E2E-P1-M6-001 — Run Revenue Distribution

| Field | Value |
|-------|-------|
| Module | M6 — Revenue Distribution & Accounting |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `revenue:execute` |

**Preconditions:** Valuation statements posted (TC-E2E-P1-M4-002)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/revenue/distribution/run` — trigger distribution run for Month 1 | `distributionRunId` returned |
| 2 | Poll until `completed` | `runStatus: completed` |
| 3 | `GET /api/v1/revenue/distribution/runs/{id}/owners` | Owner distribution records returned for all owners |
| 4 | Verify WC1: Our Company WI, JV Partner WI, Farmer Ted RI distributions present | 3 distributions for WC1 |
| 5 | Verify WC2: 4 distributions including Neighbor ORRI | 4 distributions for WC2 |
| 6 | Verify JV Partner TIK: `netValue: 0` | TIK owner not paid |
| 7 | Verify Farmer Ted: `grossValue = settlement.grossValue × 0.125`; severance tax correctly shared | Values match formula |
| 8 | `GET /api/v1/revenue/documents` — list RADs | RADs generated for the run |
| 9 | `POST /api/v1/revenue/documents/{id}/gl-posting` — submit RAD to GL | `postingStatus: posted`; `externalDocumentNumber` returned from SAP FI |

**Edge Case (TC-E2E-P1-M6-001E):** Farmer Ted has no payment method configured → distribution record shows `suspenseFlag: true`, `suspenseReason: "No valid payment method"`.

---

### 3.6 First Month Operations — Payment Processing

#### TC-E2E-P1-M7-001 — Execute Payment Run

| Field | Value |
|-------|-------|
| Module | M7 — Payment Processing & Check Input |
| Type | Happy Path |
| Priority | P2 — High |
| Auth Scope | `payments:execute` |

**Preconditions:** Revenue distribution completed (TC-E2E-P1-M6-001); Owner Attribute Groups configured with minimum threshold

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/payments/runs` — trigger payment run, scope: all ventures, Month 1 | `paymentRunId` returned |
| 2 | `GET /api/v1/payments/runs/{id}` — validate step | `runStep: validate`; validation errors list empty |
| 3 | Advance to extract step | All owners with payable distributions extracted |
| 4 | `GET /api/v1/payments/runs/{id}/variance` — variance analysis | Prior period comparison shown; Month 1 has no prior so baseline established |
| 5 | Advance to journalize | GL payment entries created |
| 6 | Advance to finalize | Owner payment records locked |
| 7 | Advance to output | Check file or EFT file generated |
| 8 | `GET /api/v1/payments/runs/{id}/owner-payments` | Payment records with `paymentStatus: issued` |
| 9 | Verify: Farmer Ted's distribution in suspense → `paymentStatus: suspended` | Suspended distribution not in payment file |

**Negative Test (TC-E2E-P1-M7-001N):** Attempt to output payment run when journalize step has not completed → HTTP 409 "Payment run is not in finalizable state".

---

## 4. Phase 2 Test Cases — Balancing, Ownership Changes & Check Input

### 4.1 Balancing Agreement

#### TC-E2E-P2-M5-001 — Create and Generate PBA Statement

| Field | Value |
|-------|-------|
| Module | M5 — Balancing Workplace |
| Type | Happy Path |
| Priority | P2 — High |
| Auth Scope | `balancing:write` |

**Preconditions:** Month 1 CA completed; WCs and contracts configured

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/balancing/pbas` — create Product Balancing Agreement, linked to both WCs | `pbaId` returned |
| 2 | `POST /api/v1/balancing/pbas/{id}/entities` — assign WC1 and WC2 | Entities linked |
| 3 | `POST /api/v1/balancing/pbas/{id}/statements/generate` — generate statement for Month 1 | Statement generated with over/under positions |
| 4 | `GET /api/v1/balancing/pbas/{id}/statements/{stmtId}` | Statement detail shows cumulative over/short per owner |
| 5 | Verify: JV Partner TIK → balance position reflected (they took in-kind, no cash shortfall) | TIK balance correctly modeled |

---

### 4.2 Ownership Change — Owner Transfer

#### TC-E2E-P2-M2-001 — Transfer Royalty Interest to Two New Owners

| Field | Value |
|-------|-------|
| Module | M2 — Ownership & DOI |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `ownership:write` |

**Preconditions:** Month 1 closed; Farmer Ted (RI) active on WC1 DOI

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Create two new vendor BPs: Son 1 and Son 2 | Two new BPs created |
| 2 | Son 2: no valid mailing address — `paymentStatus: suspend`, `suspenseReason: "No valid address"` | Suspense flag set |
| 3 | `POST /api/v1/ownership/transfer-requests` — `transferType: with_funds`; source: Farmer Ted; `relativeTransfer: 100%`; effective: first day of Month 2 | `transferRequestId` returned |
| 4 | `GET /api/v1/ownership/doi/{doiId}/owners` → get current Farmer Ted interest rows; add them to transfer request | Interest rows linked |
| 5 | Add new owners to transfer: Son 1 `RI: 0.0625`, Son 2 `RI: 0.0625` (split 50/50 of 1/8) | New owner interests staged |
| 6 | `POST /api/v1/ownership/transfer-requests/{id}/release` | `requestStatus: released` |
| 7 | `POST /api/v1/ownership/transfer-requests/{id}/approve` | `requestStatus: approved` |
| 8 | `POST /api/v1/ownership/transfer-requests/{id}/execute` | Execution job started |
| 9 | Poll until `executionStatus: completed` | |
| 10 | `GET /api/v1/ownership/doi/{doiId}/owners?effectiveDate=<month2-start>` | Farmer Ted no longer active; Son 1 and Son 2 appear at 0.0625 each |
| 11 | `GET /api/v1/ownership/chain-of-title/{doiId}` | Chain of title entry shows transfer event with from/to owners and effective date |

**Negative Test (TC-E2E-P2-M2-001N):** Attempt transfer where new owner interests sum to more than Farmer Ted's original interest → HTTP 422 "New owner interests exceed transferred interest".

**Edge Case (TC-E2E-P2-M2-001E):** Son 2 is in suspense — distributions for Son 2's share should accumulate in suspense balance, not generate a payment.

---

### 4.3 Prior Period Adjustments

#### TC-E2E-P2-M4-001 — Correct DOI and Process Prior Period Notifications

| Field | Value |
|-------|-------|
| Module | M4 — Contracts, Pricing & Valuation / M2 |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `ownership:write`, `valuation:execute` |

**Preconditions:** Month 1 is closed; owner transfer executed

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Create a correction owner transfer request (type: correction, not with_funds) adjusting WC2 ORRI decimal for Month 1 | `transferRequestId` returned |
| 2 | Release, approve, and execute the correction | `executionStatus: completed` |
| 3 | `GET /api/v1/ownership/ppns?ventureId={id}&period=<month1>` | PPNs generated — one for CA, one for VL (valuation) |
| 4 | Review PPN impact: `GET /api/v1/ownership/ppns/{ppnId}/impact` | Impact detail shows affected owner distributions |
| 5 | Process CA PPN: `POST /api/v1/allocation/runs` with `ppaMode: true`, `ppnId: {caId}` | CA re-executed for Month 1 |
| 6 | Process VL PPN: `POST /api/v1/valuation/runs` with `ppaMode: true`, `ppnId: {vlId}` | Valuation re-run for Month 1 |
| 7 | `GET /api/v1/valuation/runs/{id}/je-summary` | JE summary shows delta: only affected owner and accounts changed |
| 8 | Post the PPA valuation run | `statementStatus: posted`; prior-period adjustment entries in GL |

---

#### TC-E2E-P2-M4-002 — Price Change Triggering PPN

| Field | Value |
|-------|-------|
| Module | M4 — Contracts, Pricing & Valuation |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `valuation:write`, `valuation:execute` |

**Preconditions:** Month 1 posted; gas price requires correction

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `PUT /api/v1/valuation/formula-prices/{id}` — update gas spot price for Month 1 | System detects this affects a closed period |
| 2 | Response includes `ppnGenerated: true`, `ppnId: {id}` | PPN created automatically |
| 3 | `GET /api/v1/ownership/ppns/{ppnId}/impact` | Impact detail shows gas DN affected; oil DN not impacted |
| 4 | Provide processing rationale: `PUT /api/v1/ownership/ppns/{ppnId}` with `disposition: process`, `rationale: "Corrected gas price per purchaser statement"` | PPN rationale saved |
| 5 | Process PPN through DN Workplace: `POST /api/v1/valuation/runs` with `ppaMode: true` | Re-valuation runs for gas only |
| 6 | `GET /api/v1/valuation/runs/{id}/je-detail?compare=true` | Comparison report shows line-level differences vs. original |
| 7 | Post the corrected run | `statementStatus: posted`; delta entries in GL |

**Edge Case (TC-E2E-P2-M4-002E):** Verify JV Partner TIK gas shows no cash impact (price change affects their volume position only) — `netValue: 0` on TIK lines even after price correction.

---

### 4.4 Check Input — Royalty Income from JV Partner

#### TC-E2E-P2-M7-001 — Setup Check Input Cross-References

| Field | Value |
|-------|-------|
| Module | M7 — Payment Processing & Check Input |
| Type | Happy Path |
| Priority | P2 — High |
| Auth Scope | `payments:write` |

**Preconditions:** JV Partner BP active; DOI for non-operated interest exists

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `GET /api/v1/payments/check-layouts` | Retrieve available check layout templates |
| 2 | `POST /api/v1/payments/remitter-layout-xref` — map JV Partner remitter to the default check layout | Cross-reference created |
| 3 | `POST /api/v1/payments/pdx` — create PDX record for JV Partner: link to our DOI; set `specialDistribution: true`, `distributionBasis: net_amount`, `distributeTo: our_WI_only` | PDX record created; `specialDistribution: true` |
| 4 | `GET /api/v1/payments/pdx?remitterId={jvPartnerId}` | PDX record returned with correct DOI link |

---

#### TC-E2E-P2-M7-002 — Enter and Process Inbound Check (PPD)

| Field | Value |
|-------|-------|
| Module | M7 — Payment Processing & Check Input |
| Type | Happy Path |
| Priority | P2 — High |
| Auth Scope | `payments:write`, `payments:execute` |

**Preconditions:** Check layout and PDX cross-reference configured (TC-E2E-P2-M7-001)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/payments/incoming-checks` — enter check: `remitterId: {jvPartner}`, `checkNumber`, `checkDate`, `checkAmount`, `period: <month1>` | Check record created; `checkStatus: entered` |
| 2 | `GET /api/v1/payments/incoming-checks/{id}` | Check record returned with line items per product |
| 3 | `POST /api/v1/payments/incoming-checks/{id}/validate` | Validation passes; PDX cross-reference resolved; DOI matched |
| 4 | `POST /api/v1/payments/incoming-checks/{id}/value` — run valuation against the check | Check valued; distribution lines calculated using special distribution (net basis) |
| 5 | Verify: Burdened owner (JV Partner's share of our overhead) shown with `payCode: BALANCING`, `netValue: 0` | Balancing owner receives distribution but is not paid |
| 6 | Verify: Our WI shows correct net distribution | Our interest = check net amount |
| 7 | `POST /api/v1/payments/incoming-checks/{id}/post` | `checkStatus: posted`; accounting entries committed to GL |
| 8 | `GET /api/v1/payments/incoming-checks/{id}/accounting-detail` | Debit cash / credit revenue entries visible |

**Negative Test (TC-E2E-P2-M7-002N):** Enter check with `remitterId` that has no PDX cross-reference → HTTP 422 "No PDX cross-reference found for remitter".

---

### 4.5 Outside Operated Property

#### TC-E2E-P2-M1-001 — Setup Non-Allocated Delivery Network (Outside Operated)

| Field | Value |
|-------|-------|
| Module | M1 — Production & Volume Allocation |
| Type | Happy Path |
| Priority | P3 — Medium |
| Auth Scope | `production:write` |

**Preconditions:** OO property BP (the operator) configured as a business partner

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Create a minimal DN with a single Measurement Point (our sales point only) | DN created |
| 2 | Set DN to `allocationMethod: non_allocated` | DN will accept manually entered volumes; no allocation run needed |
| 3 | `POST /api/v1/production/mp-volumes` — manually enter MP-level gas volume as reported by operator | Volumes recorded directly at MP level |
| 4 | `GET /api/v1/production/mp-volumes?period=<month1>` | Volumes returned; no WC-level allocation records |

---

#### TC-E2E-P2-M2-002 — Outside Operated DOI with Balancing Owner

| Field | Value |
|-------|-------|
| Module | M2 — Ownership & DOI |
| Type | Happy Path |
| Priority | P3 — Medium |
| Auth Scope | `ownership:write` |

**Preconditions:** OO DN created (TC-E2E-P2-M1-001)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Create generic Balancing Owner BP: `ownerType: WI`, `payCode: BALANCING` | BP created |
| 2 | Create DOI for OO venture: Our WI interest only (our burdens) + Balancing Owner to represent the 100% view | DOI created |
| 3 | Verify: `decimalInterest` of Balancing Owner fills the gap to 1.0 | DOI balanced at 1.0 |
| 4 | `POST /api/v1/valuation/runs` for OO DN using `valuationMethod: system_valued` | Valuation run triggered |
| 5 | `GET /api/v1/valuation/runs/{id}/settlement-statements` | Our WI distributions correct; Balancing Owner shows distribution but no payment |

---

## 5. Regulatory Reporting Test Cases

#### TC-E2E-P1-M8-001 — Generate ONRR-2014 Report (Federal Lease)

| Field | Value |
|-------|-------|
| Module | M8 — Regulatory, Tax & Royalty Reporting |
| Type | Happy Path |
| Priority | P2 — High |
| Auth Scope | `regulatory:execute` |

**Preconditions:** Month 1 valuation posted; venture linked to ONRR lease hierarchy; payor code configured

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/regulatory/reports/run` — `templateCode: ONRR-2014`, `period: <month1>` | `reportRunId` returned |
| 2 | Poll until `runStatus: review` | |
| 3 | `GET /api/v1/regulatory/reports/runs/{id}/lines` | TC 01 lines generated for each PPN group |
| 4 | Verify: transportation allowance present → TC 11 line generated | TC 11 present |
| 5 | Verify: processing allowance present → TC 13 line generated; not mapped to transport bucket | TC 13 present; `transactionType: processing_allowance` |
| 6 | `GET /api/v1/regulatory/reports/runs/{id}/lines?tcCode=TC15` | No TC 15 lines (production was non-zero) |
| 7 | `PUT /api/v1/regulatory/reports/runs/{id}/approve` | `runStatus: approved` |
| 8 | `POST /api/v1/regulatory/reports/runs/{id}/submit` | `runStatus: submitted`; `submissionReference` returned |

**Negative Test (TC-E2E-P1-M8-001N):** Run ONRR report when payor code is not configured → HTTP 422 "ONRR payor code not found for company code".

---

#### TC-E2E-P1-M8-002 — Generate State Severance Tax Report

| Field | Value |
|-------|-------|
| Module | M8 — Regulatory, Tax & Royalty Reporting |
| Type | Happy Path |
| Priority | P2 — High |
| Auth Scope | `regulatory:execute` |

**Preconditions:** State tax rates configured for the chosen simulation state; Month 1 valuation posted

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/regulatory/reports/run` — `templateCode: <state>-SEV`, `period: <month1>` | `reportRunId` returned |
| 2 | Poll until `runStatus: review` | |
| 3 | `GET /api/v1/regulatory/reports/runs/{id}/lines` | Lines returned: taxable volume, taxable value, tax due |
| 4 | Verify: exemptions applied where applicable (royalty exemptions, stripper well) | Exempt volumes excluded from taxable base |
| 5 | Verify: tax amounts reconcile to M4 severance tax entries | Tax amounts match valuation |
| 6 | Approve and submit | `runStatus: submitted` |

---

## 6. Administration Test Cases

#### TC-E2E-P1-M9-001 — Onboarding Validation Before Go-Live

| Field | Value |
|-------|-------|
| Module | M9 — Administration & ILM |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `admin:onboarding` |

**Preconditions:** All master data setup complete for the test venture

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `POST /api/v1/admin/onboarding/checklists` — create checklist for new venture | `checklistId` returned |
| 2 | `POST /api/v1/admin/onboarding/checklists/{id}/run` | All checks executed |
| 3 | Verify `DOI-BALANCE` passed: both DOIs sum to 1.0 | `checkStatus: passed` |
| 4 | Verify `TAX-RATE-CONFIG` passed: severance tax rates present for the simulation state | `checkStatus: passed` |
| 5 | Verify `PAY-METHOD-ALL-OWNERS` — Son 2 in suspense (no address) → check shows warning, not failure | `checkStatus: passed` with warning annotation |
| 6 | `GET /api/v1/admin/onboarding/checklists/{id}` | `checklistStatus: passed` (or failed with specific items) |

**Negative Test (TC-E2E-P1-M9-001N):** Run checklist when DOI does not sum to 1.0 → `DOI-BALANCE` check returns `failed`; `checklistStatus: failed`; venture activation blocked.

---

#### TC-E2E-P1-M9-002 — Accounting Period Close Sequence

| Field | Value |
|-------|-------|
| Module | M9 — Administration & ILM |
| Type | Happy Path |
| Priority | P1 — Blocker |
| Auth Scope | `admin:calendar` |

**Preconditions:** All Month 1 close steps completed (production, valuation, distribution, payments, regulatory)

**Test Steps:**

| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | `PUT /api/v1/admin/calendar/{companyCode}/{period}/soft-close` | `periodStatus: soft_closed`; no new postings allowed |
| 2 | Attempt to post a new valuation run for Month 1 | HTTP 409 "Period is soft-closed; postings not permitted" |
| 3 | `PUT /api/v1/admin/calendar/{companyCode}/{period}/hard-close` | `periodStatus: hard_closed` |
| 4 | Attempt to reopen the period via standard close endpoint | HTTP 403 "Hard-closed period requires admin override" |
| 5 | `GET /api/v1/admin/calendar/{companyCode}` | Month 1 shows `hard_closed`; Month 2 shows `open` |

---

## 7. Test Data Summary

| Entity | Phase 1 | Phase 2 |
|--------|---------|---------|
| Business Partners | 10 (6 customers, 4 vendors) | +2 (Son 1, Son 2) |
| Wells | 2 | — |
| Well Completions | 2 | — |
| Delivery Networks | 2 (Oil, Gas) | +1 (OO — non-allocated) |
| Contracts | 3 (Oil, Gas WI, Gas TIK) | — |
| DOIs | 2 (WC1, WC2) | +1 (OO) |
| Owner Interests | 7 (3 + 4 across 2 DOIs) | +2 (Son 1, Son 2 replace Farmer Ted) |
| Production Months | 1 | +1 (Month 2) |
| PPNs Generated | 0 | 2 (DOI correction + price change) |
| Check Input Records | 0 | 1 (JV Partner royalty check) |

---

## 8. Test Execution Order

```
Phase 1
  ├── M2: Create BPs → JV Venture
  ├── M1: Create Wells → WCs → DNs
  ├── M2: Create DOIs → Division Orders → Approve → Check-In
  ├── M4: Create Contracts → Formulas → VCRs
  ├── M1: Record Measurements → Run Production Allocation
  ├── M3: Generate Template → Enter Actuals → Run CA
  ├── M4: Enter Prices → Run Valuation (draft) → Review → Post
  ├── M6: Run Distribution → Post RADs
  ├── M7: Run Payment Processing
  ├── M8: Generate ONRR-2014 → State Severance Tax Report
  └── M9: Onboarding Validation → Period Close

Phase 2
  ├── M5: Create PBA → Generate Statement
  ├── M2: Owner Transfer (Farmer Ted → Son 1 + Son 2)
  ├── M2: Correction Transfer → PPNs Generated
  ├── M4: Process CA PPN → Process VL PPN → Post PPA
  ├── M4: Price Change → PPN → Re-Valuation → Post
  ├── M7: Setup Check Input (PDX, Remitter XRef)
  ├── M7: Enter Check → Validate → Value → Post
  ├── M1: OO Non-Allocated DN → Enter MP Volumes
  ├── M2: OO DOI with Balancing Owner
  └── M9: Period Close (Month 2)
```
