# EnerFusion Upstream Accounting — Upstream Accounting System
## Documentation Index

> **Platform:** React 18 + Node.js | **Infra:** Azure AKS + PostgreSQL | **Pattern:** Modular Microservices + RBAC

---

## Document Map

| # | Document | Description |
|---|----------|-------------|
| 01 | [System Overview](./01-System-Overview.md) | Business context, module map, end-to-end process flow |
| 02 | [Solution Architecture](./02-Solution-Architecture.md) | Frontend, backend, infra, containerization, AKS topology |
| 03 | [Module Specifications](./03-Module-Specifications/) | Detailed functional requirements per module |
| 04 | [Data Models](./04-Data-Models.md) | All entities, attributes, relationships, PostgreSQL schema guidance |
| 05 | [API Design](./05-API-Design.md) | RESTful API contracts per module for external integrations |
| 06 | [RBAC & Security](./06-RBAC-Security.md) | Role taxonomy, permission matrix, implementation patterns |
| 07 | [NFR & Performance](./07-NFR-Performance.md) | Scalability, availability, data volume, caching, resilience |
| 08 | [Deployment Architecture](./08-Deployment-Architecture.md) | Azure AKS setup, Helm charts, CI/CD, observability |

---

## Module Index

| Module | Domain | Key Capability |
|--------|--------|----------------|
| M1 | Production & Volume Allocation | Wellhead volumes → delivery network allocation |
| M2 | Ownership & DOI | Division of interest, JVA, ownership transfers |
| M3 | Contractual Allocation & Product Control | Sales contract allocation, nominations, imbalances |
| M4 | Contracts, Pricing & Valuation | Formula pricing, tax calc, settlement statements |
| M5 | Balancing Workplace | Product imbalance resolution, PBA management |
| M6 | Revenue Distribution & Accounting | Owner revenue distribution, interest calculation |
| M7 | Payment Processing & Check Input | Check input/output, AR, escheat/escrow |
| M8 | Regulatory, Tax & Royalty Reporting | State + federal statutory report generation |
| M9 | Administration & ILM | Data archiving, personal data protection, automation |

---

## End-to-End Data Flow

```
[Wellhead Production]
       ↓
[M1: Production Allocation] — volumes per well/MP
       ↓
[M2: Ownership / DOI] — who owns what %
       ↓
[M3: Contractual Allocation] — allocate to sales contracts
       ↓
[M4: Valuation & Pricing] — apply price, tax, deductions
       ↓
[M5: Balancing] — resolve imbalances
       ↓
[M6: Revenue Distribution] — distribute to owners
       ↓
[M7: Payment Processing] — cut checks, manage AR
       ↓
[M8: Regulatory Reporting] — state & federal compliance
```
