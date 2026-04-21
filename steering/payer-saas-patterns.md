# Payer SaaS Patterns

## Segment Overview

Payer SaaS includes multi-tenant platforms built for health plans, third-party administrators (TPAs), clearinghouses, and insurance technology companies. Tenants are health plans, employer groups, or payer organizations. Key workloads: claims processing, prior authorization, member portals, enrollment, eligibility verification, provider network management, and regulatory reporting.

**Honest scope note:** This file covers infrastructure, compliance, and interoperability patterns for payer SaaS on AWS. It does NOT cover claims adjudication business logic, benefits configuration rules engines, or actuarial modeling. Those are domain-specific and vary significantly by payer type.

## X12 EDI on AWS

X12 Electronic Data Interchange is the standard for electronic healthcare transactions in the US, mandated by HIPAA for covered transactions.

### Key Transaction Types

| Transaction | X12 Code | Direction | Purpose |
|------------|----------|-----------|---------|
| Eligibility Inquiry | 270 | Provider → Payer | Check if a patient is covered |
| Eligibility Response | 271 | Payer → Provider | Confirm coverage and benefits |
| Claim Submission | 837 (P/I/D) | Provider → Payer | Submit professional, institutional, or dental claims |
| Claim Status Inquiry | 276 | Provider → Payer | Check status of a submitted claim |
| Claim Status Response | 277 | Payer → Provider | Report claim status |
| Prior Auth Request | 278 | Provider → Payer | Request prior authorization |
| Prior Auth Response | 278 | Payer → Provider | Approve/deny/pend prior auth |
| Remittance Advice | 835 | Payer → Provider | Explain payment for claims |
| Enrollment | 834 | Employer → Payer | Enroll/disenroll members |
| Premium Payment | 820 | Employer → Payer | Report premium payments |

### EDI Processing Pipeline on AWS

```
EDI Source (clearinghouse/provider) → S3 (raw EDI files)
    → Lambda (parse X12 → structured data)
    → DynamoDB/RDS (transaction store)
    → Business Logic (adjudication, eligibility check, etc.)
    → Lambda (structured data → X12 response)
    → S3 (outbound EDI files)
    → EDI Destination (clearinghouse/provider)
```

**Multi-tenant routing:** Each tenant (health plan) receives EDI from different providers/clearinghouses. Route inbound EDI to the correct tenant based on:
- ISA/GS envelope headers (receiver ID maps to tenant)
- Dedicated S3 prefixes per tenant
- Separate SQS queues per tenant for processing isolation

**Parsing libraries:** X12 parsing is non-trivial. Consider open-source libraries (Stedi, x12-parser) or AWS partner solutions rather than building from scratch.

### EDI as PHI
X12 transactions contain PHI (patient names, dates of birth, diagnoses, procedures). All EDI processing infrastructure must be HIPAA-compliant:
- S3 buckets encrypted with KMS
- Lambda functions in VPC (if accessing PHI data stores)
- DynamoDB/RDS encrypted at rest
- All transit encrypted (TLS)
- Audit logging for all EDI processing (CloudTrail + application logs)

## Claims Processing

### Paper-to-Electronic Claims
Despite EDI mandates, ~10% of claims still arrive as paper (CMS-1500 for professional, UB-04 for institutional).

**AWS pattern:**
```
Paper claim (scanned PDF) → S3
    → Amazon Textract (OCR + form extraction)
    → Lambda (map extracted fields to X12 837 structure)
    → Claims processing pipeline (same as electronic)
```

**Textract capabilities:**
- Extracts structured data from CMS-1500 and UB-04 forms
- Identifies key fields: patient info, provider info, diagnosis codes, procedure codes, charges
- Confidence scores per field — low-confidence fields flagged for manual review

**Reference:** [Automating Paper-to-Electronic Healthcare Claims Processing with AWS](https://aws.amazon.com/blogs/storage/automating-paper-to-electronic-healthcare-claims-processing-with-aws/)

### Medical Coding with AI
Amazon Comprehend Medical can extract medical entities from clinical text and map them to standard code systems:
- ICD-10-CM (diagnosis codes)
- RxNorm (medication codes)
- SNOMED CT (clinical terms)

**Use case in payer SaaS:** Validate submitted diagnosis and procedure codes against clinical documentation. Flag mismatches for review. This is a revenue integrity / fraud detection pattern.

### Claims Adjudication Architecture
Claims adjudication (determining payment for a claim) is the core business logic of a payer platform. The architecture pattern:

```
Claim received → Eligibility check → Benefits lookup → Rules engine → Payment determination → Remittance
```

**Multi-tenant considerations:**
- Each tenant (health plan) has different benefits configurations, fee schedules, and adjudication rules
- Rules engine must be tenant-scoped — Tenant A's rules never apply to Tenant B's claims
- Benefits configuration is tenant-specific data — store in per-tenant data partitions
- Fee schedules are tenant-specific — per-tenant tables or per-tenant partitions

**This file does not prescribe a specific rules engine architecture.** Claims adjudication logic varies significantly by payer type (commercial, Medicare Advantage, Medicaid, dental, vision). The infrastructure patterns (multi-tenant data partitioning, isolation, scaling) from the core SaaS steering files apply.

## CMS-0057-F Compliance

The CMS Interoperability and Prior Authorization Final Rule mandates FHIR-based APIs for payers. **Compliance deadline: January 1, 2027.**

### Who Must Comply
- Medicare Advantage organizations
- Medicaid managed care plans
- CHIP managed care entities
- Qualified Health Plan (QHP) issuers on the Marketplace

### Required APIs

**Patient Access API**
- Give patients access to their claims, encounters, clinical data, and prior auth decisions via FHIR
- Must support US Core profiles + CARIN Blue Button IG (for claims/EOB data)
- Patient authenticates via OAuth 2.0 and authorizes third-party apps

**Provider Access API**
- Give in-network providers access to patient data for treatment purposes
- Payer must make data available within 1 business day of receiving it
- Provider authenticates via OAuth 2.0 with appropriate scopes

**Payer-to-Payer API**
- Exchange patient data between payers when a patient switches health plans
- Must support US Core profiles
- Triggered by patient request (opt-in)

**Prior Authorization API**
- Accept and respond to prior auth requests via FHIR
- Must implement Da Vinci Implementation Guides:
  - CRD (Coverage Requirements Discovery): provider queries payer for coverage requirements
  - DTR (Documentation Templates and Rules): payer provides documentation templates
  - PAS (Prior Authorization Support): provider submits prior auth request, payer responds
- Must provide prior auth decisions within 72 hours (urgent) or 7 days (standard)
- Must include specific denial reasons

### Architecture for CMS-0057-F

**FHIR data store:** HealthLake as the backend for all four APIs. Store claims as FHIR ExplanationOfBenefit resources, clinical data as US Core resources, prior auth as FHIR Claim/ClaimResponse.

**API layer:** API Gateway + Lambda (or ECS) implementing the FHIR API endpoints. SMART on FHIR for patient and provider authentication.

**Multi-tenant:** Each tenant (health plan) has their own member population. FHIR data must be partitioned by tenant. Use HealthLake data-store-per-tenant for strong isolation, or shared data store with tenant metadata for cost efficiency.

**Timeline:** If you're building payer SaaS, these APIs must be in your product by Jan 2027. Start now.

**Reference:** [CMS-0057-F Final Rule](https://www.cms.gov/priorities/burden-reduction/overview/interoperability/policies-regulations/cms-interoperability-prior-authorization-final-rule-cms-0057-f)

## Prior Authorization Automation

Prior authorization is one of the most burdensome processes in healthcare. CMS-0057-F mandates automation via FHIR, and AI can further accelerate it.

### FHIR-Based Prior Auth (Da Vinci PAS)
```
Provider EHR → CRD (coverage check) → DTR (documentation) → PAS (submit prior auth)
    ↓                                                              ↓
Payer SaaS receives FHIR Claim resource → Rules engine → Auto-approve / Pend for review / Deny
    ↓
FHIR ClaimResponse returned to provider
```

### AI-Assisted Prior Auth with Bedrock AgentCore
AWS published a reference pattern for multi-agent RCM using Bedrock AgentCore:
- Document processing agent: extracts clinical information from submitted documentation
- Medical coding agent: validates diagnosis and procedure codes
- Coverage determination agent: checks against plan benefits and medical policies
- Decision agent: synthesizes inputs and recommends approve/deny/pend

**Multi-tenant:** Each tenant's medical policies and benefits are different. The agents must be configured per tenant (different knowledge bases, different rules).

**Reference:** [Transform Healthcare Revenue Cycle Management with Amazon Bedrock AgentCore](https://aws.amazon.com/blogs/industries/transform-healthcare-revenue-cycle-management-with-amazon-bedrock-agentcore/)

## Member Portal Patterns

Health plan members (patients) need a portal to view their benefits, claims, ID cards, and find providers.

### Architecture
- Frontend: React/Next.js served via CloudFront
- Authentication: Cognito with member identity (email/phone + MFA)
- Backend: API Gateway + Lambda/ECS
- Data: HealthLake (FHIR) for clinical/claims data, DynamoDB for member profiles and preferences
- FHIR Patient Access API: the same API required by CMS-0057-F serves the member portal

### Multi-Tenant Member Identity
- Members belong to a specific health plan (tenant)
- Member ID is unique within a tenant but may not be globally unique
- Cognito custom attribute: `tenant_id` (health plan) + `member_id`
- Members should only see their own data within their health plan's tenant

## Multi-Tenant Considerations for Payer SaaS

### Tenant = Health Plan
In payer SaaS, the tenant is typically a health plan or TPA. Each tenant has:
- Their own member population
- Their own benefits configuration
- Their own fee schedules and contracts
- Their own provider network
- Their own regulatory requirements (state-specific Medicaid rules, etc.)

### Data Sensitivity
- Member data (demographics, enrollment) — PHI
- Claims data (diagnoses, procedures, charges) — PHI
- Provider network data (NPI, addresses, contracts) — may contain business-sensitive data but not PHI
- Benefits configuration — business-sensitive but not PHI
- Fee schedules — highly business-sensitive (competitive advantage)

### Isolation Recommendations
- Member and claims data: bridge or silo (per-tenant storage with per-tenant KMS keys)
- Provider network data: pool acceptable (less sensitive, often shared across plans)
- Benefits configuration: per-tenant storage (business-critical, tenant-specific)
- Fee schedules: per-tenant storage with strict access control (competitive data)

## Common Mistakes

1. **Not planning for CMS-0057-F.** The Jan 2027 deadline is real and the scope is large (four FHIR APIs + Da Vinci IGs). If you're building payer SaaS and haven't started, you're behind.

2. **Building a custom X12 parser.** X12 parsing is complex and error-prone. Use established libraries or partner solutions. Focus your engineering on business logic, not parsing.

3. **Treating EDI files as non-PHI.** X12 transactions contain patient names, dates of birth, diagnoses, and procedures. All EDI infrastructure must be HIPAA-compliant.

4. **No tenant isolation for benefits and fee schedules.** These are competitively sensitive. If Tenant A can see Tenant B's fee schedules, you've lost both customers.

5. **Ignoring state-specific Medicaid rules.** Each state has different Medicaid managed care requirements. If your tenants include Medicaid plans, your platform must support state-specific configuration.

6. **Prior auth automation without human review.** Even with AI-assisted prior auth, clinical decisions (especially denials) should have human review. Automated denials without clinical review create regulatory and legal risk.

## Discovery Questions for This Domain

**Payer type:**
- What types of health plans do your tenants operate? (Commercial, Medicare Advantage, Medicaid, dental, vision?)
- How many members per tenant? (Thousands to millions — affects data volume and performance requirements)
- Do tenants operate in multiple states? (State-specific regulatory requirements)

**EDI:**
- What X12 transaction types do you need to support? (837, 835, 270/271, 276/277, 278?)
- How do you receive EDI today? (Direct from providers, via clearinghouse, both?)
- What volume of transactions per day? (Affects pipeline sizing)

**CMS-0057-F:**
- Are your tenants subject to CMS-0057-F? (Medicare Advantage, Medicaid, CHIP, QHP?)
- Have you started implementing the required FHIR APIs? (Patient Access, Provider Access, Payer-to-Payer, Prior Auth?)
- Are you using Da Vinci Implementation Guides? (CRD, DTR, PAS?)

**Claims:**
- Do you handle claims adjudication or just claims routing/clearinghouse functions?
- Do you receive paper claims? (Textract for OCR)
- Do you need AI-assisted medical coding or prior auth? (Comprehend Medical, Bedrock)

## References

- [CMS-0057-F Final Rule](https://www.cms.gov/priorities/burden-reduction/overview/interoperability/policies-regulations/cms-interoperability-prior-authorization-final-rule-cms-0057-f)
- [Automating Paper-to-Electronic Healthcare Claims Processing with AWS](https://aws.amazon.com/blogs/storage/automating-paper-to-electronic-healthcare-claims-processing-with-aws/)
- [Transform Healthcare RCM with Amazon Bedrock AgentCore](https://aws.amazon.com/blogs/industries/transform-healthcare-revenue-cycle-management-with-amazon-bedrock-agentcore/)
- [Cloud Solutions for Healthcare Payors](https://aws.amazon.com/health/payors/)
- [Build a Member-360 Unified View Using Data Mesh with Amazon HealthLake](https://aws.amazon.com/blogs/industries/build-a-member-360-unified-view-using-data-mesh-with-amazon-healthlake/)
- [HealthEdge HealthRules Payer on AWS](https://healthedge.com/resources/press-releases/healthedge-healthrules-payer-works-with-aws-to-set-a-new-scalability-benchmark-expanding-to-support-health-plans-with-more-than-40-million)
- [Da Vinci Prior Authorization Support IG](https://www.hl7.org/fhir/us/davinci-pas/)
- [CARIN Blue Button Implementation Guide](https://build.fhir.org/ig/HL7/carin-bb/)
