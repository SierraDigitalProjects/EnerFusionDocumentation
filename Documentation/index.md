# EnerFusion Upstream Accounting

## Upstream Accounting Platform

> **Platform:** React 18 + Node.js 20 | **Infrastructure:** Azure AKS + PostgreSQL | **Pattern:** Modular Microservices + RBAC

EnerFusion Upstream Accounting is a cloud-native upstream accounting platform for North American energy operators. It manages the complete revenue lifecycle — from physical wellhead production through statutory compliance reporting and owner payment disbursement.

---

## End-to-End Process Flow

```
[M10: Partner Onboarding]                ← Business partner registration, JOA validation, system access provisioning
       │
       ▼
[Wellhead Production]
       │
       ▼
[M1: Production & Volume Allocation]     ← Well completion volumes, measurement point data, downtime adjustments
       │
       ▼
[M2: Ownership & DOI]                    ← Division of interest, working / royalty / override interest assignments
       │
       ▼
[M3: Contractual Allocation]             ← Contract volumes, product control, network imbalances, NGL component splits
       │
       ▼
[M4: Contracts, Pricing & Valuation]     ← Pricing formulas, deductions, tax schedules, valued settlement statements
       │                         │
       ▼                         ▼
[M5: Balancing Workplace]   [M11: Joint Venture Accounting]
← Over/under-take positions,    ← AFE elections, JIB billing,
  cash settlements,               cash calls, COPAS audit trails
  make-up volumes
       │                         │
       └───────────┬─────────────┘
                   ▼
[M6: Revenue Distribution & Accounting] ← Owner revenue entitlements, GL postings (RADs), late-payment interest
       │
       ▼
[M7: Payment Processing & Check Input]  ← ACH / check / wire disbursements, AR aging, purchaser remittances, escheat
       │
       ▼
[M8: Regulatory, Tax & Royalty Reporting] ← ONRR-2014, state severance tax, ad valorem, production reports

─────────────────────────────────────────────────────────────────────────────────
[M9: Administration & ILM]              ← Cross-cutting: RBAC, period control, automation, audit logs, ILM archiving
─────────────────────────────────────────────────────────────────────────────────
```

---

## Module Index

| # | Module | Domain | Key Output |
|---|--------|--------|------------|
| M1 | Production & Volume Allocation | Field Operations | Allocated well completion volumes |
| M2 | Ownership & DOI | Land Administration | Validated division of interest records |
| M3 | Contractual Allocation & Product Control | Commercial | Contract-level volumes, entitlements, imbalances |
| M4 | Contracts, Pricing & Valuation | Finance | Settlement statements, run statements |
| M5 | Balancing Workplace | Finance | Resolved owner imbalance positions |
| M6 | Revenue Distribution & Accounting | Finance | Owner distributions, GL postings |
| M7 | Payment Processing & Check Input | Finance | Check register, AR aging, escheat filings |
| M8 | Regulatory, Tax & Royalty Reporting | Compliance | State/federal statutory filings |
| M9 | Administration & ILM | IT/Ops | Archived data, anonymised records, automation config |
| M10 | Partner Onboarding | Partner Operations | 7-step JV partner onboarding workflow, document collection, system access |
| M11 | Joint Venture Accounting (JVA) | Joint Venture Finance | AFE elections, JIB billing, cash call management, COPAS audit |

---

## Primary User Roles

| Role | Module Access | Responsibilities |
|------|--------------|-----------------|
| Production Engineer | M1 | Daily volumes, downtime, well tests |
| Land / Ownership Administrator | M2 | DOI setup, ownership transfers, JVA |
| Commercial / Contract Manager | M3, M4 | Contracts, pricing formulas, nominations |
| Revenue Accountant | M4, M5, M6, M7 | Valuation, distribution, check processing |
| Compliance Officer | M8 | State and federal regulatory filings |
| System Administrator | M9, All | RBAC, ILM, automation, audit |
| Partner Onboarding Analyst | M10 | Partner registration, document verification, system access provisioning |
| JV Accountant | M11 | AFE elections, JIB billing, cash call tracking, COPAS audit management |

---

## AI-Driven Process Improvement

AI capabilities can be embedded across every module to reduce manual effort, catch errors earlier, and surface insights that are impractical to derive manually at scale.

| Module | AI Opportunity | Expected Improvement |
|--------|---------------|----------------------|
| **M1 — Production & Volume Allocation** | Anomaly detection on well completion volumes; ML-based downtime classification; automated variance flagging against historical baselines | Faster identification of measurement errors, reduced manual review of allocation exceptions |
| **M2 — Ownership & DOI** | NLP extraction of ownership data from JOA / lease documents; automated conflict detection across overlapping DOI records; change-of-ownership suggestion from title runsheets | Reduced data entry errors, faster DOI setup, earlier detection of ownership disputes |
| **M3 — Contractual Allocation** | Predictive contract nomination based on historical takes and market signals; automated imbalance root-cause categorisation; NGL component split validation against plant statements | More accurate nominations, fewer balancing exceptions, faster month-end close |
| **M4 — Contracts, Pricing & Valuation** | Automated price index reconciliation against third-party feeds (Platts, OPIS); ML detection of unusual deduction patterns; prior-period adjustment prediction | Reduced pricing disputes, faster settlement approval, proactive PPA identification |
| **M5 — Balancing Workplace** | Imbalance position forecasting; automated make-up volume scheduling recommendations; cash settlement vs. make-up trade-off analysis | Faster imbalance resolution, optimised make-up scheduling, reduced cash exposure |
| **M6 — Revenue Distribution & Accounting** | Automated RAD exception triage; duplicate distribution detection; late-payment interest calculation audit; GL posting anomaly detection | Fewer distribution errors reaching M7, faster accountant review cycle |
| **M7 — Payment Processing & Check Input** | Intelligent remittance matching (CDEX/820 to open AR); duplicate payment detection; escheat risk scoring for aged unclaimed items; ACH return pattern analysis | Higher straight-through processing rates, lower escheat exposure, faster AR reconciliation |
| **M8 — Regulatory, Tax & Royalty Reporting** | Automated ONRR-2014 pre-validation against filing rules; severance tax rate change monitoring; ad valorem true-up variance analysis | Reduced regulatory filing errors, proactive compliance alerts, faster audit responses |
| **M9 — Administration & ILM** | AI-assisted RBAC role recommendation based on job function; anomalous access pattern detection; intelligent ILM retention classification for unstructured data | Tighter access governance, earlier security anomaly detection, reduced ILM manual classification |
| **M10 — Partner Onboarding** | Automated document completeness checks (W-9, insurance certs); entity name / TIN validation against IRS/SAP master data; risk scoring for new JV partners | Faster onboarding cycle, reduced compliance gaps, earlier flagging of high-risk partners |
| **M11 — Joint Venture Accounting** | AFE cost-to-complete forecasting; JIB dispute prediction based on historical partner patterns; automated COPAS audit exception flagging; cash call timing optimisation | Better AFE budget control, fewer JIB disputes, proactive COPAS compliance |

### Cross-Cutting AI Capabilities

- **Natural Language Query** — users can query production volumes, owner statements, and compliance status in plain English without navigating module-specific screens.
- **Predictive Close** — ML models trained on historical month-end patterns flag which modules are likely to miss close deadlines, enabling proactive intervention.
- **Audit Trail Summarisation** — LLM-generated summaries of change history for any entity (well, owner, contract) reduce auditor preparation time.
- **Regulatory Change Monitoring** — automated ingestion of ONRR, state agency, and COPAS bulletins with AI-generated impact assessments mapped to affected modules.
- **Operator Copilot** — in-app AI assistant contextual to the user's current module, role, and open work items — surfaces next-best actions and explains calculation logic on demand.

---

## Document Map

| Document | Description |
|----------|-------------|
| [Benefits](benefits.md) | Business and technical value drivers |
| [Assumptions](assumptions.md) | Constraints and baseline conditions |
| [Scope](scope.md) | In-scope and out-of-scope boundaries |
| [Master Data Overview](master-data/overview.md) | Entity ownership model |
| [Functional Requirements](requirements/functional.md) | Module-level functional requirements |
| [Non-Functional Requirements](requirements/non-functional.md) | Performance, scalability, security |
| [Pilot Scope](pilot-scope.md) | Minimum viable solution scope, deferred features, integration stubs, and acceptance criteria |
| [E2E Test Cases — Build-A-Biz](test-cases/e2e-scenario.md) | End-to-end scenario test cases across all modules (Phase 1 & 2) |
