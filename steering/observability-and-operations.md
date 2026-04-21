# Observability & Operations

## The SaaS Observability Challenge

In a multi-tenant environment, standard monitoring isn't enough. You need tenant-aware observability — the ability to see how each tenant is using the system, what load they're placing on it, and whether any tenant is impacting others.

The SaaS Lens design principle: "Instrument, capture, and analyze tenant metrics." This data is core to analyzing tenant trends that directly impact the business, architectural, and operational health of a SaaS company.

## Tenant-Aware Logging

### Principle: Every Log Entry Must Include Tenant Context

Every log statement should include the tenant ID. Without it, you can't filter logs by tenant when debugging issues, and you can't correlate log events across services for a specific tenant.

### Implementation Pattern

**Structured logging with tenant context:**
- Extract tenant ID from JWT/request context at the entry point
- Store in request-scoped context (thread-local, async context, middleware)
- Logging framework automatically includes tenant ID in every log entry

**Log format example (JSON structured logging):**
```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "INFO",
  "tenant_id": "abc123",
  "tenant_tier": "professional",
  "user_id": "user-456",
  "service": "order-service",
  "message": "Order created",
  "order_id": "ord-789",
  "trace_id": "1-abc-def"
}
```

### CloudWatch Log Groups Strategy

**Option 1: Single log group, filter by tenant ID**
- All tenants' logs in one log group per service
- Use CloudWatch Insights to query by tenant_id field
- Simpler to manage, but harder to set per-tenant retention or access controls

**Option 2: Log group per tenant (silo model)**
- Separate log group per tenant: `/saas/tenant-abc123/order-service`
- Per-tenant retention policies, per-tenant access controls
- Operational overhead scales with tenant count

**Recommendation:** Use Option 1 (single log group) for pool model, Option 2 for silo model. For bridge model, use single log group with structured tenant context.

## PHI in Logs — Healthcare Warning

**CRITICAL:** The most common HIPAA violation in healthcare SaaS is PHI in application logs. If your logs contain patient names, MRNs, SSNs, diagnoses, or other identifiers in plaintext, those logs are PHI — subject to all HIPAA controls (encryption, access control, audit, retention).

### Prevention
- **Structured logging with internal IDs only:** Log tenant_id, user_id, record_id — never patient_name, mrn, ssn, diagnosis
- **Log sanitization middleware:** Before writing any log entry, scan for PHI patterns and redact
- **Comprehend Medical in log pipeline:** Run DetectPHI on log entries to catch accidental PHI inclusion
- **Macie on log buckets:** Enable Macie scanning on S3 buckets where logs are archived to detect PHI that slipped through

### If PHI Must Be in Logs
In rare cases (clinical audit trails, debugging clinical workflows), logs may intentionally contain PHI:
- Those log groups must be encrypted with KMS
- Access restricted to authorized roles only (not all developers)
- Retention: 6+ years per HIPAA (see `audit-logging-and-access.md`)
- Treated as PHI data stores in your HIPAA Service Eligibility Matrix

### HIPAA Audit Log Retention
HIPAA requires retention of audit logs for 6 years minimum. Some state laws require longer (TX HB 300, NY SHIELD). Set CloudWatch Log Group retention and S3 lifecycle policies explicitly — do not rely on defaults.

### Healthcare-Specific Operational Metrics
In addition to standard SaaS metrics, healthcare SaaS should track:
- PHI access events per tenant (volume, patterns, anomalies)
- Consent status changes per tenant
- Break-the-glass events (should be rare — high frequency is a red flag)
- FHIR API latency and error rates per tenant (if using HealthLake)
- DICOM retrieval latency per tenant (if using HealthImaging)
- AI inference volume and latency per tenant (if using Bedrock for clinical AI)
- De-identification pipeline throughput and error rates

## Tenant-Partitioned Metrics

### CloudWatch Custom Metrics with Tenant Dimensions

Publish metrics with tenant ID as a dimension to enable per-tenant dashboards:

**Key metrics to capture per tenant:**
- API request count and latency (by endpoint)
- Error rate (4xx, 5xx)
- Database consumed capacity (DynamoDB RCU/WCU per tenant)
- Storage consumed
- Compute duration (Lambda duration per tenant)
- Active users / sessions

**Cost warning:** CloudWatch custom metrics are priced per unique metric (combination of namespace + metric name + dimensions). High tenant count × many metrics × many dimensions = significant cost. Consider:
- Aggregating to hourly/daily granularity for cost metrics
- Using CloudWatch Embedded Metric Format (EMF) for high-cardinality metrics — it's more cost-effective
- Reserving per-tenant CloudWatch metrics for top-tier tenants; use aggregated metrics for basic tier

### Embedded Metric Format (EMF)

EMF lets you embed metric data in structured log entries. CloudWatch automatically extracts metrics from the logs. This is more cost-effective than PutMetricData for high-cardinality scenarios.

**How it works:** Your application writes a specially formatted JSON log entry. CloudWatch Logs automatically creates metrics from it without additional API calls.

## Distributed Tracing with Tenant Context

### AWS X-Ray / OpenTelemetry

Add tenant ID as an annotation or attribute on every trace segment. This enables:
- Filtering traces by tenant to debug tenant-specific issues
- Identifying which tenants are generating the most latency
- Correlating cross-service calls for a specific tenant

**X-Ray annotation:** `tenant_id: abc123` (annotations are indexed and searchable)

**OpenTelemetry attribute:** Add `tenant.id` as a span attribute

**For silo model with cross-account deployments:** Use AWS Distro for OpenTelemetry (ADOT) to collect traces across tenant accounts and aggregate in a central observability account.

**Reference:** [Tracing Cross-Account Tenant Activities for SaaS Solutions with AWS Distro for OpenTelemetry](https://aws.amazon.com/blogs/apn/tracing-cross-account-tenant-activities-for-saas-solutions-with-aws-distro-for-open-telemetry/)

## Noisy Neighbor Detection and Mitigation

### What Is a Noisy Neighbor?

A tenant whose usage pattern consumes disproportionate shared resources, degrading performance for other tenants. Examples:
- A tenant running a bulk import that saturates DynamoDB throughput
- A tenant with a runaway process making thousands of API calls per second
- A tenant storing massive objects that fill the shared cache

### Detection

**Metrics to monitor:**
- Per-tenant API call rate (sudden spikes)
- Per-tenant DynamoDB consumed capacity vs. provisioned
- Per-tenant Lambda concurrent executions
- Per-tenant error rate (may indicate they're hitting limits)
- Overall system latency correlated with per-tenant activity

**Alerting:** Set CloudWatch alarms on per-tenant metrics that exceed expected thresholds for their tier. Alert when a single tenant's consumption exceeds X% of total capacity.

### Mitigation

**Preventive (design-time):**
- Throttling per tier via API Gateway usage plans
- DynamoDB auto-scaling with per-tenant monitoring
- Lambda reserved concurrency for critical functions
- Queue-based load leveling (SQS) to absorb spikes

**Reactive (runtime):**
- Automatically throttle the noisy tenant (increase throttling limits temporarily)
- Move the noisy tenant to dedicated resources (silo) if the problem persists
- Notify the tenant that they're exceeding their tier limits

## Throttling and Fairness Patterns

### API Gateway Usage Plans

The primary mechanism for per-tenant throttling:
- Rate limit: sustained requests per second
- Burst limit: maximum concurrent requests
- Quota: total requests per day/week/month

Map usage plans to tiers. Each tenant gets an API key associated with their tier's plan.

### Application-Level Fairness

For more granular control than API Gateway provides, implement fairness at the application level using patterns from the AWS Builders Library.

**Token Bucket Algorithm:**
- Each tenant has a token bucket with a configured rate and burst capacity
- Requests consume tokens; when empty, requests are rejected (429)
- Tokens refill at the configured rate
- Can be implemented locally (per-instance) or distributed (shared state)

**Local vs. Distributed Admission Control:**

| Approach | How It Works | Best For |
|----------|-------------|----------|
| Local (per-instance) | Each instance enforces limits independently, dividing quota by fleet size | Uniform traffic distribution, simpler implementation |
| Distributed (shared state) | Centralized rate tracking (e.g., ElastiCache/Redis) | Non-uniform traffic, precise enforcement |
| Hybrid | Local enforcement with periodic sync to central state | Balance of precision and performance |

**Reference:** [Fairness in Multi-Tenant Systems (AWS Builders Library)](https://aws.amazon.com/builders-library/fairness-in-multi-tenant-systems/)

### SQS Fair Queues

For message-processing workloads, Amazon SQS Fair Queues automatically reduce noisy-neighbor impact by ensuring fair processing across message groups (which can map to tenants).

## Operational Dashboards

### What to Build

**System-level dashboard (SaaS operator view):**
- Total active tenants
- Onboarding success/failure rate
- System-wide error rate and latency
- Resource utilization (compute, storage, database)
- Top 10 tenants by resource consumption
- Tenants approaching tier limits

**Per-tenant dashboard (for debugging and customer support):**
- Tenant's API usage over time
- Tenant's error rate and latency
- Tenant's storage consumption
- Tenant's current tier and limits
- Recent tenant activity log

**Business metrics dashboard:**
- Cost per tenant trend
- Revenue per tenant vs. cost per tenant (margin)
- Tenant growth rate
- Feature adoption by tier
- Churn indicators (declining usage)

## Common Mistakes

1. **Not including tenant context in logs from day one** — Retrofitting tenant context into logging is tedious. Make it part of your logging middleware from the start.

2. **Monitoring only at the system level** — System-level metrics hide tenant-specific problems. A system-wide p99 latency of 200ms might mask one tenant experiencing 2s latency.

3. **Not setting up noisy neighbor alerts** — If you don't monitor per-tenant consumption, you won't know a noisy neighbor is degrading the system until other tenants complain.

4. **Over-investing in per-tenant metrics for all tenants** — CloudWatch custom metrics at high cardinality are expensive. Use EMF or sampling for basic-tier tenants; detailed metrics for premium tenants.

5. **Ignoring the business metrics** — Technical metrics tell you if the system is healthy. Business metrics (cost per tenant, margin, feature adoption) tell you if the business is healthy. Both matter.

## Discovery Questions for This Domain

When the conversation is about observability, monitoring, or operations:

**Current state:**
- Do your logs include tenant ID today? (If not, this is the single highest-priority fix)
- What monitoring tools are you using? (CloudWatch, Datadog, Grafana, New Relic, custom?)
- Can you currently debug an issue for a specific tenant? (Filter logs, traces, metrics by tenant ID?)

**Noisy neighbor:**
- Have you experienced noisy neighbor issues? (One tenant degrading performance for others)
- Do you have per-tenant usage metrics that would detect a noisy tenant? (Per-tenant API call rate, DB throughput, error rate)
- Do you have alerts that fire when a single tenant's consumption spikes?

**Operational model:**
- Who operates the SaaS platform? (Dedicated SRE/platform team? The same team that builds features?)
- Do you have operational dashboards? (System-level? Per-tenant? Business metrics?)
- How do you handle tenant-specific incidents today? (Can you isolate the impact to one tenant, or does troubleshooting affect everyone?)

**Scale:**
- How many tenants do you need to monitor? (This affects whether per-tenant CloudWatch metrics are cost-effective)
- Do you need tenant-facing usage dashboards? (Showing tenants their own consumption)

## References

- [Capturing and Visualizing Multi-Tenant Metrics Inside a SaaS Application on AWS](https://aws.amazon.com/blogs/apn/capturing-and-visualizing-multi-tenant-metrics-inside-a-saas-application-on-aws/)
- [SaaS Lens — General Design Principles (Instrument, Capture, Analyze)](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/general-design-principles.html)
- [SaaS Lens — How Do You Prevent One Tenant from Adversely Impacting Another?](https://wa.aws.amazon.com/saas.question.PERF_1.en.html)
- [SaaS Lens — How Do You Limit a Tenant's Ability to Impact Availability?](https://wa.aws.amazon.com/saas.question.REL_1.en.html)
- [Fairness in Multi-Tenant Systems (AWS Builders Library)](https://aws.amazon.com/builders-library/fairness-in-multi-tenant-systems/)
