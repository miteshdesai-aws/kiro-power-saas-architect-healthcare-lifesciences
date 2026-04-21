# Resilience, Deployment & Migration

## Cell-Based Architecture for SaaS

### What Is a Cell?

A cell is an independent, self-contained replica of your system. Each cell serves a subset of tenants. A failure in one cell (bad deployment, resource exhaustion, bug) only affects the tenants in that cell, not the entire system.

Think of it like bulkheads in a ship — if one compartment floods, the others stay dry.

### When to Use Cell-Based Architecture

- **High tenant count** (hundreds to thousands) where a single failure affecting all tenants is unacceptable
- **Blast radius reduction** is a priority — you want to limit the impact of bad deployments
- **Regulatory requirements** for fault isolation
- **You've outgrown simple pool model** and need more resilience without going full silo

**Don't use cells for:**
- Small tenant count (< 50) — silo model is simpler
- Early-stage SaaS — adds complexity before you need it
- When simple multi-AZ deployment provides sufficient resilience

### Cell Design Principles

- Each cell is a complete, independent stack (compute, storage, routing)
- Cells are identical in configuration — no cell-specific customization
- Tenants are assigned to cells during onboarding and don't move unless explicitly migrated
- A thin routing layer (cell router) directs tenant requests to the correct cell
- Deployments are rolled out cell-by-cell (canary across cells)

### Cell Router

The cell router is the only shared component. It maps tenant ID → cell and routes requests accordingly. This must be highly available and low-latency.

**Implementation options:**
- DynamoDB lookup table (tenant_id → cell_endpoint)
- CloudFront + Lambda@Edge for routing at the edge
- Route 53 with weighted routing for cell-level failover

**Reference:** [Guidance for Cell-Based Architecture on AWS](https://aws.amazon.com/solutions/guidance/cell-based-architecture-on-aws/)
**Reference:** [Reducing the Scope of Impact with Cell-Based Architecture](https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/reducing-scope-of-impact-with-cell-based-architecture.html)

## Shuffle Sharding

### What Is Shuffle Sharding?

Instead of assigning each tenant to one cell (simple sharding), assign each tenant to a unique combination of cells. This dramatically reduces the probability that two tenants share the exact same failure domain.

**Example:** With 8 cells and 2-cell assignments per tenant, there are C(8,2) = 28 unique combinations. Two random tenants have only a 1/28 chance of sharing the same cell pair. If one cell fails, only tenants assigned to combinations including that cell are affected — and they still have their second cell as fallback.

**When to use:** Large-scale SaaS where even cell-level blast radius is too broad. This is how AWS itself isolates workloads in services like Route 53.

**Reference:** [Workload Isolation Using Shuffle-Sharding (AWS Builders Library)](https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/)
**Reference:** [Shuffle Sharding: Massive and Magical Fault Isolation](https://aws.amazon.com/blogs/architecture/shuffle-sharding-massive-and-magical-fault-isolation/)

## CI/CD for Multi-Tenant SaaS

### Core Principle: All Tenants Run the Same Version

This is a defining characteristic of SaaS. If different tenants run different versions, you don't have SaaS — you have hosted software. A single pipeline deploys the same code to all tenants.

### Pool Model CI/CD

Simplest model. One environment, one deployment:

1. Code commit → build → test → deploy to shared environment
2. All tenants immediately get the new version
3. Use feature flags for gradual feature rollout if needed

**Risk:** A bad deployment affects all tenants simultaneously.
**Mitigation:** Canary deployments (deploy to a small percentage of traffic first), automated rollback on error rate spike.

### Silo Model CI/CD

More complex. Must deploy to every tenant's environment:

1. Code commit → build → test
2. Deploy to canary tenant(s) first
3. Monitor canary for errors
4. If healthy, deploy to remaining tenants (in waves or all at once)
5. Each tenant environment is updated independently

**Challenge:** Deployment time scales with tenant count. 100 tenants = 100 deployments.
**Solution:** Parallelize deployments. Use AWS CodePipeline with parallel actions, or Step Functions for orchestration.

**For account-per-tenant:** Cross-account deployment pipelines. The deployment pipeline in the tooling account assumes roles in each tenant account to deploy.

**Reference:** [Building a Cross-Account CI/CD Pipeline for Single-Tenant SaaS](https://aws.amazon.com/blogs/devops/cross-account-ci-cd-pipeline-single-tenant-saas/)

### Cell-Based CI/CD

Deploy cell-by-cell:

1. Deploy to Cell 1 (canary cell)
2. Monitor Cell 1 for errors (automated health checks)
3. If healthy, deploy to Cell 2, Cell 3, etc.
4. If unhealthy, stop rollout and rollback Cell 1

This limits blast radius to one cell's tenants during deployment.

### Zero-Downtime Deployment Strategies

- **Blue/Green:** Maintain two identical environments. Deploy to green, switch traffic from blue to green. Instant rollback by switching back.
- **Canary:** Route a small percentage of traffic to the new version. Gradually increase if healthy.
- **Rolling:** Update instances/containers one at a time. Each update is verified before proceeding.

**For Lambda:** Use Lambda aliases with weighted routing for canary deployments. CodeDeploy supports this natively.

**For ECS/EKS:** Use CodeDeploy blue/green deployments with ECS, or Kubernetes rolling updates with EKS.

## Multi-Account Strategy

### When to Use Multiple AWS Accounts

- **Account-per-tenant (silo):** Strongest isolation, clearest cost attribution, but highest operational overhead
- **Environment separation:** Dev, staging, production in separate accounts (standard practice)
- **Control plane vs. application plane:** Control plane in a management account, application plane in tenant accounts
- **Blast radius:** Separate accounts limit the impact of IAM misconfigurations or resource limit exhaustion

### AWS Organizations Structure for SaaS

```
Management Account (root)
├── Security OU
│   ├── Log Archive Account
│   └── Security Tooling Account
├── Infrastructure OU
│   ├── Shared Services Account (control plane)
│   └── CI/CD Tooling Account
├── Tenant OU (for silo model)
│   ├── Tenant-ABC Account
│   ├── Tenant-DEF Account
│   └── ... (one per tenant)
└── Workload OU (for pool model)
    ├── Production Account
    ├── Staging Account
    └── Development Account
```

### Service Control Policies (SCPs)

Apply SCPs at the Tenant OU level to enforce guardrails across all tenant accounts:
- Restrict regions (data residency)
- Prevent disabling CloudTrail or GuardDuty
- Limit which services can be used
- Prevent account-level settings changes

**Reference:** [Managing the Account Lifecycle in Account-Per-Tenant SaaS Environments](https://aws.amazon.com/blogs/mt/managing-the-account-lifecycle-in-account-per-tenant-saas-environments-on-aws/)
**Reference:** [6,000 AWS Accounts, Three People, One Platform: Lessons Learned](https://aws.amazon.com/blogs/architecture/6000-aws-accounts-three-people-one-platform-lessons-learned/)

## Multi-Region Considerations

### When You Need Multi-Region

- **Data residency:** GDPR requires EU customer data to stay in EU regions
- **Latency:** Tenants in different geographies need low-latency access
- **Disaster recovery:** Business continuity requires cross-region failover

### Multi-Region SaaS Patterns

**Pattern 1: Region-per-tenant-group**
- EU tenants in eu-west-1, US tenants in us-east-1
- Each region has a complete deployment (control plane + application plane)
- Tenant routing at DNS level (Route 53 geolocation routing)

**Pattern 2: Active-active multi-region**
- Full deployment in multiple regions
- Tenants can be served from any region
- Data replication between regions (DynamoDB Global Tables, Aurora Global Database)
- Most complex, highest availability

**Pattern 3: Active-passive (DR)**
- Primary region serves all traffic
- Secondary region on standby for failover
- Data replicated asynchronously
- Simpler but with RTO/RPO tradeoffs

**Key consideration:** The control plane should be aware of which region each tenant is assigned to. Tenant routing must account for region assignment.

## Monolith-to-SaaS Migration

### The Journey

Most SaaS products don't start as SaaS. They start as on-premises software or single-tenant hosted applications. The migration to SaaS is a journey, not a big bang.

### Migration Approach

**Phase 1: Lift and Containerize**
- Move the monolith to AWS (EC2 or containers)
- Don't rewrite — just get it running in the cloud
- This is not SaaS yet, but it's the foundation

**Phase 2: Build the Control Plane**
- This is what makes it SaaS
- Implement tenant management, identity, onboarding, billing
- The monolith becomes the "application plane" managed by the control plane
- All tenants are now onboarded, managed, and operated through a single system

**Phase 3: Strangler Fig — Extract Services**
- Incrementally extract microservices from the monolith
- Each extracted service can adopt the appropriate tenancy model
- Use API Gateway to route between monolith and new services
- The monolith shrinks over time as services are extracted

**Phase 4: Optimize Tenancy**
- Now that services are independent, optimize tenancy per service
- Move high-sensitivity services to silo, keep low-sensitivity in pool
- Implement proper isolation, metering, and tiering

### Common Migration Mistakes

1. **Trying to rewrite everything at once** — The strangler fig pattern exists for a reason. Extract incrementally.

2. **Skipping the control plane** — Without a control plane, you just have "hosted software in the cloud." The control plane is what makes it SaaS.

3. **Allowing per-tenant customization** — If tenants have custom code or configurations that differ from the standard product, you can't deploy a single version to all tenants. Eliminate customization before or during migration.

4. **Not planning for tenant data migration** — Existing customers need to be migrated into the new multi-tenant data model. Plan this carefully — it's often the hardest part.

5. **Treating it as a purely technical project** — SaaS migration changes the business model (pricing, support, operations). Involve product and business teams, not just engineering.

**Reference:** [Migrating Applications to SaaS: Rethinking Your Design](https://aws.amazon.com/blogs/apn/migrating-applications-to-saas-rethinking-your-design/)
**Reference:** [SaaS Migrations: Changing the On-Premises Mindset](https://community.aws/content/2jevXepf76ZMAdPRYxk9nkWVPBc/saas-migrations-changing-the-on-premises-mindset)

## Compute Model Selection

### Serverless (Lambda)

**Multi-tenant characteristics:**
- Natural per-invocation isolation (each invocation runs in its own execution environment)
- Scales to zero — no cost when tenants aren't active
- No server management — operational simplicity
- Cold starts can affect latency for infrequent tenants

**SaaS considerations:**
- Pool model: single Lambda function serves all tenants. Tenant context from JWT.
- Silo model: separate Lambda functions per tenant (possible but unusual — more common to use separate accounts)
- Concurrency limits: set reserved concurrency to prevent one tenant from consuming all capacity

**Best for:** Event-driven workloads, variable traffic, small teams, early-stage SaaS.

### Containers (ECS/Fargate)

**Multi-tenant characteristics:**
- Shared task definitions (pool) or per-tenant task definitions (silo)
- Fargate removes server management; EC2 launch type gives more control
- Better for sustained workloads than Lambda (cost-effective at consistent load)

**SaaS considerations:**
- Pool model: shared ECS service, tenant context in request headers
- Silo model: separate ECS service per tenant, or separate task definitions
- Bridge model: shared ECS service with per-tenant storage backends

**Best for:** Sustained workloads, applications with long-running processes, teams with container experience.

### Kubernetes (EKS)

**Multi-tenant characteristics:**
- Namespace-per-tenant isolation (logical isolation within a cluster)
- Network policies for inter-tenant network isolation
- Resource quotas per namespace to prevent noisy neighbor
- Most operational overhead, most flexibility

**SaaS considerations:**
- Pool model: shared namespace, tenant context in requests
- Silo model: namespace per tenant, or cluster per tenant (strongest isolation)
- Requires Kubernetes expertise on the team

**Best for:** Complex applications, teams with Kubernetes experience, need for maximum flexibility.

**Reference:** [SaaS Deployment Architectures with Amazon EKS](https://aws.amazon.com/blogs/containers/saas-deployment-architectures-with-amazon-eks/)
**Reference:** [Building a Multi-Tenant SaaS Solution Using AWS Serverless Services](https://aws.amazon.com/blogs/apn/building-a-multi-tenant-saas-solution-using-aws-serverless-services/)

## Healthcare Deployment Considerations

### CI/CD for FDA-Regulated SaaS
If your clinical SaaS includes FDA-cleared features (SaMD), deployment must satisfy regulatory requirements. See `clinical-saas-and-imaging.md` for detailed FDA/SaMD guidance. Key architectural implications:

- **Validated deployment pipeline:** Every deployment must be traceable (commit → build → test → deploy) with artifacts stored as evidence. Use CodePipeline or Step Functions for orchestration with immutable build artifacts in S3.
- **Change control as code:** Generate change control documentation automatically from commit messages, PR descriptions, test results, and deployment logs. Store in S3 with Object Lock.
- **Rollback requirements:** Every deployment must be rollbackable. Rollback procedure documented and tested. Rollback events documented with reason and impact.
- **Feature flags for regulatory separation:** Deploy code continuously, but activate FDA-regulated features only after validation. Use LaunchDarkly, AWS AppConfig, or custom feature flags to separate deployment from feature activation.

### Validated CI/CD Pipeline Architecture for GxP

A validated pipeline is itself a qualified system (GAMP Category 4). Qualify it once, inherit that qualification for every release within the validated envelope. See `gxp-compliance-generic.md` for the full validation framework.

**Pipeline stages with evidence capture:**

```
Source (Git) → Build → Unit Tests → SAST/DAST → Integration Tests → Regression Tests
    ↓            ↓         ↓            ↓              ↓                  ↓
   [PR       [Immutable  [Test       [Security     [Integration      [Regression
    metadata]  artifact]  results]    scan report]  test report]      test report]
                                                                          ↓
                                                            Clinical Validation Tests
                                                                          ↓
                                                                [Clinical test report]
                                                                          ↓
                                                            Deployment Gate (Approval)
                                                                          ↓
                                                            Deploy to Production (canary)
                                                                          ↓
                                                            Production Monitoring
                                                                          ↓
                                                            [Release Package to S3 Object Lock]
```

**Each stage produces immutable evidence stored in S3 with Object Lock (Compliance Mode):**
- Build artifacts (container images, Lambda packages) signed and versioned
- Test reports with test case → requirement traceability
- Security scan reports (dependency scan, container scan, static analysis)
- Deployment logs (what was deployed, when, by which pipeline execution)
- Approval records (who approved, when, on what basis)

**Pipeline qualification evidence:**
- Pipeline-as-code (CodePipeline/GitHub Actions/Jenkins) version-controlled
- IQ: pipeline deployed matches IaC definition
- OQ: test runs demonstrate each stage functions correctly (test fixtures trigger each gate)
- PQ: pipeline handles production-representative loads and edge cases

### Immutable Release Package

For every production release, assemble an immutable release package in S3 Object Lock:

```
s3://releases-bucket/release-{version}/
├── manifest.json              (version, commit SHA, build timestamp, signed)
├── requirements/              (URS snapshot at release)
├── design/                    (FS, DS snapshot at release)
├── source/                    (git bundle for this release)
├── build-artifacts/           (container images, lambda packages with checksums)
├── test-results/
│   ├── unit-tests.xml
│   ├── integration-tests.xml
│   ├── clinical-tests.xml
│   └── regression-tests.xml
├── security-scans/
│   ├── dependency-scan.json
│   ├── container-scan.json
│   └── sast-report.json
├── traceability-matrix.csv    (requirement → test case → test result)
├── risk-analysis-delta.md     (risks addressed in this release)
├── change-control-records/    (PRs merged in this release with approvals)
├── deployment-logs/
└── approval-record.json       (release manager + QA sign-off)
```

Retention: product lifetime plus regulatory retention period (often 7+ years).

### Change Classification (Aligned with 21 CFR Part 11 / Annex 11)

See `gxp-compliance-generic.md` for the generic change classification table. For cloud-native SaaS deployments:

| Change Type | Classification | Pipeline Gate |
|-------------|---------------|---------------|
| Dependency patch (no functional change) | Standard | Automated tests must pass |
| Bug fix in non-regulated feature | Standard | Peer review + tests |
| Bug fix in regulated feature | Normal | Peer review + regression tests + QA approval |
| New non-regulated feature | Normal | Full pipeline + product approval |
| New regulated feature | Major | Full pipeline + validation protocol execution + CAB approval + activate via feature flag after regulatory validation |
| Infrastructure change affecting regulated system | Major | Full pipeline + IQ re-execution + CAB approval |
| Security hotfix | Emergency | Expedited gate with post-hoc CAB review within 48 hours |
| AI model update (within PCCP scope) | Normal | PCCP-defined validation protocol |
| AI model update (outside PCCP scope) | Major | Full re-validation; may require new FDA submission |

### Feature Flags for Regulatory Separation

Separating deployment from activation is the key pattern for reconciling SaaS velocity with GxP validation:

- All tenants receive code updates via continuous deployment
- Regulated features are flag-protected — off by default
- Feature activation per tenant or globally happens only after:
  - Validation protocols executed and passed
  - Validation Summary Report signed
  - Change control record closed
  - (If applicable) FDA PCCP requirements satisfied

**AWS options:**
- AWS AppConfig — managed feature flag and configuration service, integrates with CloudWatch for rollout monitoring
- Parameter Store with version control — simple, audit-logged via CloudTrail
- LaunchDarkly or similar third-party (requires BAA if flag targeting uses PHI)

### Deployment Approval as an Electronic Signature

For GxP deployments, the release approval is often an electronic signature under 21 CFR Part 11. See `audit-logging-and-access.md` for implementation. Requirements:
- Approver re-authenticates at time of approval (not just a button click in an open session)
- Signature captures: signer identity, timestamp, meaning ("approved for production release"), binding to the release package
- Signature is immutable and linked to the artifact being approved
- Two-signature requirement common (developer + QA, or engineering + regulatory)

### Healthcare DR Requirements
HIPAA §164.308(a)(7) requires a contingency plan including data backup, disaster recovery, and emergency mode operation.

- **RPO/RTO for PHI systems:** Define per-service based on clinical impact. Clinical data stores (HealthLake, RDS with patient records): RPO < 1 hour, RTO < 4 hours. Non-clinical services: standard SaaS RPO/RTO.
- **Multi-region for data residency:** State laws and GDPR Article 9 may require PHI to stay in specific regions. Use region-specific deployments with tenant-to-region routing in the control plane.
- **Backup encryption:** All backups of PHI must be encrypted. RDS automated backups inherit instance encryption. S3 cross-region replication must use KMS keys in the destination region.

## Discovery Questions for This Domain

When the conversation is about resilience, deployment, CI/CD, or migration:

**Deployment:**
- How do you deploy today? (Manual? CI/CD pipeline? What tools?)
- Do all tenants run the same version? (If not, this is a fundamental SaaS issue to address)
- How long does a deployment take? Does it scale with tenant count?
- Do you have canary or staged deployment strategies?
- What's your rollback strategy if a deployment goes wrong?

**Resilience:**
- What's your current blast radius? (If a bad deployment goes out, how many tenants are affected?)
- Are you using multi-AZ? Multi-region?
- Have you considered cell-based architecture? (Relevant at 100+ tenants where blast radius matters)
- What are your RPO/RTO requirements? Do they differ by tier?

**Compute model:**
- What compute platform are you using or considering? (Lambda, ECS/Fargate, EKS, EC2?)
- What drove that choice? (Team experience, workload characteristics, cost, existing investment?)
- For containers: are you using Fargate (serverless) or EC2 launch type? (Affects multi-tenant isolation options)

**Multi-account:**
- Are you using a single AWS account or multiple? (Single is fine for pool, multi-account is common for silo)
- If multi-account: how are you managing cross-account deployments?
- Do you use AWS Organizations with SCPs?

**Migration (if applicable):**
- What's the current state? (On-prem monolith? Single-tenant hosted? Already partially SaaS?)
- Do you have existing customers that need to be migrated into the multi-tenant model?
- Are there per-customer customizations that need to be eliminated before SaaS migration?
- What's the migration timeline and risk tolerance? (Big bang vs. incremental strangler fig?)

**GxP/validated deployment (healthcare SaaS):**
- Is your software SaMD or otherwise GxP-regulated? (If yes, deployment is subject to change control and validation)
- Is your CI/CD pipeline qualified as a GAMP Category 4 system?
- Do you produce an immutable release package per production release?
- Is release approval captured as a 21 CFR Part 11 electronic signature?
- Do you separate deployment from activation using feature flags for regulated features?

## References

- [Guidance for Cell-Based Architecture on AWS](https://aws.amazon.com/solutions/guidance/cell-based-architecture-on-aws/)
- [Reducing the Scope of Impact with Cell-Based Architecture](https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/)
- [Workload Isolation Using Shuffle-Sharding](https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/)
- [Organizing Your AWS Environment Using Multiple Accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/)
- [Patterns for Deploying SaaS in Remote Environments](https://aws.amazon.com/blogs/apn/patterns-for-deploying-saas-in-remote-environments/)
