# 🚀 Mastering Terraform Modules: The Ultimate Step-by-Step Runbook (Write & Consume)



If you are working with Infrastructure as Code (IaC), writing standard flat Terraform files is a great start. But to scale cloud architectures across multiple teams and environments, you must master **Terraform Modules**.

Below is an architectural breakdown of why modules are the cornerstone of scalable platform engineering, followed by a hands-on runbook to author and consume a secure-by-default AWS S3 module.

---

## 🧠 Part 1: Understanding Terraform Modules

In Terraform, a **module** is simply any directory containing a collection of `.tf` files:

* **The Root Module:** The working directory where you execute `terraform init` and `terraform apply`.
* **Child Modules:** Modular, self-contained packages of infrastructure code called from a root module (or other child modules) to abstract complexity and promote reusability.

```text
 ┌────────────────────────────────────────────────────────┐
 │                      Root Module                       │
 │                 (environments/dev)                     │
 │                                                        │
 │   module "s3_bucket" {                                 │
 │     source = "../../modules/s3-bucket"                 │
 │     bucket_name = "my-dev-bucket-992211"               │
 │   }                                                    │
 └───────────────────────────┬────────────────────────────┘
                             │
                             ▼ Ingests Parameters
 ┌────────────────────────────────────────────────────────┐
 │                      Child Module                      │
 │                  (modules/s3-bucket)                   │
 │                                                        │
 │  • Enforces AES256 Server-Side Encryption              │
 │  • Enforces S3 Block Public Access (All 4 settings)    │
 │  • Enforces S3 Bucket Versioning                       │
 └────────────────────────────────────────────────────────┘
```

---

### Why We Need Modules in Modern Cloud Environments

1. **The DRY Principle (Don't Repeat Yourself):**  
   Without modules, deploying 10 S3 buckets across 3 environments requires copying and pasting identical resource blocks 30 times. If an organizational security baseline updates, you must update 30 separate blocks. Modules allow you to write the pattern once, parameterize it with input variables, and instantiate it dynamically.
2. **Simplification and Abstraction:**  
   Building a production-grade VPC requires subnets, route tables, internet gateways, NAT gateways, and flow logs. Packaging these resources into a child module allows consumers to instantiate an entire network topology with a concise interface:
   ```hcl
   module "vpc" {
     source      = "./modules/vpc"
     environment = "prod"
   }
   ```
3. **Standardization and Security Guardrails:**  
   A central platform or DevSecOps team can write an S3 module that hardcodes default encryption, enforces bucket versioning, and blocks public access. Downstream developers consuming the module inherit compliance by default.
4. **Version Control and Decoupled Upgrades:**  
   Modules can reside in external Git repositories tagged with semantic versions (`v1.0.0`, `v2.0.0`). Application teams can pin their deployments to specific tags and evaluate upgrades on their own schedule without being disrupted by breaking upstream changes.
5. **Blast Radius Reduction and Clean Architecture:**  
   A 5,000-line monolithic `main.tf` file is difficult to read and risky to apply. Modules partition infrastructure into distinct, decoupled domains — Networking, Compute, and Databases.

---

## 🛠️ Part 2: The Step-by-Step Runbook

In this runbook, we will build a reusable, secure-by-default AWS S3 child module, and then consume it from an environment directory.

---

### 📋 Prerequisites

* Terraform CLI installed (`terraform -version`).
* AWS CLI installed and configured (`aws configure`).
* Terminal open in an empty working directory.

---

### 📂 Phase 1: Structuring the Project

Create the multi-tiered directory layout:

```bash
mkdir my-terraform-project && cd my-terraform-project
mkdir -p modules/s3-bucket
mkdir -p environments/dev
```

Configure `.gitignore` in the project root to prevent checking in state files or sensitive variable files:

```gitignore
# .gitignore
**/.terraform/*
*.tfstate
*.tfstate.*
crash.log
*.tfvars
*.tfvars.json

# DO NOT ignore the dependency lockfile:
!.terraform.lock.hcl
```

> [!NOTE]
> Always commit `.terraform.lock.hcl` to version control. It cryptographically locks provider binary versions across operating systems and CI/CD pipelines.

---

### 🏗️ Phase 2: Authoring the Child Module

Navigate into the child module directory:

```bash
cd modules/s3-bucket
```

A production-ready child module requires four standard files:
* `versions.tf`
* `variables.tf`
* `main.tf`
* `outputs.tf`

---

#### Step 1: Pin Provider Constraints (`versions.tf`)

```hcl
# modules/s3-bucket/versions.tf

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

> [!TIP]
> Pinning provider versions inside child modules prevents unexpected schema breakages when modules are consumed across different teams or environments.

---

#### Step 2: Define Input Schemas (`variables.tf`)

```hcl
# modules/s3-bucket/variables.tf

variable "bucket_name" {
  description = "The globally unique name of the S3 bucket"
  type        = string
}

variable "environment" {
  description = "The deployment environment tier (e.g., dev, prod)"
  type        = string
  default     = "dev"
}

variable "enable_versioning" {
  description = "Bucket versioning state: Enabled or Suspended"
  type        = string
  default     = "Enabled"
}
```

---

#### Step 3: Implement Infrastructure Resources (`main.tf`)

Enforce security controls directly within the module so consumers cannot bypass them:

```hcl
# modules/s3-bucket/main.tf

# 1. Base S3 Bucket
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name

  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# 2. Bucket Versioning Configuration
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = var.enable_versioning
  }
}

# 3. Default Server-Side Encryption (Non-Negotiable)
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# 4. Block All Public Access (Strict S3 Perimeter)
resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

#### Step 4: Define Output Exports (`outputs.tf`)

Export key attributes so parent callers can consume them downstream:

```hcl
# modules/s3-bucket/outputs.tf

output "bucket_arn" {
  description = "The Amazon Resource Name (ARN) of the provisioned S3 bucket"
  value       = aws_s3_bucket.this.arn
}

output "bucket_id" {
  description = "The name/ID of the provisioned S3 bucket"
  value       = aws_s3_bucket.this.id
}
```

---

### 💻 Phase 3: Consuming the Child Module

Navigate to the development environment workspace:

```bash
cd ../../environments/dev
```

Create `main.tf`:

```hcl
# environments/dev/main.tf

provider "aws" {
  region = "us-east-1"
}

module "my_first_bucket" {
  # Local relative filesystem source
  source = "../../modules/s3-bucket"

  bucket_name       = "my-unique-company-bucket-dev-992211"
  environment       = "development"
  enable_versioning = "Enabled"
}

# Reference child module outputs using: module.<MODULE_NAME>.<OUTPUT_NAME>
output "dev_bucket_arn" {
  description = "Exported ARN of the bucket from child module"
  value       = module.my_first_bucket.bucket_arn
}
```

---

#### 💡 Remote Module Consumption Pattern (Git Sourcing)

When publishing child modules to a centralized repository, consume them by referencing the Git source address and pinning a semantic release tag:

```hcl
module "my_remote_bucket" {
  # Remote Git repository source pinned to release tag v1.0.0
  source = "git::https://github.com/your-org/terraform-modules.git//s3-bucket?ref=v1.0.0"

  bucket_name       = "my-unique-company-bucket-dev-992211"
  environment       = "development"
  enable_versioning = "Enabled"
}
```

---

### ⚙️ Phase 4: Execution Lifecycle

Inside `environments/dev`:

```bash
# 1. Download providers and load child module definitions
terraform init

# 2. Format configuration syntax
terraform fmt

# 3. Validate module interfaces and arguments
terraform validate

# 4. Preview execution plan
terraform plan

# 5. Provision the resources
terraform apply -auto-approve
```

Upon completion, `dev_bucket_arn` prints directly to the console:

```text
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

dev_bucket_arn = "arn:aws:s3:::my-unique-company-bucket-dev-992211"
```

---

### 🧹 Phase 5: Verification & Cleanup

1. **Verify Compliance in AWS Console:**  
   Navigate to **Amazon S3** in the AWS Management Console. Locate the bucket and verify that **Bucket Versioning**, **Default Encryption (AES256)**, and **Block Public Access (all 4 settings)** are enabled.
2. **Decommission Resources:**
   ```bash
   terraform destroy -auto-approve
   ```

---

## ✅ Module Production Best Practices Checklist

- [ ] **Directory Separation:** Maintained distinct `modules/` and `environments/` directory trees.
- [ ] **Lockfile Tracking:** Ignored local state files while checking `.terraform.lock.hcl` into Git.
- [ ] **Provider Pinning:** Configured `versions.tf` in child modules with strict `required_providers` constraints.
- [ ] **Clean Interfaces:** Declared descriptive input types and defaults in `variables.tf`.
- [ ] **Guardrails by Default:** Enforced baseline security settings (encryption, access blocks) directly in `main.tf`.
- [ ] **Explicit Outputs:** Exported resource IDs and ARNs in `outputs.tf` for downstream consumption.
- [ ] **Semantic Pinning:** Pinned remote module sources to Git release tags (`?ref=vX.Y.Z`) in shared repositories.