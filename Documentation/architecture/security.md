# Security Architecture

## Security Layers

EnerFusion Upstream Accounting enforces security at five distinct layers. A request must clear all applicable layers before data is returned or mutated.

```
Browser / API Client
       │
       ▼
[1] Azure AD — Identity & Token Issuance
       │  Access token (JWT, 1hr TTL)
       ▼
[2] Azure API Management — Token Validation + Rate Limiting
       │  Validated claims forwarded to backend
       ▼
[3] Service Middleware — RBAC + Scope Enforcement
       │  Module × action × companyCode check
       ▼
[4] Application Layer — Business Rule Validation (Zod)
       │  Schema-valid, sanitized input
       ▼
[5] PostgreSQL — Row-Level Security (RLS)
       │  company_code isolation enforced at data layer
       ▼
Data returned / mutation applied
```

---

## Identity & Access Management

### Azure Active Directory (Entra ID)

All users authenticate via Azure AD using OAuth2 Authorization Code + PKCE. Roles are assigned as Azure AD App Roles, not managed within EnerFusion.

**Token Claims Used:**

| Claim | Usage |
|-------|-------|
| `oid` | Immutable user identifier in audit logs |
| `preferred_username` | Display name in UI, log entries |
| `roles[]` | Mapped to EnerFusion module permissions |
| `extension_companyCode` | Tenant scoping; set in PostgreSQL session |
| `exp` | Token expiry enforced by APIM |

**App Role → Module Scope Mapping:**

```typescript
// shared/auth/rolePermissionMap.ts
export const ROLE_PERMISSION_MAP: Record<string, ModulePermission[]> = {
  'PRA.Production.Reader':   [{ module: 'production',  action: 'read'    }],
  'PRA.Production.Editor':   [{ module: 'production',  action: 'write'   },
                              { module: 'production',  action: 'read'    }],
  'PRA.Production.Runner':   [{ module: 'production',  action: 'execute' }],
  'PRA.Ownership.Editor':    [{ module: 'ownership',   action: 'write'   }],
  'PRA.Ownership.Approver':  [{ module: 'ownership',   action: 'approve' }],
  'PRA.Valuation.Runner':    [{ module: 'valuation',   action: 'execute' }],
  'PRA.Compliance.Officer':  [{ module: 'regulatory',  action: 'read'    },
                              { module: 'regulatory',  action: 'submit'  }],
  'PRA.SystemAdmin':         ['*'], // All modules, all actions
};
```

---

## RBAC Middleware

Every API route declares its required permission. The `rbac.middleware.ts` validates the decoded JWT roles against the permission map cached in Redis.

```typescript
// api/middleware/rbac.middleware.ts
export const requirePermission = (module: string, action: string) =>
  async (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    const cacheKey = `perms:${req.user.oid}:${req.user.companyCode}`;
    let permissions = await redis.get(cacheKey);

    if (!permissions) {
      permissions = buildPermissions(req.user.roles);
      await redis.setex(cacheKey, 300, JSON.stringify(permissions)); // 5min TTL
    }

    const allowed = permissions.some(
      (p) => p.module === module && p.action === action
    );

    if (!allowed) {
      return next(new ForbiddenError(`${module}:${action} access denied`));
    }

    // Set PostgreSQL session variable for RLS
    await db.raw(`SET LOCAL app.company_code = ?`, [req.user.companyCode]);
    next();
  };
```

---

## Secrets Management

All secrets are stored in Azure Key Vault and mounted into pods using the CSI Secret Store driver. No secrets appear in Helm values files or environment variable definitions.

```yaml
# helm/svc-production/templates/deployment.yaml
volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: pra-production-secrets

# SecretProviderClass references Key Vault secrets
# Mounted at /mnt/secrets/ in the container
```

**Key Vault Secret Naming Convention:**

| Secret Name | Consumed By | Description |
|------------|-------------|-------------|
| `pra-production-db-url` | svc-production | PostgreSQL write connection string |
| `pra-production-db-read-url` | svc-production | PostgreSQL read replica URL |
| `pra-servicebus-connection` | All services | Azure Service Bus connection string |
| `pra-redis-connection` | All services | Redis connection string |
| `pra-jwt-audience` | auth-svc | Expected JWT audience claim |

---

## Network Security

### Kubernetes Network Policies

Every namespace has a default-deny policy. Services are granted explicit ingress allows.

```yaml
# Default deny all ingress in pra-prod namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: pra-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
# Allow NGINX ingress to reach BFF only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-bff
  namespace: pra-prod
spec:
  podSelector:
    matchLabels:
      app: bff
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
```

---

### mTLS (Service Mesh)

All pod-to-pod communication within the AKS cluster is encrypted via mTLS using Linkerd (or Istio). Services do not need to implement TLS certificate management themselves.

---

### Azure APIM WAF

Azure API Management is configured with an Azure Web Application Firewall (WAF) policy in Prevention mode:

- OWASP Core Rule Set 3.2
- Rate limiting: 1,000 requests/minute per client IP
- JWT validation on all routes
- IP allowlist for SCADA and SAP integration endpoints

---

## Data Security

### Column-Level Encryption for PII

Sensitive personal data fields are encrypted at the application layer before storage:

```typescript
// infrastructure/encryption/fieldEncryption.ts
import { KeyClient, CryptographyClient } from '@azure/keyvault-keys';

export async function encryptField(plaintext: string): Promise<string> {
  const result = await cryptoClient.encrypt('RSA-OAEP', Buffer.from(plaintext));
  return result.result.toString('base64');
}

export async function decryptField(ciphertext: string): Promise<string> {
  const result = await cryptoClient.decrypt(
    'RSA-OAEP',
    Buffer.from(ciphertext, 'base64')
  );
  return result.result.toString('utf-8');
}
```

Fields encrypted: `tax_id`, `bank_account_number`, `bank_routing_number`.

---

### PostgreSQL Row-Level Security

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE production.well_completions ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see rows for their company_code
CREATE POLICY well_completion_company_isolation
ON production.well_completions
USING (company_code = current_setting('app.company_code', TRUE));

-- Service role bypasses RLS for admin/migration operations
ALTER TABLE production.well_completions FORCE ROW LEVEL SECURITY;
GRANT SELECT, INSERT, UPDATE, DELETE ON production.well_completions TO pra_app_role;
-- pra_admin_role is granted BYPASSRLS for migration purposes only
```

---

## Audit Logging

Every data mutation is written to `admin_ilm.audit_log` before the transaction commits.

```sql
CREATE TABLE admin_ilm.audit_log (
  audit_id        UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  event_time      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  user_id         UUID NOT NULL,
  user_name       VARCHAR(200),
  company_code    VARCHAR(10) NOT NULL,
  module          VARCHAR(50) NOT NULL,
  table_name      VARCHAR(100) NOT NULL,
  record_id       UUID NOT NULL,
  action          VARCHAR(10) NOT NULL,  -- INSERT, UPDATE, DELETE
  before_value    JSONB,
  after_value     JSONB,
  correlation_id  VARCHAR(100),
  source_ip       INET
);

-- Partition by month for retention management
-- Append-only: no UPDATE or DELETE on audit_log
REVOKE UPDATE, DELETE ON admin_ilm.audit_log FROM pra_app_role;
```

---

## Container Image Security

CI pipeline gates on Trivy scan results before any image is pushed to ACR:

```yaml
# .github/workflows/ci.yml
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE_NAME }}:${{ github.sha }}
    format: sarif
    exit-code: 1             # Fail CI on CRITICAL or HIGH findings
    severity: CRITICAL,HIGH
    output: trivy-results.sarif
```

Multistage Dockerfile minimizes attack surface:

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npx tsc --noEmit && npm run build

# Stage 2: Runtime — no build tools, no dev dependencies
FROM node:20-alpine AS runtime
WORKDIR /app
RUN addgroup -S pra && adduser -S pra -G pra
COPY --from=builder --chown=pra:pra /app/dist ./dist
COPY --from=builder --chown=pra:pra /app/node_modules ./node_modules
USER pra
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## OWASP Controls Summary

| OWASP Risk | Control Applied |
|-----------|----------------|
| A01 Broken Access Control | RBAC middleware + PostgreSQL RLS + APIM JWT validation |
| A02 Cryptographic Failures | AES-256 column encryption for PII; CMK at rest; TLS in transit |
| A03 Injection | Zod input validation; Knex parameterized queries; no raw SQL in controllers |
| A05 Security Misconfiguration | Default-deny NetworkPolicy; no `latest` image tags; Key Vault for secrets |
| A06 Vulnerable Components | Trivy scan in CI gates on CRITICAL/HIGH CVEs |
| A07 Auth Failures | Short-lived JWT (1hr); PKCE; MSAL.js token refresh; Redis permission cache TTL |
| A09 Logging Failures | Structured audit log (append-only); OpenTelemetry traces; Azure Monitor alerts |
