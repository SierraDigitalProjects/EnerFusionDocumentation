# Centrally Managed in SAP

## Definition

**Centrally Managed** objects are master data records where **SAP ERP is the authoritative system of record**. EnerFusion Upstream Accounting consumes these records as read-only replicas. Operators must maintain these objects in SAP; changes are propagated to EnerFusion through the integration layer.

This model avoids duplicate maintenance, ensures financial consistency between the upstream platform and the corporate ERP, and keeps identity (who is a valid business partner or GL account) centralised.

---

## Business Partner (Vendor / Owner / Purchaser)

**SAP Object:** Business Partner (BP) — T-code: BP  
**SAP Table:** BUT000, BUT020, BUT100  
**EnerFusion Schema:** `shared.business_partners`

### Attributes Mastered in SAP

| Attribute | SAP Field | EnerFusion Usage |
|-----------|-----------|-----------------|
| Business Partner ID | BP Number | FK in M2 DOI, M7 Payments |
| Legal Name | Name1 / Name2 | Display in DOI lists, check payee |
| Tax ID (EIN / SSN) | Tax number 1 | Encrypted; 1099 / withholding |
| Partner Type | Category (Person / Org) | Determines withholding logic |
| Mailing Address | Standard address | Owner statement mailing |
| Payment Address | Alternative address | Override for payment disbursement |
| Bank Details (ACH) | Bank Account | ACH payment routing |
| CDEX ID | External ID | EDI 820 purchaser matching |
| Active / Blocked status | Central block | Prevents payment disbursement |

### Replication Rules

```
Trigger: BP change in SAP (BDCP2 change pointer or IDoc ADRMAS)
Method:  SAP Integration Suite → Azure API Management → EnerFusion REST
Endpoint: POST /api/v1/shared/business-partners
          PUT  /api/v1/shared/business-partners/{id}
Frequency: Near-real-time (event-driven preferred) | nightly batch fallback
Conflict:  SAP wins; EnerFusion rejects local updates to SAP-managed fields
```

### What EnerFusion Can Add Locally

The following fields are maintained only in EnerFusion and are not pushed back to SAP:

- `escheatState` — US state for unclaimed property reporting
- `ownerAttributeGroupId` — Payment threshold, withholding %, preferred payment method
- `cdexId` — CDEX/EDI integration identifier (if not in SAP BP)

---

## GL Account / Chart of Accounts (PRA Accounts)

**SAP Object:** GL Account (FS00)  
**SAP Table:** SKA1 (chart of accounts), SKB1 (company code)  
**EnerFusion Schema:** `shared.gl_accounts`

### Attributes Mastered in SAP

| Attribute | SAP Field | EnerFusion Usage |
|-----------|-----------|-----------------|
| GL Account Number | SAKNR | RAD posting reference |
| Account Description | TXT50 | Display in M6 distribution config |
| Account Type | KTOKS | Revenue vs. tax vs. deduction |
| Posting Block | XSPEB | Prevents inadvertent posting |

### Replication Rules

```
Trigger: GL Account created or changed in SAP (GLMAST IDoc)
Endpoint: POST /api/v1/shared/gl-accounts
Frequency: Nightly batch (GL changes are low-frequency)
```

EnerFusion uses GL account codes as foreign keys in Revenue Accounting Documents (RADs). The platform does **not** post directly to SAP; it produces a RAD document that the integration layer passes to `BAPI_ACC_DOCUMENT_POST`.

---

## Cost Center

**SAP Object:** Cost Center (KS01)  
**SAP Table:** CSKS  
**EnerFusion Schema:** `shared.cost_centers`

Cost centers are referenced when generating RADs so that GL postings carry the correct organizational cost assignment.

| Attribute | SAP Field | EnerFusion Usage |
|-----------|-----------|-----------------|
| Cost Center ID | KOSTL | RAD line item cost object |
| Description | KTEXT | Display label |
| Valid From / To | DATAB / DATBI | Date-effective check on RAD generation |
| Company Code | BUKRS | Tenant scoping |

---

## User Identity (Azure AD)

**System of Record:** Azure Active Directory (Entra ID)  
**EnerFusion Consumption:** OIDC/JWT claims validated per request

While not a SAP-managed object in all environments, **user identity and group membership** originate from Azure AD and are consumed by EnerFusion as authoritative. EnerFusion does not store passwords or provision users directly.

| Claim | Source | EnerFusion Usage |
|-------|--------|-----------------|
| `oid` | Azure AD | Immutable user identifier |
| `preferred_username` | Azure AD | Display in audit logs |
| `roles[]` | App Role assignment in Azure AD | RBAC permission mapping |
| `extension_companyCode` | Azure AD custom attribute | Tenant scoping in RLS |

### Role → Permission Mapping

EnerFusion maps Azure AD app roles to module-level permissions. Role assignments are made in Azure AD by the system administrator and cannot be modified within EnerFusion's own UI.

| Azure AD App Role | EnerFusion Modules | Permission Level |
|------------------|--------------------|-----------------|
| `PRA.Production.Editor` | M1 | `production:write` |
| `PRA.Ownership.Approver` | M2 | `ownership:approve` |
| `PRA.Valuation.Runner` | M4 | `valuation:execute` |
| `PRA.Compliance.Officer` | M8 | `regulatory:read`, `regulatory:submit` |
| `PRA.SystemAdmin` | All | `admin` on all modules |

---

## Integration Architecture (SAP → EnerFusion)

```
┌─────────────────────────────────────────────────────────────┐
│  SAP ERP (Source of Truth)                                  │
│                                                             │
│  Business Partner │ GL Account │ Cost Center               │
└───────────────────┬─────────────────────────────────────────┘
                    │ IDoc / Change Pointer / RFC
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  Azure Integration Layer                                    │
│  (Logic Apps or SAP Integration Suite)                      │
│                                                             │
│  Transform → Validate → Forward                             │
└───────────────────┬─────────────────────────────────────────┘
                    │ HTTPS REST API
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  EnerFusion — Azure API Management                          │
│  → svc-admin (M9) — /api/v1/shared/*                       │
│  → shared.* PostgreSQL schema (read-only in app layer)      │
└─────────────────────────────────────────────────────────────┘
```

---

## Governance Rules

1. **No local edits** — EnerFusion UI renders centrally managed fields as read-only. Attempting a `PUT` on SAP-managed fields via API returns `HTTP 422 Unprocessable Entity` with an RFC 7807 error body.
2. **Audit on sync** — Every replication event is written to `admin_ilm.integration_sync_log` with timestamp, source system, record ID, and changed fields.
3. **Orphan detection** — A nightly job in M9 flags EnerFusion records whose SAP BP has been archived; these are surfaced in the Admin UI for cleanup.
4. **No delete propagation** — SAP BP archiving does not automatically delete EnerFusion records. The Admin user must explicitly block or archive the partner in EnerFusion to prevent payment disbursement.
