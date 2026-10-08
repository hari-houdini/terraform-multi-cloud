# terraform-multi-cloud: Project Context

## Project Summary

**terraform-multi-cloud** is a **reference architecture** for multi-cloud infrastructure using Terraform. It demonstrates patterns for deploying identical or complementary workloads across AWS, GCP, and Azure using a unified IaC framework. The project is currently in **documentation-first phase**: it contains reference patterns, decision records, and a learning POC (Payload CMS + AWS), but not yet live infrastructure.

**Target audience**: Developers learning cloud architecture and Terraform patterns; teams evaluating multi-cloud strategies.

---

## Current State

### Phase: Documentation & Learning POCs

- ✅ **Multi-cloud architecture patterns** documented (`docs/ARCHITECTURE.md`, `docs/LOCAL_SETUP.md`, `docs/CI_CD.md`)
- ✅ **Terraform module structure** designed (VPC, compute, database, storage, IAM, serverless, messaging per cloud)
- ✅ **Payload CMS learning POC** with journey documentation and architecture diagrams
- ⏳ **Actual Terraform code**: Not yet implemented (next phase)
- ⏳ **Live deployments**: Not yet tested

### Key Documentation

- `docs/ARCHITECTURE.md` — Multi-cloud service mapping (AWS ↔ GCP ↔ Azure)
- `docs/LOCAL_SETUP.md` — Floci emulator setup for local development
- `docs/CI_CD.md` — GitHub Actions patterns for Terratest + Checkov
- `docs/CMS_JOURNEY.md` — End-to-end user/data flows for Payload CMS POC
- `docs/CMS_ARCHITECTURE_DIAGRAM.md` — Service interactions and sequence diagrams
- `docs/diagrams/*.mmd` — Mermaid diagrams (architecture, auth, caching, etc.)

---

## Architecture Overview

### Multi-Cloud Service Mapping

The project maps **equivalent services** across three clouds:

| Layer | AWS | GCP | Azure |
|-------|-----|-----|-------|
| **Networking** | VPC + Security Groups | VPC + Firewall | VNET + NSG |
| **Compute** | EC2 + Auto Scaling | Compute Engine | VMs + Scale Sets |
| **Serverless** | Lambda | Cloud Run | Functions |
| **Database** | RDS (SQL) + DynamoDB | Cloud SQL + Firestore | Azure Database + Cosmos |
| **Storage** | S3 | Cloud Storage | Blob Storage |
| **CDN** | CloudFront | Cloud CDN | Azure CDN |
| **Load Balancing** | ALB/NLB | Cloud LB | App Gateway |
| **Messaging** | SNS/SQS | Pub/Sub | Service Bus |
| **Identity** | Cognito + IAM | Cloud Identity | Azure AD |

### Terraform Module Structure (Planned)

Each cloud has a **modular architecture** under `terraform/{aws|gcp|azure}/`:

```
terraform/{cloud}/
├── modules/
│   ├── vpc/              # Networking
│   ├── compute/          # Instances
│   ├── database/         # Managed databases
│   ├── storage/          # Object storage
│   ├── iam/              # Identity & access
│   ├── networking/       # Load balancers
│   ├── compute-serverless/ # Lambda/Cloud Run/Functions
│   └── messaging/        # Pub/Sub, SNS/SQS, Service Bus
├── dev/                  # Development environment
├── staging/              # Staging environment
└── prod/                 # Production environment
```

**Design principle**: Modules are cloud-agnostic _templates_, not cloud-specific implementations. A `vpc` module defines the _concept_ of a VPC (networking, isolation, security groups), and each cloud's implementation fills in the details.

---

## Payload CMS Learning POC

The project includes a **learning POC** demonstrating multi-cloud infrastructure with a real application.

### POC Architecture

- **Application**: Payload CMS (headless, REST API)
- **ORM**: Prisma (schema-first, type-safe)
- **Database**: RDS PostgreSQL (portable across clouds)
- **Media Storage**: S3 / Cloud Storage / Blob Storage
- **CDN**: CloudFront / Cloud CDN / Azure CDN
- **Auth**: Cognito / Cloud Identity / Azure AD
- **Compute**: Lambda / Cloud Run / Functions (serverless, scales to zero)

### POC Scope

The CMS demonstrates **three user journeys**:
1. **Admin publishes blog post** — Cognito auth → API Gateway → Lambda → Prisma → RDS (post saved) → S3 (image uploaded) → CloudFront (cache invalidated)
2. **Public reads blog post** — CloudFront (cached) → API Gateway → Prisma (auto-joins author/categories/tags) → S3 images
3. **Admin forgot password** — Cognito + email reset → RDS update via Prisma

Each journey shows _which AWS service handles each step_ and _why_, teaching infrastructure patterns through a real workflow.

---

## Key Constraints

### What This Project Is

- A **reference architecture**: shows patterns, not production-ready code
- A **learning resource**: for developers new to multi-cloud or Terraform
- **Documentation-first**: patterns are documented before code is written
- **Single ORM choice**: Prisma (PostgreSQL) as the learning focus for CMS POC

### What This Project Is NOT

- A **complete framework** for every use case
- **Production-ready infrastructure** (no deployed resources yet)
- A **competitor to existing platforms** (Terraform Cloud, Spacelift, Pulumi)
- **Cloud-agnostic boilerplate** (patterns are cloud-specific, intentionally)

### Design Constraints

1. **Terraform as the only IaC tool** — all clouds defined in HCL, no Pulumi/CDK/ARM
2. **Modular, not monolithic** — each cloud's modules are independent and testable
3. **Local emulation first** — Floci emulators (localstack, fake-gcp, fake-azure) for offline development
4. **CI/CD patterns, not secrets management** — GitHub Actions + Terratest, but no vault setup
5. **Single database per POC** — PostgreSQL only (DynamoDB is mentioned as alternative, not implemented)

---

## Non-Goals

- Full disaster recovery / high availability patterns
- Cost optimization across clouds
- Multi-region failover
- Enterprise compliance (SOC2, HIPAA, etc.)
- Custom resource development (only managed services)

---

## Critical Code Paths

### For Module Developers

1. **Writing a module** → Create `terraform/{cloud}/modules/{service}/main.tf` + `variables.tf` + `outputs.tf`
2. **Testing a module** → Write Terratest in Go under `test/` with localstack/Floci
3. **Integrating across modules** → Use data sources to reference outputs from other modules

### For POC Developers

1. **Payload CMS setup** → Create Node.js + Prisma project under `payload-cms/`
2. **Prisma schema** → Define data model (Posts, Authors, Categories, Tags) in `schema.prisma`
3. **Deploying CMS** → Use Terraform to spin up RDS, Lambda/EC2, S3, Cognito

### For Documentation Readers

1. **Starting point** → `docs/ARCHITECTURE.md` (service mapping)
2. **Local setup** → `docs/LOCAL_SETUP.md` (Floci emulators)
3. **CI/CD patterns** → `docs/CI_CD.md` (GitHub Actions)
4. **CMS POC details** → `docs/CMS_JOURNEY.md` (user flows) + `docs/CMS_ARCHITECTURE_DIAGRAM.md` (service interactions)

---

## Design Principles & Trade-Offs

### Principle: Transparency Over Abstraction

**Trade-off**: Modules show cloud-specific details (security groups, IAM roles) rather than hiding them. This makes code longer but teaches cloud concepts.

**Why**: A reference architecture's job is to teach, not to abstract. Developers must understand _why_ a security group is needed, not just that it exists.

### Principle: Modular, Not Monolithic

**Trade-off**: Each cloud is separate (no shared code between AWS/GCP/Azure modules). This means duplication.

**Why**: Each cloud has different operational models. Forced sharing would create a leaky abstraction where modules must know about all three clouds to work correctly.

### Principle: Documentation-First

**Trade-off**: Patterns are documented before code is written. This means code may lag behind docs.

**Why**: A reference architecture's value is in the _pattern_, not the _code_. Documenting the pattern first forces clarity before implementation details muddy it.

### Principle: Local Development First

**Trade-off**: Emulators (localstack, Floci) cannot reproduce all cloud behavior. Some bugs only surface in real clouds.

**Why**: Learning should not require AWS/GCP/Azure accounts. Emulators teach the pattern without the cost.

### Principle: Single Application, Multiple Clouds

**Trade-off**: The Payload CMS POC is designed for AWS first. Multi-cloud deployment is future work.

**Why**: Easier to learn with one complete example than three partial ones. Once AWS pattern is clear, the Azure/GCP ports become pattern-matching exercises.

---

## How Agents Should Use This Context

### Before Writing Terraform Code
- Check `docs/ARCHITECTURE.md` for the service-mapping reference
- Review the appropriate module structure under `terraform/{cloud}/modules/`
- Look for existing modules that solve similar problems (don't duplicate)

### Before Writing CMS Application Code
- Read `docs/CMS_JOURNEY.md` to understand the user flows
- Review `docs/CMS_ARCHITECTURE_DIAGRAM.md` for service interactions
- Check the Prisma schema in `payload-cms/` (if it exists)

### Before Adding Documentation
- Follow the principle: **documentation-first design**
- Use `docs/` for architecture and patterns; use ADRs in `docs/adr/` for decisions
- Keep examples concrete and tied to the Payload CMS POC or actual modules

### Before Proposing Changes
- Ask: does this change align with the "reference architecture" goal, or is it adding production-specific concerns?
- Verify it fits the modular, multi-cloud structure (not AWS-only, not overly abstracted)
- Check if it belongs in documentation before code

---

## Next Phases (Not Yet Started)

### Phase 2: Implement Terraform Modules
- [ ] AWS modules (VPC, EC2, RDS, S3, CloudFront, IAM, etc.)
- [ ] GCP modules (equivalent services)
- [ ] Azure modules (equivalent services)
- [ ] Local Floci emulator setup

### Phase 3: Build Payload CMS Application
- [ ] Node.js + Payload CMS scaffolding
- [ ] Prisma schema with Posts, Authors, Categories, Tags
- [ ] S3 upload handling (presigned URLs)
- [ ] Cognito integration

### Phase 4: Deploy & Test
- [ ] Terraform deployments to AWS/GCP/Azure
- [ ] Terratest coverage for each module
- [ ] End-to-end CMS workflow testing
- [ ] GitHub Actions CI/CD validation

### Phase 5: Multi-Cloud POC
- [ ] Same CMS deployed on GCP and Azure
- [ ] Comparison of deployment differences
- [ ] Cost analysis across clouds
