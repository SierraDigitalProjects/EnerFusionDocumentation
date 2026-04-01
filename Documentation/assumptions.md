# Assumptions

This document captures the baseline conditions, constraints, and decisions that underpin the EnerFusion Upstream Accounting design. Any assumption that proves invalid may require a scope or architecture change.

---

## Business Assumptions

| # | Assumption | Impact if Wrong |
|---|------------|-----------------|
| B1 | The operator's accounting operations are limited to **North American onshore/offshore** assets in US states covered by M8 | Additional state or international regulatory report formats would be required |
| B2 | The company continues to use **SAP FI/CO** (or an equivalent ERP) as the general ledger system of record | EnerFusion Upstream Accounting posts Revenue Accounting Documents (RADs) to an external GL via API; if SAP is decommissioned, an alternative GL integration must be designed |
| B3 | Production volumes originate from **SCADA or field data capture systems** that feed EnerFusion M1 via API or flat file | If volumes are entered manually at scale, the UI must accommodate bulk entry workflows |
| B4 | Purchaser check receipts are delivered via **CDEX/EDI 820** format | Other EDI or non-standard check formats require mapping configuration in M7 |
| B5 | Owner payments are issued as **ACH transfers or physical checks** — no cryptocurrency or non-standard disbursement | Alternative disbursement channels require M7 extension |
| B6 | All operators and purchasers are **US-domiciled entities** for tax withholding purposes | Foreign owners require FIRPTA withholding logic beyond the current scope |
| B7 | The company operates under a **working interest operator** model — it both operates and distributes revenue to non-operating interest owners | Pure royalty company or non-op workflows may require module configuration changes |
| B8 | **Gas balancing** under PBAs is settled in-kind (volume make-up) rather than cash | Cash settlement at PBA expiration requires M5 extension |

---

## Technology Assumptions

| # | Assumption | Impact if Wrong |
|---|------------|-----------------|
| T1 | **Azure** is the approved cloud platform | Migrating compute (AKS), messaging (Service Bus), monitoring (Azure Monitor), and key management (Key Vault) to another cloud provider requires re-architecture of all infra components |
| T2 | **Azure Active Directory (Entra ID)** is the identity provider | SAML-based or non-Azure OIDC providers require MSAL.js replacement and APIM token validation changes |
| T3 | **PostgreSQL 15+** (Azure Database for PostgreSQL — Flexible Server) is the primary database engine | Switching to a different RDBMS requires re-writing Knex migrations, Row-Level Security policies, and schema-per-module setup |
| T4 | Network latency between AKS services and the PostgreSQL Flexible Server is **< 2ms** (same Azure region) | Cross-region database deployment degrades settlement and allocation batch performance |
| T5 | The React SPA is deployed as a **containerized static bundle** served from an AKS pod behind NGINX ingress | CDN-based SPA delivery is not assumed; future CDN caching of assets can be added without breaking the backend |
| T6 | Inter-module events are processed via **Azure Service Bus Standard tier or above** | Azure Service Bus Basic tier does not support topics/subscriptions required by the event architecture |
| T7 | Container images are scanned by **Trivy** before push to Azure Container Registry (ACR) | Alternative image scanning tools (Snyk, Clair) are compatible but not pre-configured in CI |
| T8 | **Node.js 20 LTS** is used for all backend services; **TypeScript 5.x** is the language | Downgrading runtimes requires dependency audit and potential removal of ESM-only packages |

---

## Data Assumptions

| # | Assumption | Impact if Wrong |
|---|------------|-----------------|
| D1 | A single PostgreSQL cluster hosts all module schemas, isolated via schema-level permissions and RLS | Very high data volumes (>500 GB per module) may require schema sharding or dedicated clusters per module |
| D2 | Historical production and ownership data from the prior system (SAP PRA) is **migrated to EnerFusion before go-live** via a one-time ETL | Parallel-run scenarios require both systems to be live simultaneously, adding integration complexity |
| D3 | Reporting periods are **calendar months** — no split-period or non-standard fiscal periods | Non-calendar-month accounting periods require reporting period model changes across all modules |
| D4 | Well API numbers are treated as the **unique national identifier** for wells — no internal well numbering scheme conflicts | Duplicate or un-assigned API numbers require a deduplication process in M1 master data |
| D5 | DOI decimal interests are stored to **8 decimal places** — sufficient precision for all royalty and interest calculations | Some lease instruments specify interest to 10+ decimal places; schema changes to `DECIMAL(10,10)` may be required |

---

## Security & Compliance Assumptions

| # | Assumption | Impact if Wrong |
|---|------------|-----------------|
| S1 | All **secrets** (DB passwords, service bus connection strings, API keys) are stored in **Azure Key Vault** and mounted into pods via CSI driver | Hardcoded secrets in Helm values or environment files would fail security review |
| S2 | **Multi-tenancy** is enforced at the `company_code` level — one AKS cluster may serve multiple operating companies if they are subsidiaries of the same parent | Serving unrelated companies on a shared cluster requires a stricter namespace-per-tenant isolation model and separate Key Vaults |
| S3 | **GDPR / personal data** in the context of this platform is limited to owner (business partner) mailing addresses, tax IDs, and payment details | Medical, biometric, or other sensitive personal data is out of scope for ILM anonymization rules |
| S4 | All state regulatory filings are **generated within EnerFusion** and submitted manually or via state agency e-filing portals by the compliance officer | Automated e-filing API integrations with state agencies are out of scope for the initial release |

---

## Operational Assumptions

| # | Assumption | Impact if Wrong |
|---|------------|-----------------|
| O1 | Month-end close processing (allocation runs, valuation, distribution) completes within a **4-hour window** on the AKS batch node pool | Larger data volumes or complex formula pricing may require batch job optimization or extended windows |
| O2 | The platform is operated by **internal DevOps** using the provided Helm charts and GitHub Actions pipelines | Managed service deployments (e.g., Azure DevOps, third-party MSP) require pipeline adaptation |
| O3 | A **read replica** is available for reporting and batch query workloads — primary is write-only during batch runs | Single-instance PostgreSQL removes the ability to offload heavy reporting queries during month-end |
| O4 | **24×7 availability** is required only for API services — batch jobs may be scheduled off-peak (e.g., 10 PM – 4 AM CT) | Real-time valuation requirements (e.g., intraday pricing) would require always-on batch compute |
