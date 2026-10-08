# Local Development Setup

This guide walks you through setting up your local development environment to work with the Terraform multi-cloud project using Floci emulators.

## Prerequisites

- **Docker** & **Docker Compose** installed (for running Floci emulators)
- **Terraform** (v1.5+) installed and in `PATH`
- **Make** installed (for running Makefile targets)
- **AWS CLI** (optional, for testing/debugging AWS resources in Floci)
- **Go** (optional, for running Terratest locally)

### Installation Quick Start

#### macOS (Homebrew)

```bash
brew install terraform docker docker-compose make
# Docker Desktop also installs Docker and Docker Compose
brew install awscli  # optional
brew install go      # optional
```

#### Linux (Ubuntu/Debian)

```bash
sudo apt-get update
sudo apt-get install -y terraform docker.io docker-compose make
sudo apt-get install -y awscli  # optional
sudo apt-get install -y golang-go  # optional
```

#### Windows

- Install **Docker Desktop** from https://www.docker.com/products/docker-desktop
- Install **Terraform** from https://www.terraform.io/downloads or via Chocolatey: `choco install terraform`
- Install **Make** via Chocolatey: `choco install make`

## Project Setup

### 1. Clone the Repository

```bash
git clone https://github.com/yourusername/terraform-multi-cloud.git
cd terraform-multi-cloud
```

### 2. Start Floci Emulators

All three cloud emulators run via Docker Compose:

```bash
make dev-up
```

This starts:
- **Floci (AWS)** on `http://localhost:4566`
- **Floci-Az (Azure)** on `http://localhost:4567`
- **Floci-GCP (GCP)** on `http://localhost:4568`

To verify they're running:

```bash
docker ps | grep floci
```

You should see three containers: `floci`, `floci-az`, `floci-gcp`.

### 3. Initialize Terraform for a Cloud

Let's start with AWS dev environment:

```bash
cd terraform/aws/dev
terraform init
```

This creates:
- `.terraform/` directory with provider plugins
- `.terraform.lock.hcl` (lock file for reproducible runs)

Repeat for other clouds/environments as needed:

```bash
cd ../../gcp/dev && terraform init
cd ../../azure/dev && terraform init
```

### 4. Validate Configuration

Check that all Terraform files are syntactically correct:

```bash
make validate
```

### 5. (Optional) Run Code Formatting Check

```bash
make fmt-check
```

To auto-fix formatting issues:

```bash
make fmt-fix
```

## Development Workflow

### Plan Changes

Before applying, always review the plan:

```bash
make plan-aws-dev
```

This shows exactly which resources will be created/modified/destroyed. Review carefully before applying.

### Apply Changes

```bash
make apply-aws-dev
```

Terraform will prompt for confirmation. Type `yes` to proceed.

### Verify Resources in Floci

After applying, verify resources exist using AWS CLI pointing to the Floci endpoint:

```bash
# List S3 buckets
aws s3 ls --endpoint-url http://localhost:4566

# List EC2 instances
aws ec2 describe-instances \
  --endpoint-url http://localhost:4566 \
  --region us-east-1

# Describe RDS databases
aws rds describe-db-instances \
  --endpoint-url http://localhost:4566 \
  --region us-east-1
```

**Note**: Floci uses default AWS credentials (dummy values). Environment setup is not required.

### Destroy Resources

When done testing, clean up:

```bash
make destroy-aws-dev
```

Type `yes` to confirm destruction.

## Environment Variables

### AWS-Specific

For AWS CLI commands against Floci, no auth needed, but you can set these:

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
```

### Terraform-Specific

To enable debug logging:

```bash
export TF_LOG=DEBUG
terraform plan  # Outputs verbose logs
unset TF_LOG
```

## Troubleshooting

### Floci Containers Won't Start

**Error**: `docker: permission denied while trying to connect to the Docker daemon`

**Solution**: Add your user to the `docker` group (Linux):

```bash
sudo usermod -aG docker $USER
newgrp docker  # Apply group membership
```

**Error**: `Port 4566 is already in use`

**Solution**: Another service is using the port. Stop it:

```bash
# Check what's using port 4566
lsof -i :4566
# Kill the process or stop the old container
docker stop $(docker ps -q --filter "expose=4566")
```

### Terraform Init Fails

**Error**: `Error: Failed to install provider`

**Solution**: Ensure Floci is running and accessible:

```bash
curl http://localhost:4566/_localstack/health
```

If Floci isn't responding, restart it:

```bash
make dev-down
make dev-up
sleep 5  # Wait for services to be ready
terraform init
```

### Terraform Apply Fails with Network Error

**Error**: `error: InvalidClientTokenId` or connection refused

**Solution**: 
1. Ensure Floci is running: `docker ps | grep floci`
2. Check Floci logs: `docker logs -f floci`
3. Restart Floci: `make dev-down && make dev-up`

### State File Conflicts

**Error**: `.terraform.lock.hcl` has different provider versions

**Solution**: Re-initialize and re-lock:

```bash
rm .terraform.lock.hcl
terraform init
```

### Floci Memory Limits

If Floci containers are slow or crash:

**Error**: `OOMKilled` or `exit code 137`

**Solution**: Increase Docker memory limits:

```bash
# Edit docker-compose.yml to add resource limits:
services:
  floci-aws:
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
```

Then restart:

```bash
make dev-down
make dev-up
```

### Can't Connect to GCP Emulator

**Error**: "unknown" or "connection refused" for GCP resources

**Solution**: Floci-GCP may take longer to start. Wait a moment and retry:

```bash
docker logs floci-gcp  # Check initialization
terraform plan -chdir=terraform/gcp/dev
```

## Advanced: Cloud-Specific Debugging

### AWS (Floci)

```bash
# List all S3 buckets
aws s3api list-buckets --endpoint-url http://localhost:4566

# Describe VPCs
aws ec2 describe-vpcs --endpoint-url http://localhost:4566 --region us-east-1

# Check security groups
aws ec2 describe-security-groups --endpoint-url http://localhost:4566 --region us-east-1
```

### GCP (Floci-GCP)

GCP emulator doesn't have a standard CLI; use Terraform directly to verify:

```bash
cd terraform/gcp/dev
terraform state list  # Show all resources
terraform state show google_compute_network.main  # Show specific resource details
```

### Azure (Floci-Az)

Azure CLI can point to the emulator (if supported), but Terraform is preferred:

```bash
cd terraform/azure/dev
terraform state list
terraform state show azurerm_virtual_network.main
```

## Cleaning Up

### Stop Floci (Keep Containers)

```bash
make dev-down
```

### Remove All Local State Files

```bash
find terraform -name ".terraform" -type d -exec rm -rf {} + 2>/dev/null || true
find terraform -name ".terraform.lock.hcl" -delete
find terraform -name "terraform.tfstate*" -delete
```

### Full Reset

```bash
make dev-down
rm -rf terraform/**/.terraform
rm -rf terraform/**/.terraform.lock.hcl
rm -rf terraform/**/terraform.tfstate*
```

## Next Steps

1. Review `ARCHITECTURE.md` to understand module structure
2. Start with the AWS dev environment to get familiar
3. Implement the VPC module and apply it locally
4. Move to other modules (compute, database, etc.)
5. Once comfortable, explore GCP and Azure equivalents
6. Read `CI_CD.md` to understand GitHub Actions workflows
