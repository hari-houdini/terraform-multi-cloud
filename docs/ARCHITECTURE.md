# Architecture & Design Decisions

## Overview

This project demonstrates multi-cloud infrastructure-as-code patterns using Terraform with three separate cloud providers: AWS, GCP, and Azure. Each provider has its own fully independent Terraform configuration while maintaining a consistent logical structure for learning purposes.

## Why Separate Cloud Directories (Not Shared Abstractions)?

### Decision

We use **separate, independent Terraform directories per cloud** (`terraform/aws/`, `terraform/gcp/`, `terraform/azure/`) rather than a single abstraction layer.

### Rationale

1. **Clarity for Learning**: Each cloud's implementation is self-contained and explicit. You can understand AWS concepts without being abstracted by GCP or Azure layers.

2. **Provider Differences**: While the logical multi-tier architecture is the same, cloud providers differ significantly:
   - AWS uses Security Groups; Azure uses Network Security Groups; GCP uses Firewall Rules
   - AWS ALB != GCP Cloud Load Balancer != Azure Application Gateway (different features, syntax, capabilities)
   - Instance naming, disk management, IAM models all differ
   - A shared abstraction would hide these differences or become overly complex

3. **Real-World Practice**: Teams typically maintain separate Terraform state and code per cloud environment. This mirrors actual production scenarios.

4. **Future Refactoring**: Once you're comfortable with all three clouds independently, you can refactor into a unified abstraction layer if needed.

## Module Structure

Each cloud has identical **conceptual modules** but cloud-specific implementations:

```
terraform/{aws|gcp|azure}/
├── modules/
│   ├── vpc/              # Networking layer
│   ├── compute/          # Compute instances
│   ├── database/         # Managed databases
│   ├── networking/       # Load balancers, DNS
│   ├── storage/          # Object storage
│   ├── iam/              # Identity & access
│   ├── compute-serverless/  # Serverless compute
│   └── messaging/        # Pub/Sub and queues
├── dev/
├── staging/
└── prod/
```

### Module Responsibilities

#### **vpc**
- Create virtual network/VPC
- Define subnets (public/private)
- Configure routing tables
- Set up gateways (IGW, NAT, VPN)

**AWS**: `aws_vpc`, `aws_subnet`, `aws_route_table`, `aws_internet_gateway`
**GCP**: `google_compute_network`, `google_compute_subnetwork`, `google_compute_route`
**Azure**: `azurerm_virtual_network`, `azurerm_subnet`, `azurerm_route_table`, `azurerm_nat_gateway`

#### **compute**
- Launch instances (EC2 / Compute Engine / VMs)
- Configure security groups / firewall rules
- Set up user data / initialization scripts
- Manage instance metadata and tags

**AWS**: `aws_instance`, `aws_security_group`, `aws_launch_template` (for ASG)
**GCP**: `google_compute_instance`, `google_compute_firewall`, `google_compute_instance_template`
**Azure**: `azurerm_virtual_machine`, `azurerm_network_security_group`, `azurerm_virtual_machine_scale_set`

#### **database**
- Provision managed relational database
- Configure multi-AZ / regional replication
- Set up backups and monitoring
- Create database instances and initial schemas

**AWS**: `aws_rds_cluster`, `aws_rds_cluster_instance`, `aws_db_parameter_group`
**GCP**: `google_sql_database_instance`, `google_sql_database`, `google_sql_user`
**Azure**: `azurerm_mssql_server`, `azurerm_mssql_database`, or `azurerm_postgresql_server`

#### **networking**
- Deploy load balancers (ALB / CLB / App Gateway)
- Configure target groups / backend pools
- Set up health checks
- Optional: DNS configuration

**AWS**: `aws_lb`, `aws_lb_target_group`, `aws_lb_listener`, `aws_route53_record`
**GCP**: `google_compute_health_check`, `google_compute_backend_service`, `google_compute_url_map`, `google_compute_global_forwarding_rule`
**Azure**: `azurerm_application_gateway`, `azurerm_application_gateway_backend_address_pool`, `azurerm_lb`

#### **storage**
- Object storage buckets (S3 / GCS / Blob Storage)
- Configure bucket policies / access control
- Enable versioning, lifecycle rules
- Optional: CDN configuration

**AWS**: `aws_s3_bucket`, `aws_s3_bucket_versioning`, `aws_s3_bucket_lifecycle_configuration`, `aws_cloudfront_distribution`
**GCP**: `google_storage_bucket`, `google_storage_bucket_object`, `google_compute_backend_bucket` (for CDN)
**Azure**: `azurerm_storage_account`, `azurerm_storage_container`, `azurerm_cdn_profile`

#### **iam**
- Define roles, service accounts, identities
- Create policies and permission assignments
- Set up trust relationships / assume roles
- Instance profiles / workload identities

**AWS**: `aws_iam_role`, `aws_iam_policy`, `aws_iam_role_policy_attachment`, `aws_instance_profile`
**GCP**: `google_service_account`, `google_project_iam_member`, `google_service_account_iam_member`
**Azure**: `azurerm_user_assigned_identity`, `azurerm_role_assignment`, `azurerm_key_vault_access_policy`

#### **compute-serverless**
- Serverless compute (Lambda / Cloud Run / Function Apps)
- Define function code, triggers, environment variables
- Configure concurrency and resource limits
- Optional: API Gateway / Cloud Tasks

**AWS**: `aws_lambda_function`, `aws_apigatewayv2_api`, `aws_apigatewayv2_integration`, `aws_lambda_permission`
**GCP**: `google_cloud_run_service`, `google_cloud_scheduler_job`, `google_cloud_run_service_iam_member`
**Azure**: `azurerm_function_app`, `azurerm_function_app_function`, `azurerm_app_service_plan`

#### **messaging**
- Pub/Sub or queue services
- Topics, subscriptions, queues
- Retention policies and delivery settings

**AWS**: `aws_sns_topic`, `aws_sqs_queue`, `aws_sns_topic_subscription`
**GCP**: `google_pubsub_topic`, `google_pubsub_subscription`, `google_pubsub_topic_iam_member`
**Azure**: `azurerm_servicebus_namespace`, `azurerm_servicebus_topic`, `azurerm_servicebus_queue`, `azurerm_servicebus_subscription`

## Environment-Specific Layers (dev, staging, prod)

Each cloud has **three identical structural environments** with different variable sets:

```
terraform/aws/
├── dev/
│   ├── main.tf               # Instantiate modules
│   ├── terraform.tfvars      # Small instances, minimal redundancy
│   └── backend.tf            # Local state (.tfstate in .gitignore)
├── staging/
│   ├── main.tf
│   ├── terraform.tfvars      # Medium instances, some redundancy
│   └── backend.tf
└── prod/
    ├── main.tf
    ├── terraform.tfvars      # Large instances, full HA, multi-AZ
    └── backend.tf
```

### Example: Differences in `terraform.tfvars`

**aws/dev/terraform.tfvars**:
```hcl
environment = "dev"
instance_count = 1
instance_type = "t2.micro"
rds_instance_class = "db.t2.micro"
enable_multi_az = false
enable_backup = false
enable_monitoring = false
```

**aws/prod/terraform.tfvars**:
```hcl
environment = "prod"
instance_count = 3
instance_type = "t3.large"
rds_instance_class = "db.r6i.xlarge"
enable_multi_az = true
enable_backup = true
enable_monitoring = true
backup_retention_days = 30
```

The `main.tf` in each environment instantiates modules with these variables, demonstrating **environment promotion**: same code, different resource tiers.

## Naming Conventions

### Resources

```
<project>-<environment>-<resource-type>-<identifier>
```

**Examples**:
- `terraform-dev-vpc-main`
- `terraform-prod-rds-postgres`
- `terraform-staging-alb-web`

### Tags

All resources include consistent tags:

```hcl
tags = {
  Project     = "terraform-multi-cloud"
  Environment = var.environment
  ManagedBy   = "Terraform"
  CreatedAt   = timestamp()
}
```

### Variables & Outputs

**Variables** (inputs to modules): descriptive, type-annotated
```hcl
variable "instance_count" {
  description = "Number of compute instances to launch"
  type        = number
  default     = 1
}
```

**Outputs** (return values from modules):
```hcl
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}
```

## Adding a New Resource / Module

1. **Create module directory**: `terraform/aws/modules/my-new-module/`
2. **Create `main.tf`**: Define resources using cloud-specific providers
3. **Create `variables.tf`**: Define all input variables with descriptions
4. **Create `outputs.tf`**: Export values needed by other modules
5. **Repeat for GCP and Azure** with equivalent cloud-native resources
6. **Instantiate in `terraform/{aws,gcp,azure}/{dev,staging,prod}/main.tf`**:
   ```hcl
   module "my_new_module" {
     source = "../modules/my-new-module"
     
     # Pass variables
     vpc_id = module.vpc.vpc_id
     environment = var.environment
   }
   ```
7. **Test locally**: `make plan-aws-dev`, review output
8. **Add Terratest**: Create a test in `tests/terratest/aws_test.go` to verify the resource exists

## State Management

### Local (dev)

`.tfstate` files are **git-ignored** (see `.gitignore`). Developers work with local state on their machine:

```bash
cd terraform/aws/dev
terraform init     # Creates .terraform/ and local tfstate
terraform plan
terraform apply
```

### CI/CD (staging, prod)

GitHub Actions uses **remote state backends** pointing to Floci emulator endpoints. State is stored in Floci's S3 emulation:

```hcl
# terraform/aws/staging/backend.tf
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "aws/staging/terraform.tfstate"
    region         = "us-east-1"
    endpoint       = "http://localhost:4566"  # Points to Floci
    skip_credentials_validation = true
    skip_region_validation      = true
  }
}
```

## Cloud-Specific Notes

### AWS
- Uses VPC for networking, EC2 for compute, RDS for databases
- Security Groups define ingress/egress; no explicit "subnets per AZ" needed (single subnet = AZ agnostic)
- ALB is standard for HA load balancing
- IAM roles are essential for EC2 → AWS service permissions

### GCP
- Uses VPC Networks (global construct), but subnets are regional
- Firewall rules are stateful (only egress rules needed in most cases)
- Compute Engine for VMs; similar pricing to AWS but with different instance families
- Service Accounts for workload identity; no instance profiles
- Cloud SQL supports MySQL, PostgreSQL, SQL Server

### Azure
- vNets are regional; subnets are contained within a vNet per region
- Network Security Groups (NSGs) are stateless (require both ingress AND egress rules)
- VMs are more tightly integrated with storage accounts
- Managed Identities (Azure's answer to service accounts) are simpler than AWS IAM
- Azure Database for MySQL/PostgreSQL uses different naming (not "RDS")

## Next Steps

1. Implement the VPC module for one cloud (e.g., AWS) to establish the pattern
2. Mirror that pattern for GCP and Azure
3. Implement compute, database, and networking modules
4. Add Terratest to verify resource creation
5. Set up GitHub Actions workflows
6. Test the full dev → staging → prod flow locally with Floci
