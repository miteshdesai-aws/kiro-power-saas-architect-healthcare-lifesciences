# SaaS Lens & Healthcare Industry Lens Architecture Review

## How to Use This File

This steering file provides a structured assessment checklist combining the AWS Well-Architected SaaS Lens with the AWS Healthcare Industry Lens. Use it to conduct formal architecture reviews for healthcare SaaS applications on AWS.

When conducting a review:
1. Walk through each pillar's questions with the customer
2. For each question, assess the current state (Not Implemented, Partially, Fully)
3. Include the Healthcare Lens questions (marked with HCL-) for healthcare SaaS
4. For GxP-regulated systems (SaMD, eClinical, pharmacovigilance, regulated labs), also include the GxP questions (marked with HCL-OPS-3 and HCL-SEC-4). For deeper GxP coverage, load `gxp-compliance-generic.md`.
5. Record findings and recommendations
6. Prioritize findings by risk and effort — use the healthcare severity calibration
7. Produce a summary report

Each pillar contains standard SaaS Lens questions followed by Healthcare Industry Lens questions. For healthcare SaaS, BOTH sets of questions apply. For GxP-regulated healthcare SaaS, GxP questions add a third layer.

## Pillar 1: Operational Excellence

### OPS-1: How are you managing tenant-aware operations?

Assessment questions:
- Is there a unified control plane for managing all tenants?
- Can you onboard, configure, and offboard tenants without manual intervention?
- Do you have a single operational dashboard that shows tenant health across the system?
- Are operational runbooks tenant-aware (can you troubleshoot a specific tenant's issue)?

Findings to look for:
- Manual tenant management processes (spreadsheets, tickets, scripts)
- No centralized view of tenant health
- Operational procedures that require knowing the tenant's infrastructure details
- Different operational processes for different tenants

### OPS-2: How are you deploying updates to your SaaS environment?

Assessment questions:
- Do all tenants run the same version of the application?
- Is deployment automated end-to-end (no manual steps)?
- Do you have canary or staged deployment strategies?
- Can you roll back a deployment without affecting tenant data?
- How long does a deployment take? Does it scale with tenant count?

Findings to look for:
- Tenants running different versions (this is not SaaS)
- Manual deployment steps
- No rollback capability
- Deployment time that grows linearly with tenant count (silo without automation)

### OPS-3: How are new tenants onboarded to your system?

Assessment questions:
- Is onboarding fully automated (single API call or self-service)?
- What is the time from signup to first use? (minutes, hours, days?)
- Does onboarding handle all aspects: identity, resources, billing, routing?
- What happens if onboarding partially fails? Is there rollback/retry?
- Can you onboard 100 tenants in a day without manual intervention?

Findings to look for:
- Manual onboarding steps (creating databases, configuring users by hand)
- Onboarding that takes hours or days
- No error handling for partial onboarding failures
- Onboarding process that differs by tenant tier without automation

### HCL-OPS-1: How are you managing PHI access and audit trails operationally?

Assessment questions:
- Do you have operational visibility into PHI access patterns per tenant?
- Can you produce an audit trail for a specific patient's data access over the past year?
- Are break-the-glass events reviewed within 24-48 hours?
- Is there an operational runbook for HIPAA breach detection and notification?
- How do you handle tenant offboarding with PHI data deletion and right-to-erasure?

Findings to look for:
- No application-level PHI access logging (only infrastructure logs)
- No break-the-glass procedure or review process
- No documented breach notification procedure
- Tenant offboarding doesn't include PHI deletion verification
- Audit logs not retained for 6+ years

### HCL-OPS-2: Is onboarding HIPAA-compliant end-to-end?

Assessment questions:
- Does onboarding provision per-tenant KMS keys for PHI encryption?
- Does onboarding configure audit logging for the new tenant's PHI stores?
- Is BAA status verified before a tenant can process PHI?
- Does onboarding set up tenant-specific FHIR data stores (if applicable)?
- Is the onboarding process itself audit-logged?

Findings to look for:
- Tenants can process PHI before BAA is confirmed
- No per-tenant encryption keys provisioned during onboarding
- Audit logging not configured until after tenant is active
- FHIR data stores shared without explicit tenant isolation setup

### HCL-OPS-3: If GxP applies, is the system validated and change-controlled?

Assessment questions only relevant if the system is SaMD, eClinical, pharmacovigilance, regulated lab, or otherwise GxP-regulated:
- Have components been categorized per GAMP 5 (Category 1/3/4/5)?
- Is there a current Validation Plan with IQ/OQ/PQ protocols executed?
- Is there a current Traceability Matrix linking requirements → design → tests → results?
- Is the CI/CD pipeline itself qualified as a Category 4 system?
- Are changes controlled via formal Change Control Records with approval signatures?
- Is there a Supplier Qualification Register covering AWS and third-party components?
- For SaMD AI: is there a Predetermined Change Control Plan (PCCP) for retraining?

Findings to look for:
- No GAMP 5 categorization (validation scope is undefined)
- Validation done once at initial release, not maintained per release
- Traceability gaps (requirements without tests, tests without requirements)
- Unqualified CI/CD pipeline used for regulated releases
- Ad-hoc changes without formal change control
- Third-party components in regulated paths without supplier qualification
- AI retraining without PCCP (triggers new FDA submission per retrain)

## Pillar 2: Security

### SEC-1: How are you associating tenant context with users?

Assessment questions:
- Is every authenticated user bound to a tenant ID?
- Is tenant context included in the JWT/token as a claim?
- Does tenant context flow through all layers of the architecture?
- Is the tenant context trustworthy (signed JWT, validated by infrastructure)?
- Can a user modify their tenant context (e.g., change tenant ID in the token)?

Findings to look for:
- Tenant context stored only in application state (not in the token)
- Tenant ID passed as a query parameter or header that can be spoofed
- Services that don't validate tenant context
- Async operations (queues, events) that lose tenant context

### SEC-2: How are you isolating tenant resources?

Assessment questions:
- What isolation mechanism is used? (IAM policies, network isolation, resource separation)
- Is isolation enforced at the infrastructure level (not just application code)?
- For pool model: are dynamic IAM policies or ABAC used to scope access?
- For silo model: are resources in separate accounts/VPCs?
- Have you tested that cross-tenant access is actually blocked?

Findings to look for:
- Isolation relies solely on application-level filtering (WHERE tenant_id = X)
- No IAM-based enforcement for shared resources
- No automated testing of isolation boundaries
- Inconsistent isolation across services (some isolated, some not)

### SEC-3: How are you managing tenant data encryption?

Assessment questions:
- Is data encrypted at rest and in transit?
- For silo model: do tenants have their own encryption keys (CMKs)?
- For pool model: is there a strategy for per-tenant key management?
- Can a tenant's data be decrypted independently of other tenants?

Findings to look for:
- Shared encryption keys across all tenants in pool model
- No encryption at rest
- No key rotation strategy

### HCL-SEC-1: How are you enforcing PHI isolation specifically?

Assessment questions:
- Is PHI isolation enforced at the infrastructure level (IAM, KMS, resource boundaries) — not just application code?
- Are per-tenant KMS keys used for PHI data stores?
- Can you demonstrate to a HITRUST assessor that Tenant A cannot access Tenant B's PHI even if application code has a bug?
- For HealthLake: are you using data-store-per-tenant or shared with SMART on FHIR scoping?
- For HealthImaging: are you using data-store-per-tenant?
- Is PHI isolation tested automatically in CI/CD?

Findings to look for:
- PHI isolation relies solely on application-level WHERE clauses
- Shared KMS keys across tenants for PHI data stores
- No automated PHI isolation testing
- HealthLake shared data store without SMART on FHIR scoping
- No per-tenant KMS key policy scoping

### HCL-SEC-2: Do you have a BAA with AWS and all subprocessors?

Assessment questions:
- Is the AWS BAA executed for all accounts processing PHI?
- Are all AWS services used for PHI on the HIPAA Eligible Services list?
- Do you have BAAs with all third-party services that handle PHI? (Analytics, email, SMS, error tracking)
- Do your tenants (customers) have BAAs with you?
- Is there a BAA inventory that's reviewed periodically?

Findings to look for:
- AWS BAA not executed
- Non-HIPAA-eligible services used for PHI workloads
- Third-party services handling PHI without BAAs
- No BAA inventory or tracking
- No periodic BAA review process

### HCL-SEC-3: How are you handling PHI in logs and non-production environments?

Assessment questions:
- Do application logs contain PHI in plaintext? (Patient names, MRNs, diagnoses)
- Are log groups containing PHI encrypted with KMS?
- Is real PHI used in development, staging, or QA environments?
- Do you scan for PHI in non-PHI data stores? (Macie, Comprehend Medical)

Findings to look for:
- PHI in plaintext application logs (most common HIPAA violation in SaaS)
- Log groups not encrypted
- Real PHI in non-production environments
- No PHI scanning or detection in place

### HCL-SEC-4: If GxP applies, are audit trails Part 11 / Annex 11 compliant?

Assessment questions only relevant for GxP-regulated systems:
- Do modifications to regulated records capture old value, new value, and reason for change?
- Are audit logs tamper-evident via cryptographic hash chains or CloudTrail log file validation?
- Are compute nodes NTP-synchronized with documented time sources?
- If electronic signatures apply: are signers re-authenticated at signing time?
- Are signatures cryptographically bound to record content (modifying the record invalidates the signature)?
- Do signatures capture signer identity + timestamp + meaning?

Findings to look for:
- Audit logs missing old/new value or reason for change on modifications (21 CFR Part 11 violation)
- No tamper evidence on audit logs (logs could be modified without detection)
- Clock skew not monitored
- Session validity used instead of fresh re-authentication for signatures
- "Signed: true" flag used instead of cryptographic signature binding

## Pillar 3: Reliability

### REL-1: How do you limit a tenant's ability to impact other tenants?

Assessment questions:
- Are there per-tenant or per-tier throttling limits?
- How are throttling limits enforced? (API Gateway usage plans, application-level)
- What happens when a tenant exceeds their limits? (429 response, queuing, degraded service)
- Have you tested noisy neighbor scenarios?
- Is there monitoring that detects when one tenant is consuming disproportionate resources?

Findings to look for:
- No throttling at all (any tenant can consume unlimited resources)
- Throttling only at the system level (not per-tenant)
- No noisy neighbor detection or alerting
- No testing of throttling enforcement

### REL-2: How do you handle tenant-specific failures?

Assessment questions:
- Can a failure in one tenant's resources affect other tenants?
- For silo model: is there blast radius isolation between tenant environments?
- For pool model: can a bad data record from one tenant crash the service for all?
- Do you use cell-based architecture or similar fault isolation patterns?

Findings to look for:
- Single points of failure that affect all tenants
- No fault isolation between tenants in pool model
- Error handling that doesn't account for tenant-specific failures

### REL-3: How are you testing multi-tenant capabilities?

Assessment questions:
- Do you have automated tests that validate tenant isolation?
- Do you test with multiple concurrent tenants (not just single-tenant testing)?
- Do you simulate noisy neighbor scenarios in testing?
- Do you test tenant onboarding and offboarding end-to-end?
- Do you test tier changes (pool to silo migration)?

Findings to look for:
- All testing is single-tenant (no multi-tenant test scenarios)
- No isolation validation tests
- No load testing with tenant-aware traffic patterns
- Onboarding/offboarding not tested in CI/CD

### HCL-REL-1: Can a tenant-specific failure expose PHI?

Assessment questions:
- If a service crashes, could error messages or stack traces contain PHI?
- If a tenant's data store fails, does the fallback path maintain PHI isolation?
- Are HIPAA contingency plan requirements met? (Data backup, disaster recovery, emergency mode operation)
- What are the RPO/RTO for PHI data stores specifically?

Findings to look for:
- Error responses that include PHI (patient data in 500 error bodies)
- Fallback paths that bypass tenant isolation
- No documented HIPAA contingency plan
- RPO/RTO not defined for PHI systems
- PHI backups not encrypted

### HCL-REL-2: Do you test cross-tenant PHI access specifically?

Assessment questions:
- Do automated tests attempt to read Tenant B's PHI with Tenant A's credentials?
- Do tests verify cross-tenant KMS key usage fails?
- Do tests verify cross-tenant HealthLake/HealthImaging data store access fails?
- For 42 CFR Part 2 data: do tests verify consent-gated access works correctly?
- Are isolation test results stored as HITRUST evidence?

Findings to look for:
- No PHI-specific isolation tests (only generic cross-tenant tests)
- No KMS cross-tenant key usage tests
- No consent-gated access tests for SUD data
- Isolation test results not retained for compliance evidence

## Pillar 4: Performance Efficiency

### PERF-1: How do you prevent one tenant from adversely impacting another?

Assessment questions:
- Are compute resources shared or dedicated per tenant?
- For shared compute: how do you prevent one tenant from consuming all capacity?
- For shared databases: how do you prevent one tenant from saturating throughput?
- Do you use reserved concurrency (Lambda) or resource quotas (EKS) per tenant?
- Is there auto-scaling that responds to per-tenant load patterns?

Findings to look for:
- Shared resources with no per-tenant limits
- Auto-scaling based only on aggregate metrics (not tenant-aware)
- Database hot partitions caused by a single tenant
- No reserved capacity for critical tenants

### PERF-2: How does your architecture scale with tenant growth?

Assessment questions:
- What happens when you go from 10 to 100 to 1000 tenants?
- Does onboarding time increase with tenant count?
- Does deployment time increase with tenant count?
- Are there hard limits (IAM policies, database connections, account limits) that will block growth?
- Have you identified the first scaling bottleneck?

Findings to look for:
- Architecture that works for 10 tenants but breaks at 100
- Linear scaling of operational overhead with tenant count
- Approaching AWS service limits without a mitigation plan
- No capacity planning for tenant growth

### HCL-PERF-1: Does FHIR/DICOM performance meet clinical workflow requirements?

Assessment questions:
- What is the FHIR API latency per tenant? Does it degrade with tenant count?
- For imaging: what is the DICOM study retrieval latency? Is it sub-second for radiology workflow?
- Are HealthLake query patterns optimized per tenant? (Index usage, search parameter tuning)
- For shared HealthLake data stores: does one tenant's query volume impact another's latency?
- Are DICOM pre-fetch strategies in place for viewer performance?

Findings to look for:
- FHIR API latency > 500ms for clinical workflows (too slow for point-of-care)
- DICOM retrieval latency > 2 seconds (unacceptable for radiology reading)
- No per-tenant HealthLake performance monitoring
- No pre-fetch strategy for imaging viewers
- Shared HealthLake data store with noisy neighbor latency issues

## Pillar 5: Cost Optimization

### COST-1: How are you measuring tenant consumption?

Assessment questions:
- Can you measure the cost of serving each tenant?
- For pool model: do you have a metering pipeline that captures per-tenant usage?
- For silo model: are resources tagged for cost allocation?
- Do you know which tenants are profitable and which are not?
- Is metering data used to inform pricing decisions?

Findings to look for:
- No per-tenant cost visibility
- Pricing based on guesswork rather than actual cost data
- No metering pipeline for pool model
- Resources not tagged for cost allocation in silo model

### COST-2: How are you optimizing costs across tenancy models?

Assessment questions:
- Are you using the right tenancy model for each tier? (pool for basic, silo for enterprise)
- Are idle silo resources scaled down or stopped?
- For pool model: are you right-sizing shared resources based on actual usage?
- Do you have a strategy for moving tenants between tiers based on usage?

Findings to look for:
- All tenants in silo model regardless of revenue (over-spending)
- All tenants in pool model regardless of requirements (under-isolating)
- Idle resources in silo tenant environments
- No right-sizing based on actual tenant usage patterns

### HCL-COST-1: Can you attribute PHI storage and processing costs per tenant?

Assessment questions:
- For HealthLake: can you measure per-tenant FHIR API costs and storage costs?
- For HealthImaging: can you measure per-tenant imaging storage and retrieval costs?
- For Bedrock (clinical AI): can you attribute AI inference costs per tenant? (Application inference profiles)
- Are per-tenant KMS key costs tracked? (Each CMK has a monthly cost)
- Are audit log storage costs attributed per tenant or treated as shared overhead?

Findings to look for:
- No visibility into HealthLake/HealthImaging costs per tenant
- AI inference costs not attributed per tenant (subsidizing heavy AI users)
- KMS key costs not factored into per-tenant cost model
- Audit log storage costs growing unchecked (CloudTrail data events can be expensive)

## Default Severity Calibration

Not all findings are equal. Use this default priority weighting when the conversation doesn't provide enough context to override. Healthcare findings (HCL-) are integrated into the standard calibration.

**Critical (address immediately — security, data integrity, or PHI exposure risk):**
- SEC-2: No infrastructure-level tenant isolation (relying only on application-level filtering)
- SEC-1: Tenant context can be spoofed or is not validated
- HCL-SEC-1: PHI isolation relies solely on application-level filtering (HIPAA violation risk)
- HCL-SEC-2: No BAA with AWS or using non-HIPAA-eligible services for PHI
- HCL-SEC-3: PHI in plaintext application logs
- REL-1: No per-tenant throttling (any tenant can consume unlimited shared resources)

**High (address within weeks — significant operational, business, or compliance risk):**
- OPS-3: Manual tenant onboarding (scaling bottleneck)
- HCL-OPS-1: No PHI access audit trail (cannot demonstrate compliance to auditors)
- HCL-OPS-2: Onboarding doesn't provision per-tenant encryption or audit logging
- SEC-3: No encryption at rest, or shared encryption keys with no key policy scoping
- HCL-SEC-2: Third-party services handling PHI without BAAs
- REL-3: No automated multi-tenant testing (isolation not validated)
- HCL-REL-2: No PHI-specific isolation testing
- PERF-1: Shared resources with no per-tenant limits (noisy neighbor risk)
- COST-1: No per-tenant cost visibility (pricing decisions are guesswork)

**Medium (address within a quarter — maturity gaps):**
- OPS-1: No unified control plane (tenant management is ad-hoc)
- OPS-2: Manual deployments or tenants running different versions
- HCL-REL-1: No documented HIPAA contingency plan (backup, DR, emergency mode)
- REL-2: No fault isolation between tenants (single blast radius)
- PERF-2: No capacity planning for tenant growth
- HCL-PERF-1: FHIR/DICOM latency not meeting clinical workflow requirements
- HCL-COST-1: No per-tenant visibility into HealthLake/HealthImaging/Bedrock costs

**Low (address when capacity allows — optimization opportunities):**
- COST-2: Not optimizing costs across tenancy models
- Multi-region not implemented (unless data residency is a requirement, in which case it's High)
- No Marketplace integration (unless it's a business priority)
- Missing operational dashboards (useful but not urgent)

**Healthcare override rules:**
- Any finding involving PHI exposure is automatically Critical, regardless of the standard calibration
- Missing BAA (AWS or subprocessor) is always Critical — it's a HIPAA violation
- PHI in plaintext logs is always Critical — it's the most common OCR finding
- No audit trail for PHI access is always High — it's a HITRUST blocker

**GxP override rules (only if GxP applies):**
- Missing old-value/new-value/reason-for-change on regulated record modifications is Critical — 21 CFR Part 11 violation
- Unqualified CI/CD pipeline used for regulated releases is High — compromises every release's validation evidence
- No validation coverage for a Class B or C SaMD component is Critical — patient safety + FDA finding risk
- AI retraining without PCCP for cleared SaMD is Critical — triggers need for new FDA submission
- No electronic signature binding (modifying record doesn't invalidate signature) is Critical — Part 11 §11.50/§11.70 violation
- Broken traceability (untraced requirements, orphaned tests) is High — validation evidence is incomplete

This calibration assumes a production healthcare SaaS with real tenants and real PHI. For pre-launch systems, shift standard SaaS findings down one level (Critical → High, etc.) but keep all HCL-SEC findings at their stated level — PHI security doesn't get a grace period.

## Producing the Review Report

After walking through all pillars, produce a findings report with this structure:

### Report Template

**Executive Summary:** 2-3 sentences on overall SaaS maturity, healthcare compliance posture, and top risks.

**Findings by Priority:**

For each finding:
- Pillar: (which WA pillar)
- Question: (which assessment question — include HCL- prefix for healthcare findings)
- Current State: (what you observed)
- Risk: (what could go wrong — for healthcare, specify if PHI exposure is possible)
- Recommendation: (what to do)
- Effort: (Low / Medium / High)
- Priority: (Critical / High / Medium / Low)

**Prioritization guidance:**
- Critical: PHI exposure risk, missing BAA, PHI in logs, no infrastructure-level isolation, spoofable tenant context; for GxP: missing old/new/reason on modifications, no e-signature binding, no validation for Class B/C components, AI retrain without PCCP
- High: No PHI audit trail, no throttling, no onboarding automation, no encryption, no isolation testing; for GxP: unqualified CI/CD for regulated releases, broken traceability
- Medium: No HIPAA contingency plan, no multi-tenant testing, manual deployments, no capacity planning; for GxP: supplier qualification gaps for Category 4/5 components
- Low: Missing operational dashboards, no Marketplace integration, cost optimization gaps

**Quick Wins:** List 3-5 findings that are low effort but high impact.

**Roadmap:** Suggest a phased approach to address findings (Phase 1: Critical, Phase 2: High, etc.)

## References

- SaaS Lens Full Document: https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/saas-lens.html
- Healthcare Industry Lens: https://docs.aws.amazon.com/wellarchitected/latest/healthcare-industry-lens/healthcare-industry-lens.html
- Healthcare Industry Lens — Security Pillar: https://docs.aws.amazon.com/wellarchitected/latest/healthcare-industry-lens/detective-controls.html
- SaaS Lens General Design Principles: https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/general-design-principles.html
- SaaS Lens Identity and Access Management: https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/identity-and-access-management.html
- SaaS Lens Foundations: https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/foundations.html
- SaaS Lens Expenditure Awareness: https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/expenditure-awareness.html
- SaaS Lens Multi-Tenant Microservices: https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/multi-tenant-microservices.html
- Testing SaaS Solutions on AWS: https://aws.amazon.com/blogs/apn/testing-saas-solutions-on-aws
- How Are You Testing Multi-Tenant Capabilities: https://wa.aws.amazon.com/saas.question.REL_3.en.html
- HIPAA on AWS: https://aws.amazon.com/compliance/hipaa-compliance/
- HIPAA Eligible Services: https://aws.amazon.com/compliance/hipaa-eligible-services-reference/
- AWS HITRUST SRM: https://aws.amazon.com/blogs/security/aws-hitrust-shared-responsibility-matrix-for-hitrust-csf-v11-2-now-available/
- GxP on AWS: https://aws.amazon.com/compliance/gxp-part-11-annex-11/
- 21 CFR Part 11: https://www.ecfr.gov/current/title-21/chapter-I/subchapter-A/part-11
- EU Annex 11: https://health.ec.europa.eu/document/download/6de3c89d-0d79-4ff2-832b-a9bf40eac09b_en