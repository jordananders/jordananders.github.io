---
layout: default
title:  "Infrastructure as Code with Terraform"
date:   2025-11-18 12:00:00
categories: DevOps Terraform IaC
---

Terraform enables declarative infrastructure management across any cloud. Here are best practices for modules, state management, and production workflows.

## Project Structure

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
├── modules/
│   ├── vpc/
│   ├── eks/
│   └── rds/
└── global/
    └── iam/
```

## Module Best Practices

### Creating Reusable Modules

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-vpc"
  })
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-private-${count.index + 1}"
    Type = "private"
  })
}
```

```hcl
# modules/vpc/variables.tf
variable "cidr_block" {
  description = "VPC CIDR block"
  type        = string
}

variable "private_subnets" {
  description = "List of private subnet CIDRs"
  type        = list(string)
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "project" {
  description = "Project name"
  type        = string
}

variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}
```

### Using Modules

```hcl
# environments/prod/main.tf
module "vpc" {
  source = "../../modules/vpc"

  cidr_block      = "10.0.0.0/16"
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  environment     = "prod"
  project         = "myapp"

  tags = {
    Team = "platform"
  }
}

module "eks" {
  source = "../../modules/eks"

  cluster_name   = "myapp-prod"
  vpc_id         = module.vpc.vpc_id
  subnet_ids     = module.vpc.private_subnet_ids
  node_count     = 5
}
```

### Version Pinning

```hcl
# Always pin module versions
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"
  # ...
}

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## State Management

### Remote State with S3

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### State Locking with DynamoDB

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Accessing Remote State

```hcl
# Reference state from another workspace
data "terraform_remote_state" "vpc" {
  backend = "s3"

  config = {
    bucket = "mycompany-terraform-state"
    key    = "prod/vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.vpc.outputs.private_subnet_ids[0]
}
```

## Environment Separation

### Using Workspaces

```bash
# Create workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Switch workspace
terraform workspace select prod

# Apply
terraform apply
```

```hcl
# Reference workspace in config
locals {
  environment = terraform.workspace

  instance_count = {
    dev     = 1
    staging = 2
    prod    = 5
  }
}

resource "aws_instance" "app" {
  count         = local.instance_count[local.environment]
  instance_type = local.environment == "prod" ? "t3.large" : "t3.small"
}
```

### Using tfvars Files

```hcl
# environments/prod/terraform.tfvars
environment    = "prod"
instance_count = 5
instance_type  = "t3.large"
enable_backup  = true
```

```bash
terraform apply -var-file="environments/prod/terraform.tfvars"
```

## Testing

### Terratest

```go
// test/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVpcModule(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "cidr_block":  "10.0.0.0/16",
            "environment": "test",
        },
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcId)
}
```

### Validation

```hcl
variable "environment" {
  type        = string
  description = "Environment name"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "cidr_block" {
  type        = string
  description = "VPC CIDR block"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}
```

## CI/CD Pipeline

### GitHub Actions

```yaml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.5.0

      - name: Terraform Format
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: terraform init
        working-directory: environments/prod

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -out=tfplan
        working-directory: environments/prod

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan
        working-directory: environments/prod
```

## Security Best Practices

### Sensitive Variables

```hcl
variable "db_password" {
  type        = string
  sensitive   = true
  description = "Database password"
}

output "connection_string" {
  value     = "postgres://user:${var.db_password}@${aws_db_instance.main.endpoint}"
  sensitive = true
}
```

### Using Secrets Manager

```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

## Best Practices Summary

1. **Use modules**: Break infrastructure into reusable components
2. **Remote state**: Store state in S3/GCS with locking
3. **Version pin**: Lock provider and module versions
4. **Separate environments**: Use workspaces or directories
5. **Test**: Use Terratest for module validation
6. **CI/CD**: Automate plan and apply
7. **Secure secrets**: Use sensitive flag and secrets managers

## Resources

- [Terraform Best Practices](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices)
- [Terraform Module Registry](https://registry.terraform.io/)
- [Terratest](https://terratest.gruntwork.io/)
- [Spacelift Best Practices](https://spacelift.io/blog/terraform-best-practices)

---

*Questions about Terraform IaC? [Let me know](mailto:jordan@jordananderson.us).*
