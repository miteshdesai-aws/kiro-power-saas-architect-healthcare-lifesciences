# Tenant Isolation Strategies

## Why Isolation Is Non-Negotiable

Tenant isolation is the most critical security concern in multi-tenant SaaS. Without it, one tenant can access another tenant's data — whether through a bug, a misconfiguration, or a malicious actor. The AWS SaaS Tenant Isolation Strategies whitepaper is clear: isolation must be enforced at the infrastructure level, not just the application level. Application-level checks ("if tenant_id == current_tenant") are necessary but insufficient — they are one bug away from a cross-tenant data breach.

The goal: even if application code has a bug, the underlying infrastructure prevents cross-tenant access.

## Isolation Models by Tenancy Type

### Silo Isolation (Dedicated Resources)

The simplest isolation model. Each tenant's resources are in separate infrastructure (separate accounts, separate VPCs, separate database instances). Isolation is enforced by resource boundaries.

**Implementation patterns:**
- **Account-per-tenant**: Each tenant gets their own AWS account via AWS Organizations. SCPs provide guardrails. This is the strongest isolation — IAM boundaries are at the account level.
- **VPC-per-tenant**: Tenants share an account but have separate VPCs. Network isolation via security groups and NACLs.
- **Resource-per-tenant**: Separate DynamoDB tables, separate RDS instances, separate S3 buckets per tenant.

**Isolation enforcement:** IAM policies attached to compute roles restrict access to only that tenant's resources. Since resources are dedicated, the policies are straightforward.

**When to use:** Enterprise tier, regulated industries, customers who contractually require dedicated infrastructure.

### Pool Isolation (Shared Resources)

The most complex isolation model. All tenants share the same resources. Isolation must be enforced at runtime through IAM policies, application logic, and database-level controls.

**Implementation patterns:**

#### Runtime IAM Policy Generation (Dynamic Policies)
The recommended AWS approach for pool isolation. Instead of creating static IAM policies per tenant (which hits IAM limits at scale), generate scoped credentials at runtime:

1. Request arrives with JWT containing tenant ID
2. Application extracts tenant context from JWT
3. Application calls STS AssumeRole with a session policy scoped to the current tenant
4. The scoped credentials only allow access to the current tenant's data
5. All subsequent AWS SDK calls use these scoped credentials

**Example flow for DynamoDB pool isolation:**
- Base role has permission to access the shared DynamoDB table
- Session policy adds a condition: `dynamodb:LeadingKeys` must match the tenant's partition key prefix
- Even if application code has a bug, the IAM policy prevents reading another tenant's rows

#### Token Vending Machine (TVM)
A service (typically a Lambda function) that vends tenant-scoped credentials. The application calls the TVM with the tenant context, and the TVM returns short-lived credentials scoped to that tenant.

**When to use TVM vs. inline STS:**
- TVM: when you want to centralize credential generation logic, when multiple services need scoped credentials, when you want to cache and reuse credentials
- Inline STS: simpler for single-service scenarios, fewer moving parts

#### Application-Level Isolation (Necessary but Insufficient)
Every database query includes a tenant ID filter. Every API response is filtered by tenant. This is the baseline — but it must be backed by IAM or database-level enforcement.

**Never rely solely on application-level isolation.** It's one missed WHERE clause away from a data leak.

### Bridge Isolation (Mixed)

Combines silo and pool patterns. Typically: shared compute with pool-style IAM isolation, plus dedicated storage with silo-style resource boundaries.

**Example:** Lambda functions are shared (pool), but each tenant has their own DynamoDB table (silo). The Lambda role can only access the current tenant's table via a session policy.

## IAM-Based Isolation Patterns

### Static Policies (Silo)
Create an IAM role per tenant with policies that restrict access to that tenant's resources.

```
Effect: Allow
Action: dynamodb:*
Resource: arn:aws:dynamodb:*:*:table/tenant-{tenant-id}-*
```

**Scaling limit:** IAM has limits on the number of policies and roles per account. This works for tens of tenants, not thousands.

### Dynamic Policies with STS (Pool)
Generate session policies at runtime using STS AssumeRole with a policy document.

The session policy narrows the base role's permissions to the current tenant:

```
Effect: Allow
Action: dynamodb:GetItem, dynamodb:PutItem, dynamodb:Query
Resource: arn:aws:dynamodb:*:*:table/SharedTable
Condition:
  ForAllValues:StringEquals:
    dynamodb:LeadingKeys: ["TENANT#{tenant-id}"]
```

**Key advantage:** One base role, unlimited tenants. The session policy is generated per-request, so no IAM scaling limits.

**Key consideration:** Session policies have a size limit (2048 characters for inline, larger for managed). Keep policies focused.

**Code example — Lambda function generating tenant-scoped credentials for PHI access:**
```typescript
import { STSClient, AssumeRoleCommand } from '@aws-sdk/client-sts';

const sts = new STSClient({});

export async function getTenantScopedCredentials(tenantId: string, baseRoleArn: string) {
  const sessionPolicy = JSON.stringify({
    Version: '2012-10-17',
    Statement: [{
      Effect: 'Allow',
      Action: ['dynamodb:GetItem', 'dynamodb:PutItem', 'dynamodb:Query'],
      Resource: 'arn:aws:dynamodb:*:*:table/PatientRecords',
      Condition: {
        'ForAllValues:StringEquals': {
          'dynamodb:LeadingKeys': [`TENANT#${tenantId}`]
        }
      }
    }, {
      Effect: 'Allow',
      Action: ['kms:Decrypt', 'kms:GenerateDataKey'],
      Resource: `arn:aws:kms:*:*:key/*`, // further scoped by key policy
      Condition: {
        StringEquals: { 'kms:RequestAlias': `alias/tenant-${tenantId}-phi` }
      }
    }]
  });

  const { Credentials } = await sts.send(new AssumeRoleCommand({
    RoleArn: baseRoleArn,
    RoleSessionName: `tenant-${tenantId}-${Date.now()}`,
    Policy: sessionPolicy,
    DurationSeconds: 900, // 15 min — short-lived for PHI access
  }));

  return Credentials;
}
```
*Note: This scopes both DynamoDB access (LeadingKeys) and KMS key usage to the tenant. Even if application code has a bug, IAM + KMS prevent cross-tenant PHI access.*

### ABAC (Attribute-Based Access Control)
Use resource tags and session tags to control access without per-tenant policies.

1. Tag resources with `TenantId` tag
2. When assuming role, pass `TenantId` as a session tag
3. IAM policy uses `aws:PrincipalTag/TenantId` condition to match resource tags

```
Effect: Allow
Action: s3:GetObject
Resource: arn:aws:s3:::shared-bucket/*
Condition:
  StringEquals:
    s3:ExistingObjectTag/TenantId: "${aws:PrincipalTag/TenantId}"
```

**When to use ABAC over dynamic policies:**
- When you have many resource types to isolate (ABAC scales better across services)
- When resources are tagged consistently
- When you want a single policy that works across all tenants

**Reference:** [How to implement SaaS tenant isolation with ABAC and AWS IAM](https://aws.amazon.com/blogs/security/how-to-implement-saas-tenant-isolation-with-abac-and-aws-iam)

## Amazon Verified Permissions & Cedar

For fine-grained application-level authorization beyond IAM (which controls AWS resource access), Amazon Verified Permissions provides a policy decision point (PDP) using the Cedar policy language.

### When to Use Verified Permissions vs. IAM

| Concern | Use IAM | Use Verified Permissions |
|---------|---------|--------------------------|
| AWS resource access (DynamoDB, S3, etc.) | ✅ | ❌ |
| Application-level permissions (can user X edit document Y?) | ❌ | ✅ |
| Tenant-scoped AWS API calls | ✅ | ❌ |
| Feature-level access control per tier | ❌ | ✅ |
| Role-based access within a tenant (admin, viewer, editor) | ❌ | ✅ |

### Multi-Tenant Patterns with Verified Permissions

**Per-tenant policy store:** Each tenant gets their own policy store. Strongest isolation — policies for tenant A can never affect tenant B. Higher operational overhead.

**Shared policy store with tenant context:** Single policy store with policies that include tenant conditions. Simpler to manage, but policies must be carefully written to prevent cross-tenant access.

**Example Cedar policy (shared store):**
```
permit(
  principal,
  action == Action::"ViewDocument",
  resource
) when {
  principal.tenant == resource.tenant
};
```

**Reference:** [SaaS access control using Amazon Verified Permissions with a per-tenant policy store](https://aws.amazon.com/blogs/security/saas-access-control-using-amazon-verified-permissions-with-a-per-tenant-policy-store/)

## Testing Isolation

Isolation must be tested continuously, not just at design time. The SaaS Lens reliability pillar specifically calls this out.

### What to Test

1. **Cross-tenant data access:** Authenticate as Tenant A, attempt to read Tenant B's data. Must fail.
2. **Cross-tenant API access:** Use Tenant A's JWT to call APIs that should be scoped to Tenant A. Verify no Tenant B data leaks.
3. **Privilege escalation:** Attempt to modify the tenant ID in the JWT or session. Verify the system rejects it.
4. **IAM policy enforcement:** Bypass application code and call AWS APIs directly with the scoped credentials. Verify IAM blocks cross-tenant access.
5. **Database-level isolation:** For pool model with RLS, attempt direct database queries crossing tenant boundaries.

### How to Test

- Automated integration tests that run on every deployment
- Include cross-tenant access attempts in your CI/CD pipeline
- Use separate test tenants with known data to verify isolation
- Periodically run isolation tests in production (with test tenants)

## PHI Isolation Requirements

Tenant isolation in healthcare SaaS is not just a best practice — it's a HIPAA requirement. The HIPAA Security Rule mandates access controls that restrict ePHI access to authorized persons only (§164.312(a)(1)). In a multi-tenant environment, this means one tenant must NEVER be able to access another tenant's PHI, even if application code has a bug.

**The healthcare isolation bar:**
- Infrastructure-level enforcement is mandatory for PHI. Application-level filtering (WHERE tenant_id = X) is necessary but insufficient.
- IAM-based isolation (dynamic policies, ABAC, or resource-level policies) must back every PHI data access path.
- Encryption with per-tenant KMS keys provides a second isolation layer — even if IAM is misconfigured, the wrong tenant's key won't decrypt the data.
- Audit trails must prove isolation is working — every PHI access must be logged with tenant context. See `audit-logging-and-access.md`.

**The compliance test:** Can you demonstrate to a HITRUST assessor that Tenant A cannot access Tenant B's PHI, even if a developer introduces a bug in the data access layer? If the answer relies solely on application code correctness, you fail.

## HealthLake Isolation Patterns

AWS HealthLake (FHIR data store) supports multi-tenant patterns with different isolation characteristics:

### Data Store Per Tenant (Silo)
Each tenant gets their own HealthLake data store. Strongest isolation — separate FHIR endpoints, separate encryption keys, separate access policies.

**Isolation enforcement:** IAM resource policy on each data store restricts access to the tenant's roles. KMS key policy scoped to the tenant.

**Pros:** Strongest isolation, per-tenant encryption, simple compliance story, easy tenant offboarding (delete the data store)
**Cons:** Higher cost (per-data-store pricing), operational overhead scales with tenant count, cross-tenant analytics requires aggregation layer

**Best for:** Enterprise health system tenants, regulated environments, tenants with contractual isolation requirements.

### Shared Data Store with Tenant Metadata (Pool)
All tenants share one HealthLake data store. Tenant context is stored as FHIR resource metadata (tags or extensions). Access control via SMART on FHIR scopes and application-level filtering.

**Isolation enforcement:** SMART on FHIR authorization scopes tenant access. Application layer filters by tenant metadata. IAM session policies scope API access.

**Pros:** Lower cost, simpler operations, easier cross-tenant analytics
**Cons:** Weaker isolation (relies on application + SMART scopes, not resource boundaries), shared encryption key, harder tenant offboarding (must delete individual FHIR resources)

**Best for:** High tenant count, SMB clinics, cost-sensitive, lower compliance requirements.

### Hybrid
Shared data store for most tenants, dedicated data stores for enterprise tenants. The control plane routes FHIR requests to the correct data store based on tenant configuration.

**Best for:** Mixed SMB + enterprise customer base.

## HealthImaging Isolation Patterns

AWS HealthImaging stores DICOM imaging data in data stores. Each data store is a natural isolation boundary.

### Data Store Per Tenant (Recommended for Healthcare)
Each tenant (hospital, imaging center) gets their own HealthImaging data store. This is the natural model because imaging data is large, tenant-specific, and highly sensitive.

**Isolation enforcement:** IAM resource policy per data store. KMS CMK per tenant for encryption. DICOM import jobs scoped to the tenant's data store.

**Pros:** Strongest isolation, per-tenant encryption, per-tenant storage metrics, clean tenant offboarding
**Cons:** Data store limits per account (check current limits), operational overhead

### Shared Data Store with Image Set Tagging (Pool)
All tenants share one data store. Image sets are tagged with tenant metadata. Access control via IAM conditions on image set tags.

**Pros:** Simpler operations for high tenant count
**Cons:** Shared encryption, weaker isolation, harder to attribute storage costs per tenant

**Recommendation:** Use data-store-per-tenant for healthcare. Imaging data is too sensitive and too large for pool model to be practical.

## Isolation Testing for PHI

Standard isolation testing (as described above) applies, but healthcare adds specific requirements:

**PHI-specific test scenarios:**
1. Authenticate as Tenant A clinician, attempt to read Tenant B patient's FHIR resources. Must return 403/empty.
2. Authenticate as Tenant A, attempt to use Tenant B's KMS key to decrypt data. Must fail.
3. Authenticate as Tenant A, attempt to access Tenant B's HealthLake data store. Must fail.
4. Authenticate as Tenant A, attempt to access Tenant B's HealthImaging image sets. Must fail.
5. For pool model: query shared DynamoDB table with Tenant A's scoped credentials, verify zero Tenant B items returned.
6. For 42 CFR Part 2 data: verify that even within a tenant, SUD records are only accessible with active consent.

**Automation:** These tests must run in CI/CD on every deployment. A failed isolation test should block the deployment. Document test results for HITRUST evidence.

**Frequency:** Automated on every deployment + quarterly manual penetration testing focused on cross-tenant access.

## Common Mistakes

1. **Relying only on application-level checks** — "We filter by tenant_id in every query" is not isolation. It's a bug away from a breach. Back it with IAM.

2. **Not testing isolation** — If you don't have automated tests that attempt cross-tenant access, you don't know if isolation works.

3. **Forgetting about secondary access paths** — You isolated DynamoDB but forgot that the S3 bucket with exports is shared without tenant scoping.

4. **Static IAM policies at scale** — Creating a role per tenant works for 50 tenants. At 5,000 tenants, you'll hit IAM limits. Use dynamic policies or ABAC.

5. **Ignoring the JWT trust chain** — If the JWT can be forged or the tenant ID can be modified, all downstream isolation is compromised. Validate JWTs properly.

6. **No per-tenant KMS keys for PHI.** Shared encryption keys across tenants weaken the isolation story. Per-tenant CMKs provide cryptographic isolation that survives application bugs. This is the healthcare default.

7. **Not testing isolation for PHI specifically.** Generic isolation tests check cross-tenant API access. Healthcare isolation tests must also verify: cross-tenant KMS key usage fails, cross-tenant HealthLake/HealthImaging access fails, and 42 CFR Part 2 consent-gated access works correctly.

## Discovery Questions for This Domain

When the conversation is about tenant isolation, ask these:

**Current state:**
- How is tenant isolation enforced today? (Application-level filtering only? IAM policies? Network isolation? Nothing yet?)
- Do you have automated tests that verify one tenant can't access another's data?
- Have you had any cross-tenant data incidents or near-misses?

**Requirements:**
- Do you have compliance requirements that mandate specific isolation mechanisms? (HIPAA, SOC2, FedRAMP each have different expectations)
- Do any customers require proof of isolation? (Audit reports, penetration test results, architecture diagrams showing separation)
- What AWS services are you using for storage? (The isolation mechanism differs per service — DynamoDB LeadingKeys vs. RDS RLS vs. S3 bucket policies)

**Implementation:**
- Are you using IAM session policies or ABAC for runtime isolation, or is it purely application-level?
- How is tenant context propagated through your stack? (JWT claims? Headers? Message attributes?)
- For async workflows (queues, events, batch jobs) — does tenant context survive the async boundary?

**Scale:**
- How many tenants do you have or expect? (This determines whether static IAM policies or dynamic STS policies are appropriate)
- Do you need different isolation levels for different tiers? (Pool isolation for basic, resource-level isolation for enterprise)

## References

- [SaaS Tenant Isolation Strategies Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/saas-tenant-isolation-strategies.html)
- [Isolating SaaS Tenants with Dynamically Generated IAM Policies](https://aws.amazon.com/blogs/apn/isolating-saas-tenants-with-dynamically-generated-iam-policies/)
- [Implement SaaS Tenant Isolation for S3 Using a Lambda Token Vending Machine](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/implement-saas-tenant-isolation-for-amazon-s3-by-using-an-aws-lambda-token-vending-machine.html)
- [Security Practices in AWS Multi-Tenant SaaS Environments](https://aws.amazon.com/blogs/security/security-practices-in-aws-multi-tenant-saas-environments/)
- [Multi-Tenant Authorization and API Access Control](https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/introduction.html)
- [SaaS Lens — Identity and Access Management](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/identity-and-access-management.html)
