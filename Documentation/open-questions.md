# Open Questions

This document tracks outstanding decisions, unresolved design questions, and known risks that must be resolved before or during implementation. Each item has an owner and a target resolution date.

Status: **Open** | **In Progress** | **Resolved** | **Deferred**

---

## Business & Functional

| # | Question | Status | Owner | Target | Notes |
|---|----------|--------|-------|--------|-------|
| OQ-B-001 | Will cash balancing at PBA expiration be required in Phase 1, or is in-kind balancing sufficient for launch? | Open | Product Owner | Phase 1 planning | In-kind assumed in current M5 design; cash settlement requires additional pricing logic |
| OQ-B-002 | Is FIRPTA withholding for foreign (non-US) owners in scope? What percentage of the owner population is foreign-domiciled? | Open | Tax/Legal | Discovery | Current scope assumes US-domiciled owners only |
| OQ-B-003 | Which gas marketing scenarios require support — operator-marketed gas only, or third-party gas purchaser scenarios as well? | Open | Commercial | Requirement review | Affects M3 entitlement and M7 check input design |
| OQ-B-004 | Does the client require automated e-filing submission to state agency portals (Texas RRC WRAP, ONRR PAAS), or manual export + upload? | Open | Compliance | Phase 2 scope | Automated e-filing could significantly reduce compliance team effort but adds integration complexity |
| OQ-B-005 | Are non-calendar fiscal accounting periods required? | Open | Finance | Requirement validation | Current architecture assumes calendar-month reporting periods |
| OQ-B-006 | What is the maximum historical data volume to be migrated from SAP PRA? (years of history, number of wells, DOI records) | Open | IT/Migration | Data assessment | Drives migration strategy, initial database sizing, and archive policy design |
| OQ-B-007 | Will the platform serve multiple independent operating companies (multi-tenant) on a shared AKS cluster, or one operator per cluster? | Open | Architecture / IT | Pre-Phase 1 | Determines tenant isolation strategy; shared cluster requires stricter namespace + RLS configuration |
| OQ-B-008 | Is state income tax withholding from owner payments required for all states, or only states where the operator has nexus? | Open | Tax/Legal | Tax review | Owner Attribute Group supports withholding % but the list of applicable states needs to be confirmed |
| OQ-B-009 | Are NGL fractionated components (ethane, propane, iC4, nC4, natural gasoline) priced individually, or as composite NGL? | Open | Commercial | Pricing model review | Affects M4 formula pricing and M8 NGL production reporting |
| OQ-B-010 | What is the expected volume of monthly purchaser check records processed via CDEX/EDI 820? | Open | Finance / IT | Volume assessment | Drives M7 batch processing design and EDI middleware selection |

---

## Technical & Architecture

| # | Question | Status | Owner | Target | Notes |
|---|----------|--------|-------|--------|-------|
| OQ-T-001 | Will the React SPA be deployed as a containerized bundle (current design) or served from Azure CDN / Static Web Apps? | Open | Architecture | Phase 1 design | CDN reduces AKS load for static assets but adds CDN cache invalidation complexity for versioned deployments |
| OQ-T-002 | What is the agreed Service Bus tier? Standard supports topics/subscriptions; Basic does not. | Open | IT / Cloud | Infrastructure provisioning | Basic tier would block the event-driven inter-module architecture |
| OQ-T-003 | Is a service mesh (Linkerd / Istio) required for mTLS, or is NGINX ingress + Azure APIM sufficient for the initial launch? | Open | Architecture | Phase 1 design | Service mesh adds operational complexity; initial launch may defer to APIM-only |
| OQ-T-004 | What price index data provider will supply NYMEX_GAS and WTI_OIL values for formula pricing? (Platts, Argus, CME Group) | Open | Commercial / IT | Data contracts | Affects formula engine index feed design in M4 |
| OQ-T-005 | Will PgBouncer run as a sidecar container per service pod, or as a shared connection pooling deployment? | Open | Architecture | Database design | Sidecar gives per-service control; shared pooler reduces resource overhead |
| OQ-T-006 | What is the target PostgreSQL Flexible Server SKU for production? (Compute tier, vCores, storage) | Open | IT / Cloud | Sizing exercise | Requires data volume assessment (OQ-B-006) and load testing results |
| OQ-T-007 | Should gRPC be used for synchronous inter-service calls (M3 → M4 volume handoff), or should all inter-module communication be async via Service Bus? | Open | Architecture | Phase 1 design | gRPC adds latency benefits for large volume payloads but increases service coupling |
| OQ-T-008 | Is Azure Application Gateway Ingress Controller (AGIC) preferred over NGINX Ingress Controller for AKS? | Open | IT / Cloud | Infrastructure decision | AGIC integrates with Azure WAF; NGINX offers more flexible routing rules |
| OQ-T-009 | What is the disaster recovery target region? (e.g., East US 2 primary → West US 3 DR) | Open | IT / Cloud | Business continuity planning | Affects geo-replication configuration for PostgreSQL and Service Bus |
| OQ-T-010 | Are there existing SAP BAPI or IDoc interfaces available for Business Partner replication, or will a custom RFC/API be needed? | Open | SAP / Integration | SAP landscape review | Affects integration layer design for centrally managed master data |

---

## Data & Migration

| # | Question | Status | Owner | Target | Notes |
|---|----------|--------|-------|--------|-------|
| OQ-D-001 | What is the data quality of existing SAP PRA DOI records? Are decimal interests balanced to 1.0? Are all owners valid BPs? | Open | Data Governance | Data quality assessment | DOI migration requires pre-validation; unbalanced interests must be resolved before import |
| OQ-D-002 | Will historical settlement statements from SAP PRA be migrated read-only to EnerFusion, or left in SAP for 7-year retention? | Open | Finance / IT | Migration strategy | Migrating historical statements simplifies AR/escheat analysis but increases migration scope |
| OQ-D-003 | Are there any proprietary file formats from third-party field data capture systems that need to be supported for M1 volume ingestion? | Open | IT / Operations | Interface inventory | Affects M1 data ingestion API design |
| OQ-D-004 | Are SAP PRA well API numbers consistent with current national API number registry? (Decommissioned wells, duplicate entries) | Open | Data Governance | Data assessment | |
| OQ-D-005 | What is the required data retention period for archived transactions? (US federal minimum is 7 years; some states require longer) | Open | Tax/Legal | Legal review | Drives M9 ILM archive policy configuration |

---

## Operations & Support

| # | Question | Status | Owner | Target | Notes |
|---|----------|--------|-------|--------|-------|
| OQ-O-001 | Is there an existing Azure DevOps or GitHub Actions pipeline infrastructure that EnerFusion should integrate with? | Open | IT / DevOps | Infra discovery | Affects CI/CD pipeline design |
| OQ-O-002 | Will a dedicated SRE / DevOps team operate the AKS cluster, or will EnerFusion be managed by the application team? | Open | IT | Org design | Affects runbook depth and operational handover documentation |
| OQ-O-003 | What is the approved process for applying regulatory report format updates when states change filing requirements? | Open | Compliance / IT | Process design | Statutory report templates in M8 must be updatable without a full application release |
| OQ-O-004 | Is Azure Monitor + Grafana the agreed observability stack, or are existing tools (Datadog, Splunk) already in use? | Open | IT / Operations | Tool standardisation | Affects observability configuration in Helm charts |

---

## ONRR-Specific Questions

| # | Question | Status | Owner | Target | Notes |
|---|----------|--------|-------|--------|-------|
| OQ-ONRR-001 | What is the complete list of federal and Indian leases requiring ONRR-2014 filing? | Open | Compliance | Pre-Phase 4 | Drives payor/company code cross-reference setup and Section 6 lease identification |
| OQ-ONRR-002 | Are any leases subject to Section 6 additional royalty provisions? What are their current rates? | Open | Compliance | ONRR master data setup | Required to configure Section 6 default rates and lease-level overrides |
| OQ-ONRR-003 | Does the operator have Indian mineral leases requiring Indian recoupment and estimate calculations? | Open | Compliance | ONRR master data setup | Indian recoupment is a separate calculation path requiring additional configuration |
| OQ-ONRR-004 | What is the ONRR payor code(s) for each company code reporting? | Open | Compliance | ONRR master data setup | Payor codes are issued by ONRR and must be registered before filing can occur |
| OQ-ONRR-005 | Are quality differential deductions (ERQADIFD) used in current valuation formulas? | Open | Commercial | Formula review | Affects TC 13 generation; must NOT be mapped to Processing Deduct bucket for federal leases |
| OQ-ONRR-006 | Will ONRR-2014 processing run at venture/DOI level or well/WC level? | Open | Compliance / IT | ONRR master data design | Venture/DOI is recommended; WC-level is more granular but operationally burdensome |

---

## Payment Processing Questions

| # | Question | Status | Owner | Target | Notes |
|---|----------|--------|-------|--------|-------|
| OQ-PP-001 | What hold codes are in use for suspended payments? How many user-defined hold reason codes are needed? | Open | Finance | M7 configuration | Hold code master list must be configured before payment processing can go live |
| OQ-PP-002 | Are there JIB (Joint Interest Billing) netting arrangements where revenue is offset against JIB receivables before issuing checks? | Open | Finance / JIB | M7 design | JIB netting adds integration complexity between M7 and AR |
| OQ-PP-003 | How many owners require non-resident alien (NRA) withholding? Which treaty rates apply? | Open | Tax / Legal | Owner master data | NRA rates are treaty-specific and require legal/tax review per country |
| OQ-PP-004 | Is stripper well 1099 classification needed? What volume thresholds qualify in each operating state? | Open | Tax | 1099 config | Stripper well credits may require manual override of standard 1099 logic |
| OQ-PP-005 | What is the minimum payment threshold policy — uniform across owners, or variable by state/owner type? | Open | Finance | Owner Attribute Group config | Directly drives suspended balance accumulation behavior |
| OQ-PP-006 | Does the operator use Common AR Point distribution? How many properties share common receivable points? | Open | Finance | Check Input config | Requires participation factor setup per common AR point |

---

## Contractual Allocation Questions

| # | Question | Status | Owner | Target | Notes |
|---|----------|--------|-------|--------|-------|
| OQ-CA-001 | Are there gas plants with keepwhole provisions? What are the contract terms? | Open | Commercial | M3 config | Keepwhole requires BTU entitlement comparison; affects NGL allocation design |
| OQ-CA-002 | Is settlement diversity (group-level valuation for selected owners) required in any marketing groups? | Open | Commercial | M3 config | Settlement diversity adds a group aggregation step before owner distribution |
| OQ-CA-003 | What rounding precision is required at owner level vs. marketing group level? Are there existing rounding discrepancies to correct? | Open | Finance | Rounding config | Must configure `roundingLevel` per marketing group |
| OQ-CA-004 | Are there unit ventures (regulatory pooling units) where tract participation factors drive allocation? | Open | Land / Commercial | Unit venture config | Unit allocation uses tract participation factors, not raw DOI decimals |

---

## Tax & Regulatory Questions

| # | Question | Status | Owner | Target | Notes |
|---|----------|--------|-------|--------|-------|
| OQ-TAX-001 | Are ad valorem true-up distributions required for Utah, Colorado, or Wyoming? What is the true-up cycle and invoice timing? | Open | Tax | M4 ad valorem config | True-up can be positive (under-accrued) or negative (recoupment); affects payment processing |
| OQ-TAX-002 | Are there Kansas operations? Kansas uses a separate tax framework requiring specific rate structure configuration | Open | Tax | State tax config | Kansas tax framework has a distinct rate structure |
| OQ-TAX-003 | Are there North Dakota operations? ND has a unique tax framework and report workplace requiring separate configuration | Open | Tax / Compliance | M8 ND config | ND tax framework configuration is separate from standard severance setup |
| OQ-TAX-004 | Does Montana apply? Montana has specific exemption configuration steps required before state reports can be generated | Open | Compliance | M8 state config | Montana exemption steps must be completed in sequence |

---

## Resolved Questions

| # | Question | Resolution | Date |
|---|----------|------------|------|
| *(None yet — to be populated as questions are resolved)* | | | |
