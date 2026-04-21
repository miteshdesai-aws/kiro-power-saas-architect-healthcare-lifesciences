# Tenancy Models & Architecture Foundations

## Core Concept: SaaS Is a Business Model, Not Just an Architecture

Per the AWS SaaS Architecture Fundamentals whitepaper: SaaS and multi-tenancy are not the same thing. SaaS is a business and delivery model. Multi-tenancy describes how resources are shared within that model. A fully siloed SaaS where every tenant has dedicated resources is still SaaS — as long as tenants are managed, operated, and deployed collectively through a unified experience.

This distinction matters because it frees you from thinking "we must share everything." The right answer is almost always a mix of shared and dedicated resources, varying by service.

## The Three Tenancy Models

### Pool Model (Shared Resources)
All tenants share the same compute, storage, and infrastructure. Tenant separation is logical (e.g., tenant ID in database rows, IAM policies scoping access).

**Strengths:**
- Lowest cost per tenant — infrastructure scales with total load, not tenant count
- Simplest deployment — one environment to manage, one pipeline
- Fastest onboarding — no new infrastructure to provision
- Best operational efficiency — single pane of glass for monitoring

**Weaknesses:**
- Noisy neighbor risk — one tenant's spike affects others
- Isolation complexity — must enforce at application and IAM level, not infrastructure level
- Compliance challenges — harder to prove data separation to auditors
- Right-to-erasure is harder — tenant data is interleaved with others
- Blast radius — a bug or bad deployment affects all tenants simultaneously

**Best for:** High tenant count (hundreds to thousands), SMB customers, cost-sensitive markets, early-stage SaaS where operational simplicity matters.

### Silo Model (Dedicated Resources)
Each tenant gets their own dedicated infrastructure — separate compute, storage, and potentially separate AWS accounts.

**Strengths:**
- Strongest isolation — infrastructure-level separation, easiest to audit
- No noisy neighbor — each tenant's resources are independent
- Simplest compliance story — "your data is in your own database/account"
- Per-tenant customization possible (though discouraged in SaaS)
- Clearest cost attribution — tag resources per tenant

**Weaknesses:**
- Highest cost per tenant — dedicated resources even when idle
- Operational overhead scales with tenant count — N tenants = N environments
- Slower onboarding — must provision infrastructure per tenant
- Deployment complexity — must deploy to every tenant environment
- Doesn't scale well beyond ~50-100 tenants without heavy automation

**Best for:** Enterprise customers with compliance requirements, regulated industries (healthcare, finance, government), premium tiers, low tenant count with high revenue per tenant.

### Bridge Model (Hybrid)
Some resources are shared, others are dedicated. The most common example: shared compute, separate storage per tenant (e.g., shared Lambda functions, separate DynamoDB tables or RDS schemas per tenant).

**Strengths:**
- Balances cost and isolation — share what's safe, isolate what's sensitive
- Flexible — can tune the shared/dedicated split per service
- Good compliance middle ground — data is separated even if compute is shared
- Easier tenant data management — backup, restore, delete per tenant

**Weaknesses:**
- More complex than either pure model — must manage both shared and dedicated resources
- Onboarding is more complex than pool — must provision some dedicated resources
- Cost is higher than pool but lower than silo

**Best for:** Mid-market customers, situations where data isolation is required but compute isolation isn't, bridge between pool (basic tier) and silo (premium tier).

## Critical Insight: Tenancy Varies Per Service

This is the most important concept from the AWS SaaS Architecture Fundamentals whitepaper.

In a real SaaS system with multiple microservices, each service can have a different tenancy model. Example:
- **Product service**: Pool model (shared compute, shared storage) — low sensitivity, high read volume
- **Order service**: Bridge model (shared compute, separate storage) — financial data needs isolation
- **Analytics service**: Pool model — aggregated data, no tenant sensitivity
- **Compliance service**: Silo model — regulatory requirement for dedicated resources

**When advising on tenancy, evaluate each service independently.** Don't pick one model for the entire system. Ask: "For this specific service, what are the isolation requirements, noisy neighbor risks, and compliance needs?"

## Control Plane vs. Application Plane

This is the foundational architectural split in any SaaS system, formalized by the SaaS Builder Toolkit (SBT).

### Control Plane (Always Shared)
The set of services that manage and operate your SaaS environment. This is shared across ALL tenants regardless of their tenancy model.

Includes:
- **Tenant management**: CRUD operations on tenants, tenant configuration
- **Identity and authentication**: User registration, login, tenant-user binding
- **Onboarding orchestration**: Automated provisioning of new tenants
- **Billing and metering**: Usage tracking, invoice generation, Marketplace integration
- **Tiering and throttling**: Defining and enforcing tier-based limits
- **Metrics and analytics**: Tenant activity dashboards, operational metrics
- **Deployment management**: Rolling out updates to all tenants

The control plane is what makes it SaaS. Without it, you just have separate deployments.

### Application Plane (Tenancy Model Varies)
The actual SaaS application that tenants use. This is where tenancy model decisions apply.

- In pool model: single shared deployment serving all tenants
- In silo model: separate deployment per tenant
- In bridge model: shared compute with tenant-specific resources

The application plane receives events from the control plane (e.g., "new tenant onboarded, provision resources") and responds accordingly.

## Tiering Strategies

Tiers are how you map business value to infrastructure decisions. Common pattern:

| Tier | Tenancy Model | Isolation | Throttling | Price |
|------|--------------|-----------|------------|-------|
| Free/Basic | Pool | Logical (IAM + app-level) | Aggressive limits | Low/free |
| Professional | Pool or Bridge | Logical + separate storage | Moderate limits | Mid |
| Enterprise | Silo or Bridge | Infrastructure-level | Generous or custom | High |

**Key principles:**
- Tiers should be defined in the control plane and enforced consistently
- Throttling limits per tier are enforced at API Gateway (usage plans) and application level
- Tenant portability: design so tenants can move between tiers (pool → silo upgrade)
- Don't over-tier early — start with 2 tiers (basic + premium), add more based on demand

## Decision Framework: Choosing a Tenancy Model

When advising a customer, walk through these factors for each service:

### 1. Compliance Requirements
- Regulatory mandate for data isolation? → Silo or Bridge (separate storage)
- Need to prove isolation to auditors? → Silo is easiest to demonstrate
- HIPAA, FedRAMP, PCI-DSS? → Likely silo for data stores, possibly pool for stateless compute

### 2. Noisy Neighbor Risk
- Service has highly variable load per tenant? → Consider silo for heavy tenants
- Service is stateless and scales horizontally? → Pool is usually fine
- Service has shared resource bottlenecks (DB connections, throughput)? → Bridge or silo

### 3. Tenant Count and Growth
- Expecting 10-50 tenants? → Silo is manageable
- Expecting 100-1000 tenants? → Pool or bridge (silo won't scale operationally)
- Expecting 1000+ tenants? → Pool is almost certainly required

### 4. Revenue Per Tenant
- High revenue per tenant (enterprise contracts)? → Silo is justified by revenue
- Low revenue per tenant (SMB, self-serve)? → Pool to keep costs viable
- Mixed? → Tiering with pool for basic, silo for enterprise

### 5. Operational Capacity
- Small team (2-5 engineers)? → Pool — you can't operate N separate environments
- Large platform team? → Silo is feasible with automation
- Using SBT or similar automation? → Silo becomes more manageable

### 6. Data Sensitivity
- Tenant data is highly sensitive (PII, financial, health)? → Silo or bridge for storage
- Tenant data is low sensitivity (preferences, configs)? → Pool is fine
- Mixed sensitivity across services? → Different models per service

## Default Recommendations

For healthcare SaaS teams that need a starting point, not a research project:

**Digital health / telehealth SaaS, small team (< 10 engineers):**
Bridge as the baseline for ALL services handling PHI — shared compute, per-tenant storage with per-tenant KMS keys. Pool only for non-PHI services (feature flags, aggregated analytics, public content). This is the healthcare default because HIPAA requires demonstrable data isolation, and bridge gives you that without silo's operational overhead. Don't silo anything until a health system contract or specific regulation requires it.

**EHR-adjacent / clinical workflow SaaS:**
Bridge for clinical data services (patient records, clinical notes, lab results). Pool for non-clinical services (scheduling, notifications, user preferences). If you integrate with hospital EHRs via FHIR, the FHIR data store (HealthLake) should be per-tenant (data-store-per-tenant model) for the strongest isolation story. Silo for any service where a health system contractually requires dedicated infrastructure.

**Clinical SaaS / imaging / radiology:**
Bridge or silo for imaging data (HealthImaging data stores are naturally per-tenant). Bridge for clinical workflow services. Pool for viewer infrastructure (shared compute for rendering). If your software is FDA-cleared (SaMD), the deployment model may be constrained by your validated environment requirements — see `clinical-saas-and-imaging.md`.

**Payer SaaS (claims, prior auth, member portals):**
Bridge for claims and member data (per-tenant storage, shared compute). Pool for reference data services (code sets, formularies, provider directories). Silo for large health plans that contractually require dedicated infrastructure. X12 EDI processing can be pool (shared pipeline) with tenant-scoped routing.

**Migration from on-prem / single-tenant hosted to healthcare SaaS:**
Start with silo (it's closest to what you have). Build the control plane. Then incrementally move services to bridge as you gain confidence. Don't try to go from single-tenant to pool in one step. For healthcare, the target state is usually bridge (not pool) because PHI isolation requirements persist even at scale.

**Key healthcare principle:** In generic SaaS, pool is the default and you silo when required. In healthcare SaaS, bridge is the default and you pool only for non-PHI services. The compliance burden of proving PHI isolation in a pure pool model is higher than the cost savings justify for most healthcare SaaS.

### Covered Entity vs Business Associate as Tenant

In healthcare SaaS, your tenants are usually one of:
- **Covered entities:** Hospitals, clinics, physician groups, health plans — they have direct HIPAA obligations
- **Business associates:** Other SaaS vendors, billing companies, clearinghouses — they handle PHI on behalf of covered entities

This distinction matters for tenancy because:
- Covered entity tenants often have stricter isolation requirements (their compliance team will audit your architecture)
- Large covered entities (health systems with 50+ hospitals) may demand silo as a contractual requirement
- Business associate tenants may be more flexible on isolation but need clear BAA chains
- Multi-covered-entity platforms (serving competing health systems) almost always need silo or strong bridge to prevent any perception of data commingling

These are starting points. Adjust based on your specific compliance, customer, and operational context.

## Common Tension Patterns

Real-world healthcare SaaS decisions involve conflicting requirements. Here's how to navigate the most common tensions:

**"We need HIPAA compliance but we're a small team targeting many clinics"**
Bridge model (shared compute, isolated data stores with per-tenant KMS keys) satisfies HIPAA requirements for most services. You don't need to silo everything. Focus silo on services where a specific regulation or contract demands it. Keep stateless, non-PHI services in pool. HIPAA compliance is about proving PHI isolation — bridge with IAM-enforced isolation and per-tenant encryption keys gives you that proof.

**"A large health system wants silo but we serve 500 small clinics in pool"**
Tiering solves this. Pool/bridge for the 95% of tenants that are small clinics. Silo (or strong bridge with dedicated storage + dedicated compute namespace) for the health system. The control plane manages both. Design your data access layer to abstract storage location. The health system pays a premium that justifies the operational cost. Be explicit in your BAA about what "dedicated" means at each tier.

**"We handle PHI but want pool for cost efficiency"**
Bridge is the healthcare answer. Pool the compute (Lambda, ECS), silo the storage (per-tenant DynamoDB tables, per-tenant RDS schemas, per-tenant S3 prefixes with per-tenant KMS keys). You get most of pool's cost benefits while maintaining PHI isolation that satisfies HITRUST assessors. Pure pool for PHI storage is possible with IAM dynamic policies + RLS, but the compliance burden of proving isolation is higher.

**"We serve competing health systems — they can't know about each other"**
This is a multi-covered-entity problem. Even in bridge model, ensure: no cross-tenant data leakage (IAM-enforced), no cross-tenant metadata leakage (tenant names, counts, usage patterns are not visible to other tenants), separate KMS keys, and consider separate HealthLake data stores. For the most sensitive cases (competing hospital networks), silo with separate AWS accounts may be the only option that satisfies both parties' legal teams.

**"We want to start simple but need HITRUST certification within a year"**
Start with bridge for PHI services, pool for non-PHI. Use per-tenant KMS keys from day one (retrofitting is painful). Enable CloudTrail data events, S3 Object Lock for audit logs, and application-level PHI access logging from the start. These are HITRUST requirements that are much harder to add later than to build in. See `healthcare-compliance-foundations.md` for the full HITRUST checklist.

**"We're building ambient documentation AI — is the AI output PHI?"**
Yes. If the input is PHI (clinical encounter audio), every intermediate and final output is PHI (transcription, AI-generated notes, coding suggestions). The entire pipeline must be treated as PHI. Use the dual-zone architecture pattern from `genai-and-phi.md`. The AI zone processes data but has no persistent PHI storage. The PHI zone stores results with full encryption and audit logging.

## Common Mistakes

1. **Picking one model for everything** — Evaluate per service. Your auth service and your analytics service have different needs.

2. **Over-siloing early** — Silo is expensive and operationally heavy. Don't silo because "it feels safer." Silo when there's a concrete requirement.

3. **Under-isolating in pool** — Pool doesn't mean "no isolation." You still need IAM-based or application-level isolation. Pool without isolation is a security incident waiting to happen.

4. **Ignoring tenant portability** — Design so tenants can move between tiers. If upgrading from pool to silo requires a data migration project, you've created a business problem.

5. **Treating SaaS as "hosted software"** — If each tenant runs a different version or has custom code, it's not SaaS. All tenants run the same version, deployed through the same pipeline.

6. **Building the application plane before the control plane** — The control plane (onboarding, identity, billing, metrics) is what makes it SaaS. Build it first or in parallel, not as an afterthought.

7. **Defaulting to pool for PHI services in healthcare SaaS** — In generic SaaS, pool is the default. In healthcare SaaS, bridge is the default for any service handling PHI. The compliance burden of proving PHI isolation in pure pool is higher than the cost savings justify.

8. **Not distinguishing covered entity vs business associate tenants** — A hospital (covered entity) has different isolation expectations than a billing company (business associate). Your tenancy model should account for this.

## Discovery Questions for This Domain

When the conversation is about tenancy models, ask these to gather the context needed for a recommendation:

**If you don't know the services yet:**
- What are the main services or components in your system? (Even a rough list helps — "we have an API, a background processor, and a dashboard" is enough to start.)
- Which of these services handle sensitive data? (PII, financial, health records, credentials)

**If you know the services but not the model:**
- For each service: does it need to store tenant-specific data, or is it stateless?
- Are there any services where one tenant's usage could spike and affect others? (batch processing, report generation, file uploads)
- Do any customers have contractual requirements for dedicated infrastructure?

**If they're choosing between models:**
- How many tenants do you expect in year 1? Year 3? (This is the single biggest factor — silo doesn't scale past ~50-100 without heavy automation)
- What's your average revenue per tenant? (If it's $50/month, silo is economically impossible. If it's $50K/month, silo is justified.)
- How big is your platform/ops team? (Silo requires operational maturity. A 3-person team can't manage 50 separate environments.)

**If they're designing tiers:**
- What differentiates your tiers from the customer's perspective? (Features? Limits? Support? Isolation?)
- Do enterprise customers explicitly ask for "dedicated" infrastructure, or is it your assumption?
- Do you need tenants to be able to upgrade tiers without downtime or data migration?

## References

- [SaaS Architecture Fundamentals — Re-defining Multi-tenancy](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/re-defining-multi-tenancy.html)
- [SaaS Architecture Fundamentals — Data Partitioning](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/data-partitioning.html)
- [SaaS Lens — General Design Principles](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/general-design-principles.html)
- [Let's Architect! Building Multi-Tenant SaaS Systems](https://aws.amazon.com/blogs/architecture/lets-architect-building-multi-tenant-saas-systems/)
- [Tenant Portability: Move Tenants Across Tiers](https://aws.amazon.com/blogs/architecture/tenant-portability-move-tenants-across-tiers-in-a-saas-application/)
