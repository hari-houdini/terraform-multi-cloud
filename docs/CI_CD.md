# CI/CD Pipeline Guide

This document explains the GitHub Actions workflows that validate and deploy infrastructure changes across all three cloud providers.

## Overview

The project uses three GitHub Actions workflows to enforce code quality and automate deployment:

1. **Validate** (on PR): Syntax check, formatting, security scanning
2. **Test** (on PR): Infrastructure tests via Terratest
3. **Apply** (on merge to main): Deploy to staging and prod environments

All workflows run against **Floci emulators** — no real AWS/GCP/Azure accounts needed.

## Workflow Files

Located in `.github/workflows/`:

```
.github/workflows/
├── validate.yml      # terraform validate, fmt, checkov
├── test.yml          # terratest for all clouds
└── apply.yml         # apply to staging/prod environments
```

## Validate Workflow

**Trigger**: Pull Request (on push to any branch)

**File**: `.github/workflows/validate.yml`

### What It Does

1. Checks out code
2. Installs Terraform
3. Runs `terraform validate` for all three clouds (AWS, GCP, Azure)
4. Checks code formatting with `terraform fmt`
5. Scans for security issues with Checkov

### Example Workflow

```yaml
name: Validate Terraform

on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Format Check
        run: make fmt-check
      
      - name: Terraform Validate
        run: make validate
      
      - name: Security Scan (Checkov)
        run: make scan
```

### How to Make It Pass

1. **Fix syntax errors**:
   ```bash
   make validate
   ```

2. **Fix formatting**:
   ```bash
   make fmt-fix
   git add terraform/
   git commit -m "Fix Terraform formatting"
   git push
   ```

3. **Fix security issues** (from Checkov):
   - Review Checkov's report
   - Either fix the issue or add a `.checkov.yaml` exception with justification
   - Push changes

## Test Workflow

**Trigger**: Pull Request (on push to any branch)

**File**: `.github/workflows/test.yml`

### What It Does

1. Starts Floci emulators (AWS, Azure, GCP) as GitHub Actions services
2. Installs Terraform and Go
3. Runs Terratest suites for each cloud
4. Reports pass/fail

### Example Workflow

```yaml
name: Infrastructure Tests

on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      floci-aws:
        image: ghcr.io/getfloci/floci:latest
        options: -p 4566:4566
        env:
          DEBUG: '0'
          SERVICES: ec2,s3,rds,elasticloadbalancing,iam,lambda,sns,sqs
      
      floci-azure:
        image: ghcr.io/getfloci/floci-az:latest
        options: -p 4567:4567
      
      floci-gcp:
        image: ghcr.io/getfloci/floci-gcp:latest
        options: -p 4568:4568
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v2
      - uses: actions/setup-go@v4
      
      - name: Run Terratest
        run: make test
        env:
          TF_LOG: INFO
```

### Understanding Test Results

Each Terratest run outputs:

```
TestAWSVPCCreated .... PASS
TestAWSComputeInstances .... PASS
TestGCPNetworking .... PASS
TestAzureDatabase .... PASS
```

If a test fails, output includes:

```
TestAWSRDSCreated .... FAIL
    Error: Resource validation failed: ...
```

### Fixing Failed Tests

1. **Review the error** in GitHub Actions logs
2. **Reproduce locally**:
   ```bash
   make dev-up
   cd tests
   go test -v -run TestAWSRDSCreated ./terratest
   ```
3. **Fix the Terraform code**:
   ```bash
   cd terraform/aws/dev
   terraform apply  # Make the fix
   ```
4. **Re-run the test**:
   ```bash
   cd /path/to/tests
   go test -v -run TestAWSRDSCreated ./terratest
   ```
5. **Push the fix**:
   ```bash
   git add terraform/ tests/
   git commit -m "Fix RDS database creation in Terratest"
   git push
   ```

## Apply Workflow

**Trigger**: Push to `main` branch (after PR merge)

**File**: `.github/workflows/apply.yml`

### What It Does

1. Starts Floci emulators as services
2. Deploys infrastructure changes to **staging** and **prod** environments
3. Creates Terraform plans and applies them
4. Outputs resource details

### Example Workflow

```yaml
name: Deploy Infrastructure

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
      floci-azure:
        image: ghcr.io/getfloci/floci-az:latest
        options: -p 4567:4567
      floci-gcp:
        image: ghcr.io/getfloci/floci-gcp:latest
        options: -p 4568:4568
    
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v2
      
      - name: Deploy AWS Staging
        run: |
          cd terraform/aws/staging
          terraform init
          terraform apply -auto-approve
      
      - name: Deploy GCP Staging
        run: |
          cd terraform/gcp/staging
          terraform init
          terraform apply -auto-approve
      
      - name: Deploy Azure Staging
        run: |
          cd terraform/azure/staging
          terraform init
          terraform apply -auto-approve
  
  apply-prod:
    needs: apply-staging
    runs-on: ubuntu-latest
    
    services:
      # ... same Floci services
    
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v2
      
      - name: Deploy AWS Prod
        run: |
          cd terraform/aws/prod
          terraform init
          terraform apply -auto-approve
      
      # ... repeat for GCP, Azure
```

### Workflow Behavior

1. **On PR merge to main**, apply workflow triggers automatically
2. **Staging is deployed first** (lower-risk environment)
3. **If staging succeeds**, prod deployment proceeds
4. **All deployments are to Floci** (no real cloud resources)

### Monitoring Deployment

In GitHub Actions UI:

1. Navigate to **Actions** tab
2. Click the latest **"Deploy Infrastructure"** workflow run
3. Click the job to see logs:
   ```
   Deploy AWS Staging
   ✓ Fetching Terraform version
   ✓ terraform init
   ✓ terraform apply
   
   Apply complete! Resources added: 15, changed: 0, destroyed: 0.
   ```

## Customizing Workflows

### Adding a New Environment

To add a `qa` environment between staging and prod:

1. Create the directory:
   ```bash
   mkdir -p terraform/{aws,gcp,azure}/qa
   ```

2. Update `.github/workflows/apply.yml` to include:
   ```yaml
   apply-qa:
     needs: apply-staging
     runs-on: ubuntu-latest
     services: [...]
     steps:
       - uses: actions/checkout@v4
       - name: Deploy AWS QA
         run: cd terraform/aws/qa && terraform init && terraform apply -auto-approve
       # ... repeat for GCP, Azure
   ```

3. Push and merge to main

### Skipping a Step

If a test is flaky or you need to skip it temporarily (only in emergencies):

Add a `if` condition in the workflow:

```yaml
- name: Security Scan (Checkov)
  if: false  # Temporarily skip
  run: make scan
```

**Never commit a skipped step** — fix the underlying issue instead.

### Adding Slack Notifications

To get notified when deployments succeed/fail:

1. Create a Slack webhook: https://api.slack.com/messaging/webhooks
2. Add GitHub secret: Settings → Secrets → `SLACK_WEBHOOK_URL`
3. Add notification step:
   ```yaml
   - name: Notify Slack
     if: always()
     uses: slackapi/slack-github-action@v1
     with:
       webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
       payload: |
         {
           "text": "Deployment ${{ job.status }}: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
         }
   ```

## Troubleshooting Workflows

### Workflow Doesn't Trigger

**Problem**: PR doesn't trigger validate workflow

**Solution**: 
- Check branch protection settings (Settings → Branches → main)
- Ensure `on: [pull_request]` is in workflow file
- Push to a new branch, create PR from that branch

### Floci Service Timeout

**Problem**: `error: connection refused` in workflow logs

**Solution**: Increase startup wait in workflow:

```yaml
- name: Wait for Floci
  run: sleep 10 && curl -f http://localhost:4566/_localstack/health || true
```

### Terraform State Lock

**Problem**: `Error: resource locked` during apply

**Solution**: This shouldn't happen in Floci, but if it does:

```bash
# Manually unlock (use carefully!)
terraform force-unlock <LOCK_ID>
```

### Large State File

**Problem**: Workflow is slow, times out

**Solution**: 
- Consider splitting into smaller workspaces
- Use `terraform state` to move resources to separate state files
- Optimize expensive resource creation (e.g., RDS can take time in emulator)

## Scaling to Real Clouds

### Switching from Floci to AWS/GCP/Azure

To deploy to real cloud accounts after learning with Floci:

1. **Update backend configuration**:
   ```hcl
   # terraform/aws/staging/backend.tf
   # Remove Floci-specific settings
   terraform {
     backend "s3" {
       bucket         = "my-terraform-state-staging"
       key            = "aws/staging/terraform.tfstate"
       region         = "us-east-1"
       # Remove endpoint and skip_*_validation
     }
   }
   ```

2. **Add cloud credentials to GitHub Secrets**:
   ```
   Settings → Secrets → New secret
   Name: AWS_ACCESS_KEY_ID
   Value: <your_key>
   
   Name: AWS_SECRET_ACCESS_KEY
   Value: <your_secret>
   ```

3. **Update workflow to use credentials**:
   ```yaml
   - name: Deploy AWS Staging
     env:
       AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
       AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
     run: cd terraform/aws/staging && terraform apply -auto-approve
   ```

4. **Remove Floci services** from workflow, keep only real cloud setup

This is a seamless transition once you're confident with Terraform patterns.

## Best Practices

1. **Always review PR validation results** before merging
2. **Run tests locally before pushing**: `make validate && make test`
3. **Keep Makefile targets in sync** with GitHub workflows
4. **Test prod changes** in staging first
5. **Monitor Floci logs** if deployments fail: `docker logs floci-aws`
6. **Commit lock files**: `.terraform.lock.hcl` ensures reproducible provider versions
7. **Use descriptive commit messages**: Reference the issue/feature being deployed

## Related Docs

- **ARCHITECTURE.md** — Module structure and design decisions
- **LOCAL_SETUP.md** — Running workflows locally with `make` commands
- **README.md** — Quick start and command reference
