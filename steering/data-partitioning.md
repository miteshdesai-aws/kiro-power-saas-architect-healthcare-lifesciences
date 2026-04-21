# Data Partitioning Strategies

## The Core Decision

Data partitioning is where your tenancy model meets reality. The model you chose (silo, pool, bridge) must be implemented differently for each storage service because each service has different scoping, access control, and isolation mechanisms.

**Key principle from the AWS Multi-Tenant SaaS Storage Strategies whitepaper:** Evaluate data partitioning per service, not globally. Your DynamoDB strategy might be pool while your S3 strategy is silo — and that's fine.

## DynamoDB Strategies

DynamoDB has no concept of a "database instance" — all tables are global to an account within a region. This changes how you think about silo vs. pool.

### Pool Model: Tenant ID in Partition Key

All tenants share one table. Tenant ID is the leading element of the partition key.

**Partition key design:** `TENANT#<tenant-id>` as the partition key, or composite key with tenant ID as the first element.

**Example:**
- PK: `TENANT#abc123`, SK: `ORDER#2024-001`
- PK: `TENANT#abc123`, SK: `PRODUCT#widget-42`

**Isolation enforcement:**
- IAM session policy with `dynamodb:LeadingKeys` condition restricts access to items matching the tenant's partition key prefix
- Even if application code has a bug, IAM prevents cross-tenant reads

**Pros:** Single table to manage, cost-efficient, scales with total load
**Cons:** Hot partition risk if one tenant dominates, harder to do per-tenant backup/restore, right-to-erasure requires scanning all items

**When to use:** High tenant count, uniform access patterns, cost-sensitive.

### Silo Model: Table Per Tenant

Each tenant gets their own DynamoDB table(s). Table names include the tenant ID (e.g., `tenant-abc123-orders`).

**Isolation enforcement:**
- IAM policy restricts the role to only access tables matching the tenant's naming pattern
- Resource-level isolation — no possibility of cross-tenant access

**Pros:** Strongest isolation, per-tenant throughput settings, easy backup/restore per tenant, simple right-to-erasure (delete the table)
**Cons:** Operational overhead (managing N × M tables for N tenants and M table types), harder to query across tenants, table creation adds onboarding latency

**When to use:** Low tenant count, enterprise tier, compliance requirements, tenants with very different throughput needs.

### Bridge Model: Shared Table with Strong Partitioning

Shared table like pool, but with additional isolation mechanisms. Often combined with separate GSIs or separate tables for specific high-sensitivity data.

**Example:** Shared `orders` table (pool) but separate `compliance-docs` table per tenant (silo).

**Reference:** [Partitioning Pooled Multi-Tenant SaaS Data with Amazon DynamoDB](https://aws.amazon.com/blogs/apn/partitioning-pooled-multi-tenant-saas-data-with-amazon-dynamodb/)
**Reference:** [Amazon DynamoDB Data Modeling for Multi-Tenancy — Part 1](https://aws.amazon.com/blogs/database/amazon-dynamodb-data-modeling-for-multi-tenancy-part-1/)

## RDS / Aurora Strategies

Relational databases have a more natural mapping to silo, bridge, and pool because they have the concept of instances, databases, and schemas.

### Pool Model: Shared Schema with Row-Level Security

All tenants share the same database, same schema, same tables. Every table has a `tenant_id` column. Row-level security (RLS) enforces isolation at the database level.

**PostgreSQL RLS implementation:**
1. Create a policy that filters rows by tenant ID
2. Set the tenant context at the beginning of each connection/transaction (e.g., `SET app.current_tenant = 'abc123'`)
3. RLS policy automatically filters all queries to the current tenant

**Pros:** Most cost-efficient, single database to manage, standard SQL
**Cons:** RLS adds query overhead, shared connection pool limits, noisy neighbor on shared instance, complex right-to-erasure, harder to audit isolation

**When to use:** High tenant count, cost-sensitive, PostgreSQL-based stack.

**Reference:** [Multi-Tenant Data Isolation with PostgreSQL Row Level Security](https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/)

### Bridge Model: Database or Schema Per Tenant

Each tenant gets their own database (or schema) on a shared RDS/Aurora instance. Tenants share the compute (instance) but have logically separate data.

**Pros:** Clear data separation, easy per-tenant backup/restore, straightforward right-to-erasure (drop the database/schema), tenants can't see each other's tables
**Cons:** Connection management complexity (must route to correct database), instance limits on number of databases, shared instance means shared compute resources

**When to use:** Mid-market, moderate tenant count (tens to low hundreds), need data isolation without dedicated instances.

### Silo Model: Instance Per Tenant

Each tenant gets their own RDS/Aurora instance (or cluster). Complete infrastructure isolation.

**Pros:** Strongest isolation, per-tenant performance tuning, independent scaling, simplest compliance story
**Cons:** Highest cost (paying for idle instances), operational overhead, slower onboarding (instance provisioning takes minutes)

**When to use:** Enterprise tier, regulated industries, tenants with very different performance requirements.

### Scaling Considerations

**Database sharding:** When pool or bridge models hit the limits of a single instance (connections, storage, IOPS), you may need to shard — distribute tenants across multiple instances. This requires a tenant-to-shard routing layer.

**Tenant migration:** As tenants grow, you may need to move them from a shared instance to a dedicated one. Design your data access layer to support this routing change without application code changes.

**Reference:** [Scale Your Relational Database for SaaS, Part 2: Sharding and Routing](https://aws.amazon.com/blogs/database/scale-your-relational-database-for-saas-part-2-sharding-and-routing/)
**Reference:** [Choose the Right PostgreSQL Data Access Pattern for Your SaaS Application](https://aws.amazon.com/blogs/database/choose-the-right-postgresql-data-access-pattern-for-your-saas-application/)

## S3 Strategies

### Pool Model: Shared Bucket with Prefix Per Tenant

All tenants share one S3 bucket. Objects are organized by tenant prefix: `s3://shared-bucket/tenant-abc123/...`

**Isolation enforcement:**
- IAM session policy with `s3:prefix` condition restricts access to the tenant's prefix
- Or use ABAC with object tags (`TenantId` tag on each object)

**Pros:** Single bucket to manage, simple
**Cons:** Bucket-level settings (versioning, lifecycle) apply to all tenants, harder to do per-tenant access logging

### Silo Model: Bucket Per Tenant

Each tenant gets their own S3 bucket: `s3://tenant-abc123-data/...`

**Isolation enforcement:** IAM policy restricts access to the tenant's bucket. Resource-level isolation.

**Pros:** Strongest isolation, per-tenant bucket policies, per-tenant access logging, easy right-to-erasure (delete the bucket)
**Cons:** AWS account limit on number of buckets (default 100, can be increased), operational overhead

### Bridge Model: Shared Bucket with Strong IAM + Separate Buckets for Sensitive Data

Most data in a shared bucket with prefix isolation. Sensitive data (PII, compliance docs) in per-tenant buckets.

## HealthLake Strategies (FHIR)

AWS HealthLake is a HIPAA-eligible, fully managed FHIR R4 data store. Multi-tenant patterns are covered in detail in `fhir-and-interop.md`. Summary for data partitioning decisions:

### Pool Model: Shared Data Store
All tenants share one HealthLake data store. Tenant context in FHIR resource metadata (tags/extensions). Access scoped via SMART on FHIR + IAM session policies.

**Pros:** Lower cost, simpler operations, easier cross-tenant analytics via zero-ETL to Iceberg
**Cons:** Weaker isolation, shared KMS key, harder tenant offboarding
**Best for:** High tenant count (100+), SMB clinics, cost-sensitive

### Silo Model: Data Store Per Tenant
Each tenant gets their own HealthLake data store with separate FHIR endpoint and KMS key.

**Pros:** Strongest isolation, per-tenant encryption, simple compliance story, easy offboarding
**Cons:** Higher cost (per-data-store pricing), operational overhead, cross-tenant analytics requires aggregation
**Best for:** Enterprise health systems, < 100 tenants, contractual isolation requirements

### Healthcare Default
Data store per tenant (silo) for enterprise health system tenants. Shared data store (pool) acceptable for high-volume SMB clinics with strong SMART on FHIR scoping. Hybrid for mixed customer base.

## HealthImaging Strategies (DICOM)

AWS HealthImaging stores medical imaging data (DICOM P10) at petabyte scale with sub-second retrieval. See `clinical-saas-and-imaging.md` for deep coverage.

### Data Store Per Tenant (Recommended)
Each tenant gets their own HealthImaging data store. This is the natural model — imaging data is large, tenant-specific, and highly sensitive.

**Isolation:** IAM resource policy per data store, KMS CMK per tenant
**Storage tiering:** S3 Intelligent-Tiering for aging studies (hot → warm → cold automatically)
**Backup:** HealthImaging export to S3 for backup/archive

### Shared Data Store (Not Recommended for Healthcare)
Possible but not recommended. Imaging data volumes are too large and too sensitive for pool model to be practical. Per-tenant data stores are the healthcare default.

## OpenSearch Strategies

Amazon OpenSearch Service is used in healthcare SaaS for clinical search (patient lookup, diagnosis search, medication search), log analytics, and operational dashboards.

### Pool Model: Shared Index with Tenant Field
All tenants share one OpenSearch index. Every document includes a `tenant_id` field. Queries always filter by tenant.

**Isolation:** Document-level security (DLS) in OpenSearch restricts query results to the current tenant's documents. IAM session policies scope API access.

**Pros:** Lowest cost, simplest operations
**Cons:** Weaker isolation (relies on DLS), shared cluster resources (noisy neighbor risk), harder to attribute costs per tenant
**Best for:** High tenant count, search over non-PHI data, operational logs

### Bridge Model: Index Per Tenant
Each tenant gets their own index (or index pattern) on a shared OpenSearch domain. Tenant routing via index naming convention.

**Isolation:** IAM policies restrict access to the tenant's index pattern. Separate indices provide data-level separation.

**Pros:** Stronger isolation than shared index, per-tenant index settings, easier tenant offboarding (delete the index)
**Cons:** Index count limits (shard limits per domain), operational overhead at high tenant count
**Best for:** Moderate tenant count, PHI search workloads

### Silo Model: Domain Per Tenant
Each tenant gets their own OpenSearch domain. Strongest isolation but highest cost.

**Best for:** Enterprise tenants with dedicated search requirements, very large data volumes per tenant

### OpenSearch Serverless Collection Groups
OpenSearch Serverless introduced collection groups for multi-tenant workloads — per-tenant encryption with shared compute. This balances cost and isolation effectively.

**Reference:** [Build a Multi-Tenant Healthcare System with Amazon OpenSearch Service](https://aws.amazon.com/blogs/big-data/build-a-multi-tenant-healthcare-system-with-amazon-opensearch-service/)
**Reference:** [Amazon OpenSearch Serverless Collection Groups for Multi-Tenant Workloads](https://aws.amazon.com/blogs/big-data/amazon-opensearch-serverless-introduces-collection-groups-to-optimize-cost-for-multi-tenant-workloads/)

## PHI Encryption Key Strategy

For healthcare SaaS, encryption key management is a data partitioning decision, not just a security decision. See `phi-data-handling.md` for detailed guidance. Summary:

| Storage Service | Healthcare Default Key Strategy |
|----------------|-------------------------------|
| DynamoDB (silo — per-tenant table) | Per-tenant CMK |
| DynamoDB (pool — shared table) | AWS-managed key + IAM isolation (per-tenant CMK not practical for shared table) |
| RDS/Aurora (silo — per-tenant instance) | Per-tenant CMK |
| RDS/Aurora (bridge — per-tenant schema) | Shared instance CMK (encryption is instance-level, not schema-level) |
| S3 | Per-tenant CMK (via bucket policy or object-level KMS key) |
| HealthLake | Per-data-store CMK (aligns with per-tenant data stores) |
| HealthImaging | Per-data-store CMK |
| OpenSearch | Per-domain CMK (silo) or shared domain CMK with DLS (pool/bridge) |

## Backup and Recovery Implications

The tenancy model dramatically affects backup and recovery:

| Model | Backup | Per-Tenant Restore | Right to Erasure |
|-------|--------|-------------------|------------------|
| Silo | Standard (per-resource backup) | Straightforward — restore the tenant's resource | Delete the resource |
| Bridge | Per-database/schema backup | Restore specific database/schema | Drop the database/schema |
| Pool | Full table/database backup | Complex — must extract tenant's data from shared backup | Scan and delete all tenant rows |

**For pool model:** Consider maintaining a separate export mechanism that can extract a single tenant's data. This is needed for both restore scenarios and right-to-erasure compliance.

**Reference:** [Managed Database Backup and Recovery in a Multi-Tenant SaaS Application](https://aws.amazon.com/blogs/database/managed-database-backup-and-recovery-in-a-multi-tenant-saas-application/)

## Data Migration Between Models

When a tenant upgrades from pool to silo (or vice versa), you need to migrate their data.

**Pool → Silo migration:**
1. Create dedicated resources for the tenant
2. Export tenant's data from shared resources (query by tenant ID)
3. Import into dedicated resources
4. Update routing to point to new resources
5. Verify data integrity
6. Delete tenant's data from shared resources

**Design for this from day one:** Use a data access layer that abstracts the storage location. The application code calls `getOrders(tenantId)` — the data access layer knows whether to query the shared table or the tenant's dedicated table based on the tenant's current configuration.

## Performance Considerations

### DynamoDB Hot Partitions
In pool model, if one tenant generates significantly more traffic than others, their partition key prefix can become a hot partition. Mitigations:
- Use write sharding (add a random suffix to partition keys)
- Monitor per-tenant consumed capacity
- Consider moving hot tenants to dedicated tables (silo)

### RDS Connection Exhaustion
In pool model, all tenants share the database connection pool. A tenant with many concurrent users can exhaust connections. Mitigations:
- Use RDS Proxy for connection pooling
- Set per-tenant connection limits at the application level
- Monitor connections per tenant

### S3 Request Rate
S3 has per-prefix request rate limits. In pool model with prefix-per-tenant, a high-traffic tenant can hit these limits. Mitigations:
- Distribute objects across multiple prefixes (add date or hash to prefix)
- Monitor per-prefix request rates

## Discovery Questions for This Domain

When the conversation is about data partitioning or database design:

**Current state:**
- What storage services are you using or planning? (DynamoDB, RDS/Aurora, S3, ElastiCache, OpenSearch?)
- For each service: what data does it hold, and how sensitive is it?
- Do you have an existing schema or data model, or is this greenfield?

**Partitioning decisions:**
- For each storage service: does the data need to be isolated per tenant, or can it be shared? (This often maps to data sensitivity — PII and financial data usually need isolation)
- Do you need per-tenant backup and restore? (If yes, pool model makes this very hard)
- Do you have right-to-erasure requirements (GDPR, CCPA)? (This strongly favors bridge or silo for data stores)
- What's the expected data volume per tenant? (Highly variable volumes favor silo to avoid hot partitions)

**Access patterns:**
- Do you need cross-tenant queries? (Analytics, admin dashboards, reporting — these are harder with silo)
- What's the read/write ratio? (Affects DynamoDB partition key design and RDS connection pooling)
- Are there batch operations that process large amounts of tenant data? (Bulk imports, exports, migrations)

**Scale and performance:**
- How many tenants will share a single database instance? (RDS connection limits, DynamoDB partition throughput)
- Do you expect significant variance in data volume across tenants? (If one tenant has 100x the data of others, pool model creates hot partitions)
- Do you need to support tenant data migration between models? (Pool → silo for tier upgrades)

## References

- [Multi-Tenant SaaS Storage Strategies Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/multi-tenant-saas-storage-strategies/multi-tenant-saas-storage-strategies.html)
- [Multitenancy on DynamoDB](https://docs.aws.amazon.com/whitepapers/latest/multi-tenant-saas-storage-strategies/multitenancy-on-dynamodb.html)
- [Multitenancy on Amazon RDS](https://docs.aws.amazon.com/whitepapers/latest/multi-tenant-saas-storage-strategies/multitenancy-on-rds.html)
- [SaaS Architecture Fundamentals — Data Partitioning](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/data-partitioning.html)
