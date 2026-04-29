# Cost Attribution, Billing & Metering

## Why Cost-Per-Tenant Matters

The SaaS Lens Cost Optimization pillar emphasizes "expenditure awareness" — knowing how much each tenant costs you to serve. Without this, you can't:
- Price your tiers profitably (you might be losing money on your biggest tenant)
- Identify tenants whose usage doesn't match their tier
- Make informed decisions about when to move tenants between pool and silo
- Optimize infrastructure spending based on actual tenant consumption patterns

Cost attribution is a business capability, not just a technical exercise.

## Cost Attribution by Tenancy Model

### Silo Model: Straightforward

Each tenant has dedicated resources. Tag all resources with the tenant ID. Use AWS Cost Explorer, Cost and Usage Reports, or AWS Budgets to see per-tenant costs.

**Implementation:**
- Tag all tenant resources: `TenantId: abc123`
- Use AWS Cost Allocation Tags to enable cost tracking
- Create Cost Explorer reports filtered by tenant tag
- Set up AWS Budgets alerts per tenant if needed

**This is the simplest model.** The cost of serving Tenant A = the cost of Tenant A's tagged resources.

### Pool Model: Complex

All tenants share resources. You can't use resource tags because the resources aren't tenant-specific. Instead, you need to:

1. **Instrument your application** to capture per-tenant usage metrics
2. **Aggregate usage data** into a metering pipeline
3. **Apportion shared infrastructure costs** based on tenant consumption

**The approach:**
- Inject tenant context into every operation (API calls, database queries, compute time)
- Capture tenant-level metrics: API call count, data transfer, storage consumed, compute time
- Correlate tenant metrics with infrastructure costs to calculate cost-per-tenant

**Example:** If your shared Lambda function costs $1,000/month and Tenant A made 30% of the invocations, Tenant A's attributed compute cost is ~$300.

**This is approximate, not exact.** Pool model cost attribution is always an estimation. The goal is "directionally correct" — enough to inform pricing and identify outliers.

**Reference:** [SaaS Cost Attribution: How to Align Technology with Business](https://aws.amazon.com/blogs/apn/saas-cost-attribution-how-to-align-technology-with-business/)
**Reference:** [Optimizing Cost Per Tenant Visibility in SaaS Solutions](https://aws.amazon.com/blogs/apn/optimizing-cost-per-tenant-visibility-in-saas-solutions/)

### Bridge Model: Mixed

Dedicated resources (storage) are attributed directly via tags. Shared resources (compute) are attributed via usage metrics, same as pool model.

## Metering Architecture

Metering captures tenant usage data for billing and cost attribution.

### What to Meter

Depends on your pricing model, but common dimensions:
- **API calls**: Number of requests per tenant (by endpoint or total)
- **Compute time**: Lambda duration, ECS task time consumed per tenant
- **Storage**: Data stored per tenant (DynamoDB consumed storage, S3 object size)
- **Data transfer**: Bytes transferred per tenant
- **Features**: Usage of specific features (reports generated, users created, integrations configured)
- **Seats/Users**: Number of active users per tenant

**Healthcare-specific metering dimensions:**
- FHIR API calls per tenant (HealthLake read/write operations)
- Imaging studies stored and retrieved per tenant (HealthImaging)
- Clinical encounters processed per tenant
- Patient records accessed per tenant
- AI inferences per tenant (Bedrock invocations for clinical AI)
- De-identification operations per tenant
- EDI transactions processed per tenant (for payer SaaS)

**Healthcare pricing note:** HealthLake and HealthImaging have their own pricing models (per-request + storage). Factor these into per-tenant cost attribution. For pool model HealthLake (shared data store), apportion costs by tenant's resource count or request volume.

### Metering Pipeline Pattern

```
Application → Metering Events → Aggregation → Storage → Billing/Dashboard
```

**Detailed flow:**
1. **Capture**: Application emits metering events with tenant context (tenant ID, dimension, quantity, timestamp)
2. **Transport**: Events sent to Kinesis Data Streams or SQS for buffering
3. **Aggregate**: Lambda consumer aggregates events per tenant per time window (hourly, daily)
4. **Store**: Aggregated metrics stored in DynamoDB or Timestream
5. **Consume**: Billing service reads aggregated metrics to generate invoices; dashboards display usage

**Key design decisions:**
- **Real-time vs. batch**: Real-time metering (Kinesis) for usage-based billing; batch (S3 + Athena) for cost attribution analytics
- **Granularity**: Meter at the finest grain you might need for billing. You can always aggregate up, but you can't disaggregate.
- **Idempotency**: Metering events may be delivered more than once. Design aggregation to handle duplicates.

### CloudWatch Custom Metrics for Tenant Usage

For simpler metering needs, publish custom CloudWatch metrics with tenant ID as a dimension:

```
Namespace: SaaS/TenantMetrics
MetricName: ApiCallCount
Dimensions: TenantId=abc123, Endpoint=/api/orders
Value: 1
```

**Pros:** Simple, integrates with CloudWatch dashboards and alarms
**Cons:** CloudWatch custom metrics have cost implications at high cardinality (many tenants × many dimensions), not suitable for billing-grade metering

**Use CloudWatch metrics for operational visibility. Use a dedicated metering pipeline for billing.**

## Tiering and Throttling

### Defining Tiers

Tiers map business value to infrastructure and feature access:

| Dimension | Basic | Professional | Enterprise |
|-----------|-------|-------------|------------|
| API rate limit | 100 req/min | 1,000 req/min | 10,000 req/min or custom |
| Storage | 1 GB | 10 GB | Unlimited |
| Users | 5 | 50 | Unlimited |
| Features | Core only | Core + advanced | All + custom integrations |
| Support | Community | Email | Dedicated |
| Tenancy model | Pool | Pool or Bridge | Silo |

### Enforcing Throttling with API Gateway

API Gateway Usage Plans are the primary mechanism for per-tenant throttling:

1. Create a usage plan per tier (Basic, Pro, Enterprise)
2. Set rate limits and burst limits per plan
3. Create an API key per tenant
4. Associate each tenant's API key with their tier's usage plan
5. API Gateway automatically enforces the limits

**For more granular control:** Use Lambda authorizer to check tenant tier and apply custom throttling logic (e.g., different limits per endpoint based on tier).

**Reference:** [Enabling Tiering and Throttling in a Multi-Tenant Amazon EKS SaaS Solution Using Amazon API Gateway](https://aws.amazon.com/blogs/apn/enabling-tiering-and-throttling-in-a-multi-tenant-amazon-eks-saas-solution-using-amazon-api-gateway/)
**Reference:** [Throttling a Tiered, Multi-Tenant REST API at Scale Using API Gateway: Part 1](https://aws.amazon.com/blogs/architecture/throttling-a-tiered-multi-tenant-rest-api-at-scale-using-api-gateway-part-1/)

## AWS Marketplace Integration

If you plan to sell your SaaS through AWS Marketplace, the architecture needs to support Marketplace billing integration.

### SaaS Listing Types

**SaaS Subscriptions:** Customer subscribes and pays a recurring fee. You report usage via `BatchMeterUsage` API for consumption-based dimensions.

**SaaS Contracts:** Customer commits to an upfront contract (annual, multi-year). Can include usage-based overages.

### Pricing Models on Marketplace

- **Tiered contracts**: Customer selects one tier from predefined options
- **Feature-based contracts**: Customer purchases specific features
- **Usage-based (consumption)**: Customer pays based on actual usage, reported via `BatchMeterUsage`
- **Hybrid**: Base contract + usage-based overages

You can define up to 24 pricing dimensions (e.g., API calls, storage GB, users).

### Integration Architecture

1. Customer subscribes via AWS Marketplace
2. Marketplace sends SNS notification to your SaaS
3. Your onboarding service creates the tenant (using the Marketplace customer ID)
4. Your metering service periodically calls `BatchMeterUsage` to report consumption
5. AWS Marketplace handles billing and payment collection
6. You receive payouts from Marketplace

**Key consideration:** Your metering pipeline must be able to report usage in the dimensions you defined in your Marketplace listing. Design metering and Marketplace integration together, not separately.

**Reference:** [Step-by-Step Guide to SaaS Integration with AWS Marketplace](https://aws.amazon.com/blogs/awsmarketplace/step-by-step-guide-to-saas-integration-with-aws-marketplace/)
**Reference:** [Implementing SaaS Contract Pricing Models in AWS Marketplace](https://aws.amazon.com/blogs/awsmarketplace/implementing-saas-contract-pricing-models-in-aws-marketplace/)

## Common Mistakes

1. **Not metering from day one** — Retrofitting metering into an existing application is painful. Instrument tenant usage from the start, even if you don't use it for billing yet.

2. **Metering at the wrong granularity** — If you meter total API calls but later want to bill per-endpoint, you can't disaggregate. Meter at the finest grain you might need.

3. **Ignoring cost attribution in pool model** — "We'll figure out costs later" leads to pricing that doesn't reflect actual costs. You might be subsidizing your most expensive tenants.

4. **Over-engineering tiers early** — Start with 2 tiers (basic + premium). Add more when you have data on how tenants actually use the system.

5. **Not connecting metering to throttling** — If you meter usage but don't enforce limits, tenants have no incentive to stay within their tier. Metering and throttling should be two sides of the same coin.

## Discovery Questions for This Domain

When the conversation is about cost attribution, billing, or metering:

**Pricing model:**
- How do you charge customers today? (Flat subscription? Usage-based? Per-seat? Hybrid?)
- Do you have defined tiers, or is pricing custom per customer?
- Are you planning to sell through AWS Marketplace? (This significantly affects metering architecture)

**Cost visibility:**
- Can you currently measure the cost of serving each tenant? (Most teams can't — that's normal, but it's a gap to close)
- For pool/bridge services: do you have any per-tenant usage metrics today? (API call counts, storage consumed, compute time)
- For silo services: are resources tagged with tenant ID for cost allocation?

**Metering:**
- What usage dimensions matter for your billing? (API calls, storage, users, features, compute time?)
- Do you need real-time usage visibility (for usage-based billing) or is daily/weekly aggregation sufficient?
- Do you need to enforce usage limits (quotas) or just track usage for billing?

**Throttling:**
- Do you have per-tenant or per-tier rate limits today?
- How are limits enforced? (API Gateway usage plans? Application-level? Not at all?)
- What should happen when a tenant exceeds their limits? (Hard block? Overage charges? Notification only?)

**Healthcare metering (if applicable):**
- Which healthcare-specific dimensions do you need to meter per tenant? (FHIR API calls to HealthLake, imaging studies stored/retrieved in HealthImaging, AI inferences through Bedrock for clinical features, EDI transactions for payer SaaS, de-identification operations)
- For HealthLake shared data store (pool model): how do you apportion storage and request costs across tenants? (By resource count, by request volume, by tenant tag)
- For clinical AI costs: are you using Bedrock Application Inference Profiles per tenant or per tier?
- Do any tenants require detailed cost transparency reports? (Enterprise health systems often ask for per-service cost breakdowns as part of the BAA or vendor review)

## References

- [SaaS Lens — Expenditure Awareness](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/expenditure-awareness.html)
- [Calculating Tenant Costs in SaaS Environments](https://aws.amazon.com/blogs/apn/calculating-tenant-costs-in-saas-environments/)
- [Optimizing SaaS Tenant Workflows and Costs](https://aws.amazon.com/blogs/apn/optimizing-saas-tenant-workflows-and-costs/)
- [Optimizing the Cost of Your SaaS Environment with the SaaS Lens](https://aws.amazon.com/blogs/apn/optimizing-the-cost-of-your-saas-environment-with-the-aws-well-architected-saas-lens/)
- [AWS Marketplace SaaS Listing Guide](https://aws.amazon.com/startups/learn/aws-marketplace-listing-a-step-by-step-guide-for-startups/)
