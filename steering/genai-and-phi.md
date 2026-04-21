# Generative AI & PHI

## Why This Is the Sharpest Edge

GenAI on clinical data is the fastest-moving area in healthcare SaaS. Ambient documentation, clinical decision support, medical coding automation, patient communication — all require AI processing of PHI. The architectural challenge: how do you use foundation models on the most sensitive data in healthcare without creating a compliance nightmare?

## Bedrock HIPAA Eligibility

Amazon Bedrock is HIPAA-eligible and can be included under the AWS BAA. This means you CAN process PHI through Bedrock — but only with proper configuration.

**What's covered:**
- Model invocation (InvokeModel, InvokeModelWithResponseStream)
- Bedrock Agents
- Bedrock Knowledge Bases
- Bedrock Guardrails
- Model invocation logging (to S3 or CloudWatch — required for PHI audit trails)

**Critical configuration:**
- Enable model invocation logging — every prompt and response must be logged for PHI audit trails
- Logs must be encrypted (KMS) and retained per HIPAA requirements (6+ years)
- Do NOT use models from providers that don't have a BAA with AWS for PHI workloads — verify model provider eligibility
- Use VPC endpoints for Bedrock to keep traffic off the public internet

**Reference:** [HIPAA Compliance for Generative AI Solutions on AWS](https://aws.amazon.com/blogs/industries/hipaa-compliance-for-generative-ai-solutions-on-aws/)

## The Dual-Zone Architecture Pattern

This is the recommended architecture for AI processing of PHI. It creates a strict boundary between the AI processing layer and the PHI storage layer.

### Zone 1: AI Zone (Processing)
- Contains: Bedrock invocations, Lambda functions for prompt construction, response parsing
- Has: IAM permissions to call Bedrock APIs
- Does NOT have: IAM permissions to directly access PHI data stores (DynamoDB, HealthLake, S3 buckets with patient data)
- PHI enters this zone only through controlled interfaces (Lambda functions that retrieve specific data, construct prompts, and pass to Bedrock)
- PHI exits this zone only through controlled interfaces (Lambda functions that parse responses and write to PHI stores)

### Zone 2: PHI Zone (Storage)
- Contains: HealthLake, DynamoDB, S3 buckets with patient data, RDS with clinical records
- Has: IAM permissions to access PHI data stores
- Does NOT have: IAM permissions to call Bedrock APIs directly
- Data retrieval functions in this zone extract specific PHI needed for AI processing and pass it to the AI zone

### The Boundary
- Strict IAM separation: AI zone roles cannot assume PHI zone roles and vice versa
- Data flows through Lambda functions that act as controlled gates
- Every cross-zone data transfer is logged
- The AI zone never persists PHI — it processes and returns results to the PHI zone for storage

### Why This Pattern Matters
- If the AI zone is compromised, the attacker cannot access PHI data stores directly
- If a prompt injection attack manipulates the model, the model cannot exfiltrate data because it has no direct access to PHI stores
- Audit trail clearly shows what PHI entered the AI zone and what came out
- Compliance story: "PHI is processed by AI but never stored in the AI layer"

**Reference:** [Dual-Zone AI Architecture for PHI Protection](https://ai.efsnetworks.com/case-studies/dual-zone-ai-phi-architecture/)

## RAG Over PHI with Tenant Isolation

Retrieval-Augmented Generation (RAG) over clinical data is a common pattern — the model retrieves relevant patient records before generating a response. In multi-tenant healthcare SaaS, this requires strict tenant isolation in the retrieval layer.

### Per-Tenant Knowledge Bases
- Each tenant gets their own Bedrock Knowledge Base backed by their own data source (S3 bucket, HealthLake data store)
- IAM policies restrict each knowledge base to the tenant's data
- No cross-tenant retrieval is possible — the knowledge base only contains one tenant's data

**Pros:** Strongest isolation, simple compliance story
**Cons:** Higher cost, operational overhead scales with tenant count

### Shared Knowledge Base with Tenant Filtering
- Single knowledge base with documents tagged by tenant ID
- At query time, apply a metadata filter to restrict retrieval to the current tenant's documents
- IAM session policies scope the Bedrock API call to the current tenant

**Pros:** Lower cost, simpler operations
**Cons:** Relies on metadata filtering for isolation (weaker than resource-level separation), risk of cross-tenant retrieval if filter is misconfigured

**Recommendation for healthcare:** Per-tenant knowledge bases for PHI. The compliance risk of cross-tenant retrieval in a shared knowledge base is too high for clinical data.

### Preventing Cross-Tenant Context Leakage
- Never include Tenant A's data in a prompt sent on behalf of Tenant B
- Clear conversation history between tenant sessions (no shared memory across tenants)
- Bedrock Agents sessions must be tenant-scoped — use session attributes to bind to tenant
- Monitor for prompt injection attempts that try to extract data from other tenants

## Audio as PHI in AI Pipelines

This is critical for ambient documentation and telehealth SaaS.

### The PHI Chain
```
Clinical encounter audio → Transcribe Medical → Clinical text → Bedrock → SOAP note / Summary
       (PHI)                    (PHI)              (PHI)         (PHI)         (PHI)
```

Every step in this chain handles PHI. Every step must be:
- Encrypted (in transit and at rest)
- Access-controlled (tenant-scoped)
- Audit-logged (who processed what, when)
- Retained per HIPAA requirements

### Architecture Pattern
1. **Audio capture:** Chime SDK (HIPAA-eligible) for telehealth, or client-side recording for ambient documentation
2. **Audio storage:** S3 with per-tenant KMS key, lifecycle policies for retention
3. **Transcription:** Transcribe Medical (HIPAA-eligible) — outputs clinical text with medical terminology recognition
4. **AI processing:** Bedrock (dual-zone pattern) — generates SOAP notes, summaries, coding suggestions
5. **Output storage:** HealthLake (FHIR DocumentReference) or DynamoDB — the AI-generated note is PHI and must be stored with full PHI controls
6. **Clinician review:** AI output is presented to the clinician for review and approval before it becomes part of the medical record

### Key Requirement: Human-in-the-Loop
AI-generated clinical content must be reviewed by a clinician before it's finalized. This is both a clinical safety requirement and a liability consideration. The architecture must support:
- Draft state: AI output is stored as a draft, not yet part of the official record
- Review workflow: clinician reviews, edits, and approves
- Approval state: approved output becomes part of the medical record
- Audit trail: who generated, who reviewed, who approved, what was changed

## AI Inference Audit Trails

Every AI inference on patient data must be logged and reproducible. This is a HIPAA audit requirement and a clinical safety requirement.

### What to Log
```json
{
  "timestamp": "2025-01-15T14:30:00Z",
  "event_type": "AI_INFERENCE",
  "tenant_id": "tenant-abc123",
  "user_id": "user-dr-smith-456",
  "patient_record_id": "record-789",
  "model_id": "anthropic.claude-3-sonnet",
  "model_version": "v1",
  "input_type": "clinical_note_transcription",
  "input_hash": "sha256:abc123...",
  "output_type": "soap_note_draft",
  "output_hash": "sha256:def456...",
  "guardrails_applied": ["phi-filter", "clinical-safety"],
  "guardrails_triggered": false,
  "inference_duration_ms": 2340,
  "approval_status": "pending_review"
}
```

### Why Reproducibility Matters
- If a clinical decision was influenced by AI output, the organization must be able to reconstruct what the AI produced and what data it was based on
- For FDA-regulated AI (SaMD), reproducibility is a regulatory requirement — see `clinical-saas-and-imaging.md`
- Store the input hash and output hash so you can verify that the logged inference matches the actual data

### Model Invocation Logging
Enable Bedrock model invocation logging to capture prompts and responses:
- Log to S3 (encrypted with KMS, Object Lock for immutability)
- Retention: 6+ years per HIPAA
- Access: restricted to compliance and audit roles only (prompts contain PHI)
- Note: invocation logs themselves are PHI because they contain patient data in prompts

## Amazon Connect Health

Amazon Connect Health (announced March 2026) is a purpose-built agentic AI service for clinical documentation, patient insights, and medical coding within EHR workflows.

### What It Does
- Ambient clinical documentation: listens to clinical encounters, generates structured notes
- Patient insights: summarizes patient history from EHR data
- Medical coding: suggests ICD-10 and CPT codes from clinical documentation
- EHR integration: works within existing EHR workflows (Epic, Cerner)

### Architecture Implications
- HIPAA-eligible — can process PHI
- Integrates with existing EHR via FHIR/HL7
- Reduces the need to build custom ambient documentation pipelines
- Consider Connect Health vs custom Bedrock pipeline: Connect Health is faster to deploy but less customizable; custom pipeline gives full control but requires more engineering

### When to Use Connect Health vs Custom
- **Use Connect Health:** Standard clinical documentation, medical coding, patient summarization. You want to ship fast and don't need deep customization.
- **Use custom Bedrock pipeline:** Specialized clinical workflows, custom AI models, unique output formats, need for fine-grained control over prompts and guardrails, multi-tenant cost attribution at the inference level.

## Multi-Tenant Bedrock Patterns

### Per-Tenant Guardrails
Bedrock Guardrails can be configured per tenant to enforce different content policies:
- Enterprise health system tenant: strict clinical safety guardrails, no speculative diagnoses
- Research-focused tenant: more permissive for exploratory analysis
- Implementation: store guardrail configuration per tenant in control plane, apply at inference time

### Per-Tenant Cost Tracking
Use Bedrock Application Inference Profiles to track AI usage and cost per tenant:
- Create an inference profile per tenant (or per tier)
- Route invocations through the tenant's profile
- Cost Explorer shows per-profile costs → per-tenant AI cost attribution
- Useful for usage-based billing of AI features

**Reference:** [Manage Multi-Tenant Amazon Bedrock Costs Using Application Inference Profiles](https://aws.amazon.com/blogs/machine-learning/manage-multi-tenant-amazon-bedrock-costs-using-application-inference-profiles/)

### Tenant Isolation in Bedrock Agents
- Each tenant should have isolated agent sessions (no shared memory across tenants)
- Use session attributes to bind agent sessions to tenant context
- Knowledge bases should be per-tenant for PHI (see RAG section above)
- Action groups should validate tenant context before executing actions

**Reference:** [Implementing Tenant Isolation Using Agents for Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/implementing-tenant-isolation-using-agents-for-amazon-bedrock-in-a-multi-tenant-environment/)

## Responsible AI for Clinical Decisions

### Hallucination Risk in Clinical Context
AI hallucinations in clinical settings can cause patient harm. Unlike a chatbot hallucinating a wrong fact, a clinical AI hallucinating a diagnosis or medication can lead to incorrect treatment.

**Mitigations:**
- Always human-in-the-loop for clinical decisions — AI suggests, clinician decides
- Confidence scoring: if the model's confidence is below a threshold, flag for manual review
- Citation/evidence linking: AI output should reference the source data it was based on
- Guardrails: use Bedrock Guardrails to block speculative diagnoses, unsupported treatment recommendations
- Validation: compare AI output against clinical guidelines (e.g., does the suggested ICD-10 code match the documented symptoms?)

### Clinical Safety Testing
- Test AI outputs against known clinical scenarios with expected outcomes
- Include edge cases: rare conditions, conflicting symptoms, incomplete data
- Monitor AI output quality in production: track clinician edit rates (high edit rate = low AI quality)
- Feedback loop: clinician corrections should inform model improvement

### Regulatory Considerations
- If your AI provides clinical decision support that meets FDA's SaMD definition, it may require FDA clearance — see `clinical-saas-and-imaging.md`
- FDA's CDS exemption criteria: the AI must (1) not be intended to acquire/analyze medical images or signals, (2) display the basis for recommendations, (3) not be intended for urgent situations, and (4) be intended for a healthcare professional to independently review. If all four criteria are met, it may be exempt from FDA regulation.

## GxP for Clinical AI

If your AI-assisted clinical feature is SaMD or supports a GxP workflow (eClinical data review, pharmacovigilance signal detection, regulated lab decision support), the full GxP framework applies. See `gxp-compliance-generic.md` for generic GxP and `clinical-saas-and-imaging.md` for SaMD-specific validation. This section covers GxP concerns unique to generative AI on PHI.

### GAMP 5 Categorization for AI Components

| Component | GAMP Category | Validation Scope |
|-----------|---------------|------------------|
| Bedrock managed service | Category 4 (configured) | IQ/OQ — verify model access, guardrail configuration, logging |
| Foundation model (Claude, Titan, etc.) | Category 4 (COTS) with Category 5 usage | Qualify the model version; validate your prompts/usage as Category 5 |
| Prompt templates | Category 5 (custom) | Version-controlled, tested with golden dataset, change-controlled |
| Guardrails configuration | Category 5 (custom) | Tested against known adversarial inputs |
| Knowledge base content and indexing | Category 5 (custom) | Content provenance documented, indexing process validated |
| Inference post-processing | Category 5 (custom) | Full lifecycle validation |
| Human-in-the-loop review workflow | Category 5 (custom) | Validated — this is often the control that brings the AI within CDS exemption or reduces risk class |

### Good Machine Learning Practice (GMLP)

FDA/Health Canada/MHRA joint guiding principles for AI/ML medical device development. Ten principles summarized:

1. Multi-disciplinary expertise leveraged throughout the lifecycle
2. Good software engineering and security practices
3. Clinical study participants and datasets representative of intended patient population
4. Training data sets independent of test sets
5. Selected reference datasets based on best available methods
6. Model design tailored to available data and reflective of intended use
7. Focus on performance of human-AI team
8. Testing demonstrates device performance during clinically relevant conditions
9. Users provided clear, essential information
10. Deployed models monitored for performance; retraining risks managed

**Architectural implications:**
- Dataset management tools (dataset versioning, lineage, bias analysis)
- Separate training and test data storage with access controls
- Model registry with versioning, performance metrics, and validation evidence
- Production monitoring for model drift and performance degradation
- Clear UI presentation of AI output with uncertainty/confidence communicated

### Predetermined Change Control Plan (PCCP)

For AI that retrains or updates post-deployment, FDA permits a PCCP approved at clearance time. The PCCP describes:

- **Modifications protocol** — what kinds of changes are allowed (retraining with new data, threshold tuning)
- **Performance requirements** — minimum performance thresholds the updated model must meet
- **Verification and validation** — how the change will be verified (tests, datasets, acceptance criteria)
- **Impact assessment** — how the change affects safety, effectiveness, intended use

**Architectural implications:**
- Automated model retraining pipeline with validation gates
- Model registry captures every model version with validation results
- Deployment gate checks PCCP compliance before activating a new model version
- Rollback capability to previous validated model version
- All model versions auditable — which patients had inferences from which model version

### AI Inference Audit Trail — ALCOA+ Applied

The generic AI inference audit trail (earlier in this file) aligns with ALCOA+. Additional GxP-specific fields:

```json
{
  "timestamp": "2025-01-15T14:30:00Z",
  "event_type": "AI_INFERENCE",
  "tenant_id": "tenant-abc123",
  "user_id": "user-dr-smith-456",
  "patient_record_id": "record-789",
  "model_id": "anthropic.claude-3-sonnet",
  "model_version": "v1",
  "model_validation_id": "val-2025-001",
  "prompt_template_version": "prompt-v3.2",
  "guardrail_version": "guardrail-v1.5",
  "input_type": "clinical_note_transcription",
  "input_hash": "sha256:abc123...",
  "output_type": "soap_note_draft",
  "output_hash": "sha256:def456...",
  "guardrails_applied": ["phi-filter", "clinical-safety"],
  "guardrails_triggered": false,
  "inference_duration_ms": 2340,
  "approval_status": "pending_review",
  "reviewer_id": null,
  "review_timestamp": null,
  "review_outcome": null,
  "reason_for_change": null
}
```

New fields vs. generic:
- `model_validation_id` — links to validation evidence for this model version (traceability)
- `prompt_template_version` and `guardrail_version` — reproducibility requires knowing the exact prompt and guardrails used
- `reviewer_id`, `review_timestamp`, `review_outcome` — human review audit (signature application)
- `reason_for_change` — if the clinician modified AI output, why (required for 21 CFR Part 11 compliance on amendments)

### Validation Dataset Management

For SaMD AI, datasets are as important as code:
- Training data stored in S3 with immutable versioning (S3 Object Lock)
- Test data strictly separated from training data — different buckets, different IAM roles
- Dataset metadata (source, collection date, annotation provenance, demographics, biases) stored alongside data
- Dataset access audit-logged (CloudTrail data events)
- De-identification status documented per dataset (Safe Harbor, Expert Determination, or re-identifiable)

### Human-in-the-Loop as a GxP Control

The clinician review step is often THE control that:
- Brings AI within FDA's CDS exemption (criterion 4: healthcare professional independently reviews)
- Reduces risk classification from Class C to Class B (clinician can override)
- Satisfies 21 CFR Part 11 if the clinician's approval constitutes the electronic signature

**Architecture requirements for HITL as a GxP control:**
- Clinician review cannot be bypassed (no auto-commit of AI output to medical record)
- Review action captured as an electronic signature event (signer identity, timestamp, meaning)
- Review action is atomically bound to the record being reviewed
- Modifications by the clinician logged with old value / new value / reason
- Audit trail retained per regulatory retention period

See `audit-logging-and-access.md` for 21 CFR Part 11 electronic signature implementation.

## Common Mistakes

1. **Processing PHI through non-HIPAA-eligible AI services.** Verify that the specific Bedrock model and feature you're using is covered under the BAA. Not all AI services are HIPAA-eligible.

2. **No model invocation logging.** If you can't prove what the AI was asked and what it responded, you can't audit PHI processing through the AI layer. Enable invocation logging from day one.

3. **Shared RAG knowledge bases for PHI.** Cross-tenant retrieval in a shared knowledge base is a PHI breach. Use per-tenant knowledge bases for clinical data.

4. **AI output treated as final without clinician review.** AI-generated clinical content must be reviewed by a qualified clinician before becoming part of the medical record. Build the review workflow into the architecture.

5. **Forgetting that prompts are PHI.** If the prompt contains patient data, the prompt itself is PHI. Invocation logs must be encrypted, access-controlled, and retained per HIPAA.

6. **No cross-tenant session isolation in Bedrock Agents.** If agent memory persists across tenant sessions, Tenant B's query could retrieve context from Tenant A's session. Clear sessions at tenant boundaries.

7. **Ignoring prompt injection in clinical context.** A prompt injection that causes the model to output incorrect clinical information is a patient safety risk, not just a security risk. Use Bedrock Guardrails and input validation.

8. **No model version traceability.** If you can't say which model version generated a specific clinical output, you can't reproduce or audit it. Log `model_id` and `model_version` on every inference; treat model updates as versioned artifacts.

9. **Treating prompt templates as configuration, not code.** For GxP AI, prompt templates are Category 5 custom software. They need version control, testing, change control, and validation — not ad-hoc edits in a config file.

10. **No PCCP for retraining AI.** If your model updates periodically with new data and you don't have a PCCP approved at FDA clearance, every update triggers a new 510(k). Plan PCCP into the lifecycle from day one.

## Discovery Questions for This Domain

**AI use case:**
- What clinical AI features are you building? (Ambient documentation, clinical decision support, medical coding, patient communication, imaging analysis?)
- Is the AI output used for clinical decisions? (If yes, human-in-the-loop is required)
- Does the AI process PHI? (If yes, dual-zone architecture and full PHI controls apply)

**Architecture:**
- Are you using Bedrock or a self-hosted model? (Bedrock is HIPAA-eligible; self-hosted requires your own compliance controls)
- Do you need RAG over clinical data? (Per-tenant knowledge bases for PHI)
- Do you need per-tenant AI cost tracking? (Application inference profiles)
- Are you building ambient documentation? (Audio → transcription → AI pipeline, all PHI)

**Compliance:**
- Is model invocation logging enabled? (Required for PHI audit trails)
- Are invocation logs encrypted and retained per HIPAA? (6+ years, immutable)
- Does the AI output meet FDA's CDS exemption criteria? (If not, may need FDA clearance)

**Multi-tenant:**
- How are you isolating AI processing across tenants? (Per-tenant knowledge bases, session isolation, guardrails)
- How are you attributing AI costs per tenant? (Inference profiles, metering)
- Do different tenants need different AI guardrails? (Enterprise vs SMB, clinical vs research)

**GxP for AI:**
- Is the AI component SaMD? (If yes, IEC 62304 software lifecycle applies)
- Have you categorized AI components per GAMP 5? (Category 4 for Bedrock, Category 5 for your prompts/guardrails)
- Do you have a validation dataset separate from training data?
- For retraining AI: do you have a Predetermined Change Control Plan?
- Is the human-in-the-loop review captured as an electronic signature event? (21 CFR Part 11)
- Are prompt templates and guardrail configurations version-controlled with change control?

## References

- [HIPAA Compliance for Generative AI Solutions on AWS](https://aws.amazon.com/blogs/industries/hipaa-compliance-for-generative-ai-solutions-on-aws/)
- [Implementing Tenant Isolation Using Agents for Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/implementing-tenant-isolation-using-agents-for-amazon-bedrock-in-a-multi-tenant-environment/)
- [Manage Multi-Tenant Amazon Bedrock Costs Using Application Inference Profiles](https://aws.amazon.com/blogs/machine-learning/manage-multi-tenant-amazon-bedrock-costs-using-application-inference-profiles/)
- [Build a Multi-Tenant Chatbot with RAG Using Amazon Bedrock and Amazon EKS](https://aws.amazon.com/blogs/containers/build-a-multi-tenant-chatbot-with-rag-using-amazon-bedrock-and-amazon-eks/)
- [Transform Healthcare Revenue Cycle Management with Amazon Bedrock AgentCore](https://aws.amazon.com/blogs/industries/transform-healthcare-revenue-cycle-management-with-amazon-bedrock-agentcore/)
- [How Amazon Connect Health Brings Agentic AI to the Point of Care](https://aws.amazon.com/blogs/industries/how-amazon-connect-health-brings-agentic-ai-to-the-point-of-care/)
- [Orchestrating Clinical Generative AI Workflows Using AWS Step Functions](https://aws.amazon.com/blogs/industries/orchestrating-clinical-generative-ai-workflows-using-aws-step-functions/)
- [Build an Internal SaaS Service with Cost and Usage Tracking for Foundation Models on Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/build-an-internal-saas-service-with-cost-and-usage-tracking-for-foundation-models-on-amazon-bedrock/)
