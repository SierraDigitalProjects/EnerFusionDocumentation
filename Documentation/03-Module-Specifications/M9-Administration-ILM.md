# M9 — Administration & Information Lifecycle Management

---

## 1. Module Purpose

Provides the platform-wide administrative layer and data governance engine. M9 is responsible for: user and role administration, automation rulesets (workflow bots), accounting period calendar management, information lifecycle management (ILM) archiving, personal data protection (blocking and anonymization for GDPR/CCPA compliance), audit log management, integration health monitoring, and onboarding validation of master data before a new venture or company code can go live.

This module is the **operational control plane** — it does not generate revenue or production data, but it governs access, enforces retention policies, maintains the integrity of the period calendar, and ensures the platform can be audited end to end.

---

## 2. Core Concepts

| Concept | Definition |
|---------|------------|
| **Automation Ruleset (Bot)** | A configurable rule that triggers automated actions (e.g., auto-approve, auto-suspend, auto-notify) when defined conditions are met, without requiring manual user intervention |
| **ILM Archive Policy** | A data retention and archiving rule that specifies how long transactional data is retained in the active database before being moved to cold storage or purged |
| **Accounting Period Calendar** | The master calendar defining open, closed, and future periods for each company code; controls which modules can post transactions |
| **Personal Data Record** | A catalog entry identifying where personally identifiable information (PII) is stored in the system, enabling blocking and anonymization workflows |
| **Data Blocking** | Restricting access to personal data for a specific individual upon request (GDPR right to restriction), without deleting the data |
| **Anonymization** | Irreversibly removing or pseudonymizing PII from archived records when the retention period has expired |
| **Onboarding Validation** | A checklist-driven workflow that verifies all required master data is configured before a new venture, company code, or regulatory scope is activated |
| **Integration Sync Log** | A record of each data synchronization event between EnerFusion and external systems (SAP FI, ONRR portals, bank ACH); used for troubleshooting and audit |
| **Audit Log** | An immutable, append-only record of every data change and access event; used for compliance and forensic review |

---

## 3. Master Data

### 3.1 Role Definition

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| roleId | UUID | Yes | PK |
| roleName | String(50) | Yes | Unique display name |
| roleDescription | String(200) | No | |
| moduleScopes | String[] | Yes | Array of permission scopes (e.g., `revenue:read`, `payments:execute`) |
| dataAccessLevel | Enum(company, venture, all) | Yes | Restricts data visibility |
| companyCodeRestrictions | String[] | No | If populated, role only sees these company codes |
| isSystemRole | Boolean | Yes | System roles cannot be deleted |
| createdBy | UUID | FK → User |
| effectiveDate | Date | Yes | |

### 3.2 User Role Assignment

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| assignmentId | UUID | Yes | PK |
| userId | UUID | Yes | FK → Azure AD user (via XSUAA claim) |
| roleId | UUID | Yes | FK → RoleDefinition |
| companyCode | String(10) | No | Scope-limiting override |
| assignedBy | UUID | FK → User (admin) |
| effectiveDate | Date | Yes | |
| expirationDate | Date | No | For temporary access grants |
| justification | String(500) | No | Reason for temporary access |

### 3.3 Accounting Period Calendar

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| calendarId | UUID | Yes | PK |
| companyCode | String(10) | Yes | |
| period | YearMonth | Yes | |
| periodStatus | Enum(future, open, soft_closed, hard_closed) | Yes | Controls posting eligibility |
| productionCloseDate | Date | No | When M1 production data is locked |
| valuationCloseDate | Date | No | When M4 settlement is locked |
| distributionCloseDate | Date | No | When M6 distribution run is finalized |
| paymentCloseDate | Date | No | When M7 payment run is finalized |
| regulatoryCloseDate | Date | No | When M8 reports are submitted |
| openedBy | UUID | FK → User |
| closedBy | UUID | FK → User |
| closedAt | Timestamp | |

**Period Status Transitions:**

```
future → open → soft_closed → hard_closed
                     ↑              ↑
                 (reopen)      (cannot reopen;
                  allowed)      requires admin override)
```

### 3.4 Automation Ruleset

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| rulesetId | UUID | Yes | PK |
| rulesetName | String(100) | Yes | |
| moduleTarget | Enum(M1,M2,M3,M4,M5,M6,M7,M8) | Yes | Module this rule applies to |
| triggerEvent | String(50) | Yes | Event or condition that fires the rule |
| conditionExpression | JSONB | Yes | Evaluated condition (field, operator, value) |
| actionType | Enum(auto_approve, auto_suspend, notify, escalate, reject) | Yes | |
| actionParameters | JSONB | Yes | Parameters for the action (e.g., target user, threshold) |
| priority | Integer | Yes | Evaluation order; lower = higher priority |
| isActive | Boolean | Yes | |
| effectiveDate | Date | Yes | |

**Example Automation Rulesets:**

| Ruleset | Trigger | Condition | Action |
|---------|---------|-----------|--------|
| Auto-suspend small owner | Payment run extraction | `netValue < 10.00` | `auto_suspend` with reason "Below threshold" |
| Auto-approve distribution runs | Distribution run completed | `validationErrorCount = 0 AND totalDistributed < 1000000` | `auto_approve` |
| Notify on late interest | Interest calculation | `overdueDays > 30` | `notify` Revenue Accountant |
| Escalate stale AR | AR aging | `arAge > 365` | `escalate` to Finance Director |
| Auto-approve DOI transfer | DOI transfer submitted | `decimalInterestChange < 0.001` | `auto_approve` |

### 3.5 ILM Archive Policy

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| policyId | UUID | Yes | PK |
| policyName | String(100) | Yes | |
| targetEntity | String(100) | Yes | Database table or entity name |
| retentionActiveYears | Integer | Yes | Years in active PostgreSQL DB |
| retentionColdYears | Integer | Yes | Years in Azure Blob cold storage |
| retentionTotalYears | Integer | Yes | Total retention before deletion |
| archiveMethod | Enum(pg_partitioning, blob_export, logical_delete) | Yes | |
| purgable | Boolean | Yes | Whether records can be permanently deleted |
| legalHoldOverride | Boolean | Yes | If true, legal hold blocks purge |
| regulatoryBasis | String(200) | No | Legal/regulatory authority for retention period |

**Recommended Retention Schedule:**

| Entity | Active (years) | Cold (years) | Total | Basis |
|--------|---------------|-------------|-------|-------|
| Settlement Statements | 3 | 4 | 7 | IRS / state tax |
| Owner Distribution Records | 3 | 4 | 7 | IRS / state |
| ONRR-2014 Records | 5 | 2 | 7 | ONRR regulation |
| Payment Records | 3 | 4 | 7 | IRS / UCC |
| Audit Logs | 5 | 5 | 10 | SOX / internal policy |
| DOI History | 10 | 10 | 20 | Chain of title |
| Production Volume Records | 3 | 4 | 7 | State conservation |
| Escheat Records | 5 | 20 | 25 | State unclaimed property law |

### 3.6 Personal Data Record

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| personalDataId | UUID | Yes | PK |
| entityType | String(50) | Yes | Table or entity containing PII |
| entityId | UUID | Yes | FK → the specific record |
| businessPartnerId | UUID | Yes | FK → the data subject |
| dataCategories | String[] | Yes | Types of PII (e.g., name, tax_id, bank_account, address) |
| processingPurpose | String(200) | Yes | Legal basis for holding the data |
| blockingStatus | Enum(active, blocked, anonymized) | Yes | |
| blockedAt | Timestamp | No | |
| blockedBy | UUID | No | FK → User |
| anonymizedAt | Timestamp | No | |
| retentionExpiryDate | Date | Yes | When data may be purged |

---

## 4. Transactional Entities

### 4.1 Audit Log Entry

| Attribute | Type | Description |
|-----------|------|-------------|
| auditId | UUID | PK (append-only; no updates or deletes) |
| eventTimestamp | Timestamp | When the event occurred |
| userId | UUID | Acting user (from JWT claim) |
| sessionId | String(50) | Azure AD session identifier |
| traceId | String(50) | OpenTelemetry trace ID |
| moduleCode | String(5) | M1–M9 |
| entityType | String(50) | Table or entity affected |
| entityId | UUID | Record affected |
| action | Enum(create, update, delete, read, approve, reject, submit, archive, block, anonymize) | |
| fieldChanges | JSONB | Before/after values for each changed field (null for read/approve events) |
| ipAddress | String(45) | Requester IP (IPv4 or IPv6) |
| userAgent | String(200) | Browser / client identifier |
| companyCode | String(10) | |
| outcome | Enum(success, failure, denied) | |

### 4.2 Onboarding Validation Checklist

| Attribute | Type | Description |
|-----------|------|-------------|
| checklistId | UUID | PK |
| checklistType | Enum(new_venture, new_company_code, new_regulatory_scope) | |
| companyCode | String(10) | |
| ventureId | UUID | FK → Venture (if applicable) |
| checklistStatus | Enum(in_progress, passed, failed, waived) | |
| createdAt | Timestamp | |
| completedAt | Timestamp | |
| approvedBy | UUID | FK → User |

### 4.3 Onboarding Check Item

| Attribute | Type | Description |
|-----------|------|-------------|
| checkItemId | UUID | PK |
| checklistId | UUID | FK → OnboardingValidationChecklist |
| checkCode | String(20) | Unique check identifier (e.g., DOI-BALANCE, TAX-RATE-CONFIG) |
| checkDescription | String(200) | Human-readable description |
| checkCategory | Enum(master_data, doi_integrity, tax_config, payment_config, regulatory_config) | |
| checkStatus | Enum(pending, passed, failed, waived) | |
| checkResult | JSONB | Detailed result (errors found, records checked) |
| waiveReason | String(300) | Required if waived |
| waivedBy | UUID | FK → User |

**Standard Onboarding Check Codes:**

| Check Code | Category | Description |
|------------|----------|-------------|
| DOI-BALANCE | doi_integrity | All DOI interests for venture sum to 1.0 |
| DOI-OWNER-EXISTS | doi_integrity | All owner BP IDs resolve to active Business Partners |
| TAX-RATE-CONFIG | tax_config | Severance tax rates configured for venture's state(s) |
| PAY-METHOD-ALL-OWNERS | payment_config | All owners have valid payment method or explicit suspense reason |
| MIN-THRESHOLD-CONFIG | payment_config | Minimum payment threshold configured per owner type |
| ONRR-LEASE-CONFIG | regulatory_config | ONRR lease hierarchy configured for federal/Indian leases |
| ONRR-PPN-ASSIGNED | regulatory_config | PPNs assigned for all federal lease + product combinations |
| GL-MAPPING-COMPLETE | master_data | GL account mapping exists for all product × transaction types |
| INT-RATE-CONFIG | master_data | Late-payment interest rates configured for applicable states |
| ESCHEAT-CONFIG | payment_config | Escheat configuration set for applicable states |

### 4.4 Integration Sync Log

| Attribute | Type | Description |
|-----------|------|-------------|
| syncLogId | UUID | PK |
| integrationName | String(50) | e.g., SAP_BP_REPLICATION, ONRR_SUBMISSION, ACH_TRANSMIT |
| direction | Enum(inbound, outbound) | |
| syncTimestamp | Timestamp | |
| recordCount | Integer | Records processed |
| successCount | Integer | |
| failureCount | Integer | |
| syncStatus | Enum(success, partial, failed) | |
| errorDetail | JSONB | Error messages for failed records |
| externalReference | String(100) | Confirmation or transaction ID from external system |
| retriesAttempted | Integer | Number of retry attempts |

### 4.5 Archive Run

| Attribute | Type | Description |
|-----------|------|-------------|
| archiveRunId | UUID | PK |
| policyId | UUID | FK → ILMArchivePolicy |
| period | YearMonth | Period being archived |
| runStatus | Enum(pending, running, completed, failed) | |
| recordsArchived | Integer | |
| archiveLocation | String(500) | Azure Blob Storage URI |
| startedAt | Timestamp | |
| completedAt | Timestamp | |
| triggeredBy | Enum(scheduled, manual) | |

---

## 5. Business Processing Rules

### 5.1 Period Close Sequence

```
FUNCTION closeAccountingPeriod(companyCode, period):
  calendar = fetch AccountingPeriodCalendar for companyCode + period
  IF calendar.periodStatus != 'open':
    THROW "Period is not open for closing"
  
  // Validate all prerequisite steps are complete
  CHECK: M1 production allocations finalized (productionCloseDate SET)
  CHECK: M4 settlement statements all 'posted' (valuationCloseDate SET)
  CHECK: M6 distribution run completed (distributionCloseDate SET)
  CHECK: M7 payment run finalized (paymentCloseDate SET)
  CHECK: M8 regulatory reports submitted (regulatoryCloseDate SET)
  
  IF any check fails:
    RETURN list of blocking items; do NOT close
  
  // Soft close first — blocks new postings, allows corrections
  UPDATE calendar SET periodStatus = 'soft_closed'
  
  // After accountant review window (configurable days):
  UPDATE calendar SET periodStatus = 'hard_closed', closedBy = userId, closedAt = now()
  
  PUBLISH event: ua.admin.period.closed
```

### 5.2 Automation Ruleset Evaluation Engine

```
FUNCTION evaluateRulesets(moduleTarget, triggerEvent, record):
  rulesets = fetch active Automation Rulesets WHERE
    moduleTarget = moduleTarget AND triggerEvent = triggerEvent
  ORDER BY priority ASC
  
  FOR each ruleset:
    conditionMet = evaluateCondition(ruleset.conditionExpression, record)
    
    IF conditionMet:
      EXECUTE action(ruleset.actionType, ruleset.actionParameters, record)
      
      // Log the automated action to audit log
      PERSIST AuditLogEntry (action = ruleset.actionType, entityId = record.id)
      
      IF ruleset.actionType IN ('auto_approve', 'reject'):
        BREAK  // Stop evaluating further rules
```

### 5.3 ILM Archiving Workflow

```
FUNCTION runArchivePolicy(policyId, period):
  policy = fetch ILMArchivePolicy
  cutoffDate = period.lastDay - policy.retentionActiveYears years
  
  // Identify records eligible for archiving
  eligibleRecords = SELECT * FROM policy.targetEntity
    WHERE periodDate <= cutoffDate
    AND legalHold = FALSE
  
  // Check for personal data records — anonymize before archiving
  FOR each record in eligibleRecords:
    personalDataRecords = fetch PersonalDataRecord WHERE entityId = record.id
    IF personalDataRecords exist AND record.date > retentionExpiryDate:
      anonymize(record, personalDataRecords)
  
  // Export to Azure Blob cold storage (Parquet format)
  archiveFile = exportToBlob(eligibleRecords, policy.archiveMethod)
  
  // Delete from active database after successful export
  IF archiveFile.checksum VERIFIED:
    DELETE from policy.targetEntity WHERE id IN eligibleRecords.ids
    PERSIST ArchiveRun (archiveLocation = archiveFile.uri, recordsArchived = count)
```

### 5.4 Personal Data Blocking and Anonymization

```
FUNCTION blockPersonalData(businessPartnerId, requestReason):
  // Find all PII records for this data subject
  records = fetch PersonalDataRecord WHERE businessPartnerId = businessPartnerId
    AND blockingStatus = 'active'
  
  FOR each record:
    UPDATE PersonalDataRecord SET blockingStatus = 'blocked', blockedAt = now()
    // Restrict API access — RLS policy will enforce blockingStatus
    
    PERSIST AuditLogEntry (action = 'block', entityId = record.personalDataId)
  
  NOTIFY data protection officer

FUNCTION anonymizePersonalData(businessPartnerId):
  records = fetch PersonalDataRecord WHERE businessPartnerId = businessPartnerId
    AND retentionExpiryDate <= today()
  
  FOR each record:
    entity = fetch entity(record.entityType, record.entityId)
    
    // Replace PII fields with pseudonymous tokens
    IF 'name' IN record.dataCategories:
      entity.ownerName = HASH(entity.ownerName + salt)
    IF 'tax_id' IN record.dataCategories:
      entity.taxId = '[ANONYMIZED]'
    IF 'bank_account' IN record.dataCategories:
      entity.bankAccount = '[ANONYMIZED]'
    IF 'address' IN record.dataCategories:
      entity.mailingAddress = NULL
    
    PERSIST entity
    UPDATE PersonalDataRecord SET blockingStatus = 'anonymized', anonymizedAt = now()
```

### 5.5 Onboarding Validation Workflow

```
FUNCTION runOnboardingValidation(checklistId):
  checklist = fetch OnboardingValidationChecklist
  checkItems = fetch OnboardingCheckItem WHERE checklistId = checklistId
  
  FOR each checkItem WHERE checkStatus = 'pending':
    result = executeCheck(checkItem.checkCode, checklist.ventureId, checklist.companyCode)
    
    UPDATE checkItem SET
      checkStatus = result.passed ? 'passed' : 'failed',
      checkResult = result.detail
  
  failedItems = filter checkItems WHERE checkStatus = 'failed'
  
  IF failedItems IS EMPTY:
    UPDATE checklist SET checklistStatus = 'passed'
    // Venture/company code may now be activated
  ELSE:
    UPDATE checklist SET checklistStatus = 'failed'
    NOTIFY onboarding owner with failedItems list
    // Activation blocked until all items pass or are waived
```

---

## 6. RBAC Administration UI

The M9 administration interface provides:

| Function | Description | Required Scope |
|----------|-------------|----------------|
| Role Management | Create, edit, deactivate roles; assign permission scopes | `admin:roles` |
| User Assignment | Assign/revoke roles; set effective dates; view current assignments | `admin:users` |
| Period Calendar | Open/soft-close/hard-close accounting periods; view close sequence | `admin:calendar` |
| Automation Rulesets | Create/edit/activate/deactivate rulesets; test condition evaluation | `admin:automation` |
| Archive Policy | View and manage ILM archive policies; trigger manual archive runs | `admin:ilm` |
| Personal Data | View blocking status; initiate blocking/anonymization; DSAR tracking | `admin:personal_data` |
| Audit Log Viewer | Search and export audit log entries (read-only) | `admin:audit_read` |
| Integration Monitor | View sync logs, failure rates, retry status per integration | `admin:integrations` |
| Onboarding Validator | Create checklists, run validations, waive items with justification | `admin:onboarding` |

---

## 7. API Endpoints

| Method | Path | Description | Auth Scope |
|--------|------|-------------|-----------|
| GET | `/api/v1/admin/roles` | List role definitions | `admin:roles` |
| POST | `/api/v1/admin/roles` | Create role | `admin:roles` |
| PUT | `/api/v1/admin/roles/{id}` | Update role | `admin:roles` |
| DELETE | `/api/v1/admin/roles/{id}` | Deactivate role | `admin:roles` |
| GET | `/api/v1/admin/users/{userId}/assignments` | User's current role assignments | `admin:users` |
| POST | `/api/v1/admin/users/{userId}/assignments` | Assign role to user | `admin:users` |
| DELETE | `/api/v1/admin/users/{userId}/assignments/{id}` | Revoke role | `admin:users` |
| GET | `/api/v1/admin/calendar/{companyCode}` | Period calendar for company | `admin:calendar` |
| PUT | `/api/v1/admin/calendar/{companyCode}/{period}/open` | Open a period | `admin:calendar` |
| PUT | `/api/v1/admin/calendar/{companyCode}/{period}/soft-close` | Soft-close a period | `admin:calendar` |
| PUT | `/api/v1/admin/calendar/{companyCode}/{period}/hard-close` | Hard-close a period | `admin:calendar` |
| GET | `/api/v1/admin/automation/rulesets` | List automation rulesets | `admin:automation` |
| POST | `/api/v1/admin/automation/rulesets` | Create ruleset | `admin:automation` |
| PUT | `/api/v1/admin/automation/rulesets/{id}` | Update ruleset | `admin:automation` |
| POST | `/api/v1/admin/automation/rulesets/{id}/test` | Test ruleset against sample record | `admin:automation` |
| GET | `/api/v1/admin/ilm/policies` | List archive policies | `admin:ilm` |
| POST | `/api/v1/admin/ilm/runs` | Trigger manual archive run | `admin:ilm` |
| GET | `/api/v1/admin/ilm/runs/{id}` | Archive run status | `admin:ilm` |
| GET | `/api/v1/admin/personal-data/{bpId}` | Personal data catalog for BP | `admin:personal_data` |
| POST | `/api/v1/admin/personal-data/{bpId}/block` | Block personal data | `admin:personal_data` |
| POST | `/api/v1/admin/personal-data/{bpId}/anonymize` | Anonymize personal data | `admin:personal_data` |
| GET | `/api/v1/admin/audit-log` | Search audit log | `admin:audit_read` |
| GET | `/api/v1/admin/integrations/sync-log` | Integration sync history | `admin:integrations` |
| GET | `/api/v1/admin/onboarding/checklists` | List onboarding checklists | `admin:onboarding` |
| POST | `/api/v1/admin/onboarding/checklists` | Create checklist | `admin:onboarding` |
| POST | `/api/v1/admin/onboarding/checklists/{id}/run` | Execute validation checks | `admin:onboarding` |
| PUT | `/api/v1/admin/onboarding/checklists/{id}/items/{itemId}/waive` | Waive a check item | `admin:onboarding` |

---

## 8. Reports

| Report | Description | Trigger |
|--------|-------------|---------|
| User Access Report | All current role assignments with effective dates; highlights temporary access | On demand / quarterly |
| Role Permission Matrix | Cross-reference of roles vs. permission scopes | On demand |
| Period Close Status | Current status of all open periods; blocking items for unclosed periods | Daily / on demand |
| Automation Ruleset Activity | Rules fired, actions taken, and records affected per period | Monthly / on demand |
| Archive Run Summary | Records archived per policy, storage location, and verification status | Monthly |
| Data Retention Calendar | Upcoming archive and purge dates by entity type | On demand |
| Personal Data Subject Report | All PII records for a given business partner; used for DSAR response | On demand |
| Blocked/Anonymized Data Report | Data subjects with active blocks or completed anonymization | On demand |
| Audit Log Export | Filtered export of audit log entries (by user, module, date range, entity) | On demand / compliance |
| Integration Health Report | Sync success/failure rates by integration; retry history | Daily / on demand |
| Onboarding Validation Report | Checklist results; passed, failed, and waived items by venture/company code | On demand |
| Stale Access Report | Users with role assignments past their expiration date or not logged in > 90 days | Weekly |
