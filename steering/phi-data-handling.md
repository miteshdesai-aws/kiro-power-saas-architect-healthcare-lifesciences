# PHI Data Handling

## What Counts as PHI

Protected Health Information (PHI) is any individually identifiable health information created, received, maintained, or transmitted by a covered entity or business associate. ePHI is PHI in electronic form.

### The 18 HIPAA Identifiers
If health data includes any of these, it's PHI:
1. Names
2. Geographic data smaller than state
3. Dates (except year) related to an individual
4. Phone numbers
5. Fax numbers
6. Email addresses
7. Social Security numbers
8. Medical record numbers
9. Health plan beneficiary numbers
10. Account numbers
11. Certificate/license numbers
12. Vehicle identifiers and serial numbers
13. Device identifiers and serial numbers
14. Web URLs
15. IP addresses
16. Biometric identifiers
17. Full-face photographs
18. Any other unique identifying number or code

### What People Forget Is PHI
- **Audio recordings of clinical encounters** — a therapist-patient video session recording is PHI
- **DICOM images** — medical images contain patient identifiers in metadata headers
- **AI-generated clinical notes** — if generated from PHI input, the output is PHI
- **Transcription output** — speech-to-text of a clinical encounter is PHI
- **Device telemetry with patient context** — RPM data linked to a patient is PHI
- **FHIR resources** — Patient, Observation, Condition, MedicationRequest all contain PHI
- **Log entries containing patient data** — if your application logs include patient names, MRNs, or other identifiers, those logs are PHI

## Encryption Key Strategy

### Per-Tenant KMS Keys vs Shared Keys

**Recommendation for healthcare SaaS:** Use per-tenant KMS Customer Managed Keys (CMKs) for data stores containing PHI. This is the healthcare default, not the generic SaaS default.

**Why per-tenant keys:**
- Strongest cryptographic isolation — even if application logic has a bug, Tenant A's key cannot decrypt Tenant B's data
- Simplifies tenant offboarding — disable/schedule deletion of the tenant's key and all their data becomes inaccessible
- Audit trail per key — CloudTrail logs show exactly which tenant's data was accessed
- Compliance story — "each tenant's data is encrypted with their own key" satisfies auditors

**When shared keys are acceptable:**
- Non-PHI data stores (configuration, feature flags, aggregated analytics)
- Pool-model services where per-tenant keys aren't feasible (shared DynamoDB table — use IAM isolation instead)
- Cost-sensitive early-stage SaaS with < 10 tenants (KMS key costs are per-key per-month)

**KMS key policy pattern for tenant isolation:**
- Each tenant's CMK has a key policy that restricts usage to IAM roles associated with that tenant
- Application assumes a tenant-scoped role before accessing encrypted data
- Even with the correct IAM permissions, the wrong tenant's key won't decrypt the data

**CDK example — per-tenant KMS key with scoped key policy:**
```typescript
const tenantKey = new kms.Key(this, `TenantKey-${tenantId}`, {
  alias: `alias/tenant-${tenantId}-phi`,
  description: `PHI encryption key for tenant ${tenantId}`,
  enableKeyRotation: true,
  removalPolicy: RemovalPolicy.RETAIN, // never auto-delete PHI keys
});

tenantKey.addToResourcePolicy(new iam.PolicyStatement({
  sid: 'AllowTenantRoleOnly',
  effect: iam.Effect.ALLOW,
  principals: [new iam.ArnPrincipal(tenantRoleArn)],
  actions: ['kms:Decrypt', 'kms:Encrypt', 'kms:GenerateDataKey'],
  resources: ['*'],
}));
```

**Terraform example — per-tenant KMS key:**
```hcl
resource "aws_kms_key" "tenant_phi" {
  description         = "PHI encryption key for tenant ${var.tenant_id}"
  enable_key_rotation = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowTenantRoleOnly"
      Effect    = "Allow"
      Principal = { AWS = var.tenant_role_arn }
      Action    = ["kms:Decrypt", "kms:Encrypt", "kms:GenerateDataKey"]
      Resource  = "*"
    }]
  })
}

resource "aws_kms_alias" "tenant_phi" {
  name          = "alias/tenant-${var.tenant_id}-phi"
  target_key_id = aws_kms_key.tenant_phi.key_id
}
```

### Envelope Encryption
For large data objects (DICOM images, audio recordings, documents):
- Generate a data encryption key (DEK) using the tenant's CMK
- Encrypt the data with the DEK
- Store the encrypted DEK alongside the encrypted data
- To decrypt: use the tenant's CMK to decrypt the DEK, then use the DEK to decrypt the data

This avoids sending large payloads to KMS and keeps per-request KMS costs low.

## Encryption at Rest — Per Service

| Service | How to Enable | Key Type | Notes |
|---------|--------------|----------|-------|
| S3 | Default bucket encryption (SSE-KMS) | Per-tenant CMK | Block all public access. Enable versioning. |
| DynamoDB | Table-level encryption | Per-tenant CMK (silo) or AWS-managed (pool with IAM isolation) | Enable point-in-time recovery |
| RDS/Aurora | Instance-level encryption at creation | Per-tenant CMK (silo/bridge) | Cannot add encryption after creation — must snapshot/restore |
| HealthLake | Data store encryption | KMS CMK | Per data store — aligns with per-tenant data stores |
| HealthImaging | Data store encryption | KMS CMK | Per data store |
| OpenSearch | Domain-level encryption | KMS CMK | Enable node-to-node encryption too |
| ElastiCache (Redis) | Cluster-level encryption | KMS CMK | Enable in-transit encryption too. Do NOT cache PHI in Memcached (no encryption support) |
| EBS | Volume-level encryption | KMS CMK | Set account default to encrypt all new volumes |
| Kinesis | Stream-level encryption | KMS CMK | For metering/event streams carrying PHI |
| SQS | Queue-level encryption | KMS CMK | For async workflows carrying PHI |

## Encryption in Transit

- TLS 1.2+ everywhere. No exceptions for PHI workloads.
- ALB listeners: HTTPS only, redirect HTTP → HTTPS
- API Gateway: HTTPS by default (enforce minimum TLS version)
- VPC endpoints: keep AWS API traffic off the public internet
- PrivateLink: for health system connectivity (no public internet traversal)
- Database connections: enforce SSL (RDS `rds.force_ssl`, Aurora `require_secure_transport`)
- Inter-service communication: TLS between microservices, even within VPC

## Audio and Video as PHI

This is critical for telehealth and ambient documentation SaaS.

### Telehealth Session Recordings
- Amazon Chime SDK is HIPAA-eligible for real-time video
- Session recordings stored in S3 are PHI — encrypt with per-tenant CMK
- Retention policies: configure S3 lifecycle rules per tenant/tier (some states require specific retention periods for medical records)
- Access control: only authorized clinicians and the patient should access recordings
- Audit: log every access to recording objects via S3 data events in CloudTrail

### Ambient Documentation Audio
- Audio captured during clinical encounters is PHI from the moment it's recorded
- Transcribe Medical is HIPAA-eligible for speech-to-text
- The transcription output is PHI
- AI-generated notes (from Bedrock) derived from the transcription are PHI
- The entire pipeline (audio → transcription → LLM → clinical note) must be encrypted, access-controlled, and audit-logged at every step
- See `genai-and-phi.md` for the dual-zone architecture pattern

### RPM Device Data
- Telemetry from remote patient monitoring devices linked to a patient is PHI
- IoT Core is HIPAA-eligible — use it for device ingestion
- Encrypt device-to-cloud communication (TLS, device certificates)
- Tenant isolation at the IoT layer: per-tenant IoT policies or topic namespacing

## De-identification

### When You Need It
- Analytics and reporting on aggregated data across tenants
- Research datasets
- Development and testing environments (never use real PHI in dev/test)
- AI/ML model training
- Sharing data with third parties who don't have a BAA

### Safe Harbor Method (18 Identifiers)
Remove all 18 HIPAA identifiers listed above. The remaining data is considered de-identified and no longer subject to HIPAA. This is the simpler method but removes more data.

### Expert Determination Method
A qualified statistical expert determines that the risk of re-identification is "very small." Allows retaining more data elements but requires expert engagement.

### AWS Services for De-identification

**Amazon Comprehend Medical:**
- DetectPHI API identifies PHI entities in clinical text
- Use to scan text data and flag/mask PHI before it leaves the PHI boundary
- Supports: names, dates, addresses, ages, phone/fax/email, MRNs, SSNs, URLs, device IDs

**Amazon Macie:**
- Scans S3 buckets for PHI/PII using managed data identifiers
- Detects: health insurance numbers, medical record numbers, DEA numbers, NPI numbers
- Use for continuous monitoring: alert if PHI appears in buckets that should only contain de-identified data

**HealthLake De-identified Copy:**
- HealthLake can create a de-identified copy of a FHIR data store
- Automatically removes PHI from FHIR resources
- Useful for analytics pipelines that need FHIR data without PHI

### Tokenization
Replace PHI with tokens (random identifiers) that can be reversed only by the token vault.

**Pattern:**
- Token vault (DynamoDB with per-tenant encryption) maps tokens ↔ PHI values
- Non-clinical systems (analytics, billing, reporting) receive tokenized data
- Only clinical systems with appropriate access can de-tokenize
- Token vault access is strictly controlled and audit-logged

**When to use tokenization vs de-identification:**
- Tokenization: when you need to re-link data to the patient later (e.g., billing needs to eventually generate a statement with the patient's name)
- De-identification: when you never need to re-link (e.g., population health analytics, ML training data)

## Right to Access (HIPAA §164.524)

Patients have the right to access their PHI. Your SaaS must support this.

**Architecture implications:**
- API endpoint for patient data export (all PHI for a specific patient)
- FHIR `$export` operation for bulk data access (if using HealthLake)
- Must respond within 30 days (one 30-day extension allowed)
- Must provide data in the format requested by the patient (if readily producible)
- Audit log: record every access request and fulfillment

## Right to Amendment (HIPAA §164.526)

Patients can request corrections to their PHI.

**Architecture implications:**
- Workflow for amendment requests (receive → review → approve/deny → apply)
- If denied, must allow patient to submit a statement of disagreement
- Amendments must be propagated to anyone the original data was disclosed to
- Audit trail for all amendment actions

## GxP Considerations — When PHI Is Also a Regulated Record

If your healthcare SaaS is also GxP-regulated (SaMD outputs, clinical trial data, pharmacovigilance, regulated lab results), data handling requirements expand beyond HIPAA. The record is simultaneously PHI AND a GxP regulated record — both frameworks apply. See `gxp-compliance-generic.md` for ALCOA+ and generic data integrity principles.

### HIPAA vs GxP Data Handling

| Concern | HIPAA | GxP |
|---------|-------|-----|
| Encryption at rest | Required | Required |
| Access control | Required | Required |
| Audit access | Required | Required + reason for change on modifications |
| Retention | 6 years | Per predicate rule — often longer |
| Right to deletion | GDPR/state laws may require | Generally prohibited for regulated records during retention period |
| Amendment | Allowed per §164.526 | Allowed but with full audit + reason + signature if required |
| Original vs copy | Not distinguished | Must preserve original; copies clearly marked |
| Data migration | Not addressed | Must be validated; migration evidence retained |

**Tension:** HIPAA/GDPR may require deletion; GxP may prohibit deletion during retention. Resolution: GxP retention typically supersedes for regulated records within the retention window. Document this in your data retention policy and ensure patient-facing consent captures the limitation.

### Immutability of Regulated Records

For GxP-regulated PHI:
- Original records never overwritten — updates create new versions
- Prior versions retained for the full regulatory retention period
- Deletion during retention period only via formally-approved deviation
- Cryptographic binding (hash or signature) proves record content is unaltered

**S3 pattern for regulated PHI:**
- Versioning enabled
- Object Lock in Compliance Mode with retention period = regulatory retention
- Previous versions retained (don't delete old versions during the retention period)
- Each version written with server-side encryption using per-tenant KMS CMK

**DynamoDB pattern for regulated PHI:**
- Append-only model — new items created for updates; never overwrite
- Sort key includes version number or timestamp
- Point-in-time recovery enabled
- Or: DynamoDB Streams → S3 with Object Lock for immutable archive

### Data Migration for GxP

Moving regulated PHI between storage systems (tier changes, modernization, re-architecture) requires:
- Documented migration plan approved before execution
- Record-by-record integrity verification (source hash = destination hash)
- Migration log as immutable evidence
- Parallel operation period where both systems retain the data
- Formal sign-off that migration is complete and source can be decommissioned
- Original source system retained until retention period expires, OR migrated data formally qualified as the new source of truth

**This is a common gap** — teams migrate data and delete the source without integrity verification or documentation. For GxP, this is a finding.

### Test Data and Development Environments

The HIPAA rule "never use real PHI in dev/test" applies equally under GxP, with additional GxP nuances:
- Development/test environments are not validated — cannot be used to produce or verify regulated records
- Synthetic data must be clearly marked as synthetic (no confusion with production records)
- If production data is needed for testing, use de-identified or tokenized copies
- Production-to-test data flows must be documented and controlled
- Test data stores themselves need access control but reduced validation scope

### Cryptographic Erasure as a Compliance Tool

For tenant offboarding of regulated PHI, cryptographic erasure via KMS key deletion is a powerful pattern (see `sbt-toolkit.md` for the offboarding sequence):
- Encrypt tenant data with per-tenant KMS CMK
- When tenant exits retention period, schedule KMS key for deletion (7-30 day waiting period)
- After deletion: all data encrypted with the key becomes permanently inaccessible
- This is stronger than data deletion — the data remains on disk but cannot be decrypted
- Document the cryptographic erasure as the data deletion event for audit purposes

**GxP consideration:** Cryptographic erasure must not occur before the regulatory retention period expires. Set KMS key scheduled deletion based on retention period, not tenant offboarding date.

## Common Mistakes

1. **PHI in application logs.** The most common HIPAA violation in SaaS. Never log patient names, MRNs, SSNs, or other identifiers in plaintext. Use tenant IDs and internal record IDs in logs. Scan logs with Comprehend Medical or Macie.

2. **Using real PHI in dev/test environments.** Always use de-identified or synthetic data for development and testing. If you must use real data, the dev environment must have the same HIPAA controls as production.

3. **Shared KMS keys across tenants for PHI.** Per-tenant keys are the healthcare default. Shared keys make tenant offboarding messy and weaken the isolation story for auditors.

4. **Forgetting that AI outputs are PHI.** If the input is PHI, the output is PHI. AI-generated clinical notes, summaries, and recommendations derived from patient data are all PHI.

5. **Not encrypting ElastiCache.** If you cache PHI for performance (patient records, session data), the cache must be encrypted at rest and in transit. Redis supports both. Memcached does not — never cache PHI in Memcached.

6. **Ignoring audio/video as PHI.** Telehealth recordings and ambient documentation audio are PHI. They need the same encryption, access control, and audit logging as any other PHI data store.

## Discovery Questions for This Domain

**Data classification:**
- What types of PHI does your system handle? (Clinical records, claims, imaging, audio/video, device telemetry?)
- Do you handle any of the 18 HIPAA identifiers directly, or do you receive pre-tokenized data?
- Do you store PHI in any non-production environments? (Dev, staging, QA?)

**Encryption:**
- Is encryption at rest enabled on all data stores? (Check each service individually)
- Are you using per-tenant KMS keys or shared keys for PHI?
- Is encryption in transit enforced everywhere? (TLS on all connections, SSL required for database connections?)

**De-identification:**
- Do you need de-identified data for analytics, research, or ML training?
- Which method are you using or planning? (Safe Harbor, Expert Determination, tokenization?)
- How do you prevent PHI from leaking into non-PHI environments? (Macie scanning, Comprehend Medical in pipelines?)

**Patient rights:**
- Can patients request access to their PHI through your system?
- Can patients request amendments to their PHI?
- How long does it take to fulfill an access request today?

**GxP-specific (if regulated):**
- Is any of your PHI also a GxP regulated record (SaMD output, clinical trial data, regulated lab result)?
- Are original records preserved as immutable versions (never overwritten)?
- Is your retention policy aligned with the longer of HIPAA or GxP predicate rule requirements?
- Are data migrations between storage systems validated and documented?
- Do you use cryptographic erasure (per-tenant KMS key deletion) for tenant offboarding? If so, is it gated by regulatory retention?

## References

- [HIPAA Security Rule — Technical Safeguards](https://www.hhs.gov/hipaa/for-professionals/security/guidance/index.html)
- [Common Techniques to Detect PHI and PII Using AWS Services](https://aws.amazon.com/blogs/industries/common-techniques-to-detect-phi-and-pii-data-using-aws-services/)
- [Philips and AWS Automate PHI De-identification](https://aws.amazon.com/blogs/industries/philips-and-aws-automate-phi-de-identification-with-machine-learning/)
- [Masking Patient Data with DataMasque for HealthLake](https://aws.amazon.com/blogs/awsmarketplace/masking-patient-data-datamasques-template-amazon-healthlake/)
- [Identifying Sensitive Healthcare Data with Comprehend Medical](https://aws.amazon.com/blogs/machine-learning/identifying-and-working-with-sensitive-healthcare-data-with-amazon-comprehend-medical/)
- [Amazon Macie — PHI Data Identifiers](https://docs.aws.amazon.com/macie/latest/user/mdis-reference-phi.html)
- [HIPAA Right of Access](https://www.hhs.gov/hipaa/for-professionals/privacy/guidance/access/index.html)
