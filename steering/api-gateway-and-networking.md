# API Gateway & Networking Patterns

## Why This Matters

API Gateway and networking are the front door of your SaaS. Every tenant request flows through them. The patterns you choose here affect tenant routing, isolation enforcement, throttling, custom branding (custom domains), and how enterprise tenants connect privately. These decisions are often made implicitly and regretted later.

## API Gateway Multi-Tenant Patterns

### Pattern 1: Single Shared API (Pool)

One API Gateway REST API or HTTP API serves all tenants. Tenant context is extracted from the JWT by a Lambda authorizer or Cognito authorizer.

**How it works:**
- All tenants hit the same API endpoint (e.g., `api.yoursaas.com`)
- Authorizer extracts tenant ID from JWT claims
- Backend services receive tenant context and scope operations accordingly
- Usage plans + API keys enforce per-tenant throttling

**Pros:**
- Simplest to manage — one API, one deployment
- Single WAF configuration, single CloudFront distribution
- Easy to add new endpoints (all tenants get them immediately)
- Lowest cost

**Cons:**
- All tenants share API Gateway throttling limits (account-level)
- Custom domain per tenant requires CloudFront + routing logic
- A misconfigured authorizer affects all tenants
- Harder to give enterprise tenants dedicated capacity

**Best for:** Pool model, high tenant count, uniform API access patterns.

### Pattern 2: API Per Tenant (Silo)

Each tenant gets their own API Gateway deployment. Typically used with account-per-tenant or when tenants need completely independent API configurations.

**How it works:**
- Each tenant has a dedicated API: `tenant-abc.api.yoursaas.com`
- Tenant routing at DNS level (Route 53 or CloudFront)
- Each API has its own authorizer, stages, and throttling configuration
- Can be in the same account or separate accounts

**Pros:**
- Strongest isolation — no shared API infrastructure
- Per-tenant API configuration (custom rate limits, custom stages)
- Independent scaling and throttling
- Clearest blast radius — a bad deployment to one API doesn't affect others

**Cons:**
- Operational overhead scales with tenant count
- API Gateway has account-level limits on number of APIs
- Deployment must target each API individually
- Higher cost (each API has its own CloudWatch logs, WAF association, etc.)

**Best for:** Silo model, enterprise tier, account-per-tenant, tenants with custom API requirements.

### Pattern 3: Shared API with Per-Tenant Stages or Resource Policies (Bridge)

One API Gateway with tenant-specific configurations layered on top. Uses stages, resource policies, or Lambda authorizer logic to differentiate tenant treatment.

**How it works:**
- Single API definition, but enterprise tenants may get dedicated stages
- Lambda authorizer applies tier-specific logic (throttling, feature gating)
- Resource policies can restrict access by source VPC or IP for enterprise tenants
- Usage plans differentiate throttling by tier

**Best for:** Bridge model, mixed SMB + enterprise, when you need some per-tenant customization without full API-per-tenant overhead.

## Tenant Routing Strategies

### Subdomain-Based Routing

Each tenant gets a subdomain: `tenant-abc.yoursaas.com`

**Implementation:**
- Wildcard DNS record: `*.yoursaas.com → CloudFront distribution`
- CloudFront + Lambda@Edge extracts tenant from subdomain
- Routes to the correct backend (shared API for pool, dedicated API for silo)
- ACM wildcard certificate covers all subdomains

**Pros:** Clean tenant branding, easy to identify tenant from URL, works with browser cookie scoping
**Cons:** Wildcard certificate management, DNS propagation for new tenants, more complex routing logic

### Path-Based Routing

Tenant context in the URL path: `api.yoursaas.com/v1/tenants/{tenant-id}/orders`

**Implementation:**
- Single domain, single API
- Tenant ID extracted from path parameter
- Authorizer validates that the JWT tenant claim matches the path tenant ID

**Pros:** Simplest DNS setup, no per-tenant DNS configuration
**Cons:** Tenant ID visible in URL (may be undesirable), path-based routing can be awkward for some API designs

### Header-Based Routing

Tenant context in a custom header: `X-Tenant-Id: abc123`

**Implementation:**
- Single domain, single API
- Tenant ID in request header (set by client SDK or frontend)
- Authorizer validates header matches JWT tenant claim
- API Gateway can route to different integrations based on header (with HTTP API + VPC Link)

**Pros:** Clean URLs, flexible routing
**Cons:** Requires client to set header correctly, header can be spoofed (must validate against JWT)

### Custom Domain Per Tenant

Enterprise tenants bring their own domain: `api.enterprise-customer.com → your SaaS`

**Implementation:**
- Tenant provides their domain and creates a CNAME to your CloudFront distribution
- CloudFront with SNI routes based on the Host header
- Lambda@Edge maps the custom domain to a tenant ID
- ACM certificate per custom domain (or use CloudFront's default certificate with the tenant's own cert)

**Pros:** White-label capability, enterprise branding
**Cons:** Certificate management per tenant, DNS dependency on tenant, more complex onboarding

## VPC Design for Multi-Tenant

### Shared VPC (Pool)

All tenants share a single VPC. Tenant isolation is at the application and IAM level, not the network level.

**Design:**
- Public subnets: ALB, NAT Gateway
- Private subnets: ECS/EKS tasks, Lambda (if VPC-attached), RDS
- Security groups scope access by service, not by tenant
- No network-level tenant isolation

**When to use:** Pool model, serverless-heavy architectures (Lambda doesn't need VPC for most use cases), cost-sensitive.

### VPC Per Tenant (Silo)

Each tenant gets their own VPC. Strongest network isolation.

**Design:**
- Separate VPC per tenant (or per tenant account)
- Transit Gateway connects tenant VPCs to shared services (control plane, monitoring)
- Each VPC has its own security groups, NACLs, and route tables
- VPC peering or PrivateLink for cross-VPC communication

**When to use:** Enterprise tier, regulated industries, account-per-tenant model.

**Scaling consideration:** VPC peering has a limit of 125 peering connections per VPC. Transit Gateway scales better for many tenant VPCs.

### Shared VPC with Namespace Isolation (Bridge)

Single VPC, but with network-level separation using security groups, NACLs, or EKS network policies.

**Design:**
- Shared VPC with subnet groups per tier or per tenant group
- Security groups restrict traffic between tenant workloads
- EKS: network policies enforce pod-to-pod isolation per namespace
- ECS: security groups on Fargate tasks scope network access

**When to use:** Bridge model, EKS-based architectures, when you need some network isolation without full VPC-per-tenant overhead.

## PrivateLink for Enterprise Tenants

Enterprise tenants often require private connectivity — their traffic should not traverse the public internet.

### Pattern: VPC Endpoint Service

1. Create a Network Load Balancer (NLB) in front of your SaaS backend
2. Create a VPC Endpoint Service pointing to the NLB
3. Enterprise tenant creates a VPC Interface Endpoint in their VPC pointing to your endpoint service
4. Traffic flows privately over AWS's network, never touching the internet

**When to offer:** Enterprise tier, regulated industries, tenants with "no public internet" policies, healthcare/finance customers.

**Consideration:** Each VPC Endpoint Service can support multiple consumer VPCs. You can allowlist specific AWS accounts to connect.

## WAF Multi-Tenant Considerations

### Shared WAF (Pool)

Single AWS WAF WebACL attached to your shared API Gateway or CloudFront distribution.

**Considerations:**
- WAF rules apply to all tenants equally
- Rate-based rules are global (not per-tenant) unless you use custom logic
- IP reputation and bot control protect all tenants

**Per-tenant rate limiting with WAF:**
WAF rate-based rules can scope by a custom key (e.g., a header value). If tenant ID is in a header, you can create rate-based rules that limit per tenant. However, this has limits on the number of custom keys.

For fine-grained per-tenant rate limiting, API Gateway usage plans are more appropriate than WAF.

### Per-Tenant WAF Rules (Silo)

Enterprise tenants may need custom WAF rules (IP allowlisting, geo-blocking, custom rate limits).

**Implementation:**
- Separate WAF WebACL per tenant (if they have dedicated API/CloudFront)
- Or use WAF rule groups to compose shared + tenant-specific rules

## CloudFront Multi-Tenant Patterns

### Single Distribution (Pool)

One CloudFront distribution serves all tenants. Wildcard domain or path-based routing.

**Cache considerations:**
- Cache key must include tenant context (tenant ID header or subdomain) to prevent cross-tenant cache poisoning
- Use CloudFront cache policies to include the tenant identifier in the cache key
- Origin request policies forward tenant context to the origin

**Critical:** If you cache responses and don't include tenant context in the cache key, Tenant A could receive Tenant B's cached response. This is a data leak.

### Distribution Per Tenant (Silo)

Each tenant gets their own CloudFront distribution. Simplest cache isolation, but highest operational overhead.

**When to use:** Enterprise tier with custom domains, tenants with specific geo-restriction requirements, when cache isolation is critical.

## Healthcare Networking Patterns

### DICOM Network Routing
Clinical SaaS that handles medical imaging needs to support DICOM protocols alongside standard HTTPS:

- **DICOMweb (WADO-RS, STOW-RS, QIDO-RS):** RESTful HTTP — route through API Gateway or ALB with standard TLS termination. Tenant routing via path or header.
- **DIMSE (C-STORE, C-FIND, C-MOVE):** Binary TCP protocol on custom ports. Route through Network Load Balancer (NLB). Requires VPN or Direct Connect from hospital sites.
- **Hybrid routing:** API Gateway for DICOMweb + NLB for DIMSE, both routing to the same backend imaging service.

See `clinical-saas-and-imaging.md` for detailed DICOM architecture.

### PrivateLink for Health Systems
Enterprise health system tenants often require private connectivity — their traffic must not traverse the public internet. This is common in healthcare due to security policies and compliance requirements.

- Create an NLB in front of your SaaS backend
- Create a VPC Endpoint Service pointing to the NLB
- Health system creates a VPC Interface Endpoint in their VPC
- Traffic flows privately over AWS's network
- Allowlist specific AWS accounts (health system's accounts) to connect
- Per-tenant PrivateLink endpoints for silo model; shared endpoint with tenant routing for pool/bridge

### Direct Connect for High-Volume Sites
Hospital sites sending large volumes of imaging data (> 50 GB/day) should use AWS Direct Connect:
- Dedicated 1 Gbps or 10 Gbps connections
- Consistent latency and throughput (critical for DICOM transfer)
- Can be combined with VPN for encryption
- Consider AWS Direct Connect Gateway for multi-region connectivity

## Common Mistakes

1. **Not including tenant context in CloudFront cache keys** — This causes cross-tenant cache poisoning. If you cache anything, the cache key must include tenant identity.

2. **Using API Gateway account-level throttling as tenant throttling** — Account-level limits protect your account, not individual tenants. Use usage plans for per-tenant throttling.

3. **Forgetting VPC limits when planning VPC-per-tenant** — VPCs per region (default 5, can increase), subnets per VPC, security groups per VPC, Transit Gateway attachments — all have limits that matter at scale.

4. **Not planning for custom domains early** — If enterprise tenants will want custom domains, design the routing layer for it from the start. Retrofitting custom domain support is painful.

5. **Exposing tenant IDs in URLs without validation** — If the tenant ID is in the URL path, always validate it matches the authenticated tenant's JWT claim. Otherwise it's a trivial IDOR vulnerability.

6. **Skipping PrivateLink for regulated enterprise tenants** — If an enterprise tenant asks "does traffic stay on AWS's network?" and the answer is no, you may lose the deal. Plan for PrivateLink in your enterprise tier.

## Discovery Questions for This Domain

When the conversation is about API design, networking, or tenant routing:

**API Gateway:**
- How do tenants access your API today? (Single shared endpoint? Per-tenant endpoints? Custom domains?)
- Do you need per-tenant rate limiting? How granular? (Per-tenant? Per-tier? Per-endpoint?)
- Do enterprise tenants need custom API configurations (custom rate limits, IP allowlisting)?
- Are you using REST API or HTTP API? (REST API has more features for multi-tenant: usage plans, API keys, resource policies)

**Routing:**
- How do you identify which tenant a request belongs to? (Subdomain? JWT? Header? Path?)
- Do tenants need their own subdomains or custom domains? (Branding, white-label requirements)
- Do you need to route different tiers to different backends? (Pool tenants to shared, silo tenants to dedicated)

**Networking:**
- Do any tenants require private connectivity (no public internet)? (PrivateLink, VPN, Direct Connect)
- Are your backend services in a VPC? (Lambda without VPC, ECS in VPC, EKS in VPC)
- Do you need network-level isolation between tenants? (VPC-per-tenant, security groups, network policies)

**Caching and CDN:**
- Are you using CloudFront? (Static assets, API caching, custom domains)
- If caching API responses: is tenant context included in the cache key? (Critical for preventing cross-tenant data leaks)
- Do tenants need geo-specific content delivery or geo-restrictions?

**Healthcare networking (if applicable):**
- Do you need to support DICOM ingestion from hospital modalities or PACS? If yes, which protocols — DICOMweb (STOW-RS/WADO-RS/QIDO-RS over HTTPS) only, DIMSE (TCP, legacy), or both?
- For DIMSE: you'll need Network Load Balancer routing and likely VPN or Direct Connect from hospital sites. What connectivity do hospital sites have today?
- Do enterprise health system tenants require PrivateLink? (No public internet traversal is common in healthcare security policies)
- Do you have high-volume imaging sites that need AWS Direct Connect (> 50 GB/day)? Which sites, and at what bandwidth?
- For FHIR endpoints: do different tenants need to be reachable at different endpoints (per-tenant subdomains or custom domains for white-label), or is a single endpoint with tenant routing in JWT acceptable?

## References

- [Building Multi-Tenant API Gateway Solutions](https://aws.amazon.com/blogs/apn/building-multi-tenant-api-gateway-solutions/)
- [Enabling Tiering and Throttling in a Multi-Tenant Amazon EKS SaaS Solution Using Amazon API Gateway](https://aws.amazon.com/blogs/apn/enabling-tiering-and-throttling-in-a-multi-tenant-amazon-eks-saas-solution-using-amazon-api-gateway/)
- [Throttling a Tiered, Multi-Tenant REST API at Scale Using API Gateway: Part 1](https://aws.amazon.com/blogs/architecture/throttling-a-tiered-multi-tenant-rest-api-at-scale-using-api-gateway-part-1/)
- [SaaS Tenant Isolation Strategies — Network Isolation](https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/network-isolation.html)
- [AWS PrivateLink for SaaS](https://docs.aws.amazon.com/vpc/latest/privatelink/privatelink-share-your-services.html)
- [SaaS Lens — Multi-Tenant Microservices](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/multi-tenant-microservices.html)
