# EnerFusion Upstream Accounting

## Upstream Accounting Platform

> **Platform:** React 18 + Node.js 20 | **Infrastructure:** Azure AKS + PostgreSQL | **Pattern:** Modular Microservices + RBAC

EnerFusion Upstream Accounting is a cloud-native upstream accounting platform for North American energy operators. It manages the complete revenue lifecycle — from physical wellhead production through statutory compliance reporting and owner payment disbursement.

---

## End-to-End Process Flow

```
[M10: Partner Onboarding]     ← Onboard new JV partners (7-step workflow)
       │ ua.onboarding.partner.completed
       ▼
[Wellhead Production]
       │
       ▼
[M1: Production Allocation]   ← Volumes per well / measurement point
       │
       ▼
[M2: Ownership / DOI]         ← Who owns what percentage
       │
       ▼
[M3: Contractual Allocation]  ← Allocate volumes to sales contracts
       │
       ▼
[M4: Valuation & Pricing]     ← Apply price, tax, deductions
       │                ┌─────────────────────────────────┐
       ▼                ▼                                 │
[M5: Balancing]   [M11: JVA]  ← AFE elections, JIB billing, cash calls
       │
       ▼
[M6: Revenue Distribution]    ← Distribute net value to owners
       │
       ▼
[M7: Payment Processing]      ← Cut checks, manage AR, JIB netting
       │
       ▼
[M8: Regulatory Reporting]    ← State & federal compliance filings
[M9: Administration & ILM]    ← Cross-cutting: RBAC, audit, automation
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

## Document Map

| Document | Description |
|----------|-------------|
| [Benefits](benefits.md) | Business and technical value drivers |
| [Assumptions](assumptions.md) | Constraints and baseline conditions |
| [Scope](scope.md) | In-scope and out-of-scope boundaries |
| [Master Data Overview](master-data/overview.md) | Entity ownership model |
| [Functional Requirements](requirements/functional.md) | Module-level functional requirements |
| [Non-Functional Requirements](requirements/non-functional.md) | Performance, scalability, security |
| [Open Questions](open-questions.md) | Outstanding decisions and risks |
| [Module Architecture](architecture/modules.md) | Per-module design and interactions |
| [Technical Architecture](architecture/technical.md) | Frontend, backend, infra, AKS topology |
| [Operations](operations.md) | CI/CD, observability, runbooks |
| [Project Structure](project-structure.md) | Repository layout and service map |
| [Pilot Scope](pilot-scope.md) | Minimum viable solution scope, deferred features, integration stubs, and acceptance criteria |
| [E2E Test Cases — Build-A-Biz](test-cases/e2e-scenario.md) | End-to-end scenario test cases across all modules (Phase 1 & 2) |
