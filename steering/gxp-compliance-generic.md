---
inclusion: manual
---

# GxP Compliance — Generic Steering Guide for Life Sciences Projects

## Purpose

This steering document defines GxP (Good Practice) compliance requirements applicable to any life sciences software project running on AWS. It covers computerized systems subject to FDA, EMA, and other health authority regulations. Use this as a baseline and tailor retention periods, alert thresholds, and specific controls to your project's regulatory classification and risk profile.

## Applicable Regulations

| Regulation | Jurisdiction | Scope |
|------------|-------------|-------|
| FDA 21 CFR Part 11 | United States | Electronic records and electronic signatures |
| EU Annex 11 | European Union | Computerized systems used in GxP environments |
| IEC 62304 | International | Medical device software lifecycle processes |
| IEC 82304 | International | Health software — general requirements for product safety |
| ISO 13485 | International | Quality management systems for medical devices |
| ISO 14971 | International | Application of risk management to medical devices |
| ISO 27001 | International | Information security management systems |
| HIPAA | United States | Protection of health information (PHI) |
| GDPR | European Union | Personal data protection and privacy |
| ICH Q9 | International | Quality risk management for pharmaceuticals |
| ICH Q10 | International | Pharmaceutical quality system |
| GAMP 5 (ISPE) | Industry guidance | Risk-based approach to compliant GxP computerized systems |

### GAMP 5 Software Categories

Classify your system components using GAMP 5 categories to determine the appropriate level of validation effort:

| Category | Description | Validation Effort | Examples |
|----------|-------------|-------------------|----------|
| 1 | Infrastructure software | Minimal — qualified via IQ | Operating systems, databases, cloud infrastructure |
| 3 | Non-configured products | Standard — IQ/OQ | Off-the-shelf middleware, message brokers, container runtimes |
| 4 | Configured products | Moderate — IQ/OQ/PQ | ERP systems, LIMS with configuration, SaaS platforms |
| 5 | Custom applications | Full — requirements through PQ | All custom-built application code, APIs, processing pipelines |

## Data Integrity — ALCOA+ Principles

All components that create, modify, store, or transmit regulated data must enforce ALCOA+ principles. These apply to any GxP-relevant record: clinical data, manufacturing records, laboratory results, audit trails, or electronic batch records.

| Principle | Requirement | Implementation Guidance |
|-----------|-------------|------------------------|
| **Attributable** | Every action must be traceable to a specific, authenticated individual. System-initiated actions must be attributed to a named service identity. | Unique user IDs via identity provider (OpenID Connect / SAML). No shared accounts. Service accounts with distinct identities per microservice. |
| **Legible** | Records must be readable, permanent, and retrievable throughout their retention period. | Use standard, non-proprietary data formats where possible. Ensure rendering/viewing capability is maintained for the full retention period. |
| **Contemporaneous** | Records must be created at the time the activity occurs, not retroactively. | Server-side UTC timestamps. NTP synchronization on all compute nodes. Clock skew monitoring with alerts. |
| **Original** | The first-captured version of data must be preserved. Certified copies are acceptable if the process is validated. | Immutable storage for original records (S3 Object Lock, WORM storage). Processing results stored as new records, never overwriting originals. |
| **Accurate** | Data must be correct, truthful, and free from errors. | Input validation at all ingestion points. Checksums (SHA-256) for file integrity. Reconciliation checks between storage layers. |
| **Complete** | No data may be deleted, lost, or made inaccessible without an audit trail. All relevant data must be recorded. | Soft deletes only during retention period. Completeness monitoring (gap detection in sequential records). |
| **Consistent** | Data must be internally consistent across all copies, systems, and time periods. | Cross-system reconciliation jobs. Consistent timestamps across distributed components. Version-controlled schemas. |
| **Enduring** | Records must be durable for the entire regulatory retention period. | Durable storage (S3 99.999999999% durability). Backup and archive strategies aligned with retention requirements. Media migration plans for long-term archives. |
| **Available** | Records must be accessible for review, audit, and inspection throughout their lifecycle. | Multi-AZ deployments. Defined RPO/RTO. Indexed and searchable audit logs. Export capability for regulatory inspections. |

## Audit Trail Requirements

### Events That Must Be Logged

| Category | Events |
|----------|--------|
| Data lifecycle | Create, read, update, delete (soft), export, import, archive, restore |
| Authentication | Login, logout, token refresh, failed attempts, account lockout, password change |
| Authorization | Access granted, access denied, privilege escalation, role change |
| Configuration | Any change to system configuration, feature flags, processing parameters |
| System operations | Service start/stop, deployment, scaling, backup, restore, failover |
| Electronic signatures | Signature applied, signature meaning, signer identity |
| Anomalies | Integrity check failure, unauthorized access attempt, clock skew detected |

### Required Fields per Audit Event

Every audit event must include at minimum:

```
timestamp          — UTC, ISO 8601 format, millisecond precision
event_id           — globally unique identifier (UUID v4)
event_type         — standardized event classification
user_id            — authenticated user or service identity
tenant_id          — tenant/organization context (if multi-tenant)
resource_type      — type of resource affected (e.g., patient_record, configuration)
resource_id        — unique identifier of the affected resource
action             — what was done (create, read, update, delete, approve, etc.)
outcome            — success or failure
source_ip          — originating IP address
correlation_id     — request correlation ID for end-to-end tracing
previous_value_hash — SHA-256 hash of previous state (for modifications)
new_value_hash     — SHA-256 hash of new state (for modifications)
reason_for_change  — required for modifications to regulated records
```

### Audit Trail Properties

- **Immutable** — audit logs must not be modifiable or deletable by any user, including system administrators and root accounts. Use S3 Object Lock (Compliance mode) or equivalent WORM storage.
- **Tamper-evident** — integrity of audit logs must be independently verifiable. Use cryptographic hash chains, AWS CloudTrail log file validation, or signed log digests.
- **Retained** — audit logs must be retained for the full regulatory retention period. Typical minimums: 7 years (FDA), 15 years (EU clinical trials), or as defined by your regulatory classification.
- **Searchable** — audit logs must support efficient querying by user, resource, time range, and event type. Use a dedicated search/analytics layer (OpenSearch, Athena, or equivalent).
- **Exportable** — audit logs must be exportable in a human-readable format for regulatory inspections.

### Recommended Architecture

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Application  │────►│  Audit Event SDK  │────►│  Streaming Layer  │
│  Component    │     │  (shared library) │     │  (Kinesis/SQS)   │
└──────────────┘     └──────────────────┘     └────────┬─────────┘
                                                       │
                                           ┌───────────┴───────────┐
                                           ▼                       ▼
                                 ┌─────────────────┐    ┌─────────────────┐
                                 │  Immutable Store  │    │  Search Index    │
                                 │  (S3 + Object    │    │  (OpenSearch /   │
                                 │   Lock)           │    │   Athena)        │
                                 └─────────────────┘    └─────────────────┘
```

- Centralized audit event SDK ensures consistent event format across all components
- Streaming layer decouples event production from storage (prevents audit failures from blocking operations — see fail-closed vs. fail-open decision below)
- Dual-write: immutable archive (S3) for compliance + searchable index for operations

### Fail-Closed vs. Fail-Open Decision

For GxP systems, the default posture is **fail-closed**: if the audit trail cannot be written, the operation must not proceed. This ensures no unaudited actions occur. However, this creates an availability risk — audit infrastructure failure blocks all operations.

Recommended approach:
- **Fail-closed** for data modification operations (create, update, delete)
- **Fail-open with alerting** for read-only operations (query, view) — log the failure, alert immediately, and allow the read to proceed
- Streaming buffer (Kinesis/SQS) provides resilience against transient audit storage failures

## Access Control

### Authentication

- All users must be authenticated via a centralized identity provider (OpenID Connect or SAML 2.0)
- No local authentication mechanisms — all authentication flows through the IdP
- No shared accounts — every human user has a unique identity
- Service-to-service authentication via short-lived credentials (IAM roles, IRSA, workload identity) — no long-lived API keys or passwords
- Multi-factor authentication (MFA) required for all administrative access
- Session management:
  - Idle timeout: configurable, default 30 minutes
  - Absolute timeout: configurable, default 8 hours
  - Session tokens must be invalidated on logout

### Authorization

- Role-Based Access Control (RBAC) enforced at every layer of the stack:
  1. **Network layer** — security groups, NACLs restrict traffic flow
  2. **API layer** — API gateway validates tokens, enforces rate limits
  3. **Application layer** — business logic enforces role-based permissions
  4. **Data layer** — database-level controls (RLS, schema isolation, or database isolation) as defense-in-depth
  5. **Storage layer** — IAM policies and STS-scoped credentials restrict access to authorized data partitions
- Principle of least privilege — users and services receive only the minimum permissions required
- Separation of duties — no single role can both make and approve changes to regulated records
- Privilege escalation (e.g., emergency access) must be:
  - Time-limited with automatic revocation
  - Scoped to the minimum necessary access
  - Audit-logged with explicit justification
  - Reviewed by a supervisor within a defined SLA (e.g., 48 hours)

### Periodic Access Reviews

- Quarterly reviews for administrative and privileged roles
- Annual reviews for all user roles
- Automated detection of orphaned accounts (no login within 90 days) — flag for deactivation
- Access review evidence retained as part of compliance documentation

## Electronic Records and Signatures (21 CFR Part 11 / EU Annex 11)

### Electronic Records

- Electronic records must be stored in a manner that preserves their content, meaning, and context for the full retention period
- Records must be retrievable and renderable in human-readable form
- Record copies must be clearly identified as copies; the system must distinguish originals from copies
- Backup and recovery procedures must ensure no loss of electronic records

### Electronic Signatures

When electronic signatures are required by predicate rules (e.g., batch release, clinical report approval):

- Each signature must include: signer identity, date/time of signing, and the meaning of the signature (e.g., "approved", "reviewed", "authored")
- Signatures must be uniquely linked to their respective records — separating a signature from its record must be detectable
- Signed records must not be modifiable without invalidating the signature
- The system must verify the signer's identity at the time of signing (re-authentication or continuous session validation)
- Signature records must be retained for the same period as the signed record

### Signature Types

| Type | Use Case | Implementation |
|------|----------|----------------|
| Non-biometric (credential-based) | Most GxP applications | Username + password (or MFA) at time of signing; linked to record via cryptographic binding |
| Biometric | High-assurance scenarios | Biometric factor + credential; less common in cloud applications |
| Digital signature (cryptographic) | Tamper-evidence for records | PKI-based signing with certificate chain; provides non-repudiation |

## Validation and Qualification

### Risk-Based Validation (GAMP 5 V-Model)

```
User Requirements ──────────────────────────── Performance Qualification (PQ)
        │                                                    ▲
        ▼                                                    │
Functional Specification ──────────────────── Operational Qualification (OQ)
        │                                                    ▲
        ▼                                                    │
Design Specification ─────────────────────── Installation Qualification (IQ)
        │                                                    ▲
        ▼                                                    │
        └──────────── Build / Configure ─────────────────────┘
```

- Validation effort is proportional to risk (GAMP 5 category and clinical/business impact)
- Traceability matrix: every requirement must trace forward to a test case and backward to a user need
- Validation protocols must be approved before execution
- Deviations from expected results must be documented, investigated, and resolved

### Infrastructure Qualification (IQ)

- Verify that infrastructure is deployed as specified (correct AWS services, regions, configurations)
- Evidence: Terraform/CDK plan output, deployment logs, infrastructure test results
- Drift detection: automated checks that deployed infrastructure matches IaC definitions
- Re-qualification required after significant infrastructure changes

### Operational Qualification (OQ)

- Verify that the system operates correctly within defined parameters
- Evidence: functional test results, integration test results, security scan results, load test results
- Includes: failover testing, backup/restore testing, scaling behavior verification

### Performance Qualification (PQ)

- Verify that the system performs as expected under production-representative conditions
- Evidence: end-to-end test results with production-like data volumes, performance benchmarks
- Includes: tenant isolation verification, data integrity verification, audit trail completeness verification

### Continuous Validation

Traditional validation is point-in-time. For cloud-native systems with frequent deployments, adopt continuous validation:

- Automated test suites (unit, integration, E2E) run on every deployment
- Regression test suite maintained and executed as a deployment gate
- Infrastructure tests validate IQ criteria on every Terraform apply
- Monitoring and alerting serve as ongoing OQ/PQ evidence
- Periodic formal revalidation (annually or after major changes) supplements continuous evidence

## Change Management

### Change Control Process

All changes to validated systems must follow a documented change control process:

1. **Change request** — description, rationale, risk assessment, affected components
2. **Impact assessment** — which validation artifacts are affected; does this require revalidation?
3. **Approval** — appropriate authority based on change classification
4. **Implementation** — executed per approved plan; all changes via version-controlled IaC and CI/CD
5. **Verification** — automated tests + manual verification as needed
6. **Documentation** — change record updated with implementation evidence and test results
7. **Closure** — change record closed; validation status updated

### Change Classification

| Class | Risk | Approval | Timeline | Examples |
|-------|------|----------|----------|----------|
| Standard | Low, pre-assessed | Pre-approved (automated CI/CD gate) | Immediate | Dependency patches, documentation updates, non-functional changes |
| Normal | Medium | Peer review + team lead approval | Planned release cycle | Feature additions, bug fixes, configuration changes, infrastructure updates |
| Major | High | Change Advisory Board (CAB) or equivalent | Scheduled maintenance window | Architecture changes, database migrations, new third-party integrations, security model changes |
| Emergency | Critical (production issue) | Expedited approval; post-hoc CAB review within 48 hours | Immediate | Security vulnerabilities, data integrity issues, service outages |

### Infrastructure Change Control

- All infrastructure defined as code (Terraform, CDK, CloudFormation) — no manual console changes in production
- Infrastructure changes follow the same PR and approval workflow as application code
- `terraform plan` output reviewed and approved before `terraform apply`
- Automated drift detection on schedule — any drift triggers investigation and remediation
- Production environment changes require a separate approval from non-production changes

## Disaster Recovery and Business Continuity

### Backup Strategy

Define RPO (Recovery Point Objective) and RTO (Recovery Time Objective) for each data tier based on regulatory and business requirements:

| Data Tier | Typical RPO | Typical RTO | Backup Method | Retention |
|-----------|-------------|-------------|---------------|-----------|
| Primary database (clinical/regulated data) | ≤ 5 minutes | ≤ 1 hour | Continuous replication + daily snapshots | Minimum regulatory retention period (7–15 years) |
| Object storage (files, images, documents) | 0 (inherent S3 durability) | ≤ 1 hour | Cross-region replication for DR | Per data classification and retention policy |
| File storage (shared filesystems) | ≤ 15 minutes | ≤ 4 hours | Automated daily backups + on-demand | 30–90 days for operational recovery; archive for regulatory retention |
| Audit logs | 0 (S3 durability) | ≤ 1 hour | S3 Object Lock + cross-region replication | Full regulatory retention period |
| Configuration and secrets | ≤ 1 hour | ≤ 1 hour | Versioned storage (Parameter Store, Secrets Manager) | 90 days minimum |

### DR Testing

- DR failover tested at minimum quarterly — documented with results, deviations, and corrective actions
- Restore-from-backup tested monthly for each data tier
- DR test results retained as compliance evidence
- Annual tabletop exercise for business continuity scenarios

### Data Archival

- Define archival policies per data classification (active → archive → deletion after retention expiry)
- Archived data must remain retrievable and readable for the full retention period
- Archive storage (S3 Glacier, Glacier Deep Archive) must be validated for retrieval time against RTO requirements
- Media migration: plan for technology changes over long retention periods (7–15+ years)

## Encryption

### At Rest

- All regulated data encrypted at rest using AWS KMS (or equivalent key management)
- Encryption key management:
  - AWS-managed keys (SSE-S3, SSE-SQS) acceptable for non-PHI data
  - Customer-managed keys (CMK) required for PHI and regulated clinical data
  - Key rotation: automatic annual rotation for CMKs
  - Key access audited via CloudTrail
- Database encryption: Aurora/RDS encryption enabled at cluster creation (cannot be added retroactively)
- Storage encryption: S3 default encryption (SSE-KMS), FSx/EFS encryption enabled at creation

### In Transit

- TLS 1.2 or higher for all network communication — no exceptions
- Internal service-to-service communication encrypted (service mesh mTLS or VPC-internal TLS)
- Certificate management: ACM (AWS Certificate Manager) for public certificates; Private CA for internal certificates
- No plaintext protocols in production (HTTP, unencrypted database connections, etc.)

## Observability for Compliance

### Required Monitoring

| Monitor | Purpose | Alert Threshold |
|---------|---------|-----------------|
| Audit trail completeness | Detect gaps in audit event delivery | Any gap > 5 minutes |
| Data integrity checks | Verify checksums of stored records match computed checksums | Any mismatch |
| Unauthorized access attempts | Detect potential security breaches | 5+ failed attempts in 10 minutes per user; any cross-tenant access attempt |
| Configuration drift | Detect unauthorized infrastructure changes | Any drift from IaC-defined state |
| Backup success | Verify backups complete successfully | Any backup failure |
| Certificate expiry | Prevent TLS failures | 30 days before expiry |
| Clock synchronization | Ensure timestamp accuracy | Clock skew > 1 second |
| Encryption status | Verify all storage is encrypted | Any unencrypted resource detected |

### Compliance Dashboards

Maintain dashboards that provide at-a-glance compliance posture:

- Validation status per component (qualified / pending / expired)
- Open CAPAs (Corrective and Preventive Actions) with aging
- Pending access reviews
- Audit trail health (delivery rate, gap count, storage utilization)
- DR test results (last test date, pass/fail, next scheduled)
- Change control metrics (open changes, emergency change count, mean time to close)

## Coding Standards for GxP Systems

When writing code for any GxP-regulated system:

1. **Audit before act** — emit an audit event before performing any data modification. For fail-closed operations, the modification must not proceed if the audit event cannot be published.
2. **Input validation** — validate all inputs at service boundaries. Reject malformed data early. Use schema validation (OpenAPI, JSON Schema, or equivalent) for API inputs.
3. **Error handling** — never expose internal error details (stack traces, database errors, file paths) to external clients. Log full error context server-side with correlation ID. Return standardized error codes to clients.
4. **No hardcoded credentials** — all secrets retrieved from a secrets manager (AWS Secrets Manager, HashiCorp Vault, or equivalent) or injected via environment variables by the platform. No credentials in source code, configuration files, container images, or logs.
5. **Encryption everywhere** — data encrypted at rest and in transit. No exceptions. No fallback to plaintext.
6. **Immutable regulated records** — original regulated records must never be modified after creation. Updates create new versions; deletions are soft deletes with audit trail.
7. **Correlation IDs** — every request carries a correlation ID (generated at the entry point if not present) propagated through all downstream calls for end-to-end traceability.
8. **Timestamps** — all timestamps in UTC, ISO 8601 format, millisecond precision. Server-side generation only — never trust client-submitted timestamps for regulated records.
9. **Deterministic builds** — use pinned dependency versions (lock files), signed container images, and reproducible build pipelines. Build artifacts must be traceable to source code commit.
10. **Code review** — all changes require peer review before merge. Reviewer must have domain knowledge of the affected component. Review evidence (PR approval) serves as change control documentation.

## Supplier and Third-Party Qualification

All third-party components used in a GxP system must be assessed and documented:

### Qualification Process

1. **Risk assessment** — determine the component's impact on regulated functionality (direct, indirect, or no impact)
2. **Supplier evaluation** — assess the supplier's quality system, support model, security practices, and regulatory track record
3. **Functional assessment** — verify the component meets requirements through testing or documentation review
4. **Version control** — pin to specific, qualified versions; updates require re-assessment
5. **Ongoing monitoring** — track security advisories, end-of-life announcements, and breaking changes

### Supplier Register Template

| Component | Supplier | Version | GAMP Category | Risk Impact | Qualification Evidence | Qualified Date | Review Date |
|-----------|----------|---------|---------------|-------------|----------------------|----------------|-------------|
| (name) | (vendor) | (pinned version) | 1/3/4/5 | Direct/Indirect/None | (IQ report, vendor audit, documentation review) | (date) | (next review date) |

### AWS as a GxP Supplier

- AWS publishes a [GxP on AWS whitepaper](https://aws.amazon.com/compliance/gxp-part-11-annex-11/) mapping AWS controls to 21 CFR Part 11 and EU Annex 11 requirements
- Execute a Business Associate Agreement (BAA) with AWS for HIPAA-covered workloads
- Use only [HIPAA-eligible AWS services](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/) for regulated data
- AWS shared responsibility model: AWS qualifies physical infrastructure and managed service internals; you qualify your configuration, application code, and data management practices

## Documentation Requirements

Maintain the following documentation for GxP compliance:

| Document | Purpose | Update Frequency |
|----------|---------|-----------------|
| System Description (HLD) | Describes system architecture, components, data flows | Per major release |
| Detailed Design (LLD) | Component-level design, APIs, data models | Per component change |
| User Requirements Specification (URS) | What the system must do from the user's perspective | Per major release |
| Functional Specification (FS) | How the system meets user requirements | Per major release |
| Validation Plan | Overall validation strategy, scope, acceptance criteria | Per major release |
| Validation Protocols (IQ/OQ/PQ) | Step-by-step test procedures with expected results | Per release |
| Validation Summary Report | Results of validation execution, deviations, conclusions | Per release |
| Traceability Matrix | Requirements → design → test cases → results | Per release |
| Risk Assessment | Identified risks, mitigations, residual risk acceptance | Quarterly review |
| Supplier Qualification Register | Third-party component assessments | Per new supplier or version change |
| CAPA Log | Corrective and preventive actions | As needed |
| Change Control Records | All changes to validated systems | Continuous (Git + PR records) |
| Training Records | Evidence of personnel training | Per onboarding, role change, or system update |
| SOPs | Standard operating procedures for system operation, maintenance, incident response | Annual review |
| Audit Trail Retention Policy | Defines retention periods per data classification | Annual review |
| Disaster Recovery Plan | DR procedures, RPO/RTO, escalation contacts | Annual review + post-DR-test update |

## Compliance Review Checklist

Before approving any design document or release for a GxP system, verify:

- [ ] **Data integrity**: ALCOA+ principles addressed for all regulated data stores
- [ ] **Audit trails**: all data access and modification paths emit immutable, tamper-evident audit events
- [ ] **Access control**: RBAC enforced at every layer; least privilege; no shared accounts; MFA for admin access
- [ ] **Electronic signatures**: if applicable, signatures linked to records with identity, timestamp, and meaning
- [ ] **Encryption**: at rest (KMS) and in transit (TLS 1.2+) for all data tiers — no exceptions
- [ ] **Validation**: test strategy defined with traceability to requirements; IQ/OQ/PQ protocols documented
- [ ] **Change management**: all changes via version control and CI/CD with approval gates
- [ ] **Backup and DR**: RPO/RTO defined per data tier; backup method and retention specified; DR tested quarterly
- [ ] **Observability**: compliance dashboards and alerts defined for audit trail health, data integrity, access anomalies
- [ ] **Supplier qualification**: all third-party components documented in supplier register with version pinning
- [ ] **Documentation**: all required documents exist, are current, and are under version control
- [ ] **Training**: all personnel trained on relevant SOPs and system changes
