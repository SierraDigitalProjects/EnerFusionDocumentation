# Functional Requirements

Requirements use the format: `FR-[Module]-[###]` — e.g., `FR-M1-001`.

Priority: **P1** = Must have (launch blocker) | **P2** = Should have | **P3** = Nice to have

---

## M1 — Production & Volume Allocation

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M1-001 | The system shall allow creation and management of Fields, Reservoirs, Platforms, Wells, and Well Completions as date-effective master data records | P1 | CRUD with effective dating |
| FR-M1-002 | The system shall support registration of wells using the API number as the unique national identifier | P1 | API# uniqueness enforced |
| FR-M1-003 | The system shall manage Measurement Points (meters, tank batteries, flare, fuel) with meter specifications stored as structured data | P1 | |
| FR-M1-004 | The system shall store tank strapping tables (gauge-to-volume calibration) per tank and version them by calibration date | P1 | Interpolation at volume submission |
| FR-M1-005 | The system shall manage Delivery Networks with configurable allocation tolerance (%) and allocation method (theoretical / historical / flat rate) | P1 | |
| FR-M1-006 | The system shall capture daily well completion volumes including allocated oil (BBL), gas (MCF), and water volumes | P1 | |
| FR-M1-007 | The system shall record well downtime by date with hours down and reason code; partial downtime shall reduce the theoretical calculation proportionally | P1 | |
| FR-M1-008 | The system shall store well test records (potential, adjusted, regulatory) including oil/gas/water rates, GOR, BTU, and test duration | P1 | |
| FR-M1-009 | The system shall execute the 8-step volume allocation algorithm: calculate theoreticals → sum inputs → determine outputs → compute % difference → check tolerance → apply scale factor → allocate to WCs → persist results | P1 | See algorithm in M1 spec |
| FR-M1-010 | The system shall flag allocation runs that exceed the delivery network tolerance threshold and route them for supervisor review before posting | P1 | |
| FR-M1-011 | The system shall support automated (scheduled), manually triggered, and event-driven allocation runs | P2 | Automation ruleset (M9) |
| FR-M1-012 | The system shall publish a `ua.production.volumes.allocated` event upon successful allocation run completion | P1 | Consumed by M3, M8 |
| FR-M1-013 | The system shall generate: Measurement Document, Allocation Variance Report, Well Completion Volume Summary, Downtime Analysis Report | P1 | |
| FR-M1-014 | The system shall support bulk well completion creation via Excel template upload | P2 | Async background processing |
| FR-M1-015 | The system shall enforce RBAC at the well, MP, and DN level with `production:read`, `production:write`, `production:execute`, `production:admin` scopes | P1 | |

---

## M2 — Ownership & Division of Interest

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M2-001 | The system shall manage Business Partners (individual, company, trust, government) with encrypted tax IDs and payment address override | P1 | Replicated from SAP; locally addable fields |
| FR-M2-002 | The system shall manage Upstream Lease Identifiers with state lease number, legal description, royalty rate, and operator | P1 | |
| FR-M2-003 | The system shall manage Ventures (base DOI and unit types) with date-effective owner interest records supporting WI, RI, ORRI, NPRI, PRRI interest types | P1 | |
| FR-M2-004 | The system shall validate that Working Interest decimal interests sum to exactly 1.0 (±0.00000001) before a DOI change is approved | P1 | |
| FR-M2-005 | The system shall enforce a single-checkout lock per venture — only one active checkout allowed at a time | P1 | Conflict returns HTTP 409 |
| FR-M2-006 | The system shall support an approval workflow: check out → modify → submit → approve → check in | P1 | |
| FR-M2-007 | The system shall create an immutable chain-of-title record for every approved ownership transfer | P1 | No deletes on chain-of-title |
| FR-M2-008 | The system shall support ownership transfer requests: from-owner, to-owner, effective date, interest type, decimal interest, and optional suspended fund / imbalance balance transfer | P1 | |
| FR-M2-009 | The system shall publish a `ua.ownership.doi.changed` event when a DOI change is approved | P1 | Consumed by M3, M4 |
| FR-M2-010 | The system shall manage Unit Venture tract records with participation factors | P2 | |
| FR-M2-011 | The system shall support Owner Attribute Groups defining payment method, threshold, and withholding percentages | P1 | |
| FR-M2-012 | The system shall support sliding scale royalty configurations where royalty rate varies by monthly production volume tier | P2 | |
| FR-M2-013 | The system shall generate: Schedule A (Transfer Order), Historical Ownership Report, Zero Interest Owners Report, DOI Validation Report, Chain of Title Report | P1 | |
| FR-M2-014 | The system shall support bulk DOI creation and update via Excel template upload with async processing and progress tracking | P2 | |
| FR-M2-015 | The system shall enforce RBAC with `ownership:read`, `ownership:write`, `ownership:approve`, `ownership:import` scopes | P1 | |

---

## M3 — Contractual Allocation & Product Control

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M3-001 | The system shall allocate allocated production volumes to sales contracts based on owner entitlement percentages | P1 | |
| FR-M3-002 | The system shall calculate owner entitlements using division of interest records from M2 | P1 | |
| FR-M3-003 | The system shall track daily and cumulative network imbalances per marketing group | P1 | |
| FR-M3-004 | The system shall support gas plant processing configuration: inlet MP, component splits (shrinkage, NGL recovery fractions), fuel usage, and percent-return-to-lease (PRTL) records | P1 | |
| FR-M3-005 | The system shall support manual adjustment entries for contractual allocation volumes | P2 | |
| FR-M3-006 | The system shall support daily and monthly nomination management per owner/contract | P2 | |
| FR-M3-007 | The system shall generate: CA Volume Validation Report, Marketing Group Imbalance Report, Nomination Summary | P1 | |

---

## M4 — Contracts, Pricing & Valuation

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M4-001 | The system shall manage Sales Contracts with type, buyer/seller parties, effective/expiration dates, pricing method, and royalty reimbursement flag | P1 | |
| FR-M4-002 | The system shall support Fixed Price Conditions per contract, product type, and date range | P1 | |
| FR-M4-003 | The system shall support Formula Pricing with a multi-step evaluation engine using configurable reserved words (NYMEX, WTI, BTU, transport deductions) | P1 | |
| FR-M4-004 | The system shall support API Gravity Scales for crude oil quality-based price adjustment per degree above/below base gravity | P1 | |
| FR-M4-005 | The system shall maintain State Tax Rates (severance, ad valorem, conservation) per state, product type, and effective date | P1 | |
| FR-M4-006 | The system shall support Tax Classification at the well completion level (standard, stripper, enhanced recovery, marginal) with exemption codes | P1 | |
| FR-M4-007 | The system shall support Tier Tax calculation where different rates apply to production volume thresholds | P2 | |
| FR-M4-008 | The system shall manage Valuation Cross-Reference (VCR) records that map venture/WC to contract, tax classification, and statement profile | P1 | |
| FR-M4-009 | The system shall execute valuation runs that: fetch M3 allocations → apply pricing → calculate all deductions → compute net value → persist settlement statements | P1 | |
| FR-M4-010 | The system shall generate Settlement Statements showing gross value, severance tax, ad valorem tax, transportation, marketing, processing deductions, and net value | P1 | |
| FR-M4-011 | The system shall generate Oil and Gas Run Statements per delivery run with temperature correction, gravity correction, API gravity, and BS&W | P1 | |
| FR-M4-012 | The system shall support Prior Period Adjustment (PPN) events: link to original settlement, record reason, volume and value delta, route for approval | P1 | |
| FR-M4-013 | The system shall apply marketing-free status from DOI records (owners exempt from marketing cost deductions) | P1 | |
| FR-M4-014 | The system shall publish `ua.valuation.settlement.completed` event upon successful valuation run | P1 | Consumed by M6, M7 |
| FR-M4-015 | The system shall enforce RBAC with `valuation:read`, `valuation:write`, `valuation:approve`, `valuation:execute`, `valuation:admin` scopes | P1 | |

---

## M5 — Balancing Workplace

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M5-001 | The system shall track cumulative owner imbalance positions (over/under-take) per product per venture | P1 | |
| FR-M5-002 | The system shall manage Product Balancing Agreements (PBAs) with tolerance band, make-up cap, and settlement terms | P1 | |
| FR-M5-003 | The system shall support manual balance adjustments by authorized users with mandatory reason code | P1 | |
| FR-M5-004 | The system shall generate: Balancing Exception Report, Owner Balances by Sales/Report Month, Balancing Statement | P1 | |
| FR-M5-005 | The system shall support in-kind volume make-up tracking against PBA terms | P2 | |

---

## M6 — Revenue Distribution & Accounting

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M6-001 | The system shall distribute net settlement values to each owner interest in proportion to their DOI decimal interest | P1 | |
| FR-M6-002 | The system shall calculate late-payment interest on overdue owner distributions using configurable interest rates | P1 | |
| FR-M6-003 | The system shall generate Revenue Accounting Documents (RADs) formatted for posting to the external GL system | P1 | RADs posted via API to SAP FI |
| FR-M6-004 | The system shall generate: Interest Calculation Detail Report, Owner Interest Summary Report, Summary Interest Paid Report | P1 | |
| FR-M6-005 | The system shall publish `ua.revenue.distribution.completed` event upon successful distribution run | P1 | Consumed by M7, M8 |

---

## M7 — Payment Processing & Check Input

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M7-001 | The system shall process incoming purchaser check records via CDEX/EDI 820 format, matching checks to settlement statements and ventures | P1 | |
| FR-M7-002 | The system shall support manual check input for non-EDI purchaser payments | P1 | |
| FR-M7-003 | The system shall support mass check input for bulk processing of multiple purchaser checks | P1 | |
| FR-M7-004 | The system shall generate outgoing owner payment records (check or ACH) based on M6 distribution amounts and Owner Attribute Group payment configuration | P1 | |
| FR-M7-005 | The system shall suspend payments below the owner's minimum payment threshold and accumulate until threshold is met | P1 | |
| FR-M7-006 | The system shall maintain Accounts Receivable aging for open purchaser balances | P1 | |
| FR-M7-007 | The system shall support manual AR write-offs with reason code and approver | P1 | |
| FR-M7-008 | The system shall track unclaimed owner funds and generate escheat reports when state dormancy thresholds are exceeded | P1 | |
| FR-M7-009 | The system shall generate check register, owner payment stubs, AR aging report, and escheat filing reports | P1 | |

---

## M8 — Regulatory, Tax & Royalty Reporting

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M8-001 | The system shall generate Form H-10, P1B, PR, GLO Royalty, and UT Royalty reports for Texas | P1 | |
| FR-M8-002 | The system shall generate ONRR-2014, MMS-PASR, and MMS-OGOR reports for federal ONRR compliance | P1 | |
| FR-M8-003 | The system shall generate Form 1004 for Oklahoma gross production tax | P1 | |
| FR-M8-004 | The system shall generate OGP/R5D and WR1 reports for Louisiana | P1 | |
| FR-M8-005 | The system shall generate Report C115 for New Mexico | P1 | |
| FR-M8-006 | The system shall generate Form 2 and Gross Production Tax Report for Wyoming | P1 | |
| FR-M8-007 | The system shall generate Form 7 for Arkansas and Colorado | P2 | |
| FR-M8-008 | The system shall generate Report OG110 for California | P2 | |
| FR-M8-009 | The system shall generate production reports for North Dakota | P2 | |
| FR-M8-010 | The system shall support amended report generation referencing original filing | P1 | |
| FR-M8-011 | The system shall support out-of-statute write-off processing for periods beyond the statutory filing window | P2 | |
| FR-M8-012 | The system shall maintain ONRR payor / company code cross-reference master data to map operator company codes to ONRR-issued payor identifiers | P1 | |
| FR-M8-013 | The system shall support a multi-level ONRR reporting hierarchy: Venture → Lease (with participation %) → Sales Type (with allocation %); volume and value shall be divided proportionally through each level | P1 | |
| FR-M8-014 | The system shall process ONRR-2014 in a sequential, operator-controlled workplace with the following steps: Populate → Validate → Generate → Extract → Summarize → Journalize → Finalize | P1 | Parallel processing by company code + sales date |
| FR-M8-015 | The system shall generate ONRR Transaction Codes: TC 01 (royalty), TC 11 (transport allowance), TC 13 (quality differential / marketing reimbursement), TC 15 (processing allowance), TC 37 (Section 6 severance + additional royalty), TC 38 (Section 6 additional royalty only) | P1 | |
| FR-M8-016 | The system shall support Section 6 royalty configuration with a four-level rate hierarchy: default config → lease royalty rate → master data override → ad hoc workplace override | P1 | See ONRR Processing guide |
| FR-M8-017 | The system shall support two types of ONRR Prior Period Notifications (PPNs): venture/sales-type level and venture/lease level; the venture-level PPN must supersede sales-type PPN when both exist | P1 | |
| FR-M8-018 | The system shall generate TC 13 lines when valuation formula result words ERQADIFR (quality differential reimbursement) or ERQADIFD (quality differential deduction) are used | P2 | Quality differential must NOT be collapsed into processing deduct bucket |
| FR-M8-019 | The system shall generate the ONRR-2014 output as a fixed-width ASCII file per ONRR PAAS electronic filing specification | P1 | |

---

## M7 — Payment Processing Enhancements (from Spec)

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M7-010 | The system shall validate all owners have a valid payment method, EFT banking information, and mailing address before allowing a payment run to proceed | P1 | |
| FR-M7-011 | The system shall allow payment runs to be scoped by company, owner, Delivery Network, Unit Venture, or property | P1 | |
| FR-M7-012 | The system shall support cash company groupings — associating multiple companies to a single cash company for consolidated outbound bank files | P1 | |
| FR-M7-013 | The system shall provide a pre-finalization variance analysis step where users can suspend individual payment lines, override variances, and assign hold codes before any outbound bank file is generated | P1 | |
| FR-M7-014 | The system shall support hold codes including No Print and user-defined reason codes; multiple hold codes may be assigned to a single owner | P1 | |
| FR-M7-015 | The system shall generate positive pay (outbound bank tape) and support inbound check status (bank tape reconciliation) | P1 | |
| FR-M7-016 | The system shall support user-defined summarization rules for check stub line-of-detail aggregation | P2 | |
| FR-M7-017 | The system shall support JIB (Joint Interest Billing) netting — deducting outstanding JIB receivable balances from owner revenue distributions before issuing payment | P1 | |
| FR-M7-018 | The system shall generate IRS Form 1099-MISC and 1099-NEC for qualifying owner payments with manual override capability for stripper well and special production payment structures | P1 | |
| FR-M7-019 | The system shall support user-defined recoupment rules at venture, state, vendor, and company levels | P2 | |

## M7 — Check Input Enhancements (from Spec)

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-CI-001 | The system shall maintain a Remitter/Property cross-reference with full historical versioning by effective date and change log | P1 | |
| FR-CI-002 | The system shall support mass upload of remitter/property cross-reference records via Excel template with async processing and validation report | P1 | |
| FR-CI-003 | The system shall support multi-select copy/reverse with sales date range search for correcting previously processed checks | P1 | |
| FR-CI-004 | The system shall expand CDEX company code to accommodate identifiers longer than two characters | P1 | |
| FR-CI-005 | The system shall calculate severance tax independently within Check Input when the property master data flag is enabled, following DOI accounting rules | P1 | |
| FR-CI-006 | The system shall support special disbursement (PDX) logic: distribute a net check amount to selected owners only, with no discrepancy between amount received and amount disbursed | P1 | |
| FR-CI-007 | The system shall generate a Blocked Cost Center Report prior to journalization, showing which cost centers would prevent posting | P1 | |
| FR-CI-008 | The system shall generate a Proposed Journalization Preview Report showing owner-level (who, volume, value, deducts) and account-level (GL account, cost center, amount) detail before final posting | P1 | |
| FR-CI-009 | The system shall support Common AR Point distribution using tracking participation factors when a check covers multiple properties under a shared receivable point | P2 | |
| FR-CI-010 | The system shall generate a DOI Decimal Comparison Report validating that the PDX cross-reference decimal interest agrees with the aggregated DOI owner interests | P2 | |

## M7 — AR Workplace Enhancements (from Spec)

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-AR-001 | The system shall allow AR users to flag balances as reconciled, deferred, write-off, or categorized; and to transfer balances to another AR key | P1 | |
| FR-AR-002 | The system shall support user-defined AR category codes (not a generic flag) assigned to all or part of an AR balance, and use category code as a search criterion | P1 | |
| FR-AR-003 | The system shall support bulk approval of multiple write-off rows simultaneously, with a single approval-level comment applied to all selected rows | P1 | |
| FR-AR-004 | The system shall support comment upload for batch AR reconciliation workflows; comments must be preserved when Excel downloads are re-sorted | P2 | |
| FR-AR-005 | The system shall support auto write-off scheduling more frequently than monthly, configurable as part of the Accounting Period Close process, with a maximum dollar cap on auto write-offs | P1 | |
| FR-AR-006 | The system shall provide both summary (by company/remitter) and detail AR reports with multi-value selection criteria | P1 | |
| FR-AR-007 | The system shall support an Accounting Month view of AR summary balances in addition to Sales Month view | P1 | |
| FR-AR-008 | The system shall display remitter name/description and venture/DOI name in all AR Workplace views (not ID-only) | P1 | |

## M4 — Ad Valorem True-Up (from Spec)

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-ADV-001 | The system shall support year-end ad valorem true-up processing to capture and distribute the difference between monthly accruals and the actual state/county invoice amount | P1 | Applicable: UT, CO, WY and others |
| FR-ADV-002 | The system shall accept true-up amounts as either net or gross, at the venture/DOI or unit level, by sales date (month/year), product, and marketing group via bulk upload | P1 | |
| FR-ADV-003 | The true-up distribution shall follow the same VL/RD accrual logic including rounding, account determination, affiliate treatment, and chain of title | P1 | |
| FR-ADV-004 | The true-up shall not distribute to tax-exempt, take-in-kind, or do-not-pay owners | P1 | |
| FR-ADV-005 | The system shall generate a trial combined run of accounts and totals before posting; trial runs must be saveable for review | P1 | |
| FR-ADV-006 | The true-up shall not generate new PPNs or force existing PPNs to process | P1 | |

## M3 — Contractual Allocation Enhancements (from Spec)

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M3-008 | The system shall calculate primary product entitlements at the Venture/DOI level before distributing to owner level | P1 | |
| FR-M3-009 | The system shall eliminate the subtraction method rounding routine; owners with equal decimal interests must receive equal allocated volumes | P1 | See Rounding & Alloc spec |
| FR-M3-010 | The system shall support owner-level rounding configuration within marketing groups, separate from marketing group-level rounding | P1 | |
| FR-M3-011 | The system shall support settlement diversity groups where selected owners are valued at the group level with volume/value allocated back to individuals using group participation factors | P2 | |
| FR-M3-012 | The system shall track keepwhole calculations — when marketing group volume/energy is less than keepwhole volume/energy, the system shall calculate and flag the shortfall for compensation | P2 | |
| FR-M3-013 | The system shall generate Production Volume Reconciliation (PVR) and Production Transfer Reports (PTR) | P1 | |

---

## M10 — Partner Onboarding

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M10-001 | The system shall manage a 7-step partner onboarding workflow: Company Registration → JOA & Working Interest → Financial Setup → COPAS Compliance → Document Collection → System Access Provisioning → Review & Submit | P1 | Steps must be completed sequentially; each step is independently saveable |
| FR-M10-002 | The system shall capture partner legal entity details including entity type (LLC/Corporation/LP/LLP/Trust/Individual), EIN/TIN (encrypted), DUNS number, state of formation, and primary contacts per functional role | P1 | Tax IDs encrypted at rest with AES-256 |
| FR-M10-003 | The system shall create a Joint Operating Agreement (JOA) record linking the partner to producing assets (basin, product type) with Working Interest % and Net Revenue Interest % | P1 | WI/NRI values seed M2 DOI owner interest records on completion |
| FR-M10-004 | The system shall capture and store banking information (routing, masked account, account type), payment method preference, tax classification (1099 category), W9 receipt status, and ACH prenote status | P1 | Account numbers masked in UI; stored encrypted |
| FR-M10-005 | The system shall record COPAS accounting standard configuration including version, overhead method, escalation index, material pricing method, audit period, and capture a timestamped partner acknowledgment | P1 | COPAS config feeds M11 JVA billing rules |
| FR-M10-006 | The system shall track required compliance documents (W9, bank verification, JOA executed copy, Tax ID, COPAS acknowledgment) through a pending → received → verified → complete status workflow with overdue detection | P1 | |
| FR-M10-007 | The system shall provision partner system access including portal account (read-only or elections-enabled), EDI gateway configuration (JIB 810, Cash Call 820), and notification preferences | P1 | |
| FR-M10-008 | The system shall calculate and display onboarding progress as: `(completed_steps × 100 + active_steps × 45) / 7` | P1 | |
| FR-M10-009 | The system shall publish a `ua.onboarding.partner.completed` event upon Step 7 sign-off, consumed by M2 (DOI provisioning), M7 (payment configuration), and M11 (JVA partner interest activation) | P1 | |
| FR-M10-010 | The system shall provide a Pipeline view of all active onboarding sessions with status categorization (active / in review / final review / overdue), target date tracking, and monthly completion trend chart | P1 | |
| FR-M10-011 | The system shall enforce RBAC with `onboarding:read`, `onboarding:write`, `onboarding:approve`, `onboarding:admin` scopes | P1 | |
| FR-M10-012 | The system shall generate: Onboarding Pipeline Summary, Overdue Partners Report, Document Status Report, Monthly Completion Trend | P1 | |

---

## M11 — Joint Venture Accounting (JVA)

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M11-001 | The system shall manage Authorization for Expenditure (AFE) records with type (Drilling/Workover/Facility/Plug & Abandon), authorized amount, COPAS cost breakdown (codes 100–700), election deadline, and status lifecycle (draft → issued → approved → closed) | P1 | |
| FR-M11-002 | The system shall support Non-Operating Partner (NOP) AFE elections (participate / non-consent) with deadline tracking; defaulted elections (no response by deadline) shall be treated as non-consent with configurable penalty multiplier | P1 | |
| FR-M11-003 | The system shall track budget vs. actual for each AFE by COPAS cost code (100-Labor, 200-Materials, 300-Transport, 400-Services, 600-Overhead, 700-Taxes) and alert when actuals exceed a configurable variance threshold | P1 | |
| FR-M11-004 | The system shall prepare monthly Joint Interest Bills (JIBs) for each non-operating partner calculating: direct costs × partner WI%, COPAS overhead (Fixed Rate per Well), and total JIB amount per COPAS schedule | P1 | |
| FR-M11-005 | The system shall deliver JIBs via EDI 810, email (PDF), or partner portal per partner delivery preference configured in M10 | P1 | |
| FR-M11-006 | The system shall track JIB payment status (issued → partial → paid → overdue) and publish `ua.jva.jib.issued` for M7 AR receivable tracking | P1 | |
| FR-M11-007 | The system shall manage cash call advance funding requests with due date enforcement, overdue detection, daily interest accrual on overdue balances, and receipt application against open calls | P1 | |
| FR-M11-008 | The system shall apply incoming partner payments to the oldest outstanding cash call first (configurable); overpayments credited to partner suspense account | P1 | |
| FR-M11-009 | The system shall provide a cash call forecast view showing upcoming funding requirements from open AFE-linked expenditures by partner | P2 | |
| FR-M11-010 | The system shall document COPAS audit findings with category (COPAS cost code), dollar amount, period, status (open / under review / resolved / rejected), and resolution notes; findings not resolved within the configurable window shall be flagged as overdue | P1 | |
| FR-M11-011 | The system shall activate JVA partner interest records (WI%, NRI%, COPAS config) upon receipt of `ua.onboarding.partner.completed` event from M10 | P1 | |
| FR-M11-012 | The system shall apply JIB netting in M7 — deducting outstanding JIB receivable balances from partner revenue distributions before issuing outgoing payments | P1 | FR-M7-017 cross-reference |
| FR-M11-013 | The system shall enforce RBAC with `jva:read`, `jva:write`, `jva:approve`, `jva:execute`, `jva:admin` scopes | P1 | |
| FR-M11-014 | The system shall generate: AFE Election Status Report, JIB Billing Summary, JIB Aging Report, Cash Call Forecast, Overdue Cash Call Report, COPAS Audit Finding Summary, Budget vs. Actual by AFE | P1 | |

---

## M9 — Administration & ILM

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-M9-001 | The system shall provide a centralized RBAC management UI for creating roles, assigning permissions, and managing user-role assignments | P1 | Role provisioning via Azure AD |
| FR-M9-002 | The system shall support configurable automation rulesets (bots) for allocation run scheduling, anomaly detection, and approval decision automation | P2 | |
| FR-M9-003 | The system shall archive completed transaction records to Azure Blob Storage after a configurable retention period | P2 | |
| FR-M9-004 | The system shall support blocking of personal data records associated with a business partner (right-to-be-forgotten workflow) | P2 | |
| FR-M9-005 | The system shall anonymize personal data fields (name, address, tax ID, bank details) after the applicable data retention period expires | P2 | GDPR/CCPA-aligned |
| FR-M9-006 | The system shall maintain a tamper-evident audit log of all data mutations across all modules | P1 | Who, what, when, before/after |
| FR-M9-007 | The system shall provide an onboarding validation tool that verifies master data completeness before a period's processing can begin | P2 | |
