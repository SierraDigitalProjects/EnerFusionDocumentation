# M10 — Partner Onboarding

## Module Overview

**Service:** `svc-onboarding`  
**Schema:** `onboarding`  
**Primary Users:** Land/Ownership Administrators, Finance Controllers, Legal Team  
**Domain:** Partner Operations

Partner Onboarding manages the complete lifecycle of bringing a new joint venture partner into EnerFusion's systems through a structured 7-step workflow. It serves as the entry gate for all external business partners before they can appear in M2 (Ownership/DOI) records and receive distributions from M6 (Revenue Distribution).

---

## Business Context

Before a non-operating working interest owner can receive royalty or revenue distributions, the operator must:

1. Verify the partner's legal identity and entity type
2. Establish the Joint Operating Agreement (JOA) and working interest percentages
3. Collect banking, payment, and tax documentation
4. Confirm COPAS accounting standards compliance
5. Collect required compliance documents (W9, bank verification)
6. Provision system access for partner portal and EDI
7. Complete a final review and activate the partner record

This 7-step process replaces email-based partner onboarding workflows with a structured, auditable system that integrates directly with M2 DOI records and M7 payment configuration.

---

## The 7-Step Onboarding Workflow

```
Step 1              Step 2              Step 3              Step 4
Company      →      JOA &        →      Financial    →      COPAS
Registration        Working Int.        Setup               Compliance
                                                                │
                                                                ▼
Step 7              Step 6              Step 5
Review &     ←      System       ←      Document
Submit              Access              Collection
```

### Step 1 — Company Registration

Captures the legal entity profile of the incoming partner.

| Field | Description |
|-------|-------------|
| Legal entity name | Full registered business name |
| DBA name | Doing-business-as trade name (optional) |
| Entity type | LLC / Corporation / LP / LLP / Trust / Individual |
| EIN / TIN | Federal tax identification number (encrypted at rest) |
| DUNS number | D&B identifier for credit and due diligence |
| State of formation | Jurisdiction of incorporation |
| Formation date | Date of legal formation |
| Business address | Primary registered address |
| Primary contacts | Contacts for: operations, finance, legal, billing, AFE elections |

### Step 2 — JOA & Working Interest

Creates the Joint Operating Agreement record and links the partner to producing assets.

| Field | Description |
|-------|-------------|
| JOA record | Joint Operating Agreement effective date, governing law |
| Property / asset links | Basin, formation, product type (oil/gas/NGL) |
| Working Interest % (WI) | Partner's cost-bearing interest |
| Net Revenue Interest % (NRI) | Partner's revenue-receiving interest |
| Interest summary | Calculates and validates summary totals per property |

**Validation:** WI and NRI percentages are validated for consistency. The JOA record triggers a corresponding venture/DOI interest record creation in M2 upon onboarding completion.

### Step 3 — Financial Setup

Establishes payment and tax configuration for all future distributions.

| Field | Description |
|-------|-------------|
| Bank routing number | ABA routing number for ACH |
| Account number | Masked after entry; encrypted at rest |
| Account type | Checking / Savings / Wire |
| Payment method | ACH/EFT (preferred) or check |
| Tax classification | 1099-MISC Box 2 reporting category |
| W9 receipt status | Pending / Received / Verified |
| ACH prenote status | Prenote sent / confirmed / failed |
| JIB delivery method | Email / EDI / Portal |
| Cash call delivery method | Email / EDI / Portal |
| Revenue distribution method | Per-property or consolidated |

### Step 4 — COPAS Compliance

Establishes accounting standards under which joint costs are billed and audited.

| Field | Description |
|-------|-------------|
| COPAS version | 2005 MFI-51 standard (primary) |
| Overhead method | Fixed Rate per Well (most common) |
| Escalation index | COPAS Index selection |
| Material pricing method | COPAS Schedule 3 pricing basis |
| Audit period | Number of years partner may audit billings |
| Dispute resolution terms | Escalation and resolution procedure |
| Partner acknowledgment | Digital sign-off with timestamp |

### Step 5 — Document Collection

Tracks required compliance documents through a verification workflow.

| Document | Status Flow |
|----------|------------|
| W9 / W8-BEN | Pending → Received → Verified → Complete |
| Tax ID documentation | Pending → Received → Verified → Complete |
| Bank verification letter | Pending → Received → Verified → Complete |
| JOA executed copy | Pending → Received → Verified → Complete |
| COPAS acknowledgment | Pending → Received → Verified → Complete |

- Overdue detection: flags documents not received by required date
- Analyst review queue: routes received documents for verification

### Step 6 — System Access Provisioning

Configures partner access to external-facing systems.

| Configuration | Description |
|--------------|-------------|
| Partner portal account | Creates read-only or elections-enabled user |
| Access level | Read-only / Elections-enabled |
| EDI gateway | JIB 810 (billing) and Cash Call 820 (funding) setup |
| Mineral Connect | Enablement flag |
| AFE Election Portal | Enables partner to respond to AFE elections |
| Notification config | JIB, Cash Call, AFE, Revenue statement notifications |

### Step 7 — Review & Submit

Final review of all 6 preceding steps by an authorized controller.

- Displays completion status of all steps
- Surfaces any outstanding documents or unresolved items
- Sign-off triggers status change from `active` to `completed`
- Publishes `ua.onboarding.partner.completed` event consumed by M2 (DOI provisioning) and M7 (payment config)

**Progress Formula:**
```
progress_pct = (completed_steps × 100 + active_steps × 45) / 7
```

---

## Master Data Entities

### Partner (Business Entity)

**Schema:** `onboarding.partners`

| Attribute | Type | Description |
|-----------|------|-------------|
| `partner_id` | UUID | Primary key |
| `legal_name` | VARCHAR(200) | Legal registered name |
| `dba_name` | VARCHAR(200) | Trade/doing-business-as name |
| `entity_type` | ENUM | LLC, Corporation, LP, LLP, Trust, Individual |
| `ein_encrypted` | BYTEA | AES-256 encrypted federal tax ID |
| `duns_number` | VARCHAR(9) | D&B identifier |
| `state_of_formation` | CHAR(2) | US state abbreviation |
| `formation_date` | DATE | Legal formation date |
| `status` | ENUM | prospect, active, completed, suspended |
| `company_code` | VARCHAR(10) | Tenant identifier |

### Onboarding Session

**Schema:** `onboarding.sessions`

Tracks the state of a single partner's 7-step onboarding progression.

| Attribute | Type | Description |
|-----------|------|-------------|
| `session_id` | UUID | Primary key |
| `partner_id` | UUID | FK → onboarding.partners |
| `current_step` | INT | 1–7 |
| `status` | ENUM | active, in_review, final_review, completed, overdue |
| `target_completion_date` | DATE | Expected completion |
| `completed_steps` | JSONB | Step-level completion flags |
| `assigned_analyst` | UUID | FK → admin_ilm.users |

### JOA Record

**Schema:** `onboarding.joa_records`

| Attribute | Type | Description |
|-----------|------|-------------|
| `joa_id` | UUID | Primary key |
| `partner_id` | UUID | FK → onboarding.partners |
| `effective_date` | DATE | JOA effective date |
| `basin` | VARCHAR(100) | Geographic basin |
| `product_type` | ENUM | oil, gas, ngl, mixed |
| `wi_percentage` | DECIMAL(10,8) | Working Interest decimal |
| `nri_percentage` | DECIMAL(10,8) | Net Revenue Interest decimal |
| `governing_law_state` | CHAR(2) | Jurisdiction |

### Onboarding Document

**Schema:** `onboarding.documents`

| Attribute | Type | Description |
|-----------|------|-------------|
| `document_id` | UUID | Primary key |
| `session_id` | UUID | FK → onboarding.sessions |
| `document_type` | ENUM | w9, bank_verification, joa, copas_ack, tax_id |
| `status` | ENUM | pending, received, verified, complete, overdue |
| `required_date` | DATE | Deadline for receipt |
| `received_date` | DATE | Actual receipt date |
| `verified_by` | UUID | Analyst who verified |

---

## Transactional Events

| Event | Published When | Consumed By |
|-------|---------------|-------------|
| `ua.onboarding.partner.completed` | Step 7 sign-off complete | M2 (DOI provisioning), M7 (payment config) |
| `ua.onboarding.document.overdue` | Document past required date | M9 (audit log, notifications) |

---

## Pipeline View

The Pipeline view provides a portfolio view of all active onboarding sessions:

- Status categorization: active / in review / final review / overdue
- Target date tracking with overdue flag
- Monthly completion trends chart
- Analyst workload distribution

---

## RBAC

| Scope | Permission |
|-------|-----------|
| `onboarding:read` | View partner records and session status |
| `onboarding:write` | Create and update onboarding sessions |
| `onboarding:approve` | Sign off steps and complete sessions |
| `onboarding:admin` | Manage document types and workflow configuration |

---

## Integration Points

| Module | Interaction |
|--------|------------|
| M2 — Ownership & DOI | Completed onboarding creates Business Partner and DOI owner interest records |
| M7 — Payment Processing | Financial setup (bank, payment method) pre-populates Owner Attribute Group |
| M9 — Administration & ILM | Audit log captures all step completions and document verifications |

---

## Reports

| Report | Description |
|--------|-------------|
| Onboarding Pipeline Summary | All active sessions with status and completion % |
| Overdue Partners Report | Sessions past target completion date |
| Document Status Report | Outstanding document collection by partner |
| Monthly Completion Trend | Partners completed per month |
