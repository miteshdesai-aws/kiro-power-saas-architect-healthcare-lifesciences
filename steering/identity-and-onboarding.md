# Identity, Onboarding & Tenant Lifecycle

## SaaS Identity: The Tenant-User Binding

The SaaS Lens defines "SaaS identity" as the binding of a user identity to a tenant context. Every authenticated user in a SaaS system must be associated with a tenant. This binding is the foundation for everything else — isolation, authorization, metrics, billing.

**The key principle:** Tenant context must be a first-class construct in your identity model, not an afterthought. It should flow through every layer of the architecture via the same mechanisms used for user identity (typically JWT claims).

### How Tenant Context Flows

1. User authenticates → identity provider issues JWT with tenant ID as a custom claim
2. API Gateway validates JWT → extracts tenant context
3. Lambda/ECS receives request with tenant context in the event/headers
4. Service uses tenant context to scope data access (via IAM session policies or application logic)
5. Logging, metrics, and tracing include tenant context for observability

**This flow must be automatic and unavoidable.** Developers should not need to remember to "add tenant filtering" — the infrastructure should enforce it.

## Amazon Cognito Patterns for Multi-Tenant

### Pattern 1: Single User Pool with Custom Attributes

All tenants share one Cognito user pool. Tenant ID is stored as a custom attribute on each user and included in the JWT as a custom claim.

**Pros:**
- Simplest to manage — one pool, one configuration
- Lowest cost — Cognito pricing is per-MAU, not per-pool
- Easy to implement cross-tenant admin views

**Cons:**
- All tenants share authentication configuration (password policies, MFA settings)
- Enterprise tenants can't bring their own IdP easily (SAML federation is per-pool)
- User pool limits apply across all tenants

**Best for:** SMB-focused SaaS, early stage, uniform authentication requirements.

### Pattern 2: User Pool Per Tenant

Each tenant gets their own Cognito user pool. The application routes authentication requests to the correct pool based on tenant context (subdomain, login page, etc.).

**Pros:**
- Each tenant can have custom auth settings (password policy, MFA)
- Enterprise tenants can federate their own IdP via SAML/OIDC
- Strongest identity isolation — no shared user directory
- Per-tenant user pool limits

**Cons:**
- Operational overhead scales with tenant count
- More complex routing logic
- Cross-tenant operations (admin dashboards) require querying multiple pools
- Cognito has account-level limits on number of user pools

**Best for:** Enterprise-focused SaaS, tenants with SAML/SSO requirements, regulated industries.

### Pattern 3: Single Pool with SAML Federation (Hybrid)

One user pool for most tenants, with SAML identity provider configurations for enterprise tenants who need SSO. Cognito supports multiple SAML providers per user pool.

**Pros:**
- Single pool simplicity for most tenants
- Enterprise tenants get SSO via their own IdP
- Good balance of simplicity and flexibility

**Cons:**
- Cognito limits on number of identity providers per pool (currently 300)
- All tenants still share base authentication configuration

**Best for:** SaaS with a mix of self-serve and enterprise customers.

**Reference:** [SaaS Authentication: Identity Management with Amazon Cognito User Pools](https://aws.amazon.com/blogs/security/saas-authentication-identity-management-with-amazon-cognito-user-pools/)
**Reference:** [Use SAML with Amazon Cognito to Support a Multi-Tenant Application](https://aws.amazon.com/blogs/security/use-saml-with-amazon-cognito-to-support-a-multi-tenant-application-with-a-single-user-pool/)

## Tenant Context Propagation

### The JWT as Tenant Context Carrier

The JWT issued by your identity provider should include:
- `sub`: User ID
- `custom:tenant_id`: Tenant identifier (the critical claim)
- `custom:tenant_tier`: Tenant tier (basic, pro, enterprise) — useful for throttling decisions
- `custom:roles`: User's roles within the tenant (admin, member, viewer)

### Propagation Through the Stack

**API Gateway → Lambda:**
- API Gateway validates the JWT via a Cognito authorizer or Lambda authorizer
- Lambda authorizer can enrich the context (look up tenant configuration, resolve tier)
- Tenant context is passed in the event object (requestContext.authorizer.claims)

**API Gateway → ECS/EKS:**
- JWT is forwarded in the Authorization header
- Application middleware extracts and validates tenant context
- Tenant context is stored in request-scoped context (thread-local, async context, etc.)

**Service-to-Service:**
- When one microservice calls another, tenant context must be propagated
- Pass tenant ID in headers (X-Tenant-Id) or forward the original JWT
- Never lose tenant context in async operations (SQS messages, EventBridge events must include tenant ID)

### Common Pitfall: Losing Tenant Context in Async Flows

When a request triggers an async operation (SQS message, Step Functions execution, EventBridge event), the tenant context from the original JWT is no longer available. You must explicitly include tenant ID in the message/event payload.

**Rule:** Every message, event, and async invocation must carry tenant context. If it doesn't, downstream processing can't enforce isolation or attribute metrics.

## Tenant Onboarding

### The Principle: Fully Automated, Single Process

The SaaS Lens Operational Excellence pillar is explicit: tenant onboarding must be a single automated process that runs end-to-end without manual intervention. Manual onboarding is a scaling bottleneck and an error source.

### What Onboarding Must Do

1. **Create tenant record** in the control plane (tenant ID, name, tier, configuration)
2. **Provision identity** — create user pool (if per-tenant) or configure user in shared pool
3. **Provision infrastructure** — for silo/bridge tenants, create dedicated resources (databases, buckets, etc.)
4. **Configure routing** — update tenant routing table so requests reach the right resources
5. **Set up billing** — create billing profile, configure metering, set tier-based limits
6. **Configure throttling** — apply API Gateway usage plan based on tier
7. **Send welcome/activation** — notify tenant, provide access credentials or activation link

### Orchestration Pattern: Step Functions

AWS Step Functions is the recommended orchestration engine for tenant onboarding. Each step in the onboarding process is a Lambda function or service integration, with error handling and retry logic built in.

**Why Step Functions over a single Lambda:**
- Onboarding can take minutes (especially for silo tenants with infrastructure provisioning)
- Individual steps can fail and need retry logic
- You need visibility into where onboarding is in the process
- Rollback/compensation logic for partial failures

### SBT Event-Driven Onboarding

The SaaS Builder Toolkit uses an event-driven model:
1. Control plane receives "create tenant" request
2. Control plane publishes an onboarding event (via EventBridge)
3. Application plane listens for the event and provisions tenant-specific resources
4. Application plane signals completion back to the control plane

This decouples the control plane from application-specific provisioning logic.

## Tenant Lifecycle Management

### Tenant States

| State | Description | Resources | Data |
|-------|-------------|-----------|------|
| Active | Normal operation | Running | Accessible |
| Suspended | Temporarily disabled (e.g., payment failure) | Running but access blocked | Preserved |
| Deactivated | Tenant has churned or been disabled | Stopped/scaled down | Preserved for retention period |
| Archived | Past retention period | Removed | Archived to cold storage or deleted |
| Deleted | Fully removed | Removed | Deleted (right to erasure) |

### Tenant Deactivation (Not Deletion)

When a tenant churns, don't immediately delete everything. The SaaS Lens recommends:
- Block access (disable user pool, revoke API keys)
- Preserve data for the retention period (audit, compliance, potential reactivation)
- Scale down or stop dedicated resources (silo) to reduce cost
- After retention period: archive or delete based on policy

### Tenant Portability (Tier Changes)

Tenants should be able to move between tiers. This is especially important for pool → silo upgrades (enterprise upsell).

**What tier change involves:**
- Update tenant configuration in control plane (new tier, new limits)
- For pool → silo: migrate tenant data from shared to dedicated resources
- For silo → pool: migrate tenant data from dedicated to shared resources (rare, but possible for downgrades)
- Update routing to point to new resources
- Update throttling limits
- Update billing configuration

**Design for portability from day one.** If your data model makes it impossible to extract one tenant's data from a shared database, tier upgrades become a project instead of an operation.

**Reference:** [Tenant Portability: Move Tenants Across Tiers in a SaaS Application](https://aws.amazon.com/blogs/architecture/tenant-portability-move-tenants-across-tiers-in-a-saas-application/)

### Tenant Offboarding

When a tenant is permanently removed:
1. Export tenant data if requested (data portability, GDPR right)
2. Delete tenant data from all storage (including backups if required)
3. Remove dedicated resources (silo/bridge)
4. Remove tenant from identity provider
5. Remove tenant from billing/metering
6. Update routing tables
7. Audit log the offboarding for compliance

**The hard part in pool model:** Deleting a single tenant's data from shared tables. With DynamoDB, you need to scan and delete all items with that tenant's partition key. With RDS pool model (row-level security), you need to delete all rows. This can be expensive and slow for large tenants.

**Design consideration:** If right-to-erasure is a requirement, factor this into your tenancy model decision. Silo and bridge models make deletion straightforward. Pool model makes it operationally expensive.

## Healthcare Identity Personas

In healthcare SaaS, identity is more complex than generic SaaS because you have three distinct user personas with different authentication flows, session requirements, and access patterns.

### Persona 1: Clinician (Physician, Nurse, Therapist)
- **Authentication:** Federated from the health system's IdP (Epic MyChart, Oracle Health/Cerner, ADFS, Entra ID) via SAML/OIDC. Clinicians rarely create new credentials — they use their hospital SSO.
- **Session requirements:** Short sessions (shift-based), automatic logoff per HIPAA §164.312(a)(2)(iii), re-authentication for sensitive actions.
- **Access pattern:** Read/write patient records within their tenant. May need cross-tenant access in referral scenarios (break-the-glass — see `audit-logging-and-access.md`).
- **Cognito pattern:** User pool per tenant (Pattern 2) is often required because each health system has its own IdP. Or single pool with SAML federation per tenant (Pattern 3).

### Persona 2: Patient
- **Authentication:** Self-service registration, email/phone verification, MFA encouraged. May also authenticate via SMART on FHIR patient standalone launch from a patient portal.
- **Session requirements:** Longer sessions (patient portals), but re-authentication for sensitive actions (viewing lab results, downloading records).
- **Access pattern:** Read-only access to their own records. May grant consent for data sharing. Never cross-tenant access.
- **Cognito pattern:** Can share a user pool with clinicians (different app client, different group) or separate user pool for patients. Patient identity must be bound to both a tenant AND a patient record.

### Persona 3: Organization Admin (Practice Manager, IT Admin)
- **Authentication:** Username/password with MFA required. May federate from the organization's corporate IdP.
- **Session requirements:** Standard web sessions with inactivity timeout.
- **Access pattern:** Tenant configuration, user management, billing, reporting. Should NOT have direct access to patient clinical data unless also a clinician.
- **Cognito pattern:** Same user pool as clinicians, different Cognito group with admin permissions.

### JWT Claims for Healthcare
The JWT should include healthcare-specific claims beyond the standard SaaS claims:
- `custom:tenant_id` — tenant identifier (standard SaaS)
- `custom:tenant_tier` — tier for throttling decisions (standard SaaS)
- `custom:user_role` — clinician, patient, admin (healthcare-specific)
- `custom:persona_type` — which persona (affects access patterns)
- `custom:npi` — National Provider Identifier for clinicians (if applicable)
- `custom:fhir_practitioner_id` — FHIR Practitioner resource ID (for EHR integration)

## SMART on FHIR Authentication

SMART on FHIR (Substitutable Medical Applications, Reusable Technologies) is the standard authorization framework for healthcare applications integrating with EHR systems. It layers OAuth 2.0 + OpenID Connect with healthcare-specific scopes and launch contexts.

### When You Need SMART on FHIR
- Your SaaS integrates with Epic, Cerner/Oracle Health, or other EHRs
- Clinicians launch your app from within the EHR (EHR launch)
- Patients access your app from a patient portal (standalone launch)
- You need to access FHIR resources with patient/practitioner context

### SMART Scopes
SMART defines granular scopes for FHIR resource access:
- `patient/*.read` — read all FHIR resources for the current patient
- `patient/Observation.read` — read only Observations for the current patient
- `user/*.read` — read FHIR resources accessible to the current user
- `launch` — receive launch context from the EHR
- `openid fhirUser` — get the FHIR identity of the authenticated user

### Cognito as SMART Authorization Server
Cognito can serve as the authorization server for SMART on FHIR with customization:
- Configure custom scopes matching SMART scope patterns
- Use a Lambda authorizer to validate SMART-specific claims
- Map EHR launch context to Cognito session attributes
- Note: SBT's CognitoAuth does NOT support SMART on FHIR out of the box — you'll need to extend it

### Health System IdP Federation
Enterprise health system tenants typically require SSO via their existing IdP:
- **Epic MyChart:** SAML 2.0 federation, SMART on FHIR for clinical launch
- **Oracle Health/Cerner:** SAML 2.0 or OIDC federation
- **Microsoft Entra ID (ADFS):** SAML 2.0 federation — common for hospital IT
- **Okta/Ping:** OIDC federation — common for health tech companies

Configure each as an identity provider in Cognito (per-tenant pool or shared pool with per-tenant IdP).

## Consent Management

### Why Consent Is an Identity Concern
In healthcare, consent determines what data a user can access. It's not just authorization (role-based) — it's patient-directed. A patient may consent to share their records with Provider A but not Provider B, even within the same SaaS platform.

### Consent for HIPAA
HIPAA generally allows use/disclosure of PHI for Treatment, Payment, and Healthcare Operations (TPO) without explicit patient consent. But:
- Marketing and sale of PHI require explicit authorization
- Psychotherapy notes require explicit authorization
- 42 CFR Part 2 (SUD data) requires explicit consent even for TPO

### Consent for 42 CFR Part 2
If your SaaS handles substance use disorder data:
- Capture explicit written consent before any disclosure
- Consent must specify: who can access, what data, for what purpose, expiration date
- Consent can be revoked at any time — revocation must take effect immediately
- Audit trail must record consent status at the time of every access

### FHIR Consent Resource
Model consent using the FHIR Consent resource (R4):
- `status`: active, rejected, inactive
- `scope`: patient-privacy, treatment, research
- `category`: what type of consent
- `provision`: permit or deny rules with actor, action, data, and period

**FHIR Consent resource example — 42 CFR Part 2 SUD consent:**
```json
{
  "resourceType": "Consent",
  "status": "active",
  "scope": {
    "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/consentscope", "code": "patient-privacy" }]
  },
  "category": [{
    "coding": [{ "system": "http://loinc.org", "code": "59284-0", "display": "Consent Document" }]
  }],
  "patient": { "reference": "Patient/patient-123" },
  "dateTime": "2025-01-15",
  "organization": [{ "reference": "Organization/tenant-abc123" }],
  "policy": [{
    "authority": "https://www.hhs.gov",
    "uri": "https://www.ecfr.gov/current/title-42/chapter-I/subchapter-A/part-2"
  }],
  "provision": {
    "type": "permit",
    "period": { "start": "2025-01-15", "end": "2026-01-15" },
    "actor": [{
      "role": { "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/v3-ParticipationType", "code": "PRCP" }] },
      "reference": { "reference": "Organization/tenant-abc123" }
    }],
    "action": [{
      "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/consentaction", "code": "access" }]
    }],
    "class": [{
      "system": "http://hl7.org/fhir/resource-types",
      "code": "Condition"
    }]
  }
}
```
*Note: This consent permits the tenant organization to access the patient's Condition resources (which may include SUD diagnoses) for one year. The 42 CFR Part 2 policy URI documents the regulatory basis. Your authorization middleware must check this consent before granting access to SUD-related resources.*

### Implementation Pattern
1. Patient grants/revokes consent through the application
2. Consent is stored as a FHIR Consent resource (or equivalent in your data model)
3. Authorization middleware checks consent status before granting access to protected data
4. Every access decision is logged with the consent status at that moment
5. Consent changes trigger re-evaluation of active sessions/cached permissions

## Electronic Signatures (21 CFR Part 11 / EU Annex 11)

When healthcare SaaS is also GxP-regulated (SaMD, eClinical, pharmacovigilance, regulated labs), identity flows must support electronic signatures under 21 CFR Part 11. Signatures are an identity concern because they require re-authentication and are bound to an identity at signing time. For detailed signature design, see the Electronic Signature Design artifact in `artifacts-healthcare.md` and the audit-side implementation in `audit-logging-and-access.md`.

### When E-Signatures Apply
Predicate rules (FDA, EMA, or equivalent regulations) may require an electronic signature for specific actions:
- Finalization of a regulated clinical report or SaMD output
- Amendment to a regulated record (reason-for-change plus signer attestation)
- Batch release decisions (pharmaceutical manufacturing)
- Approval of regulated lab results
- Approval of deployments to validated systems

### Identity Requirements
Per 21 CFR Part 11 Subpart C:
- Each signature is unique to one individual (no shared credentials)
- Signer identity verified at time of signing (not just session validity — re-authenticate)
- Signature links to: printed name of signer, date/time of signing, meaning of signature
- Binding to the signed record is preserved — modifying the record invalidates the signature

### Cognito Pattern for Re-authentication at Signing
- App initiates signing flow for a regulated action
- App challenges user: Cognito Custom Challenge (MFA code, biometric, or password re-entry)
- On success: Cognito issues a short-lived token (< 5 min) with a `fresh_auth` claim
- Signing endpoint verifies `fresh_auth` claim before creating the signature record
- Signature record captures: user_id (from token), printed name (from IdP profile), timestamp, meaning, record_content_hash
- Subsequent API calls no longer have `fresh_auth` — for the next signature, re-challenge

### Non-Repudiation via KMS Asymmetric Keys
For the strongest signature binding, use AWS KMS asymmetric keys:
- Each signer role (or each signer for high-assurance cases) has an asymmetric KMS key
- Signature is computed via `kms:Sign` over: signer_id + timestamp + meaning + record_content_hash
- KMS logs the signing operation in CloudTrail (separate evidence trail)
- Signature can be verified later with `kms:Verify` or the public key
- Key rotation and access audited via CloudTrail

### SBT Limitation
SBT's default CognitoAuth does not include re-authentication challenge flows for e-signatures. You'll need to add custom Lambda triggers and a signing endpoint that enforces fresh authentication. See `sbt-toolkit.md` for SBT customization patterns.

## Discovery Questions for This Domain

When the conversation is about identity, onboarding, or tenant lifecycle:

**Identity:**
- What identity provider are you using or planning to use? (Cognito, Auth0, Okta, custom, undecided?)
- Do any tenants need to bring their own identity provider (SAML/OIDC federation)?
- How do you bind users to tenants today? (Custom attribute in IdP? Lookup table? Application-level mapping?)
- Do users ever belong to multiple tenants? (This complicates the tenant-user binding significantly)
- Do you have distinct user personas? (Clinicians, patients, admins — each with different auth flows?)
- Do clinicians launch your app from within an EHR? (Triggers SMART on FHIR requirement)
- Do patients authenticate directly or through a patient portal?

**Onboarding:**
- How does a new tenant get created today? (Self-service signup? Sales-driven? Manual provisioning?)
- How long does it take from signup to first use? (If it's more than a few minutes for pool, or more than 10 minutes for silo, there's room to improve)
- What resources need to be provisioned per tenant? (This depends on the tenancy model — pool may need nothing, silo needs everything)
- What happens if onboarding partially fails? (Is there rollback? Does it leave orphaned resources?)

**Lifecycle:**
- What happens when a tenant stops paying? (Immediate lockout? Grace period? Data preserved for how long?)
- Do you need to support tenant data export? (GDPR portability, customer request)
- How do you handle tenant deletion? (Right-to-erasure requirements, data retention policies)
- Do tenants change tiers? (If yes, what changes when they upgrade/downgrade?)

**Scale:**
- How many tenants do you onboard per day/week/month? (This determines whether manual steps are acceptable)
- Do you need to support bulk onboarding? (Migration scenarios, partner channels)

**GxP (if applicable):**
- Does your system require electronic signatures under 21 CFR Part 11 or EU Annex 11?
- What actions require signatures? (Report finalization, record amendment, batch release, deployment approval?)
- Is re-authentication enforced at signing time, or are you relying on session validity?
- Are signatures cryptographically bound to record content?

## References

- [SaaS Lens — Identity and Access Management](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/identity-and-access-management.html)
- [SaaS Lens — How Are New Tenants Onboarded?](https://wa.aws.amazon.com/saas.question.OPS_3.en.html)
- [Tenant Onboarding Best Practices with the SaaS Lens](https://aws.amazon.com/blogs/apn/tenant-onboarding-best-practices-in-saas-with-the-aws-well-architected-saas-lens/)
- [Tenant Onboarding in SaaS Architecture for the Silo Model](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/tenant-onboarding-in-saas-architecture-for-the-silo-model-using-c-and-aws-cdk.html)
- [Managing the Account Lifecycle in Account-Per-Tenant Environments](https://aws.amazon.com/blogs/mt/managing-the-account-lifecycle-in-account-per-tenant-saas-environments-on-aws/)
- [SaaS Lens — User Behavior Patterns](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/user-behavior-patterns.html)
- [SaaS Authentication: Identity Management with Amazon Cognito User Pools](https://aws.amazon.com/blogs/security/saas-authentication-identity-management-with-amazon-cognito-user-pools/)
- [Use SAML with Amazon Cognito to Support a Multi-Tenant Application](https://aws.amazon.com/blogs/security/use-saml-with-amazon-cognito-to-support-a-multi-tenant-application-with-a-single-user-pool/)
- [Enhanced Interoperability with SMART on FHIR Support in Amazon HealthLake](https://aws.amazon.com/blogs/industries/enhanced-interoperability-with-smart-on-fhir-support-in-amazon-healthlake/)
- [Tenant Portability: Move Tenants Across Tiers](https://aws.amazon.com/blogs/architecture/tenant-portability-move-tenants-across-tiers-in-a-saas-application/)
