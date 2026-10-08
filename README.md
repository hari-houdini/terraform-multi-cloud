# Terraform Multi-Cloud PoC with Floci

## Context

You're building a learning/PoC project to demonstrate infrastructure-as-code patterns across AWS, GCP, and Azure using Terraform. All testing and deployment happens locally against Floci emulator variants (floci, floci-az, floci-gcp) — no real cloud accounts required. This pipeline demonstrates:

- **Multi-cloud Terraform architecture**: How to structure code for three clouds
- **Local development flow**: Docker Compose + Makefile for frictionless iteration
- **Multi-tier application**: Frontend (S3/Blob), backend (Compute), database (SQL), networking, load balancing, IAM
- **CI/CD automation**: GitHub Actions that runs all validation/testing against Floci
- **Comprehensive testing**: Code validation, security scanning (Checkov), infrastructure tests (Terratest)
- **Environment promotion**: dev → staging → prod with isolated state per environment

## Recommended Approach

### 1. Repository Structure (Monorepo)

```
terraform-multi-cloud/
├── .github/
│   └── workflows/
│       ├── validate.yml          # terraform validate, fmt, checkov
│       ├── test.yml              # terratest for all clouds
│       └── apply.yml             # apply to staging/prod (on merge to main)
├── docker-compose.yml            # Starts floci, floci-az, floci-gcp on different ports
├── Makefile                       # Unified targets for all clouds and environments
├── terraform/
│   ├── aws/
│   │   ├── dev/
│   │   ├── staging/
│   │   ├── prod/
│   │   └── modules/              # VPC, EC2, RDS, ALB, S3, IAM, Lambda, SNS/SQS
│   ├── gcp/
│   │   ├── dev/
│   │   ├── staging/
│   │   ├── prod/
│   │   └── modules/              # VPC, Compute Engine, Cloud SQL, LB, Storage, IAM, Cloud Run, Pub/Sub
│   └── azure/
│       ├── dev/
│       ├── staging/
│       ├── prod/
│       └── modules/              # vNet, VMs, Azure Database, App Gateway, Blob Storage, IAM, Function Apps, Service Bus
├── tests/
│   └── terratest/
│       ├── aws_test.go           # Terratest suite for AWS resources
│       ├── gcp_test.go           # Terratest suite for GCP resources
│       └── azure_test.go         # Terratest suite for Azure resources
├── README.md                      # Setup and usage guide
└── docs/
    ├── ARCHITECTURE.md           # Design decisions per cloud
    ├── LOCAL_SETUP.md            # How to run locally
    └── CI_CD.md                  # GitHub Actions flow
```

### 2. Docker Compose Setup

Single `docker-compose.yml` starts all three Floci variants on different ports:

```yaml
services:
  floci-aws:
    image: ghcr.io/getfloci/floci:latest
    ports:
      - "4566:4566"           # AWS endpoint
    environment:
      - DEBUG=0
      - SERVICES=ec2,s3,rds,elasticloadbalancing,iam,lambda,sns,sqs

  floci-azure:
    image: ghcr.io/getfloci/floci-az:latest
    ports:
      - "4567:4567"           # Azure emulator endpoint
    environment:
      - AZURE_STORAGE_EMULATOR_ENABLE=true

  floci-gcp:
    image: ghcr.io/getfloci/floci-gcp:latest
    ports:
      - "4568:4568"           # GCP emulator endpoint
```

Makefile targets allow selective startup:
- `make dev-up` — start all three
- `make dev-aws` — start only AWS Floci
- `make dev-down` — stop all

### 3. Terraform Structure Per Cloud

Each cloud (`terraform/{aws,gcp,azure}`) mirrors the others structurally but uses provider-native resources.

**Example: AWS**

```
terraform/aws/
├── modules/
│   ├── vpc/
│   │   ├── main.tf, variables.tf, outputs.tf
│   ├── compute/
│   │   ├── main.tf (EC2 instances, security groups)
│   ├── database/
│   │   ├── main.tf (RDS)
│   ├── networking/
│   │   ├── main.tf (ALB, route53)
│   ├── storage/
│   │   ├── main.tf (S3)
│   ├── iam/
│   │   ├── main.tf (roles, policies)
│   ├── compute-serverless/
│   │   ├── main.tf (Lambda)
│   └── messaging/
│       ├── main.tf (SNS, SQS)
├── dev/
│   ├── main.tf             # Instantiate modules for dev environment
│   ├── terraform.tfvars
│   └── backend.tf          # Local state for dev
├── staging/
│   ├── main.tf             # Same modules, different vars (more resources)
│   ├── terraform.tfvars
│   └── backend.tf
└── prod/
    ├── main.tf
    ├── terraform.tfvars
    └── backend.tf          # In CI/CD: remote backend in Floci
```

**Reusable pattern across clouds**:
- `vpc` module — networks, subnets, security
- `compute` module — instances (EC2/Compute Engine/VMs)
- `database` module — managed SQL databases
- `networking` module — load balancers, DNS
- `storage` module — object storage
- `iam` module — identities, permissions
- `compute-serverless` module — Lambda/Cloud Run/Function Apps
- `messaging` module — SNS/SQS, Pub/Sub, Service Bus

Each module is cloud-specific internally (e.g., `aws/modules/compute/` uses `aws_instance`, `gcp/modules/compute/` uses `google_compute_instance`).

### 4. Makefile Unified Targets

```makefile
# Local development
dev-up:
	docker-compose up -d

dev-down:
	docker-compose down

dev-aws:
	docker-compose up -d floci-aws

dev-gcp:
	docker-compose up -d floci-gcp

dev-azure:
	docker-compose up -d floci-azure

# Validation (runs in CI)
validate:
	terraform -chdir=terraform/aws validate
	terraform -chdir=terraform/gcp validate
	terraform -chdir=terraform/azure validate

fmt-check:
	terraform fmt -check -recursive terraform/

fmt-fix:
	terraform fmt -recursive terraform/

# Security scanning
scan:
	checkov -d terraform/aws
	checkov -d terraform/gcp
	checkov -d terraform/azure

# Per-cloud, per-environment operations
plan-aws-dev:
	terraform -chdir=terraform/aws/dev plan

apply-aws-dev:
	terraform -chdir=terraform/aws/dev apply

destroy-aws-dev:
	terraform -chdir=terraform/aws/dev destroy

# ... (repeat for each cloud × environment, or use a parameterized pattern)

# Testing
test:
	cd tests && go test -v ./terratest
```

### 5. GitHub Actions CI/CD Pipeline

Three workflows, all use Floci (no real AWS/GCP/Azure accounts):

**`.github/workflows/validate.yml`** (on PR):
```yaml
on: [pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v2
      - run: make validate
      - run: make fmt-check
      - run: make scan  # Checkov
```

**`.github/workflows/test.yml`** (on PR):
```yaml
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      floci-aws:
        image: ghcr.io/getfloci/floci:latest
        options: -p 4566:4566
      # ... floci-az, floci-gcp similarly
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v2
      - uses: actions/setup-go@v4
      - run: make test  # Terratest
```

**`.github/workflows/apply.yml`** (on merge to main, for staging/prod):
```yaml
on:
  push:
    branches: [main]
jobs:
  apply-staging:
    runs-on: ubuntu-latest
    services:
      floci-aws:
        image: ghcr.io/getfloci/floci:latest
        options: -p 4566:4566
      # ... floci-az, floci-gcp
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v2
      - run: terraform -chdir=terraform/aws/staging init
      - run: terraform -chdir=terraform/aws/staging apply -auto-approve
      # ... repeat for GCP, Azure
```

### 6. Testing Strategy

**Checkov** (free, cloud-agnostic policy engine):
- Runs on all Terraform before apply
- Catches security misconfigs, compliance issues
- Part of CI pipeline and local `make scan`

**Terratest** (Go-based, industry standard):
- Located in `tests/terratest/`
- Test files: `aws_test.go`, `gcp_test.go`, `azure_test.go`
- Example test structure:
  ```go
  func TestAWSVPCCreated(t *testing.T) {
    terraformOptions := &terraform.Options{
      TerraformDir: "../../terraform/aws/dev",
    }
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    // Assert VPC exists, has correct CIDR, etc.
  }
  ```
- Tests run against local Floci endpoints (AWS: localhost:4566, etc.)

### 7. Dummy Multi-Tier Application

**Per-cloud resources:**

| Layer | AWS | GCP | Azure |
|-------|-----|-----|-------|
| **Frontend** | S3 (static site) + CloudFront (optional) | Cloud Storage + CDN | Blob Storage + CDN |
| **Backend** | EC2 (app server) in Auto Scaling Group | Compute Engine instances + Instance Group | VMs in Scale Set |
| **Database** | RDS (MySQL/PostgreSQL) | Cloud SQL | Azure Database for MySQL/PostgreSQL |
| **Load Balancer** | Application Load Balancer (ALB) | Cloud Load Balancer | Application Gateway |
| **Networking** | VPC, subnets, route tables, security groups | VPC, subnets, firewall rules | vNet, subnets, NSGs |
| **IAM** | IAM roles, policies, instance profiles | Service accounts, roles | Managed Identities, roles |
| **Serverless** | Lambda function + API Gateway | Cloud Run | Azure Function App |
| **Messaging** | SNS (pub/sub) + SQS (queue) | Pub/Sub | Service Bus |

Each cloud's modules instantiate these with cloud-native tools.

### 8. Local Development Workflow

1. Clone repo
2. `make dev-up` → start all three Floci emulators
3. `cd terraform/aws/dev && terraform init` → initialize AWS provider (points to localhost:4566)
4. Edit code → `make validate`, `make fmt-fix`, `make scan`
5. `make plan-aws-dev` → review plan
6. `make apply-aws-dev` → deploy to local Floci
7. Test via AWS CLI pointing to localhost:4566 (e.g., `aws ec2 describe-instances --endpoint-url http://localhost:4566`)
8. `make destroy-aws-dev` → clean up

### 9. Documentation

**`README.md`**:
- Project overview
- Quick start (clone → `make dev-up`)
- Makefile command reference

**`docs/LOCAL_SETUP.md`**:
- Detailed Docker/Floci setup
- Environment variables per cloud
- Troubleshooting (port conflicts, Floci service limits)

**`docs/ARCHITECTURE.md`**:
- Why three separate cloud directories (not shared abstractions)
- Module design per cloud
- Resource naming conventions
- How to add a new resource/module

**`docs/CI_CD.md`**:
- GitHub Actions workflow overview
- How to add credentials for real clouds later (just change endpoints)
- Manual vs. automated apply gates

## Critical Files to Create

1. **`docker-compose.yml`** — Multi-Floci service definitions
2. **`Makefile`** — Tiered targets (dev-up, validate, plan-*, apply-*, scan, test)
3. **`terraform/aws/modules/`** through **`terraform/azure/modules/`** — Reusable resource modules per cloud
4. **`terraform/{aws,gcp,azure}/{dev,staging,prod}/main.tf`** — Environment-specific instantiation of modules
5. **`tests/terratest/{aws,gcp,azure}_test.go`** — Infrastructure tests
6. **`.github/workflows/validate.yml`, `test.yml`, `apply.yml`** — CI/CD pipelines
7. **`docs/` folder** — Architecture, setup, CI/CD guides

## Verification

**Local flow**:
1. `make dev-up` → all Floci services online
2. `make validate && make fmt-check && make scan` → no errors
3. `make plan-aws-dev` → shows 20–30 resources (VPC, subnets, EC2, RDS, ALB, S3, IAM, Lambda, SNS/SQS)
4. `make apply-aws-dev` → resources created in Floci
5. `aws s3 ls --endpoint-url http://localhost:4566` → see S3 bucket
6. `make destroy-aws-dev` → clean teardown

**CI/CD flow**:
1. Create PR with Terraform changes
2. GitHub Actions runs validate → test → scan workflows
3. All pass against Floci emulators
4. Merge to main triggers apply workflow
5. Staging deployment succeeds (all resources visible via Terraform state)

---

**Next steps**: Start by creating the directory structure from section 1, then build out `docker-compose.yml`, `Makefile`, and the first Terraform module (e.g., VPC for AWS).
