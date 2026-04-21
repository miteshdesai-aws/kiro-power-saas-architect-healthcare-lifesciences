# Healthcare Compliance Foundations

## Why This File Exists

Every healthcare SaaS architecture decision is constrained by compliance. HIPAA is the floor, not the ceiling. This file covers the regulatory landscape that shapes every other architectural decision in the power. Load this file early in any healthcare SaaS conversation.

## HIPAA Security Rule — Technical Safeguards Mapped to AWS

The HIPAA Security Rule (45 CFR Part 164) defines safeguards for ePHI. Technical safeguards are the ones that directly affect architecture. Each has implementation specifications that are either Required (R) or Addressable (A). Addressable does NOT mean optional — it means you must implement it or document why an equivalent alternative is appropriate.

### 164.312(a)(1) — Access Control (R)

**What it requires:** Implement technical policies and procedures to allow access only to authorized persons or software programs.

**Implementation specifications:**
- Unique User Identification (R): Every user must have a unique identifier. Map to Cognito user IDs with tenant binding via custom attributes.
- Emergency Access Procedure (R): Procedures for obtaining ePHI during an emergency. See `audit-logging-and-access.md` for break-the-glass patterns.
- Automatic Logoff (A): Terminate sessions after inactivity. Cognito token expiration + application-level session timeout.
- Encryption and Decryption (A): Encrypt ePHI. See `phi-data-handling.md` for per-service encryption strategy.

**AWS mapping:** Cognito (user identity), IAM (access policies), KMS (encryption), session policies (tenant-scoped access).

### 164.312(b) — Audit Controls (R)

**What it requires:** Implement mechanisms to record and examine activity in systems containing ePHI.

**AWS mapping:** CloudTrail (API activity), CloudWatch Logs (application logs), S3 access logging, VPC Flow Logs. See `audit-logging-and-access.md` for detailed implementation.

**Key requirement:** Logs must be retained for 6 years minimum. Many state laws require longer.

### 164.312(c)(1) — Integrity Controls (R)

**What it requires:** Implement policies to protect ePHI from improper alteration or destruction.

**AWS mapping:** S3 Object Lock (immutable storage), DynamoDB point-in-time recovery, RDS automated backups, S3 versioning, CloudTrail log file validation.

### 164.312(d) — Person or Entity Authentication (R)

**What it requires:** Verify that a person or entity seeking access to ePHI is who they claim to be.

**AWS mapping:** Cognito (MFA, password policies), SAML/OIDC federation for health system IdPs, SMART on FHIR authentication. See `identity-and-onboarding.md`.

### 164.312(e)(1) — Transmission Security (R)

**What it requires:** Protect ePHI transmitted over electronic networks.

**Implementation specifications:**
- Integrity Controls (A): Ensure ePHI is not improperly modified during transmission.
- Encryption (A): Encrypt ePHI in transit.

**AWS mapping:** TLS everywhere (ALB, API Gateway, CloudFront), VPC endpoints to keep traffic off public internet, PrivateLink for health system connectivity, ACM for certificate management.

## HIPAA Administrative Safeguards — Architecture-Relevant Sections

Not all administrative safeguards affect architecture, but these do:

### 164.308(a)(1) — Security Management Process
- **Risk Analysis (R):** Must conduct thorough assessment of risks to ePHI. Architecture decisions should be traceable to risk analysis findings. ADRs serve this purpose.
- **Risk Management (R):** Implement security measures to reduce risks. Every architectural decision for PHI services should reference the risk it mitigates.

### 164.308(a)(6) — Security Incident Procedures
- **Response and Reporting (R):** Must identify and respond to suspected or known security incidents. Architecture must support incident detection (GuardDuty, Macie, CloudTrail anomaly detection) and response (automated remediation, isolation).

### 164.308(a)(7) — Contingency Plan
- **Data Backup Plan (R):** Must create and maintain retrievable exact copies of ePHI. Affects backup strategy per storage service.
- **Disaster Recovery Plan (R):** Must establish procedures to restore lost data. Affects multi-AZ, multi-region, and RPO/RTO decisions.
- **Emergency Mode Operation Plan (R):** Must establish procedures for continued operation during emergencies. Affects high-availability architecture.

## Business Associate Agreement (BAA) Discipline

### What a BAA Is
A BAA is a contract between a covered entity (healthcare provider, health plan, clearinghouse) and a business associate (anyone who handles PHI on their behalf). As a healthcare SaaS provider, you are almost certainly a business associate. Your customers are covered entities (or other BAs).

### AWS BAA
- AWS offers a BAA as an addendum to the AWS Customer Agreement
- The BAA covers all services on the [HIPAA Eligible Services list](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/)
- You must execute the BAA before processing PHI on AWS — having an AWS account is not enough
- The BAA is account-level, not per-service

### The BAA Chain
```
Covered Entity (hospital) → BAA → Your SaaS (business associate) → BAA → AWS (subcontractor BA)
                                                                    → BAA → Other subprocessors
```

Every entity in the chain that touches PHI needs a BAA. If you use a third-party service that processes PHI (analytics, email, SMS), you need a BAA with them too.

### HIPAA Eligible Services — The Hard Guardrail

**CRITICAL RULE:** Only use HIPAA Eligible Services for workloads that process, store, or transmit PHI. The list includes ~150+ services but NOT every AWS service.

**Common services that ARE eligible:** EC2, Lambda, ECS, EKS, Fargate, S3, DynamoDB, RDS, Aurora, API Gateway, CloudFront, Cognito, KMS, CloudTrail, CloudWatch, SNS, SQS, Step Functions, EventBridge, HealthLake, HealthImaging, Bedrock, Comprehend Medical, Transcribe Medical, SageMaker, OpenSearch, ElastiCache, Kinesis, Glue, Athena, Macie, GuardDuty, Config, Systems Manager.

**Always verify against the current list.** Services are added regularly. Check: https://aws.amazon.com/compliance/hipaa-eligible-services-reference/

### Shared Responsibility Model for HIPAA

AWS secures the infrastructure (physical security, hypervisor, network). You secure everything else:
- IAM configuration (least privilege, tenant isolation policies)
- Encryption configuration (KMS keys, TLS, at-rest encryption enabled)
- Network configuration (security groups, NACLs, VPC endpoints)
- Application security (input validation, authentication, authorization)
- Logging and monitoring (CloudTrail enabled, log retention, alerting)
- Data handling (PHI classification, de-identification, access controls)

AWS being HIPAA-eligible does not make YOUR application HIPAA-compliant. You must configure services correctly.

## Breach Notification Architecture

HIPAA requires notification within 60 days of discovering a breach of unsecured PHI.

**Detection architecture:**
- GuardDuty for threat detection (anomalous API calls, compromised credentials)
- Macie for S3 PHI exposure (public buckets, unencrypted PHI)
- CloudTrail + CloudWatch Alarms for suspicious access patterns
- Application-level anomaly detection (unusual PHI access volume per tenant)

**Response architecture:**
- SNS for automated alerting to security team
- Step Functions for incident response orchestration
- S3 + Athena for forensic log analysis
- Documented runbook for breach assessment and notification

## HITRUST CSF

### What It Is
The HITRUST Common Security Framework (CSF) is the dominant compliance certification framework for healthcare technology. Most enterprise health system buyers require HITRUST certification from their SaaS vendors. It harmonizes 60+ frameworks (HIPAA, NIST, ISO 27001, PCI-DSS, etc.) into a single assessment.

### AWS HITRUST Shared Responsibility Matrix (SRM)
AWS publishes an SRM for HITRUST CSF v11.2 that maps AWS controls to HITRUST requirements. Customers can inherit AWS's certification for controls relevant to their cloud architecture.

**What this means for you:** Many HITRUST controls are already satisfied by AWS infrastructure. Your assessment scope is reduced to the controls you're responsible for (application security, access management, data handling, operational procedures).

**Reference:** [AWS HITRUST SRM for CSF v11.2](https://aws.amazon.com/blogs/security/aws-hitrust-shared-responsibility-matrix-for-hitrust-csf-v11-2-now-available/)

### Certification Process (Simplified)
1. Scope your assessment (which systems, which data, which controls)
2. Map controls to AWS-inherited vs customer-responsible
3. Implement customer-responsible controls
4. Gather evidence (CloudTrail logs, Config rules, IAM policies, encryption configs)
5. Engage a HITRUST assessor for validated assessment
6. Submit to HITRUST for certification

### SOC 2 Type II
Many healthcare SaaS buyers also require SOC 2 Type II. The Trust Services Criteria (Security, Availability, Processing Integrity, Confidentiality, Privacy) overlap significantly with HIPAA and HITRUST. If you're pursuing HITRUST, SOC 2 is often achievable with incremental effort.

## 42 CFR Part 2 — Substance Use Disorder Records

### What It Is
42 CFR Part 2 protects the confidentiality of substance use disorder (SUD) treatment records. It is STRICTER than HIPAA for this specific data type.

### Feb 2024 Final Rule Changes
The Feb 2024 Final Rule significantly aligned Part 2 with HIPAA:
- Single TPO (Treatment, Payment, Operations) consent now permitted
- Redisclosure allowed consistent with HIPAA
- BUT: SUD records still CANNOT be used in civil, criminal, administrative, or legislative proceedings against the patient without explicit consent or court order
- Compliance deadline: February 16, 2026 (now past — organizations should be compliant)

### Architecture Implications
If your SaaS handles behavioral health or SUD data:
- Implement consent-gated access: SUD records require explicit patient consent before disclosure, even within TPO
- Consent must be captured, stored, and enforced programmatically — not just a checkbox
- Audit trails must track consent status at the time of each access
- Consider data segmentation: SUD records may need to be stored separately or tagged for differential access control
- FHIR Consent resource can model Part 2 consent requirements

## State Privacy Laws

### California — CMIA (Confidentiality of Medical Information Act)
- Broader than HIPAA in some areas: covers more entity types, stricter consent requirements
- Recently expanded to cover reproductive health apps (AB 254, AB 352)
- Requires written authorization for disclosure of medical information (not just "notice")
- Penalties: $1,000 per patient for negligent disclosure, $50,000 for willful

### Texas — HB 300
- Broader definition of "covered entity" than HIPAA — includes ANY entity that handles PHI, not just HIPAA-defined covered entities
- Explicit employee training requirements (within 90 days of hire, every 2 years)
- Requires written authorization for electronic disclosure of PHI
- Penalties: $5,000-$250,000 per violation

### New York — SHIELD Act
- Broader definition of "private information" than HIPAA
- Requires "reasonable safeguards" — a defined security program with administrative, technical, and physical safeguards
- Breach notification within "most expedient time possible" (no specific day count like HIPAA's 60 days)
- Applies to any entity holding private information of NY residents, regardless of where the entity is located

### Architecture Implications
- If your SaaS serves customers in multiple states, you must comply with the strictest applicable law
- Consent mechanisms must be configurable per state/jurisdiction
- Breach notification timelines may be shorter than HIPAA's 60 days
- Data residency may be required (some state laws + GDPR Article 9 for EU patients)
- Multi-region deployment may be necessary — see `resilience-and-deployment.md`

## GxP — When Your Healthcare SaaS Is Also Life Sciences Software

GxP (Good Practice) is a collection of regulations governing computerized systems in life sciences: FDA 21 CFR Part 11, EU Annex 11, IEC 62304, ISO 13485, GAMP 5. If your healthcare SaaS touches any of the following, GxP applies in addition to HIPAA/HITRUST:

- **Software as a Medical Device (SaMD)** — AI-assisted diagnosis, triage, measurement, clinical decision support that meets FDA's SaMD definition. See `clinical-saas-and-imaging.md`.
- **Clinical trial data** — eClinical platforms, EDC systems, eTMF, randomization systems, ePRO/eCOA
- **Pharmacovigilance** — adverse event reporting, safety databases
- **Pharmaceutical manufacturing** — MES, LIMS, batch record systems
- **Regulated laboratory workflows** — pathology, diagnostic labs subject to CLIA
- **Electronic batch records or electronic signatures** — any system where 21 CFR Part 11 applies to predicate rules

### GxP vs HIPAA — Different Questions

| Concern | HIPAA asks | GxP asks |
|---------|-----------|---------|
| Data | Is PHI protected? | Is the data attributable, legible, contemporaneous, original, accurate (ALCOA+)? |
| Access | Who can see ePHI? | Who authored, reviewed, approved the record, and is the signature bound to the record? |
| Change | Is access logged? | Is every change traceable, with reason for change, old value, new value? |
| Deployment | Is it secure? | Is the system validated (IQ/OQ/PQ)? Are deployments controlled per 21 CFR Part 11? |
| Vendor | Is there a BAA? | Is AWS qualified as a GxP supplier? Is each service categorized per GAMP 5? |

Both apply simultaneously for healthcare SaaS that also falls under GxP. The GxP bar is generally higher for data integrity and change control.

### GAMP 5 Categorization of AWS Services

When building GxP-regulated SaaS on AWS, categorize components per GAMP 5 to scope validation effort:

| GAMP Category | Typical AWS Services | Validation Effort |
|--------------|---------------------|-------------------|
| Category 1 (Infrastructure) | EC2, VPC, IAM, KMS, CloudWatch, S3 (as storage) | Minimal — inherit AWS qualification via the GxP on AWS whitepaper; verify configuration |
| Category 3 (Non-configured products) | ALB, API Gateway (no custom authorizer logic), SQS, SNS | IQ/OQ — verify deployed configuration |
| Category 4 (Configured products) | HealthLake, HealthImaging, Comprehend Medical, Transcribe Medical, Cognito with custom attributes, Bedrock with guardrails | IQ/OQ/PQ — validate configuration and operational behavior |
| Category 5 (Custom applications) | Your Lambda functions, ECS tasks, custom business logic, prompt templates, Step Functions workflows | Full lifecycle validation — URS through PQ |

**Architecture implication:** Isolate Category 5 (custom) components from Category 1/3 (infrastructure/products) so validation scope is bounded. Rebuilding a Lambda shouldn't require re-qualifying DynamoDB.

### AWS as a GxP Supplier

AWS publishes a [GxP on AWS whitepaper](https://aws.amazon.com/compliance/gxp-part-11-annex-11/) mapping AWS controls to 21 CFR Part 11 and EU Annex 11. For GxP workloads:

- AWS qualifies the physical and hypervisor layers — you inherit that via shared responsibility
- You qualify: your configuration (IaC), your application code, your data handling, your change control process
- Execute the BAA AND retain the GxP whitepaper as supplier qualification evidence
- Pin to specific AWS service versions/features where the service allows it; document version in your supplier register

### 21 CFR Part 11 — Electronic Records & Signatures

Applies when a predicate rule (FDA regulation) requires records to be kept and those records are in electronic form. Architecture implications covered in `identity-and-onboarding.md` (e-signatures) and `audit-logging-and-access.md` (audit trails, immutability).

**Core requirements:**
- Closed systems: validation, audit trails, operational checks, authority checks, device checks
- Open systems: additional encryption and digital signatures
- Electronic signatures: unique to one individual, linked to record, non-repudiable, include signer identity + date/time + meaning

### EU Annex 11 — Computerized Systems

Parallel to 21 CFR Part 11 for EU. Similar requirements with some differences:
- Explicit risk management requirement (ICH Q9 alignment)
- Stronger supplier assessment expectations
- Data migration validation explicitly called out
- Business continuity must be tested

### When to Load `gxp-compliance-generic.md`

Load that steering file (manual inclusion) when:
- The conversation involves FDA-regulated software (SaMD, eClinical, pharmacovigilance)
- The customer is a pharmaceutical, biotech, or medical device company
- Requirements mention 21 CFR Part 11, Annex 11, IEC 62304, GAMP 5, or "validation"
- The customer is pursuing a 510(k), De Novo, or CE mark
- Clinical trial systems or regulated laboratory workflows are in scope

The generic GxP file covers ALCOA+, audit trail specifics, validation (IQ/OQ/PQ), change management, supplier qualification, and the full compliance checklist. This file (`healthcare-compliance-foundations.md`) only covers the healthcare SaaS intersection.

## GDPR Article 9 — Cross-Border PHI

If your SaaS serves EU patients or EU-based healthcare organizations:
- Health data is "special category" data under GDPR Article 9
- Requires explicit consent or another Article 9 basis for processing
- Data residency: EU patient data should stay in EU regions unless adequate safeguards exist
- Right to erasure (stronger than HIPAA's right to amendment)
- Data Protection Impact Assessment (DPIA) required for large-scale health data processing

**AWS mapping:** Deploy in eu-west-1 or eu-central-1 for EU data residency. Use S3 bucket policies and DynamoDB global tables with region restrictions.

## Content Freshness

| Item | Current Status | Verify At |
|---|---|---|
| HIPAA Eligible Services | ~150+ services | https://aws.amazon.com/compliance/hipaa-eligible-services-reference/ |
| 42 CFR Part 2 | Compliance deadline Feb 16, 2026 (past) | HHS/SAMHSA |
| CMS-0057-F deadlines | Jan 1, 2027 for payers | https://www.cms.gov/ |
| HITRUST CSF version | v11.2 | https://hitrustalliance.net/ |
| CA CMIA | Expanded 2024 (reproductive health) | CA legislature |
| GxP on AWS whitepaper | Published by AWS | https://aws.amazon.com/compliance/gxp-part-11-annex-11/ |
| GAMP 5 | Second edition (2022) | ISPE |
| IEC 62304 | 2015 (Amd 1) | ISO |

**Agent rule:** When citing compliance deadlines or service eligibility, add: "Verify current status at [link] as this may have changed since this power was last updated."

## Common Mistakes

1. **Assuming AWS HIPAA eligibility = your app is compliant.** AWS secures infrastructure. You secure configuration, application logic, and data handling. The shared responsibility model is non-negotiable.

2. **Not executing the BAA before processing PHI.** Having an AWS account is not enough. The BAA must be explicitly executed.

3. **Using non-HIPAA-eligible services for PHI.** If a service isn't on the list, don't use it for PHI workloads. Period.

4. **Treating HITRUST as optional.** For enterprise health system sales, HITRUST is effectively required. Budget for it early — the certification process takes 6-12 months.

5. **Ignoring state laws.** HIPAA is the federal floor. CA, TX, NY, and other states have stricter requirements. If you serve customers nationally, you must comply with the strictest applicable law.

6. **Not tracking the BAA chain.** You have a BAA with AWS, but do you have BAAs with every third-party service that touches PHI? (Analytics, email, SMS, error tracking, logging SaaS?)

7. **Treating 42 CFR Part 2 like HIPAA.** Part 2 is stricter for SUD data. If your SaaS handles behavioral health, you need consent-gated access, not just HIPAA-level controls.

## Discovery Questions for This Domain

**Regulatory landscape:**
- What regulations apply to your product? (HIPAA is assumed — ask about HITRUST, SOC 2, state-specific laws, FDA)
- Do you handle substance use disorder data? (Triggers 42 CFR Part 2 — stricter than HIPAA)
- Do you serve EU patients or organizations? (Triggers GDPR Article 9)
- Are any of your tenants government entities? (May trigger FedRAMP)

**BAA status:**
- Have you executed a BAA with AWS?
- Do you have BAAs with all third-party services that handle PHI?
- Do your customers (tenants) have BAAs with you?

**Certification goals:**
- Are you pursuing HITRUST certification? (If yes, what timeline?)
- Do your customers require SOC 2 Type II reports?
- Have you completed a HIPAA Security Risk Assessment?

**GxP applicability:**
- Is your software a Software as a Medical Device (SaMD), or does it support clinical trials, pharmacovigilance, or pharmaceutical manufacturing? (If yes, GxP applies — load `gxp-compliance-generic.md`)
- Do you need 21 CFR Part 11 compliant electronic signatures? (Batch release, clinical report approval, eSignature on records)
- Is the system subject to FDA, EMA, or other health authority inspection?
- Do you have a Quality Management System (ISO 13485) or plan to implement one?
- Have you categorized your AWS services per GAMP 5? (Category 1/3/4/5 determines validation effort)

**Current compliance posture:**
- Do you have a documented security program? (Policies, procedures, risk assessment)
- How do you handle breach detection and notification today?
- Do you have a compliance officer or team?

## References

- [HIPAA on AWS](https://aws.amazon.com/compliance/hipaa-compliance/)
- [HIPAA Eligible Services Reference](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/)
- [AWS Config Operational Best Practices for HIPAA Security](https://docs.aws.amazon.com/config/latest/developerguide/operational-best-practices-for-hipaa_security.html)
- [AWS HITRUST SRM for CSF v11.2](https://aws.amazon.com/blogs/security/aws-hitrust-shared-responsibility-matrix-for-hitrust-csf-v11-2-now-available/)
- [Healthcare Industry Lens](https://docs.aws.amazon.com/wellarchitected/latest/healthcare-industry-lens/healthcare-industry-lens.html)
- [Common Techniques to Detect PHI and PII Using AWS Services](https://aws.amazon.com/blogs/industries/common-techniques-to-detect-phi-and-pii-data-using-aws-services/)
- [HIPAA Security Rule — 45 CFR Part 164](https://www.hhs.gov/hipaa/for-professionals/security/index.html)
- [42 CFR Part 2 Final Rule (Feb 2024)](https://www.federalregister.gov/documents/2024/02/16/2024-02544/confidentiality-of-substance-use-disorder-sud-patient-records)
- [GxP on AWS — 21 CFR Part 11 and EU Annex 11](https://aws.amazon.com/compliance/gxp-part-11-annex-11/)
- [FDA 21 CFR Part 11](https://www.ecfr.gov/current/title-21/chapter-I/subchapter-A/part-11)
- [EU Annex 11 — Computerised Systems](https://health.ec.europa.eu/document/download/6de3c89d-0d79-4ff2-832b-a9bf40eac09b_en)
- [GAMP 5 (ISPE)](https://ispe.org/publications/guidance-documents/gamp-5-guide-2nd-edition)
- [IEC 62304 — Medical Device Software Lifecycle](https://www.iso.org/standard/71604.html)
