# SaaS Architecture Artifacts

## Overview

As a SaaS architect, you should produce concrete deliverables — not just advice. When the conversation reaches a decision point or the user asks for a recommendation, offer to generate the relevant artifact as a markdown file in their workspace.

Always ask the user where they want artifacts saved. Default suggestion: `docs/saas-architecture/` in the workspace root.

This file covers the **core SaaS artifacts** — those that apply to any multi-tenant SaaS on AWS. For healthcare-specific artifacts (HIPAA Eligibility Matrix, PHI Data Flow Map, BAA Inventory, Audit Log Coverage Matrix, HITRUST Control Inheritance, De-identification Strategy, Break-the-Glass Runbook), load `artifacts-healthcare.md`.

## Artifact Relationships

Artifacts overlap. When generating multiple artifacts, be aware of how they connect to avoid redundancy:

**Tenant Isolation Matrix ↔ Data Partitioning Map:**
The Isolation Matrix includes a high-level storage column per service. The Data Partitioning Map goes deeper on storage specifics (key design, backup strategy, migration paths). If both are generated, the Data Partitioning Map should reference the Isolation Matrix for the tenancy model decisions and focus on storage implementation details. Don't repeat the same service-level table in both.

**Tenant Isolation Matrix ↔ Tiering Matrix:**
The Isolation Matrix has a "Tier Mapping" section showing which services are pool/bridge/silo per tier. The Tiering Matrix covers the full tier definition (pricing, limits, features, SLA). If both exist, the Tiering Matrix should reference the Isolation Matrix for infrastructure details and focus on the business/product dimensions of tiers.

**Onboarding Flow ↔ Tenant Isolation Matrix:**
The Onboarding Flow's provisioning steps are determined by the tenancy model in the Isolation Matrix. If the Isolation Matrix exists, the Onboarding Flow should reference it and focus on the orchestration sequence, error handling, and per-tier variations.

**Cost Attribution Strategy ↔ Tiering Matrix:**
The Cost Attribution Strategy's metering dimensions should align with the Tiering Matrix's billing dimensions. If both exist, cross-reference them and flag any misalignment (e.g., a billing dimension that isn't being metered).

**SaaS Lens Review Report ↔ Everything:**
The Review Report may reference gaps that other artifacts would fill. For example, a finding of "no documented isolation strategy" naturally leads to generating a Tenant Isolation Matrix. When generating a Review Report, note which artifacts would address which findings in the roadmap section.

**General rule:** When generating an artifact and a related artifact already exists in the workspace, read it first and reference it rather than duplicating content. If the existing artifact is outdated based on the current conversation, offer to update it.


## Artifact 1: Tenant Isolation Matrix

This is the single most important artifact in a SaaS engagement. It maps every service to its tenancy model, isolation mechanism, and data strategy.

Generate this after the tenancy model discussion is complete.

### Template

```markdown
# Tenant Isolation Matrix

Generated: {date}
Product: {product name}

## Service-Level Tenancy Decisions

| Service | Tenancy Model | Compute | Storage | Isolation Mechanism | Rationale |
|---------|--------------|---------|---------|-------------------|-----------|
| {service} | Pool/Bridge/Silo | Lambda/ECS/EKS | DynamoDB/RDS/S3 | {IAM dynamic policies / ABAC / RLS / Resource separation} | {why this model for this service} |

## Isolation Enforcement Details

### {Service Name}
- **Tenancy model:** {Pool/Bridge/Silo}
- **Compute isolation:** {How compute is isolated or shared}
- **Storage isolation:** {How data is partitioned}
- **IAM strategy:** {Static policies / Dynamic STS / ABAC / N/A}
- **Data access pattern:** {How tenant context scopes queries}
- **Cross-tenant access prevention:** {Specific mechanism}
- **Isolation testing approach:** {How isolation is validated}

## Tier Mapping

| Tier | Services in Pool | Services in Bridge | Services in Silo |
|------|-----------------|-------------------|-----------------|
| Basic | {list} | {list} | - |
| Professional | {list} | {list} | {list} |
| Enterprise | {list} | {list} | {list} |

## Open Questions / Risks
- {Any unresolved decisions}
- {Known risks with chosen approach}

## Related Artifacts
<!-- If data-partitioning-map.md exists, reference it for storage implementation details instead of repeating here -->
<!-- If tiering-matrix.md exists, reference it for full tier definitions instead of duplicating tier details -->
```

### When to Generate
- After completing the tenancy model discussion for all services
- When the user asks "what should our isolation look like?"
- When doing an architecture review and documenting current state

## Artifact 2: Architecture Decision Record (ADR)

Generate an ADR for each significant architecture decision. ADRs document the why behind decisions — critical for future team members and for audit trails.

### Template

```markdown
# ADR-{number}: {Title}

**Date:** {date}
**Status:** Proposed | Accepted | Superseded
**Deciders:** {who was involved}

## Context

{What is the situation? What problem are we solving? What constraints exist?}

## Decision

{What did we decide? Be specific.}

## Rationale

{Why this option over alternatives? Reference AWS best practices where applicable.}

## Alternatives Considered

### Option A: {name}
- **Description:** {what it is}
- **Pros:** {advantages}
- **Cons:** {disadvantages}
- **Why rejected:** {specific reason}

### Option B: {name}
- **Description:** {what it is}
- **Pros:** {advantages}
- **Cons:** {disadvantages}
- **Why rejected:** {specific reason}

## Consequences

### Positive
- {benefit 1}
- {benefit 2}

### Negative (accepted trade-offs)
- {trade-off 1}
- {trade-off 2}

### Risks
- {risk 1 and mitigation}
- {risk 2 and mitigation}

## References
- {AWS whitepaper or blog post that supports this decision}

## Related Artifacts
<!-- Reference related ADRs, Isolation Matrix, or other artifacts that informed or are affected by this decision -->
```

### When to Generate
- After any significant tenancy model decision
- After choosing an isolation strategy
- After selecting a data partitioning approach
- After deciding on compute model
- When the user says "let's document this decision"

### Common ADR Topics for SaaS
- Tenancy model selection (per service or overall)
- Tenant isolation strategy
- Identity provider and multi-tenant auth approach
- Data partitioning strategy per storage service
- Control plane vs. application plane boundary
- Tiering and throttling strategy
- Cost attribution approach
- CI/CD and deployment strategy
- Multi-account vs. single-account decision
- Cell-based architecture adoption


## Artifact 3: SaaS Lens Review Report

Generate this after conducting a SaaS Lens review using the saas-lens-review.md steering file.

### Template

```markdown
# SaaS Lens Architecture Review

**Product:** {product name}
**Date:** {date}
**Reviewer:** {name}
**Review scope:** {what was reviewed}

## Executive Summary

{2-3 sentences: overall SaaS maturity level, top risks, key strengths}

**Overall Maturity:** {Early / Developing / Mature}

## Findings Summary

| # | Pillar | Finding | Risk Level | Effort | Priority |
|---|--------|---------|------------|--------|----------|
| 1 | {pillar} | {one-line finding} | Critical/High/Medium/Low | Low/Medium/High | P1/P2/P3/P4 |

## Detailed Findings

### Finding {number}: {title}

**Pillar:** {Operational Excellence / Security / Reliability / Performance / Cost Optimization}
**Risk Level:** {Critical / High / Medium / Low}

**Current State:**
{What was observed}

**Risk:**
{What could go wrong if not addressed}

**Recommendation:**
{What to do, with specific AWS guidance}

**Effort:** {Low / Medium / High}
**Priority:** {P1 / P2 / P3 / P4}

**Reference:** {Link to relevant AWS documentation}

## Quick Wins

{3-5 low-effort, high-impact improvements}

1. {Quick win with expected impact}
2. {Quick win with expected impact}
3. {Quick win with expected impact}

## Recommended Roadmap

### Phase 1: Critical (Weeks 1-4)
- {Finding X: action}
- {Finding Y: action}

### Phase 2: High Priority (Weeks 5-12)
- {Finding X: action}
- {Finding Y: action}

### Phase 3: Improvements (Weeks 13+)
- {Finding X: action}
- {Finding Y: action}

## Strengths Observed
- {What the team is doing well}
- {Existing good practices to maintain}
```

### When to Generate
- After completing a SaaS Lens review walkthrough
- When the user asks for an architecture assessment
- When the user says "review our architecture"

## Artifact 4: Onboarding Flow

Generate a tenant onboarding sequence showing every step from signup to first use.

### Template

```markdown
# Tenant Onboarding Flow

**Product:** {product name}
**Date:** {date}

## Onboarding Sequence

### Trigger
{How onboarding starts: API call, self-service signup, admin action}

### Steps

| Step | Action | Service | Failure Handling |
|------|--------|---------|-----------------|
| 1 | Create tenant record | Control Plane | Rollback: delete record |
| 2 | Provision identity | {Cognito / custom} | Rollback: delete user pool/user |
| 3 | Provision storage | {DynamoDB / RDS / S3} | Rollback: delete resources |
| 4 | Configure routing | {API GW / Route 53} | Rollback: remove route |
| 5 | Set up billing | {Billing service} | Rollback: remove billing profile |
| 6 | Apply tier limits | {API GW usage plan} | Rollback: remove usage plan |
| 7 | Send activation | {SES / SNS} | Retry |

### Orchestration
- **Engine:** {Step Functions / custom}
- **Estimated duration:** {seconds/minutes}
- **Retry strategy:** {per-step retry with exponential backoff}
- **Compensation:** {rollback steps on failure}

### Mermaid Diagram

{Generate a mermaid sequence diagram showing the flow}

## Per-Tier Variations

| Step | Basic (Pool) | Professional (Bridge) | Enterprise (Silo) |
|------|-------------|----------------------|-------------------|
| Provision storage | Skip (shared) | Create schema/table | Create instance/account |
| Configure routing | Skip (shared) | Update routing table | Create dedicated endpoint |
| Estimated time | ~5 seconds | ~30 seconds | ~5 minutes |
```

### When to Generate
- After discussing onboarding requirements
- When the user asks "how should onboarding work?"
- After tenancy model and identity decisions are made


## Artifact 5: Data Partitioning Map

Generate a per-service breakdown of storage decisions.

### Template

```markdown
# Data Partitioning Map

**Product:** {product name}
**Date:** {date}

## Storage Decisions by Service

### {Service Name}

| Aspect | Decision |
|--------|----------|
| Storage service | {DynamoDB / RDS Aurora PostgreSQL / S3} |
| Partitioning model | {Pool / Bridge / Silo} |
| Tenant key design | {e.g., PK: TENANT#id, SK: ORDER#id} |
| Isolation mechanism | {IAM LeadingKeys / RLS / Separate tables / Separate schemas} |
| Backup strategy | {Per-tenant / Shared / Point-in-time} |
| Right-to-erasure | {Delete by PK scan / Drop schema / Delete table} |
| Scaling approach | {Auto-scaling / Sharding / Read replicas} |
| Estimated data volume | {per tenant and total} |

## Cross-Service Data Flow

{How data moves between services, with tenant context preserved}

## Migration Path

{How a tenant's data moves if they change tiers}
- Pool → Bridge: {steps}
- Bridge → Silo: {steps}

## Related Artifacts
<!-- If tenant-isolation-matrix.md exists, reference it for tenancy model decisions instead of repeating here -->
<!-- If onboarding-flow.md exists, reference it for provisioning steps that create these storage resources -->
```

### When to Generate
- After data partitioning discussion is complete
- When the user asks about database design for multi-tenant

## Artifact 6: Tiering Matrix

### Template

```markdown
# Tiering Matrix

**Product:** {product name}
**Date:** {date}

## Tier Definitions

| Dimension | {Tier 1 name} | {Tier 2 name} | {Tier 3 name} |
|-----------|--------------|--------------|--------------|
| **Price** | {price} | {price} | {price} |
| **Tenancy model** | Pool | Pool/Bridge | Silo |
| **API rate limit** | {X req/min} | {X req/min} | {X req/min or custom} |
| **API burst limit** | {X} | {X} | {X} |
| **Storage quota** | {X GB} | {X GB} | {Unlimited} |
| **Users/seats** | {X} | {X} | {Unlimited} |
| **Data retention** | {X days} | {X days} | {Custom} |
| **Support** | {Community} | {Email} | {Dedicated} |
| **SLA** | {X%} | {X%} | {X%} |
| **Features** | {list} | {list} | {list} |

## Throttling Enforcement

| Tier | API Gateway Usage Plan | Application-Level Limits | Database Limits |
|------|----------------------|-------------------------|----------------|
| {tier} | {rate/burst/quota} | {specific limits} | {connection/throughput limits} |

## Upgrade Path
- {Tier 1} → {Tier 2}: {what changes, any migration needed}
- {Tier 2} → {Tier 3}: {what changes, any migration needed}

## Billing Dimensions (if Marketplace)
| Dimension | Unit | Included in Base | Overage Rate |
|-----------|------|-----------------|-------------|
| {dimension} | {unit} | {amount per tier} | {rate} |
```

### When to Generate
- After tiering discussion
- When the user asks about pricing strategy mapping to infrastructure

## Artifact 7: Cost Attribution Strategy

### Template

```markdown
# Cost Attribution Strategy

**Product:** {product name}
**Date:** {date}

## Attribution by Service

| Service | Tenancy Model | Attribution Method | Metrics Captured | Attribution Accuracy |
|---------|--------------|-------------------|-----------------|---------------------|
| {service} | Pool | Usage-based apportionment | API calls, compute duration | Approximate |
| {service} | Silo | Resource tagging | AWS Cost Explorer | Exact |
| {service} | Bridge | Hybrid (tags + usage) | Storage: exact, Compute: approximate | Mixed |

## Metering Pipeline

### Captured Metrics
| Metric | Source | Granularity | Storage |
|--------|--------|-------------|---------|
| {metric} | {where captured} | {per-request / hourly / daily} | {DynamoDB / Timestream / CloudWatch} |

### Pipeline Architecture
{Description of metering flow: capture → transport → aggregate → store → consume}

## Cost Allocation Tags
| Tag Key | Applied To | Purpose |
|---------|-----------|---------|
| TenantId | {resources} | Per-tenant cost tracking |
| TenantTier | {resources} | Tier-level cost analysis |
| Service | {resources} | Per-service cost breakdown |

## Reporting
- **Per-tenant cost report:** {frequency, tool}
- **Margin analysis:** {how revenue vs. cost is compared}
- **Anomaly detection:** {how cost spikes per tenant are detected}

## Related Artifacts
<!-- If tiering-matrix.md exists, verify metering dimensions align with billing dimensions defined there -->
<!-- If tenant-isolation-matrix.md exists, reference it for which services are pool/bridge/silo -->
```

### When to Generate
- After cost attribution discussion
- When the user asks about measuring cost per tenant


## Artifact Readiness: Progressive Discovery

These rules apply to **all** artifacts — SaaS and healthcare. The healthcare artifacts in `artifacts-healthcare.md` inherit these rules.

**Do NOT ask all questions upfront like a form.** Instead, use progressive discovery:

1. Have a natural conversation. Learn about the user's context incrementally.
2. When you offer an artifact (or the user requests one), mentally check: "Do I have enough to produce this completely?"
3. If information is missing, ask only the specific gaps — not a full questionnaire. Frame it naturally: "Before I generate the isolation matrix, I need to know a couple more things about your order service..."
4. Then generate the artifact with no gaps or placeholders.

### Required Information Per SaaS Artifact (Check Before Generating)

**Tenant Isolation Matrix** requires:
- List of services/microservices in the system
- Tenancy model decision per service (or enough context to recommend one)
- Storage service per service (DynamoDB, RDS, S3, etc.)
- Compliance/regulatory requirements
- Tier definitions (if tiering exists)

**ADR** requires:
- The specific decision being documented
- Context and constraints that led to the decision
- At least one alternative that was considered
- The rationale for choosing this option

**SaaS Lens Review Report** requires:
- Walkthrough of each pillar's questions (use saas-lens-review.md)
- Current state observations for each question
- This is gathered during the review conversation itself

**Onboarding Flow** requires:
- Tenancy model (determines what gets provisioned)
- Identity approach (Cognito pattern, SSO requirements)
- What resources need provisioning per tier
- Billing/metering integration points

**Data Partitioning Map** requires:
- List of services and their storage services
- Tenancy model per service
- Key design decisions (partition keys, schema strategy)
- Backup and right-to-erasure requirements

**Tiering Matrix** requires:
- Tier names and target customer segments
- Features per tier
- Throttling limits per tier
- Tenancy model per tier
- Pricing (even rough ranges)

**Cost Attribution Strategy** requires:
- Which services are pool vs. silo vs. bridge
- What usage metrics matter for billing
- Whether AWS Marketplace integration is planned

For healthcare-specific artifact readiness requirements (HIPAA Eligibility Matrix, PHI Data Flow Map, BAA Inventory, etc.), see `artifacts-healthcare.md`.

### Hard Rules for Artifact Quality (Apply to All Artifacts)

**HARD RULE: Do not generate an artifact if you lack the required information.**

If you check readiness and find gaps:
1. Tell the user what's missing and why it matters: "I can't generate a useful isolation matrix yet — I don't know what storage each service uses, and that determines the isolation mechanism."
2. Ask the specific missing questions.
3. Only generate after the gaps are filled.

**Never do any of these:**
- Fill gaps with generic content ("TBD", "to be determined", "depends on requirements")
- Guess at answers the user hasn't provided
- Generate a partial artifact and say "we can fill in the rest later"
- Use boilerplate from the templates without customizing to this specific customer

**If the user insists on generating before you have enough info**, explain what will be incomplete and mark those sections explicitly as "NEEDS INPUT: {what's missing and why}" so the document is honest about its gaps rather than silently generic.

**The quality bar: every artifact should be specific enough that a developer on the customer's team can read it and take action without needing to ask "but what does this mean for our system?"**

## Agent Behavior for Artifacts

### When to Offer Artifacts

- After a significant decision is made in conversation, say: "Want me to generate an ADR for this decision?"
- After completing a domain discussion, say: "I can produce a {Tenant Isolation Matrix / Data Partitioning Map / etc.} documenting what we discussed. Want me to create that?"
- After a SaaS Lens review, say: "Let me generate the review report with findings and recommendations."
- When the user asks "what should I document?" — suggest the relevant artifacts.

### How to Generate

1. Ask the user where to save: "Where should I save architecture artifacts? I'd suggest `docs/saas-architecture/`"
2. Generate the file using the template, filled with the specific decisions from the conversation
3. Use concrete details from the discussion — never leave template placeholders unfilled
4. After generating, summarize what was created and suggest next steps

### Naming Convention

- ADRs: `docs/saas-architecture/adr/ADR-001-{title}.md`
- Isolation Matrix: `docs/saas-architecture/tenant-isolation-matrix.md`
- Review Report: `docs/saas-architecture/saas-lens-review-{date}.md`
- Onboarding Flow: `docs/saas-architecture/onboarding-flow.md`
- Data Partitioning: `docs/saas-architecture/data-partitioning-map.md`
- Tiering Matrix: `docs/saas-architecture/tiering-matrix.md`
- Cost Attribution: `docs/saas-architecture/cost-attribution-strategy.md`

Healthcare artifact naming conventions are defined in `artifacts-healthcare.md`.
