# Audit Logging & Access Controls

## Why This Is Non-Negotiable

HIPAA §164.312(b) requires audit controls — mechanisms to record and examine activity in systems containing ePHI. This isn't optional or addressable; it's required. For healthcare SaaS, audit logging is the single most scrutinized control during HITRUST assessments, SOC 2 audits, and OCR investigations. If you can't prove who accessed what PHI and when, you can't prove compliance.

## What Must Be Logged

### Infrastructure-Level Events (CloudTrail)
- All API calls to AWS services handling PHI (management events)
- Data-level access to S3 objects, DynamoDB items, KMS key usage (data events)
- Console sign-in events
- IAM policy changes, role assumptions, credential usage
- Security group and NACL changes
- Encryption key creation, rotation, deletion

### Application-Level Events (Your Responsibility)
- PHI access: who accessed which patient's data, when, from where
- PHI modification: what was changed, old value vs new value (or hash), who changed it
- PHI deletion: what was deleted, who authorized it, why
- Failed access attempts: authentication failures, authorization denials, tenant boundary violations
- Consent events: consent granted, revoked, or modified (critical for 42 CFR Part 2)
- Export events: patient data exports, bulk data access, FHIR $export operations
- Break-the-glass events: emergency access grants, who authorized, time-limited scope

### What NOT to Log
- The PHI itself in log entries. Log the event (who, what resource, when, action) but NOT the patient name, MRN, SSN, or clinical content. Use internal record IDs and tenant IDs.
- Exception: if you need clinical content in logs for specific audit purposes, those logs must be treated as PHI (encrypted, access-controlled, retained per HIPAA).

## CloudTrail Configuration for Healthcare

### Organization Trail (Recommended)
For multi-account architectures (silo model), create an organization trail that aggregates all accounts:
- Covers all regions (multi-region trail)
- Includes management events AND data events for PHI services
- Delivers to a centralized S3 bucket in a dedicated log archive account
- Log file validation enabled (detects tampering)

### Data Events to Enable
These are not enabled by default and cost extra, but are required for PHI audit trails:

| Service | Data Event Type | Why |
|---------|----------------|-----|
| S3 | GetObject, PutObject, DeleteObject | Track access to PHI stored in S3 (documents, images, recordings) |
| DynamoDB | GetItem, PutItem, DeleteItem, Query, Scan | Track access to PHI in DynamoDB tables |
| KMS | Decrypt, Encrypt, GenerateDataKey | Track when tenant encryption keys are used (proxy for PHI access) |
| Lambda | Invoke | Track function invocations that process PHI |

**Cost consideration:** Data events generate high log volume. For pool-model DynamoDB tables with millions of requests, this can be expensive. Consider: enable data events for PHI-critical tables only, use KMS Decrypt events as a proxy for PHI access (cheaper than logging every DynamoDB operation).

## Immutable Audit Logs — S3 Object Lock

Audit logs must be tamper-proof. If an attacker (or insider) can delete or modify logs, the audit trail is worthless.

### S3 Object Lock Configuration
- **Compliance Mode:** No one can delete or overwrite objects during the retention period — not even the root account. Use this for production audit logs.
- **Governance Mode:** Allows users with specific IAM permissions to override. Use for non-production environments.
- **Retention Period:** Set to your longest applicable requirement (HIPAA: 6 years, some state laws: 7-10 years, your organization's policy may be longer).

### Implementation Pattern
```
CloudTrail → S3 Bucket (Object Lock: Compliance Mode, 7-year retention)
                ├── Log file validation enabled
                ├── Bucket policy: deny s3:DeleteObject, deny s3:PutBucketPolicy changes
                ├── Versioning enabled (required for Object Lock)
                └── Encrypted with dedicated KMS key (not a tenant key — this is operational)
```

**CDK example — immutable audit log bucket:**
```typescript
const auditBucket = new s3.Bucket(this, 'AuditLogBucket', {
  bucketName: `${props.appName}-audit-logs`,
  encryption: s3.BucketEncryption.KMS,
  encryptionKey: auditKmsKey,
  versioned: true, // required for Object Lock
  objectLockEnabled: true,
  objectLockDefaultRetention: s3.ObjectLockRetention.compliance(Duration.days(2555)), // ~7 years
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  removalPolicy: RemovalPolicy.RETAIN,
});

auditBucket.addToResourcePolicy(new iam.PolicyStatement({
  sid: 'DenyDeleteActions',
  effect: iam.Effect.DENY,
  principals: [new iam.AnyPrincipal()],
  actions: ['s3:DeleteObject', 's3:DeleteObjectVersion'],
  resources: [auditBucket.arnForObjects('*')],
}));
```

**CDK example — CloudTrail with data events for PHI services:**
```typescript
const trail = new cloudtrail.Trail(this, 'HipaaAuditTrail', {
  bucket: auditBucket,
  isMultiRegionTrail: true,
  enableFileValidation: true,
  encryptionKey: auditKmsKey,
});

// S3 data events for PHI buckets
trail.addS3EventSelector([{ bucket: phiDataBucket }], {
  readWriteType: cloudtrail.ReadWriteType.ALL,
  includeManagementEvents: false,
});

// DynamoDB data events for PHI tables
trail.addEventSelector(cloudtrail.DataResourceType.DYNAMODB_TABLE, [phiTable.tableArn], {
  readWriteType: cloudtrail.ReadWriteType.ALL,
});
```

### CloudWatch Logs Encryption
- All CloudWatch Log Groups containing application logs must be encrypted with KMS
- Set retention period to match your compliance requirement (6+ years)
- Never store PHI in plaintext in CloudWatch Logs — use structured logging with internal IDs only

## Application-Level Audit Trail

CloudTrail captures AWS API calls. It does NOT capture application-level PHI access (e.g., "Dr. Smith viewed Patient Jones's lab results at 2:15 PM"). You must build this.

### Audit Event Schema
```json
{
  "timestamp": "2025-01-15T14:15:00Z",
  "event_type": "PHI_ACCESS",
  "tenant_id": "tenant-abc123",
  "user_id": "user-dr-smith-456",
  "user_role": "physician",
  "patient_record_id": "record-789",
  "resource_type": "lab_result",
  "action": "read",
  "source_ip": "10.0.1.50",
  "user_agent": "clinical-app/2.1.0",
  "consent_status": "active",
  "session_id": "sess-xyz",
  "outcome": "success"
}
```

**Key fields:**
- `tenant_id` and `user_id`: who accessed (never log patient name — use internal IDs)
- `patient_record_id`: what was accessed (internal ID, not MRN)
- `action`: read, write, delete, export, amend
- `consent_status`: was consent active at time of access (critical for 42 CFR Part 2)
- `outcome`: success or failure (failed attempts are as important as successful ones)

### Storage for Audit Events
- Write to a dedicated DynamoDB table or Kinesis stream → S3 (for long-term retention)
- Separate from application data — audit logs should not be in the same table as PHI
- Encrypt with a dedicated KMS key (not a tenant key)
- Retention: 6+ years, immutable (S3 Object Lock on the archive)

## Break-the-Glass Access

### What It Is
Emergency access to PHI when normal authentication/authorization is insufficient. Example: a patient arrives unconscious in the ER, and the treating physician needs access to records from a different clinic (tenant) that the physician doesn't normally have access to.

### Architecture Pattern

**Pre-conditions:**
- Break-the-glass is a defined, documented procedure — not ad hoc
- Only specific roles can initiate (e.g., attending physician, charge nurse, system administrator)
- The system must support it without requiring a developer to modify IAM policies manually

**Implementation:**
1. User requests emergency access through the application (not by calling support)
2. Application verifies the user's role is authorized for break-the-glass
3. Application creates a time-limited elevated session (e.g., 2-hour window)
4. Elevated session grants read access to the specific patient's records across tenant boundaries
5. Every action during the elevated session is logged with `event_type: BREAK_THE_GLASS`
6. Automatic notification sent to: the patient's primary tenant admin, the compliance team, the requesting user's supervisor
7. Session expires automatically after the time limit
8. Post-incident review is mandatory — compliance team reviews the access within 24-48 hours

**AWS implementation:**
- Cognito: add user to a temporary "emergency-access" group with elevated permissions
- IAM: session policy with time-limited cross-tenant read access
- Lambda: orchestrate the grant, notification, and expiration
- Step Functions: manage the post-incident review workflow
- DynamoDB: track active break-the-glass sessions with TTL for automatic expiration

### What to Log for Break-the-Glass
- Who requested emergency access
- What justification was provided
- Who authorized it (if approval is required)
- What records were accessed during the elevated session
- When the session started and ended
- Whether post-incident review was completed

## Access Reviews

### Periodic Review Requirements
HIPAA requires ongoing review of access to ePHI. HITRUST and SOC 2 formalize this as periodic access reviews.

**What to review:**
- Who has access to PHI systems? (IAM users, roles, Cognito users)
- Are there unused accounts or permissions? (Users who left, roles no longer needed)
- Are permissions appropriate for each user's current role? (Least privilege)
- Are there any cross-tenant access grants that should have been revoked?
- Are break-the-glass sessions being reviewed within the required timeframe?

**Automation:**
- IAM Access Analyzer: identify unused permissions and external access
- Cognito: list users per tenant, identify inactive accounts (no login in 90 days)
- Config Rules: detect overly permissive IAM policies, unencrypted resources
- Custom Lambda: generate monthly access review reports per tenant

### Frequency
- Quarterly: full access review (who has access to what)
- Monthly: review break-the-glass events and cross-tenant access grants
- Continuous: automated alerting on anomalous access patterns (GuardDuty, CloudWatch Alarms)

## GxP Alignment — ALCOA+ and 21 CFR Part 11

If your healthcare SaaS is also GxP-regulated (SaMD, eClinical, pharmacovigilance, regulated labs), audit logging requirements expand beyond HIPAA §164.312(b). See `gxp-compliance-generic.md` for the full ALCOA+ framework. This section covers the healthcare-SaaS-specific intersections.

### HIPAA vs GxP Audit Trail Requirements

Both apply simultaneously for regulated healthcare SaaS. HIPAA asks "was access to ePHI logged?" GxP asks "can you reconstruct the complete record lifecycle and prove data integrity?"

| Requirement | HIPAA §164.312(b) | GxP (21 CFR Part 11 / Annex 11) |
|-------------|-------------------|--------------------------------|
| What to log | Access to ePHI | Every create/modify/delete of regulated records |
| Old vs new value | Not required | Required for modifications |
| Reason for change | Not required | Required for modifications |
| Immutability | Required | Required |
| Time synchronization | Best practice | Required (NTP, documented) |
| Retention | 6 years minimum | Per predicate rule — often 10-15+ years |
| Signature on modifications | Not required | May be required (e.g., batch record amendment) |
| Independent integrity check | Not required | Required (log file validation, hash chains) |

### Required Audit Fields for GxP

Generic HIPAA audit fields (earlier in this file) are insufficient for GxP. GxP-regulated operations on regulated records must additionally include:

```json
{
  "timestamp": "2025-01-15T14:15:00.123Z",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "REGULATED_RECORD_UPDATE",
  "tenant_id": "tenant-abc123",
  "user_id": "user-dr-smith-456",
  "user_role": "physician",
  "resource_type": "clinical_report",
  "resource_id": "record-789",
  "action": "update",
  "outcome": "success",
  "source_ip": "10.0.1.50",
  "correlation_id": "req-abc-def",
  "previous_value_hash": "sha256:abc123...",
  "new_value_hash": "sha256:def456...",
  "reason_for_change": "Clarification of measurement after re-review",
  "record_version_before": 3,
  "record_version_after": 4,
  "signature_applied": true,
  "signature_id": "sig-xyz-789",
  "system_clock_sync_source": "ntp.aws.amazon.com",
  "audit_log_checksum": "sha256:789abc..."
}
```

Key additions:
- `previous_value_hash` / `new_value_hash` — proves the log entry matches the actual change
- `reason_for_change` — required by 21 CFR Part 11 for amendments to regulated records
- `record_version_before` / `record_version_after` — enables reconstruction of record lineage
- `signature_applied` / `signature_id` — links to electronic signature if required
- `audit_log_checksum` — hash chain entry for tamper evidence

### ALCOA+ Applied to Audit Logs

| Principle | Audit Log Implementation |
|-----------|-------------------------|
| Attributable | Every event has authenticated user_id or service identity — no shared accounts |
| Legible | Structured JSON with documented schema; human-readable export supported |
| Contemporaneous | Logged at time of event, UTC, server-side timestamp, NTP-synchronized |
| Original | Original log entry immutable; corrections are new entries, not edits |
| Accurate | Event matches actual system state; hashes prevent log tampering |
| Complete | All regulated events logged; gap detection alerts on missing sequential IDs |
| Consistent | Same schema across services; correlation_id ties events across distributed components |
| Enduring | S3 Object Lock, Compliance Mode, for full regulatory retention |
| Available | Indexed in OpenSearch/Athena for inspection retrieval |

### 21 CFR Part 11 Electronic Signatures

When a predicate rule requires a signature (e.g., batch release in pharma manufacturing, clinical report approval, regulated lab result verification), the electronic signature must meet 21 CFR Part 11 Subpart C requirements.

**What an e-signature must include:**
- Printed name of the signer
- Date and time when the signature was executed
- Meaning of the signature (approved, reviewed, authored, etc.)
- Binding to the signed record — breaking the binding must be detectable

**Architecture pattern:**

```
Signing Request
     ↓
Re-authenticate signer (second factor of authentication at time of signing — not just session validity)
     ↓
Create Signature Event:
  - signer_id (from re-authentication)
  - signed_at (server UTC timestamp)
  - meaning (approved / reviewed / authored / etc.)
  - record_id and record_content_hash
  - previous_signature_id (if part of a signature sequence)
     ↓
Compute cryptographic signature over: signer_id + signed_at + meaning + record_content_hash
     ↓
Store signature with record:
  - Signature stored in record metadata (cannot be separated from record)
  - Original record state frozen at signing time
  - Any modification after signing invalidates the signature (content_hash mismatch)
     ↓
Audit log entry: E-SIGNATURE_APPLIED event
```

**AWS implementation options:**
- **Re-authentication:** Cognito challenge with MFA, OR signed JWT with short expiry (< 5 min) indicating fresh authentication
- **Signature computation:** KMS GenerateDataKey for a signing key scoped to the signer/tenant, or asymmetric KMS keys for non-repudiation
- **Storage:** Signature record in DynamoDB linked to record; record_content_hash verifies the record hasn't changed
- **Audit:** Signature event logged to the audit trail with all required fields

**Signature types per 21 CFR Part 11:**

| Type | Description | Use Case |
|------|-------------|----------|
| Non-biometric (credential-based) | Username + password (or MFA) at time of signing | Most SaaS clinical approvals |
| Biometric | Biometric factor (face, fingerprint) + credential | High-assurance, less common in cloud SaaS |
| Digital signature (cryptographic) | PKI-based with certificate chain | Tamper-evidence, non-repudiation required |

### Trusted Time — Clock Synchronization

21 CFR Part 11 §11.10(k) and Annex 11 require trustworthy time. Architecture requirements:
- All compute nodes NTP-synchronized (AWS Time Sync Service provides this on EC2/Fargate)
- Clock skew monitoring — alert on skew > 1 second
- Timestamps recorded server-side — never trust client-submitted timestamps for regulated records
- NTP source documented in supplier qualification
- Log entries include NTP source for auditability

### Deployment Approval as Electronic Signature

For GxP-regulated deployments (see `resilience-and-deployment.md`), the release approval is an electronic signature. Requirements:
- Approver re-authenticates at time of approval
- Approval captures: signer identity, timestamp, meaning (e.g., "approved for production release"), release package hash
- Two-signature requirement common for major releases (e.g., engineering lead + quality lead)
- Signatures immutable and linked to the specific release artifact

## Common Mistakes

1. **Not enabling CloudTrail data events for PHI services.** Management events alone don't tell you who accessed which S3 object or DynamoDB item. Data events are required for PHI audit trails.

2. **Audit logs that are deletable.** If logs can be deleted by an admin or attacker, they're worthless for compliance. Use S3 Object Lock in Compliance Mode.

3. **Logging PHI in audit entries.** The audit log should record the event (who, what resource ID, when, action) — not the PHI content itself. If your audit log contains patient names, it becomes a PHI data store with its own compliance requirements.

4. **No application-level audit trail.** CloudTrail captures AWS API calls. It doesn't capture "Dr. Smith viewed Patient Jones's lab results." You must build application-level audit logging.

5. **No break-the-glass procedure.** If emergency access requires a developer to manually modify IAM policies, you'll fail the HITRUST assessment. Build it into the application.

6. **Skipping access reviews.** HITRUST and SOC 2 auditors will ask for evidence of periodic access reviews. Automate as much as possible — manual reviews don't scale.

7. **Insufficient log retention.** HIPAA requires 6 years. Some state laws require longer. Set retention policies explicitly — don't rely on defaults.

8. **Treating 21 CFR Part 11 like HIPAA audit logging.** Part 11 requires old-value/new-value/reason-for-change on modifications to regulated records, plus electronic signatures where predicate rules demand them. HIPAA audit logging doesn't. If you're GxP-regulated, your HIPAA audit trail is the floor, not the ceiling.

9. **Trusting client-submitted timestamps for regulated records.** Client clocks can be wrong or manipulated. For GxP, timestamps must be server-side and NTP-synchronized.

10. **Separating electronic signatures from the records they sign.** The signature must be cryptographically bound to the record content — modifying the record must invalidate the signature. Storing a "signed: true" flag is not a signature.

## Discovery Questions for This Domain

**Current state:**
- Are CloudTrail data events enabled for S3, DynamoDB, and KMS? (If not, this is the highest-priority fix)
- Do you have application-level audit logging for PHI access? (Not just infrastructure logs)
- Are audit logs immutable? (S3 Object Lock, or equivalent)
- What is your current log retention period? (Must be 6+ years for HIPAA)

**Access control:**
- Do you have a break-the-glass procedure? (Documented, tested, auditable?)
- How often do you review who has access to PHI systems? (Quarterly minimum for HITRUST)
- Can you detect anomalous PHI access patterns? (Unusual volume, off-hours access, cross-tenant access)

**Compliance evidence:**
- Can you produce an audit trail for a specific patient's data access over the past year? (This is what OCR asks during an investigation)
- Do you have evidence of periodic access reviews? (HITRUST/SOC 2 requirement)
- Are break-the-glass events reviewed within 24-48 hours?

**GxP-specific (if regulated):**
- Are modifications to regulated records captured with old value, new value, and reason for change?
- Are audit logs cryptographically chained for tamper evidence?
- Do audit logs include record version before/after modifications?
- If electronic signatures are required: are signers re-authenticated at signing time, and are signatures bound to the record content?
- Are all compute nodes NTP-synchronized with documented time sources?
- Is your audit retention aligned with the predicate rule (often longer than HIPAA's 6 years for GxP records)?

## References

- [HIPAA Security Rule — Audit Controls §164.312(b)](https://www.hhs.gov/hipaa/for-professionals/security/guidance/index.html)
- [AWS CloudTrail Data Events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html)
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Healthcare Industry Lens — Detective Controls](https://docs.aws.amazon.com/wellarchitected/latest/healthcare-industry-lens/detective-controls.html)
- [Using CloudTrail Data Events to Audit SNS and SQS Workloads](https://aws.amazon.com/blogs/mt/using-aws-cloudtrail-data-events-to-audit-your-amazon-sns-and-amazon-sqs-workloads/)
- [IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)
- [21 CFR Part 11 — Electronic Records; Electronic Signatures](https://www.ecfr.gov/current/title-21/chapter-I/subchapter-A/part-11)
- [EU Annex 11 — Computerised Systems](https://health.ec.europa.eu/document/download/6de3c89d-0d79-4ff2-832b-a9bf40eac09b_en)
- [AWS Time Sync Service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-time.html)
