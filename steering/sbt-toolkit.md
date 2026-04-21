# SaaS Builder Toolkit (SBT)

## What Is SBT?

The SaaS Builder Toolkit (sbt-aws) is an open-source AWS CDK library that provides pre-built constructs for the foundational components of a multi-tenant SaaS application. It handles the undifferentiated heavy lifting of SaaS — tenant management, identity, onboarding orchestration, billing integration — so you can focus on your application logic.

**Repository:** [github.com/awslabs/sbt-aws](https://github.com/awslabs/sbt-aws)

**Version note:** This guidance was written against SBT v0.x (pre-1.0). SBT is actively evolving — construct names, APIs, and patterns may change. If something doesn't match the current SBT docs, check the [SBT changelog](https://github.com/awslabs/sbt-aws/releases) for breaking changes. The architectural patterns (control plane / application plane split, event-driven provisioning) are stable; the specific CDK construct interfaces may shift.

SBT is opinionated about the control plane / application plane split. It gives you a working control plane out of the box and an event-driven interface for your application plane to respond to tenant lifecycle events.

## When to Use SBT vs. Building Custom

### Use SBT When:
- You're building greenfield SaaS and want to accelerate the control plane
- Your team is already using CDK for infrastructure
- You want a proven pattern for tenant onboarding, identity, and billing
- You're comfortable with the opinions SBT makes (Cognito for auth, EventBridge for eventing, Step Functions for orchestration)
- You want to integrate with AWS Marketplace for billing

### Build Custom When:
- You have an existing control plane that works
- You need an identity provider other than Cognito (Auth0, Okta, custom)
- Your IaC tool is Terraform or SAM (SBT is CDK-only)
- You need fine-grained control over every aspect of tenant lifecycle
- Your onboarding flow has complex domain-specific steps that don't fit SBT's event model

### Hybrid Approach
Use SBT for the parts that fit (tenant management, billing) and build custom for the parts that don't. SBT's constructs are composable — you don't have to use all of them.

## Core Architecture

SBT implements the control plane / application plane split as two CDK constructs that communicate via EventBridge:

```
┌─────────────────────────────────┐     EventBridge      ┌──────────────────────────────────┐
│         Control Plane           │◄────────────────────►│        Application Plane          │
│                                 │                       │                                  │
│  - Tenant Management API        │   Onboarding Event    │  - Tenant Resource Provisioning   │
│  - User Management              │ ──────────────────►   │    (your custom logic)            │
│  - Auth (Cognito)               │                       │                                  │
│  - Billing (Marketplace)        │   Provisioning Done   │  - Tenant-Specific Infra          │
│  - Onboarding Orchestration     │ ◄──────────────────   │    (databases, buckets, etc.)     │
│  - Tier/Throttle Config         │                       │                                  │
│  - Metrics Collection           │   Offboarding Event   │  - Tenant Resource Cleanup        │
│                                 │ ──────────────────►   │    (your custom logic)            │
└─────────────────────────────────┘                       └──────────────────────────────────┘
```

The control plane is mostly SBT-provided. The application plane is where your code lives. The contract between them is a set of EventBridge events.

## Key Constructs

### ControlPlane

The main construct. Deploys the full control plane stack:

- **Tenant Management API**: REST API (API Gateway + Lambda) for CRUD operations on tenants
- **User Management**: Create, list, disable users within a tenant
- **Auth Provider**: Pluggable, but ships with CognitoAuth
- **Billing Provider**: Pluggable, but ships with Marketplace integration
- **Event Bus**: EventBridge bus for control plane ↔ application plane communication
- **Onboarding/Offboarding Orchestration**: Step Functions state machines triggered by tenant creation/deletion

**What you get out of the box:**
- `POST /tenants` — creates a tenant, triggers onboarding workflow
- `GET /tenants` — lists all tenants
- `GET /tenants/{id}` — gets tenant details
- `PUT /tenants/{id}` — updates tenant config
- `DELETE /tenants/{id}` — triggers offboarding workflow
- `POST /users` — creates a user in a tenant
- `GET /users` — lists users in a tenant
- `DELETE /users/{id}` — disables/removes a user

### CognitoAuth

The default auth provider. Sets up:
- A Cognito User Pool with `tenantId` as a custom attribute
- A Cognito App Client
- JWT-based authentication for the control plane APIs
- User creation that automatically binds users to tenants

**Customization points:**
- Password policy
- MFA configuration
- Custom attributes beyond tenantId
- Pre/post authentication Lambda triggers

**Limitation:** CognitoAuth uses a single user pool. If you need per-tenant user pools (for enterprise SAML federation), you'll need to extend or replace this construct.

**Healthcare limitation:** CognitoAuth uses a single user pool with `tenantId` as a custom attribute. This does NOT support SMART on FHIR authorization flows out of the box. If your healthcare SaaS requires SMART on FHIR (for EHR integration), you'll need to either extend CognitoAuth with custom Lambda triggers and SMART-specific scopes, or replace it with a custom auth provider that implements the SMART on FHIR specification. See `identity-and-onboarding.md` for healthcare identity patterns.

### ApplicationPlane

A lighter construct. Its main job is to wire up your application-plane logic to the control plane's events.

**What it does:**
- Listens for onboarding events from the control plane's EventBridge bus
- Routes events to your provisioning logic (Lambda functions, Step Functions, etc.)
- Signals back to the control plane when provisioning is complete or failed

**What you provide:**
- The actual provisioning logic: "when a new tenant is created, create their DynamoDB table, S3 bucket, EKS namespace, etc."
- The deprovisioning logic: "when a tenant is deleted, clean up their resources"

### CoreApplicationPlane

An extended version of ApplicationPlane that adds:
- Job orchestration via Step Functions for long-running provisioning
- Status tracking for provisioning jobs
- Built-in retry and error handling

Use CoreApplicationPlane when your provisioning involves multiple steps that can fail independently (which is almost always the case for silo or bridge tenants).

## Event-Driven Tenant Lifecycle

SBT's core pattern is event-driven. The control plane publishes events, the application plane subscribes and acts.

### Onboarding Flow

1. Admin calls `POST /tenants` with tenant details (name, tier, config)
2. Control plane creates tenant record in its internal store (DynamoDB)
3. Control plane creates user in Cognito with `tenantId` attribute
4. Control plane publishes `Onboarding` event to EventBridge
5. Application plane receives event, provisions tenant-specific resources
6. Application plane publishes `OnboardingComplete` event (or `OnboardingFailed`)
7. Control plane updates tenant status to Active (or Failed)

### Offboarding Flow

1. Admin calls `DELETE /tenants/{id}`
2. Control plane publishes `Offboarding` event
3. Application plane receives event, deprovisions tenant resources
4. Application plane publishes `OffboardingComplete` event
5. Control plane removes tenant record and disables users

### Custom Events

You can extend the event model with your own events. For example:
- `TierChanged` — when a tenant upgrades/downgrades, trigger resource migration
- `TenantSuspended` — when a tenant's payment fails, trigger access revocation
- `TenantReactivated` — when payment is restored, re-enable access

## Billing Integration

SBT includes a billing provider interface with an AWS Marketplace implementation.

### Marketplace Integration

When enabled, SBT:
- Handles the Marketplace subscription webhook (SNS → Lambda)
- Creates tenants automatically when a customer subscribes via Marketplace
- Provides a `BatchMeterUsage` integration point for reporting consumption
- Manages the Marketplace customer ↔ tenant mapping

**What you provide:**
- Your metering data: call SBT's metering API with tenant usage dimensions
- Your Marketplace listing configuration (pricing dimensions, tiers)

### Custom Billing

If you're not using Marketplace, implement the billing provider interface with your own logic (Stripe, custom invoicing, etc.). SBT will call your provider during onboarding and offboarding.

## Practical Guidance

### Getting Started

1. Install: `npm install @cdklabs/sbt-aws`
2. Create a CDK stack with `ControlPlane` and `ApplicationPlane` constructs
3. Implement your provisioning logic as Lambda functions
4. Wire provisioning functions to the ApplicationPlane's event handlers
5. Deploy with `cdk deploy`

### Extending SBT

**Custom auth provider:** Implement the `IAuth` interface if you need something other than Cognito (e.g., Auth0, Okta).

**Custom billing provider:** Implement the `IBilling` interface for non-Marketplace billing.

**Additional control plane APIs:** Add Lambda functions behind the same API Gateway for domain-specific management operations.

**Per-tenant infrastructure:** In your ApplicationPlane provisioning logic, use CDK constructs to create per-tenant resources. SBT passes the tenant context (ID, tier) in the event payload.

### What SBT Does NOT Do

- **Application code**: SBT builds the control plane and wires up events. Your actual SaaS application (the thing tenants use) is entirely your responsibility.
- **Data partitioning**: SBT doesn't create your DynamoDB tables or RDS schemas. Your provisioning logic does that.
- **Tenant isolation enforcement**: SBT doesn't create IAM policies for tenant isolation. You implement that in your application plane.
- **Observability**: SBT doesn't set up per-tenant logging or metrics. You add that to your application.
- **CI/CD**: SBT doesn't manage your deployment pipeline. It deploys via CDK, but your application deployment is separate.

SBT gives you the skeleton. You provide the muscles.

## Common Patterns with SBT

### Pool Model with SBT
- ControlPlane: standard SBT setup
- ApplicationPlane: minimal provisioning (maybe just a config entry, no dedicated resources)
- Tenant isolation: handled in your application code + IAM session policies
- Onboarding is fast (seconds) because no infrastructure is provisioned

### Bridge Model with SBT
- ControlPlane: standard SBT setup
- ApplicationPlane: provisions dedicated storage (DynamoDB table, RDS schema, S3 bucket) per tenant
- Shared compute serves all tenants
- Onboarding takes 30-60 seconds (storage provisioning)

### Silo Model with SBT
- ControlPlane: standard SBT setup
- ApplicationPlane: provisions full infrastructure per tenant (compute, storage, networking)
- May use CoreApplicationPlane for multi-step provisioning with status tracking
- Onboarding takes minutes (full infrastructure provisioning)
- Consider account-per-tenant with AWS Organizations integration

### Healthcare SaaS with SBT
- ControlPlane: standard SBT setup with CognitoAuth (extend for SMART on FHIR if needed)
- ApplicationPlane: provisions per-tenant HealthLake data stores, HealthImaging data stores, per-tenant KMS keys, and tenant-specific S3 buckets for PHI
- Onboarding includes: tenant record creation, identity provisioning, PHI storage provisioning, KMS key creation, BAA tracking entry, HIPAA eligibility verification
- Offboarding includes: PHI data export (right to access), data deletion with audit trail, KMS key scheduling for deletion, BAA termination tracking

**Healthcare onboarding sequence (Step Functions via CoreApplicationPlane):**
```
Step 1: Create per-tenant KMS CMK for PHI encryption (see phi-data-handling.md for CDK/Terraform)
Step 2: Create per-tenant HealthLake FHIR data store (encrypted with tenant's CMK)
Step 3: Create per-tenant S3 bucket for PHI documents (encrypted with tenant's CMK, Object Lock for audit)
Step 4: Configure CloudTrail data events for tenant's resources (see audit-logging-and-access.md)
Step 5: Update tenant routing table to point to tenant's resources
Step 6: Record BAA tracking entry in control plane
Step 7: Signal SBT OnboardingComplete event
```

**Healthcare offboarding sequence:**
```
Step 1: Export tenant PHI for right-to-access (encrypted export to S3)
Step 2: Verify export integrity
Step 3: Delete HealthLake data store
Step 4: Delete S3 PHI bucket (after retention period)
Step 5: Schedule KMS key deletion (7-30 day mandatory waiting period)
Step 6: Record immutable audit entry of what was deleted and when
Step 7: Mark BAA as terminated in control plane
Step 8: Signal SBT OffboardingComplete event
```

*Note: KMS key deletion has a mandatory 7-30 day waiting period. After the waiting period, all data encrypted with this key becomes permanently inaccessible — this is cryptographic erasure, the strongest form of tenant data deletion for PHI.*

For CDK implementation of individual steps, see the code snippets in `phi-data-handling.md` (KMS keys), `audit-logging-and-access.md` (CloudTrail + S3 Object Lock), and `fhir-and-interop.md` (HealthLake). Wire these into SBT's `CoreApplicationPlane` event handlers using Step Functions tasks.

## Discovery Questions for This Domain

When a user asks about SBT or control plane tooling, explore:

**Fit assessment:**
- Are you using CDK for infrastructure? (SBT is CDK-only)
- Do you already have a control plane, or are you building from scratch?
- What's your identity provider? (SBT ships with Cognito — if you need Auth0/Okta, you'll need to extend it)

**Scope:**
- Which SBT capabilities do you actually need? (tenant management, billing, onboarding orchestration, all of them?)
- Are you planning to sell through AWS Marketplace? (SBT's billing integration is a big accelerator if yes)
- How complex is your per-tenant provisioning? (simple config update vs. multi-step infrastructure creation)

**Customization:**
- Do you need per-tenant user pools for SAML/SSO? (SBT's CognitoAuth uses a single pool — you'd need to extend it)
- Do you need custom tenant lifecycle states beyond Active/Inactive? (SBT supports basic states, custom states need extension)
- Do you have existing event infrastructure (EventBridge buses, event schemas) that SBT needs to integrate with?

## References

- [SaaS Builder Toolkit GitHub Repository](https://github.com/awslabs/sbt-aws)
- [SBT Developer Guide](https://github.com/awslabs/sbt-aws/blob/main/docs/public/README.md)
- [Building a Multi-Tenant SaaS Solution Using the SaaS Builder Toolkit](https://aws.amazon.com/blogs/apn/building-a-multi-tenant-saas-solution-using-the-saas-builder-toolkit-for-aws/)
- [SaaS Architecture Fundamentals — Control Plane](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/control-plane.html)
- [SaaS Architecture Fundamentals — Application Plane](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/application-plane.html)
