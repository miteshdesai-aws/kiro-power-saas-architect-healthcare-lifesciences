# FHIR & Interoperability

## Why Interoperability Defines Healthcare SaaS

Healthcare SaaS doesn't exist in isolation. It connects to EHRs, labs, imaging systems, payers, and patient apps. The interoperability standards you support determine which customers you can sell to and which integrations you can build. FHIR R4 is the dominant standard, mandated by CMS/ONC for payers and increasingly expected by providers.

## FHIR R4 Overview

Fast Healthcare Interoperability Resources (FHIR) is an HL7 standard for exchanging healthcare data via RESTful APIs. FHIR R4 is the normative release and the version mandated by US federal regulations.

### Key Concepts
- **Resources:** The building blocks — Patient, Observation, Condition, MedicationRequest, Encounter, DiagnosticReport, ImagingStudy, Claim, ExplanationOfBenefit, Consent, etc.
- **RESTful API:** Standard CRUD operations (GET, POST, PUT, DELETE) on resources
- **Search:** Parameterized queries across resources (e.g., `GET /Patient?name=Smith&birthdate=1980-01-01`)
- **Operations:** Named operations beyond CRUD (e.g., `$export` for bulk data, `$everything` for patient summary)
- **Bundles:** Collections of resources for batch operations or transaction integrity
- **Profiles:** Constraints on base resources for specific use cases (US Core profiles are the US baseline)

### US Core Profiles
The US Core Implementation Guide defines the minimum set of FHIR profiles that US systems must support. If you're building for the US market, support US Core as your baseline. Key profiles: Patient, Practitioner, Organization, Encounter, Condition, Observation (vitals, labs, social history), MedicationRequest, AllergyIntolerance, Procedure, DiagnosticReport, DocumentReference.

## AWS HealthLake — Multi-Tenant FHIR

HealthLake is a HIPAA-eligible, fully managed FHIR R4 data store. It handles FHIR compliance, storage, and query — you focus on your application logic.

### Multi-Tenant Patterns

**Pattern 1: Data Store Per Tenant (Silo)**
Each tenant gets their own HealthLake data store with its own FHIR endpoint.

- **Isolation:** Strongest — separate endpoints, separate KMS keys, separate IAM policies
- **Cost:** Higher — per-data-store pricing, minimum cost per store regardless of usage
- **Operations:** Scales with tenant count — N tenants = N data stores to manage
- **Analytics:** Cross-tenant analytics requires aggregation layer (query each store, combine results)
- **Onboarding:** Slower — data store creation takes minutes
- **Best for:** Enterprise health systems, regulated environments, < 100 tenants

**Pattern 2: Shared Data Store with Tenant Metadata (Pool)**
All tenants share one HealthLake data store. Tenant identity is encoded in FHIR resource metadata.

- **Tenant tagging approaches:**
  - FHIR resource tags: add a `tenant` tag to every resource
  - FHIR extensions: add a tenant extension to resource metadata
  - Compartment-based: use FHIR compartments to scope access
- **Isolation:** Weaker — relies on SMART on FHIR scopes + application-level filtering + IAM session policies
- **Cost:** Lower — single data store, pay for total storage and requests
- **Operations:** Simpler — one data store to manage
- **Analytics:** Easier — all data in one store, filter by tenant tag
- **Onboarding:** Fast — just create tenant metadata, no infrastructure provisioning
- **Best for:** High tenant count (100+), SMB clinics, cost-sensitive

**Pattern 3: Hybrid**
Shared data store for most tenants, dedicated data stores for enterprise tenants. Control plane routes FHIR requests to the correct data store based on tenant configuration.

- **Best for:** Mixed SMB + enterprise customer base
- **Implementation:** API Gateway + Lambda router that checks tenant config → routes to shared or dedicated HealthLake endpoint

### HealthLake Features for Multi-Tenant
- **SMART on FHIR support:** HealthLake supports SMART on FHIR 1.0 for authorization. Use this to scope access by patient, practitioner, or tenant.
- **Zero-ETL to Apache Iceberg:** HealthLake automatically populates FHIR data into Iceberg tables governed by Lake Formation. Use for cross-tenant analytics without querying the FHIR API.
- **Bulk export ($export):** Export all resources for a tenant (or patient) as NDJSON to S3. Useful for tenant data portability and right-to-access requests.
- **De-identified copy:** Create a de-identified version of a data store for analytics/research. See `phi-data-handling.md`.

**Reference:** [Building a Multi-Tenant FHIR Server with AWS HealthLake](https://aws.amazon.com/blogs/industries/building-a-multi-tenant-fhir-server-with-aws-healthlake/)

## SMART on FHIR — Data Exchange Patterns

SMART on FHIR defines how applications access FHIR data with proper authorization. The authentication aspects are covered in `identity-and-onboarding.md`. This section covers data exchange patterns.

### EHR Launch
The clinician launches your app from within the EHR (Epic, Cerner). The EHR provides launch context (patient ID, encounter ID, practitioner ID) and an access token scoped to that context.

**Flow:**
1. Clinician clicks your app's icon in the EHR
2. EHR redirects to your app with a `launch` parameter
3. Your app exchanges the launch parameter for an authorization code
4. Your app exchanges the code for an access token with FHIR scopes
5. Your app uses the token to access FHIR resources for the current patient/encounter

**Multi-tenant implication:** The EHR launch context includes the organization (your tenant). Your app must map the EHR organization to your internal tenant ID and scope all operations accordingly.

### Standalone Launch
The user (patient or clinician) opens your app directly (not from within an EHR). Your app initiates the SMART authorization flow.

**Flow:**
1. User opens your app and selects their FHIR server (or your app knows it based on tenant config)
2. Your app redirects to the FHIR server's authorization endpoint
3. User authenticates and authorizes your app
4. Your app receives an access token with FHIR scopes
5. Your app uses the token to access FHIR resources

### Backend Services (System-to-System)
For automated data exchange without user interaction (e.g., nightly data sync, bulk import).

**Flow:**
1. Your app authenticates using a signed JWT (client credentials grant with asymmetric key)
2. FHIR server validates the JWT and issues an access token
3. Your app uses the token for bulk operations

**Multi-tenant implication:** Each tenant may have a different FHIR server endpoint. Store per-tenant FHIR server configuration (endpoint URL, client ID, signing key) in the control plane.

## HL7 v2 Integration

HL7 v2 is the legacy standard still used by most hospital systems for real-time messaging. If your SaaS integrates with hospital workflows, you'll likely need to support HL7 v2 alongside FHIR.

### Common Message Types
| Message | Trigger | Use Case |
|---------|---------|----------|
| ADT (A01-A08) | Admit, Discharge, Transfer | Patient movement notifications |
| ORM | Order entry | Lab/radiology order placement |
| ORU | Observation result | Lab results, radiology reports |
| MDM | Medical document | Clinical document notifications |
| SIU | Scheduling | Appointment scheduling |
| DFT | Detailed financial transaction | Charge posting |

### Integration Architecture
```
Hospital HL7 v2 → Interface Engine (Mirth/NextGen Connect on ECS) → FHIR Transformer → HealthLake
                                                                   → Event Bus (EventBridge) → Your Services
```

**Interface engine options:**
- **Mirth Connect / NextGen Connect:** Open-source, widely used in healthcare. Run on ECS/Fargate. Handles HL7 v2 parsing, routing, transformation.
- **AWS HealthLake Import:** HealthLake can import FHIR bundles. Transform HL7 v2 → FHIR in a Lambda function, then import to HealthLake.
- **Custom Lambda:** For simple message types, a Lambda function can parse HL7 v2 and produce FHIR resources directly.

**Multi-tenant implication:** Each tenant (hospital) sends HL7 v2 messages from their own systems. The interface engine must route messages to the correct tenant's data store. Use tenant-specific HL7 v2 endpoints or message header fields (MSH-4 sending facility) to identify the tenant.

### HL7 v2 to FHIR Transformation
- Map HL7 v2 segments to FHIR resources (PID → Patient, OBX → Observation, ORC → ServiceRequest)
- Use FHIR ConceptMap for code system mapping (HL7 v2 codes → FHIR ValueSets)
- Handle partial data: HL7 v2 messages often have incomplete data — your transformer must handle missing fields gracefully
- Maintain message provenance: store the original HL7 v2 message alongside the FHIR resources for audit

## CMS/ONC Interoperability Rules

### CMS-0057-F (Interoperability and Prior Authorization Final Rule)
This rule mandates FHIR-based APIs for payers. Compliance deadline: **January 1, 2027**.

**Required APIs:**
| API | Who Must Implement | What It Does |
|-----|-------------------|-------------|
| Patient Access API | Medicare Advantage, Medicaid, CHIP, QHP issuers | Give patients access to their claims, encounters, and clinical data via FHIR |
| Provider Access API | Same payers | Give in-network providers access to patient data for treatment |
| Payer-to-Payer API | Same payers | Exchange patient data between payers when a patient switches plans |
| Prior Authorization API | Same payers | Accept and respond to prior auth requests via FHIR (CRD, DTR, PAS) |

**Architecture implication for payer SaaS:** If you're building a payer platform, these APIs must be part of your product. Use HealthLake as the FHIR data store. Implement Da Vinci Implementation Guides (CRD, DTR, PAS) for prior authorization. See `payer-saas-patterns.md` for detailed payer architecture.

### ONC Cures Act Final Rule
Requires health IT developers (EHR vendors) to support FHIR-based APIs for patient access and prohibits information blocking. If your SaaS is certified health IT, you must comply.

**Key requirement:** Support US Core FHIR profiles for data exchange. No information blocking — you cannot restrict patient access to their data through technical or business practices.

## DICOMweb

DICOMweb is the RESTful interface for DICOM medical imaging data. It's the modern alternative to the legacy DIMSE protocol.

### Key Services
| Service | Method | Purpose |
|---------|--------|---------|
| WADO-RS | GET | Retrieve DICOM instances, metadata, or rendered images |
| STOW-RS | POST | Store DICOM instances |
| QIDO-RS | GET | Query for DICOM studies, series, instances |

### Relationship to FHIR
- FHIR `ImagingStudy` resource references DICOM studies and can include DICOMweb endpoints
- A clinical workflow might: query HealthLake for a patient's ImagingStudy resources → use the DICOMweb endpoint in the resource to retrieve images from HealthImaging
- See `clinical-saas-and-imaging.md` for deep DICOM/HealthImaging coverage

## Bulk Data Access

### FHIR $export
The FHIR Bulk Data Access specification defines how to export large datasets as NDJSON files.

**Use cases in multi-tenant healthcare SaaS:**
- **Tenant data portability:** Export all of a tenant's FHIR data when they leave (right to data portability)
- **Patient right to access:** Export all FHIR resources for a specific patient (HIPAA §164.524)
- **Analytics pipeline:** Periodic bulk export to a data lake for population health analytics
- **Backup:** Bulk export as a supplementary backup mechanism

**HealthLake $export:** HealthLake supports $export natively. Exports to S3 as NDJSON. Can scope by resource type, date range, or patient.

**Multi-tenant consideration:** Ensure $export is scoped to the requesting tenant's data. For data-store-per-tenant, this is automatic. For shared data store, the export must filter by tenant metadata.

## Common Mistakes

1. **Building a custom FHIR server instead of using HealthLake.** Unless you have very specific requirements that HealthLake can't meet, use the managed service. Building a FHIR-compliant server is a multi-year effort.

2. **Ignoring US Core profiles.** If you're building for the US market, US Core is the baseline. Customers and regulators expect it. Don't invent your own profiles when US Core covers the use case.

3. **Not supporting HL7 v2.** FHIR is the future, but HL7 v2 is the present. Most hospital systems still send ADT, ORM, and ORU messages via HL7 v2. If you can't receive HL7 v2, you can't integrate with most hospitals.

4. **Losing tenant context in FHIR operations.** Every FHIR request must be scoped to a tenant. In shared HealthLake data stores, a missing tenant filter in a search query returns all tenants' data. Use SMART on FHIR scopes and IAM session policies as guardrails.

5. **Not planning for CMS-0057-F.** If you're building payer SaaS, the Jan 2027 deadline is real. The FHIR APIs (Patient Access, Provider Access, Payer-to-Payer, Prior Auth) must be part of your product roadmap.

6. **Treating HL7 v2 → FHIR transformation as trivial.** HL7 v2 messages are inconsistent across hospitals. The same message type can have different field usage, code systems, and data quality. Budget significant effort for transformation logic and edge cases.

## Discovery Questions for This Domain

**FHIR:**
- Do you need a FHIR data store? (If yes, HealthLake is the starting point)
- Which FHIR resources do you need to support? (Patient, Observation, Condition, etc.)
- Do you need to support US Core profiles? (Yes if US market)
- Do you need SMART on FHIR for EHR integration? (Yes if integrating with Epic, Cerner, etc.)

**HL7 v2:**
- Do you need to receive HL7 v2 messages from hospital systems? (ADT, ORM, ORU?)
- Do you have an interface engine today? (Mirth, Rhapsody, custom?)
- How many hospital interfaces do you need to support? (Each hospital may have different HL7 v2 configurations)

**Regulatory:**
- Are you subject to CMS-0057-F? (Payer SaaS — Patient Access API, Prior Auth API deadlines)
- Are you certified health IT subject to ONC Cures Act? (Information blocking prohibition)
- Do you need to support patient data portability? (FHIR $export, right to access)

**Multi-tenant:**
- How will you partition FHIR data across tenants? (HealthLake data store per tenant vs shared — see `data-partitioning.md`)
- Do different tenants connect to different EHR systems? (Per-tenant FHIR server configuration)
- Do you need cross-tenant FHIR queries? (Analytics, population health — requires aggregation layer)

## References

- [Building a Multi-Tenant FHIR Server with AWS HealthLake](https://aws.amazon.com/blogs/industries/building-a-multi-tenant-fhir-server-with-aws-healthlake/)
- [Enhanced Interoperability with SMART on FHIR Support in Amazon HealthLake](https://aws.amazon.com/blogs/industries/enhanced-interoperability-with-smart-on-fhir-support-in-amazon-healthlake/)
- [AWS HealthLake Developer Guide](https://docs.aws.amazon.com/healthlake/latest/devguide/what-is.html)
- [HL7 FHIR R4 Specification](https://hl7.org/fhir/R4/)
- [US Core Implementation Guide](https://www.hl7.org/fhir/us/core/)
- [SMART on FHIR Specification](https://smarthealthit.org/)
- [CMS-0057-F Final Rule](https://www.cms.gov/priorities/burden-reduction/overview/interoperability/policies-regulations/cms-interoperability-prior-authorization-final-rule-cms-0057-f)
- [Da Vinci Implementation Guides](https://www.hl7.org/fhir/us/davinci-pas/)
- [AI-Powered Patient Profiles Using AWS HealthLake and Amazon Bedrock](https://aws.amazon.com/blogs/industries/ai-powered-patient-profiles-using-aws-healthlake-and-amazon-bedrock/)
