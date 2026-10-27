# 🚀 Stop Hardcoding Your Terraform: The Complete Guide to Variables & Outputs



If you are writing Terraform code without Input Variables and Output Values, you aren't writing dynamic Infrastructure as Code (IaC) — you are just writing a static configuration file.

To truly master Terraform, think of your code like a reusable programming function:

$$\text{Input Variables (Function Arguments)} \longrightarrow \mathbf{\text{Terraform Module}} \longrightarrow \text{Output Values (Return Statements)}$$

Below is the definitive engineering guide on why we need this separation of concerns, followed by an end-to-end hands-on runbook.

---

## 📖 Part 1: Why We Need Them

Without variables and outputs, infrastructure configurations cannot scale across environments or teams.

### 📥 Why We Need Input Variables

Input variables allow you to parameterize your Terraform modules from external inputs:

* **Reusability (DRY — Don't Repeat Yourself):** Instead of duplicating codebases for `dev`, `staging`, and `production`, you write a single golden module and pass different variable sets per environment.
* **Dynamic Sizing & Topology:** Adjust compute capacity, replicas, and instance families without touching core code (e.g., `t2.micro` in sandbox vs. `m5.large` in production).
* **Security & Secret Management:** Hardcoding database credentials or API keys directly in `.tf` files risks accidental Git leaks. Variables allow runtime injection and masking using the `sensitive = true` flag.
* **Type Safety & Custom Validation:** Variables enforce strict guardrails. You declare types (`string`, `number`, `list(string)`, `map(any)`) and attach custom `validation` rules to catch invalid inputs before running `apply`.
* **Maintainability:** When an organizational tag changes (like a `CostCenter` or `BillingCode`), you update it in one central variable definition rather than editing hundreds of individual resource blocks.

---

### 📤 Why We Need Output Values

Outputs are how Terraform exposes resource metadata and runtime data back to the operator or downstream systems:

* **Dynamic Data Extraction:** Cloud providers allocate identifiers and endpoints at runtime (e.g., Public IPv4, Load Balancer DNS names, VPC IDs). Outputs query this state data and print it directly upon completion.
* **Module Interconnection:** When composing multi-tier architectures, downstream modules require outputs from upstream modules (e.g., an RDS Database module requires the `subnet_ids` exported by a VPC module).
* **CI/CD Pipeline Glue:** Automation tools (like GitHub Actions, GitLab CI, or Ansible) can query Terraform outputs formatted as JSON (`terraform output -json`) to dynamically configure post-provisioning steps.
* **Sensitive Masking:** Outputs can be marked with `sensitive = true`, preventing database passwords and private keys from displaying in plain text in terminal logs or CI console outputs.

---

## 🛠️ Part 2: The Practical Runbook

**Scenario:** We will refactor an AWS EC2 Web Server from a rigid, hardcoded configuration into a modular, production-ready template.

---

### Step 1: The "Bad" Anti-Pattern (Hardcoded)

Consider this static `main.tf`:

```hcl
# ANTI-PATTERN: Hardcoded parameters
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "Dev-Server"
  }
}
```

* **The Problem:** Deploying a `Prod-Server` with an `m5.large` instance requires modifying the source code directly. Furthermore, after running `apply`, you must log into the AWS Web Console just to find the instance's public IP address.

---

### Step 2: Define Inputs (`variables.tf`)

Create `variables.tf` to declare the schema, types, default values, and validation rules:

```hcl
variable "instance_type" {
  description = "The compute size of the EC2 instance"
  type        = string
  default     = "t2.micro"

  # Guardrail: Prevent accidental provisioning of oversized instances
  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium"], var.instance_type)
    error_message = "Instance type must be one of: t2.micro, t2.small, or t2.medium."
  }
}

variable "environment_name" {
  description = "Deployment environment name (e.g., dev, staging, prod)"
  type        = string
}

variable "db_password" {
  description = "Administrative password for the database tier"
  type        = string
  sensitive   = true # Masks the value in CLI output and plans
}
```

---

### Step 3: Compute Derived Values (`locals.tf`)

> [!TIP]
> **Locals vs. Variables:** Variables are inputs passed from outside the module. **Locals** are intermediate constants computed within the module to prevent duplicating expressions.

Create `locals.tf`:

```hcl
locals {
  # Standardized resource prefix based on environment
  name_prefix = "${lower(var.environment_name)}-web-server"

  # Centralized tagging policy applied to all resources
  common_tags = {
    Environment = var.environment_name
    ManagedBy   = "Terraform"
    Team        = "Platform Engineering"
  }
}
```

---

### Step 4: Write Dynamic Infrastructure (`main.tf`)

Refactor `main.tf` to reference input variables via `var.<name>` and local variables via `local.<name>`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type

  tags = merge(local.common_tags, {
    Name = local.name_prefix
  })
}
```

---

### Step 5: Export Runtime Attributes (`outputs.tf`)

Create `outputs.tf` to expose runtime information and mask sensitive values:

```hcl
output "web_server_public_ip" {
  description = "The public IPv4 address assigned to the web server"
  value       = aws_instance.web.public_ip
}

output "web_server_arn" {
  description = "The Amazon Resource Name (ARN) of the server"
  value       = aws_instance.web.arn
}

output "database_password" {
  description = "The database administrative password"
  value       = var.db_password
  sensitive   = true # Prevents plaintext leak in terminal outputs
}
```

---

### Step 6: Provide Concrete Values (`terraform.tfvars`)

Create `terraform.tfvars` to supply values for declared variables:

```hcl
instance_type    = "t2.small"
environment_name = "production"
db_password      = "SuperSecretPassword123!"
```

> [!WARNING]
> **Security Rule:** Add `*.tfvars` and `*.tfvars.json` to your `.gitignore`. Files containing raw credentials must never be committed to source control.

---

### Step 7: Variable Precedence Hierarchy

Terraform evaluates variable definitions in a strict precedence order (highest to lowest). If a variable is assigned multiple times, the highest tier wins:

|   Precedence    | Source                    | How It Is Provided                                                       |
| :-------------: | :------------------------ | :----------------------------------------------------------------------- |
| **1 (Highest)** | **CLI Flags**             | `-var="instance_type=t2.large"` or `-var-file="prod.tfvars"`             |
|      **2**      | **Auto-loaded Var Files** | Any `*.auto.tfvars` or `*.auto.tfvars.json` (lexical order)              |
|      **3**      | **JSON Var File**         | `terraform.tfvars.json`                                                  |
|      **4**      | **Standard Var File**     | `terraform.tfvars`                                                       |
|      **5**      | **Environment Variables** | `TF_VAR_<variable_name>` (e.g., `export TF_VAR_environment_name="prod"`) |
| **6 (Lowest)**  | **Configuration Default** | `default = "..."` block declared in `variables.tf`                       |

#### CI/CD Pattern (Zero-Disk Secrets)
In automated CI/CD runners, avoid saving `.tfvars` files to disk. Instead, export sensitive parameters as environment variables:

```bash
export TF_VAR_db_password="${{ secrets.PROD_DB_PASSWORD }}"
export TF_VAR_environment_name="production"

terraform apply -auto-approve
```

---

### Step 8: Execute the Lifecycle

Initialize, validate, and apply the workspace:

```bash
# 1. Download provider plugins
terraform init

# 2. Validate syntax and validation rules
terraform validate

# 3. Preview execution plan
terraform plan

# 4. Deploy infrastructure
terraform apply -auto-approve
```

---

### Step 9: Observe Outputs

Upon completion, Terraform displays:

```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

database_password = <sensitive>
web_server_arn = "arn:aws:ec2:us-east-1:123456789012:instance/i-0abcdef1234567890"
web_server_public_ip = "198.51.100.24"
```

* The public IP was queried dynamically and printed immediately.
* `database_password` was evaluated and safely masked as `<sensitive>`.
* The module is now dynamic: passing a different `.tfvars` file deploys a new environment without modifying `main.tf`.

---

### Step 10: Querying Outputs Post-Apply

You can extract outputs at any point without re-running `apply`:

```bash
# Print a single output value
terraform output web_server_public_ip

# Extract raw value without quotes (ideal for shell scripts)
terraform output -raw web_server_public_ip

# Export all outputs as JSON for external tools (e.g., Ansible, jq)
terraform output -json
```

---

## 🔄 The Complete Architecture Data Flow

```text
  [ CLI Flags / TF_VAR_* / *.tfvars ]
                 │
                 ▼
          [ variables.tf ]  ───► Validates types, enforces rules
                 │
                 ▼
           [ locals.tf ]    ───► Computes derived values & common tags
                 │
                 ▼
            [ main.tf ]     ───► Provisions resources via Cloud APIs
                 │
                 ▼
          [ outputs.tf ]    ───► Exposes attributes & masks secrets
```

---

## 🎯 Quick Reference Cheatsheet

| Construct          | File Convention    | Defining Syntax                  | Referencing Syntax        | Primary Purpose                               |
| :----------------- | :----------------- | :------------------------------- | :------------------------ | :-------------------------------------------- |
| **Input Variable** | `variables.tf`     | `variable "name" { type = ... }` | `var.name`                | Parameterize module behavior from outside     |
| **Local Value**    | `locals.tf`        | `locals { name = ... }`          | `local.name`              | Internal computed constants / DRY expressions |
| **Output Value**   | `outputs.tf`       | `output "name" { value = ... }`  | `module.<name>.<output>`  | Expose data to CLI, state, or parent modules  |
| **Values File**    | `terraform.tfvars` | `name = "value"`                 | *N/A (Loaded implicitly)* | Supply environment-specific concrete values   |