# Clinical SaaS & Medical Imaging

## Segment Overview

Clinical SaaS includes any multi-tenant software delivered to healthcare providers for clinical workflows: PACS (Picture Archiving and Communication Systems), advanced visualization, radiology AI, pathology platforms, cardiology imaging, clinical decision support, and surgical planning tools. The tenants are hospitals, imaging centers, radiology groups, and health systems.

This segment has unique architectural concerns that other healthcare SaaS segments don't face: massive data volumes (DICOM studies are GB-sized, archives are PB-scale), sub-second retrieval requirements for radiologist workflow, hybrid on-prem/cloud architectures, and FDA regulatory surface for AI-assisted features.

## AWS HealthImaging

AWS HealthImaging is a HIPAA-eligible service for storing and sharing medical images at petabyte scale with sub-second retrieval.

### Key Concepts
- **Data store:** A container for medical imaging data. Each data store has its own endpoint, encryption key, and access policy. Natural tenant isolation boundary.
- **Image set:** An AWS concept grouping related DICOM instances (typically a study or series). Created during import from DICOM P10 files.
- **Image frame:** The pixel data from a DICOM instance. Retrieved separately from metadata for performance.
- **DICOM import job:** Ingests DICOM P10 files from S3 into a data store, transforming them into image sets.

### Multi-Tenant Patterns

**Data Store Per Tenant (Recommended)**
Each tenant gets their own HealthImaging data store.

- Strongest isolation — separate endpoints, separate KMS keys
- Per-tenant storage metrics and cost attribution
- Clean tenant offboarding (delete the data store)
- DICOM import jobs scoped to the tenant's data store
- Operational overhead scales with tenant count

**Shared Data Store (Not Recommended)**
All tenants share one data store. Image sets tagged with tenant metadata.

- Lower cost but weaker isolation
- Imaging data is too large and too sensitive for pool model
- Not recommended for healthcare — use data-store-per-tenant

### Storage Tiering
HealthImaging integrates with S3 storage classes for cost optimization:
- Recently accessed studies: hot storage (immediate retrieval)
- Aging studies: S3 Intelligent-Tiering automatically moves to lower-cost tiers based on access frequency
- Long-term archive: S3 Glacier Instant Retrieval for studies older than N years (still sub-second retrieval)

**Key insight:** Medical imaging data follows a predictable access pattern — high access in the first days/weeks after acquisition, declining rapidly. S3 Intelligent-Tiering handles this automatically.

### Performance
- Sub-second first-byte retrieval for image frames
- Optimized for radiology viewing workflows (hanging protocols, prior study comparison)
- HTTP/2 support for parallel frame retrieval
- Pre-fetch strategies: when a radiologist opens a study, pre-fetch the most likely next studies (priors, comparison exams)

**Reference:** [Konica Minolta Improves Radiologist Productivity Using AWS HealthImaging](https://aws.amazon.com/solutions/case-studies/konica-minolta-healthimaging-case-study/)

## PACS/VNA Integration

Most clinical SaaS doesn't replace the hospital's PACS — it integrates with it. Understanding the integration patterns is critical.

### On-Prem to Cloud Data Flow
```
Imaging Modality (CT/MRI/X-ray) → Hospital PACS/VNA → DICOM Router → AWS
                                                         ↓
                                              DICOMweb (STOW-RS) or
                                              DIMSE (C-STORE) via VPN/Direct Connect
                                                         ↓
                                              S3 (staging) → HealthImaging Import Job
```

### DICOM Protocols

**DICOMweb (Modern — Preferred)**
RESTful HTTP-based protocol. Works through standard HTTPS connections.
- STOW-RS: Store instances (upload)
- WADO-RS: Retrieve instances, metadata, or rendered images (download)
- QIDO-RS: Query for studies, series, instances (search)
- Route through API Gateway or ALB with TLS termination

**DIMSE (Legacy — Still Common)**
Binary TCP protocol on custom ports (typically 104, 11112).
- C-STORE: Send instances
- C-FIND: Query for studies
- C-MOVE: Request transfer of instances
- Requires Network Load Balancer (NLB) for TCP routing
- Often requires VPN or Direct Connect (DIMSE is not HTTP-based)

### Hybrid Architecture

Many clinical SaaS deployments are hybrid — some components on-prem, some in cloud.

**On-prem components:**
- DICOM router/gateway: receives studies from hospital modalities, forwards to cloud
- Local cache: stores recently accessed studies for low-latency viewing at the hospital site
- Interface engine: handles HL7 v2 worklist messages (ORM for orders, ORU for reports)

**Cloud components:**
- HealthImaging: long-term archive and AI processing
- Viewer backend: serves images to zero-footprint browser viewers
- AI pipeline: processes images for triage, detection, measurement
- Control plane: tenant management, billing, configuration

**AWS options for on-prem:**
- AWS Outposts: run AWS services on-prem for latency-sensitive workloads
- AWS Local Zones: AWS infrastructure closer to the hospital for lower latency
- Direct Connect: dedicated network connection for high-volume DICOM transfer (recommended for sites sending > 100 studies/day)
- Site-to-Site VPN: encrypted tunnel for lower-volume sites

### Network Considerations
- DICOM studies are large (CT: 100-500 MB, MRI: 50-200 MB, mammography: 200-500 MB per study)
- A busy imaging center generates 100-500 studies/day = 10-250 GB/day of upload
- Direct Connect is recommended for high-volume sites (> 50 GB/day)
- VPN is acceptable for lower-volume sites
- Redundant connections recommended when loss of connectivity impacts patient care

## Viewer Delivery

Radiologists and clinicians need to view medical images through your SaaS. The viewer is the primary user interface.

### Zero-Footprint Browser Viewers
Modern approach — no client-side installation required. The viewer runs in the browser.

**Architecture:**
- Viewer frontend: JavaScript/WebAssembly application served via CloudFront
- Image retrieval: DICOMweb WADO-RS calls to HealthImaging (or your image server)
- Rendering: client-side rendering (WebGL) for 2D, server-side rendering for 3D
- Pre-fetch: predict which images the radiologist will view next, pre-load them

**Performance optimization:**
- CloudFront for static viewer assets (JS, CSS, WASM)
- HTTP/2 for parallel image frame retrieval
- Progressive loading: display low-resolution first, then full resolution
- Hanging protocol pre-fetch: based on the study type, pre-load the expected image layout

### Server-Side Rendering (3D, Advanced Visualization)
For 3D reconstruction, MPR, volume rendering, and advanced visualization:

- GPU instances (G4dn, G5) for rendering
- AppStream 2.0 or NICE DCV for streaming rendered output to the browser
- Per-tenant or per-session GPU allocation
- Cost consideration: GPU instances are expensive — use auto-scaling and session-based allocation

## Medical Imaging Reference Architecture

The AWS Healthcare Industry Lens defines a reference architecture for medical imaging systems:

- Multi-AZ deployment for high availability
- Auto-scaling per tier (viewer, application, database, storage)
- Metadata in RDS/DynamoDB (study-level metadata, worklist, reporting)
- Image data in HealthImaging or S3
- Hot cache on FSx or EBS for frequently accessed studies
- S3 Intelligent-Tiering for aging studies
- Data lake (S3 + Athena/SageMaker) for AI/ML and research
- Direct Connect for high-volume hospital sites
- Hybrid with Local Zones/Outposts for latency-sensitive sites

**Reference:** [Medical Imaging System Reference Architecture](https://docs.aws.amazon.com/wellarchitected/latest/healthcare-industry-lens/medical-imaging-system-architecture.html)

## FDA/SaMD Considerations

If your clinical SaaS includes AI-assisted features (triage, detection, measurement, diagnosis suggestion), it may qualify as Software as a Medical Device (SaMD) under FDA regulation.

### When Does Your Software Qualify as SaMD?
SaMD is defined by IMDRF as "software intended to be used for one or more medical purposes that perform these purposes without being part of a hardware medical device."

**Examples that ARE SaMD:**
- AI that detects stroke on CT images and alerts the care team
- Software that measures tumor size on MRI and tracks changes over time
- Algorithm that triages chest X-rays by urgency
- AI that suggests ICD-10 codes based on radiology reports (if intended for clinical use)

**Examples that are NOT SaMD (likely):**
- PACS viewer that only displays images (no clinical analysis)
- Worklist management software
- Scheduling and reporting tools
- Image archive/storage systems

### FDA CDS Exemption
Clinical Decision Support (CDS) software may be exempt from FDA regulation if ALL four criteria are met:
1. Not intended to acquire, process, or analyze medical images or signals from a device
2. Intended to display, analyze, or print medical information about a patient
3. Intended to be used by a healthcare professional who can independently review the basis for the recommendation
4. Intended to enable the healthcare professional to independently review the basis for the recommendation

**If your AI analyzes medical images, criterion 1 is NOT met and the exemption does NOT apply.** Most radiology AI is SaMD.

### IEC 62304 — Software Lifecycle
If your software is SaMD, IEC 62304 defines the software development lifecycle requirements:
- Software safety classification (Class A, B, or C based on risk)
- Software development planning
- Software requirements analysis
- Software architectural design
- Software detailed design
- Software unit implementation and verification
- Software integration and integration testing
- Software system testing
- Software release

**SaaS implication:** IEC 62304 expects a controlled, documented development process. This doesn't prohibit CI/CD, but it requires that every release is traceable, tested, and documented.

### ISO 13485 — Quality Management System
If your software is SaMD, you need a Quality Management System (QMS) compliant with ISO 13485. This covers:
- Design controls (design input → design output → verification → validation)
- Document control
- Change management
- Risk management (ISO 14971)
- Corrective and preventive actions (CAPA)

### CI/CD for FDA-Regulated SaaS

The tension: SaaS demands rapid, continuous deployment. FDA demands validated, controlled releases. Here's how to reconcile:

**Validated deployment pipeline:**
- Every deployment is traceable: commit → build → test → deploy, with artifacts at each stage
- Automated testing includes: unit tests, integration tests, regression tests, and clinical validation tests
- Test results are stored as evidence (S3, immutable)
- Deployment approval gate: automated tests must pass before deployment proceeds
- Change control documentation generated automatically from commit messages, test results, and deployment logs

**Rollback requirements:**
- Every deployment must be rollbackable
- Rollback procedure is documented and tested
- If a rollback occurs, it's documented with reason and impact assessment

**Version management:**
- All tenants run the same version (SaaS principle)
- Version is traceable to a specific commit, build, and test run
- Model versions (for AI) are tracked separately from application versions
- Model retraining triggers a new validation cycle

**Reconciling velocity with regulation:**
- Use feature flags to separate deployment from feature activation
- Deploy code continuously, but activate FDA-regulated features only after validation is complete
- Maintain a "regulatory release" cadence (e.g., monthly) for FDA-regulated features, while non-regulated features deploy continuously
- Document the boundary between regulated and non-regulated features

### Algorithmic Audit Trails
For AI-assisted clinical features:
- Every AI inference must be logged: input data (hash), model version, output, confidence score, timestamp
- Results must be reproducible: given the same input and model version, the output should be the same
- Clinician actions on AI output must be logged: accepted, modified, rejected
- See `genai-and-phi.md` for detailed AI audit trail patterns

### Disclaimer
**This power provides architectural guidance, not regulatory advice. Engage a regulatory affairs specialist for FDA submission strategy, classification decisions, and compliance planning.**

## GxP Alignment for Clinical SaaS & SaMD

If your clinical SaaS is SaMD or otherwise falls under GxP (eClinical, pharmacovigilance, regulated labs), the generic GxP requirements apply in full. See `gxp-compliance-generic.md` for ALCOA+, audit trail standards, validation (IQ/OQ/PQ), and change management. This section covers the clinical-SaaS-specific intersections.

### GAMP 5 Categorization for a Typical Clinical SaaS

| Component | Typical GAMP Category | Validation Scope |
|-----------|----------------------|------------------|
| AWS infrastructure (EC2, VPC, IAM, KMS) | Category 1 | Inherit via GxP on AWS whitepaper; verify configuration against IaC |
| HealthImaging data stores | Category 4 (configured) | IQ/OQ — verify data store configuration, encryption, access policies, retention |
| HealthLake data stores | Category 4 (configured) | IQ/OQ — verify FHIR profile support, SMART scopes, KMS configuration |
| DICOMweb router / interface engine (Mirth) | Category 4 (configured) | IQ/OQ/PQ — validate configuration, routing rules, message transformation |
| Custom viewer application | Category 5 (custom) | Full lifecycle — URS, FS, DS, build, IQ, OQ, PQ |
| AI inference pipeline (Bedrock + custom prompts) | Category 5 (custom) | Full lifecycle + AI-specific validation (see below) |
| Prompt templates and guardrail configurations | Category 5 (custom) | Version-controlled, tested, change-controlled |

**Architecture principle:** Keep the Category 5 surface small. More custom code = more validation effort. Prefer configured managed services over custom implementations when the managed service meets the requirement.

### IEC 62304 Software Safety Classification

If your clinical SaaS is SaMD, classify each software component:

| Class | Risk | Examples |
|-------|------|----------|
| Class A | No injury possible | Administrative features, scheduling, reporting dashboards |
| Class B | Non-serious injury possible | Triage alerts that a clinician reviews, measurement tools used as references |
| Class C | Death or serious injury possible | Autonomous diagnosis, closed-loop treatment control, critical alert systems |

**Architectural implication:** Segregate Class B/C components from Class A. A viewer with a triage AI overlay (Class B) should be architecturally separable from the scheduling module (Class A) so that changes to Class A don't require Class B revalidation. Use separate microservices, separate deployment pipelines, and separate documentation packages.

### Validation Evidence Artifacts

Maintain these for every SaMD release. Most can be auto-generated from your CI/CD pipeline:

| Artifact | Source | Storage |
|----------|--------|---------|
| User Requirements Specification (URS) | Product management | Version-controlled repo + immutable snapshot per release |
| Functional Specification (FS) | Engineering | Version-controlled repo + immutable snapshot per release |
| Design Specification (DS) | Engineering | Version-controlled repo + immutable snapshot per release |
| Traceability Matrix (URS → FS → DS → tests) | Auto-generated from requirements tooling | S3 with Object Lock |
| IQ evidence | Terraform/CDK plan output, deployment logs | S3 with Object Lock |
| OQ evidence | Automated test results, integration test reports, security scans | S3 with Object Lock |
| PQ evidence | E2E test results with production-representative data, performance benchmarks | S3 with Object Lock |
| Risk Management File (ISO 14971) | QMS tool + engineering input | QMS system with full version history |
| Change Control Record | PR metadata, reviewer approvals, test results | Git + CI/CD pipeline artifacts |
| Release Package | All of the above bundled per release | S3 with Object Lock, retained for product lifetime |

### AI/ML-Specific Validation for SaMD

Traditional IEC 62304 was written for deterministic software. AI/ML SaMD adds requirements:

- **Training data provenance** — document dataset source, annotations, splits, biases
- **Model versioning** — treat each trained model as a versioned artifact with validation evidence
- **Good Machine Learning Practice (GMLP)** — FDA/Health Canada/MHRA joint guiding principles for AI/ML medical devices
- **Predetermined Change Control Plan (PCCP)** — for AI that retrains, define in advance what changes are allowed without a new submission
- **Real-world performance monitoring** — clinical AI requires ongoing performance monitoring, not just pre-release validation

See `genai-and-phi.md` for AI-specific GxP validation patterns.

### Validated CI/CD for SaMD

The tension between SaaS velocity and GxP validation is solvable with a validated pipeline. See `resilience-and-deployment.md` for validated deployment pipeline patterns. Key principles for SaMD:

- **Pipeline is qualified** — the CI/CD pipeline itself is a Category 4 configured system; validate it once, inherit that validation for every release
- **Automated validation gates** — deployment cannot proceed without passing: unit tests, integration tests, regression tests, clinical validation tests, security scans
- **Immutable evidence** — every build produces artifacts stored in S3 with Object Lock: test results, scan reports, deployment logs
- **Traceability** — every deployed version traces to: commit SHA, test run ID, reviewer approvals, release notes
- **Feature flags separate deployment from activation** — SaMD features activate only after validation completes; non-regulated features can deploy continuously

## Common Mistakes

1. **Underestimating DICOM data volumes.** A single CT study can be 500 MB. A busy imaging center generates 100+ GB/day. Plan storage, network, and cost accordingly.

2. **Not using HealthImaging for DICOM storage.** Building a custom DICOM archive on S3 is possible but you lose sub-second retrieval, automatic tiering, and DICOMweb API support. Use the managed service.

3. **Ignoring hybrid requirements.** Most hospitals need some on-prem component (DICOM router, local cache). If your architecture is cloud-only, you'll lose deals to competitors that support hybrid.

4. **Treating FDA/SaMD as an afterthought.** If your AI features require FDA clearance, the regulatory requirements affect architecture from day one (audit trails, version control, validated pipelines). Retrofitting is extremely expensive.

5. **No pre-fetch strategy for viewer performance.** Radiologists expect instant image display. Without pre-fetching (hanging protocols, prior studies), your viewer will feel slow compared to on-prem PACS.

6. **GPU costs for 3D rendering without auto-scaling.** GPU instances are expensive. If you keep them running 24/7 for occasional 3D rendering, costs will be unsustainable. Use session-based allocation with auto-scaling.

7. **Treating CI/CD velocity and GxP validation as mutually exclusive.** They're not. Build a validated pipeline once, then ship continuously within that validated envelope. See `resilience-and-deployment.md`.

8. **Classifying every component as Class C.** Over-classification wastes validation effort. A scheduling module that can't cause patient harm is Class A, even if it's in the same product as a Class B triage AI. Segregate by class.

9. **No Predetermined Change Control Plan for AI that retrains.** If your model retrains on new data, every retraining triggers full re-validation unless you have a PCCP approved by FDA in advance. Plan PCCP into the architecture from day one for retraining AI.

## Discovery Questions for This Domain

**Imaging data:**
- What imaging modalities do your tenants use? (CT, MRI, X-ray, ultrasound, mammography, pathology?)
- What's the expected data volume per tenant? (Studies/day, GB/day)
- How long must images be retained? (State laws vary: 5-10 years for adults, until age 21+ for minors)

**Integration:**
- Do tenants have existing PACS/VNA systems? (If yes, you're integrating, not replacing)
- What DICOM protocols do you need to support? (DICOMweb, DIMSE, both?)
- Do you need on-prem components? (DICOM router, local cache, edge processing)
- What connectivity do hospital sites have? (Internet, VPN, Direct Connect?)

**Viewer:**
- Do you provide a viewer? (Zero-footprint browser, thick client, both?)
- Do you need 3D rendering / advanced visualization? (GPU requirements)
- What are the latency requirements for image display? (Sub-second for radiology workflow)

**AI/FDA:**
- Does your software include AI-assisted clinical features? (Detection, triage, measurement, diagnosis?)
- Is your software FDA-cleared or seeking clearance? (510(k), De Novo?)
- Do you have a QMS (ISO 13485) in place?
- How do you manage AI model versions and validation?

**GxP specific:**
- Have you classified components per IEC 62304 software safety class (A/B/C)?
- Have you categorized AWS services per GAMP 5 (1/3/4/5)?
- Is your CI/CD pipeline qualified for regulated releases?
- For AI that retrains: do you have a Predetermined Change Control Plan (PCCP)?
- Is the Risk Management File (ISO 14971) current and version-controlled?

## References

- [AWS HealthImaging](https://aws.amazon.com/healthimaging/)
- [HealthImaging Developer Guide](https://docs.aws.amazon.com/healthimaging/latest/devguide/what-is.html)
- [Medical Imaging System Reference Architecture](https://docs.aws.amazon.com/wellarchitected/latest/healthcare-industry-lens/medical-imaging-system-architecture.html)
- [Konica Minolta HealthImaging Case Study](https://aws.amazon.com/solutions/case-studies/konica-minolta-healthimaging-case-study/)
- [Intelerad Expands Alliance with AWS for Enterprise Imaging](https://www.intelerad.com/en/press-releases/intelerad-expands-strategic-alliance-with-aws-to-advance-enterprise-imaging-in-the-cloud/)
- [Building a Scalable DICOM Ingestion Pipeline for HealthImaging](https://aws.amazon.com/blogs/apn/building-a-scalable-dicom-ingestion-pipeline-for-aws-healthimaging-with-citiustech/)
- [Simplifying Medical Imaging AI Deployments with NVIDIA NIMs and AWS](https://aws.amazon.com/blogs/industries/simplifying-medical-imaging-ai-deployments-with-nvidia-nims-and-aws-services/)
- [Philips Prototypes Near-Real-Time Inference Platform for Medical Imaging](https://aws.amazon.com/blogs/industries/philips-prototypes-a-large-scale-near-real-time-inference-platform-to-extend-medical-imaging-using-aws/)
- [FDA SaMD Guidance](https://www.fda.gov/medical-devices/software-medical-device-samd)
- [IEC 62304 Medical Device Software Lifecycle](https://www.iso.org/standard/71604.html)
- [ISO 13485 Medical Devices QMS](https://www.iso.org/standard/59752.html)
- [ISO 14971 Risk Management for Medical Devices](https://www.iso.org/standard/72704.html)
- [Good Machine Learning Practice (GMLP) Guiding Principles](https://www.fda.gov/medical-devices/software-medical-device-samd/good-machine-learning-practice-medical-device-development-guiding-principles)
- [Predetermined Change Control Plans for AI/ML-Enabled Device Software](https://www.fda.gov/regulatory-information/search-fda-guidance-documents/marketing-submission-recommendations-predetermined-change-control-plan-artificial-intelligence)
- [GxP on AWS Whitepaper](https://aws.amazon.com/compliance/gxp-part-11-annex-11/)
