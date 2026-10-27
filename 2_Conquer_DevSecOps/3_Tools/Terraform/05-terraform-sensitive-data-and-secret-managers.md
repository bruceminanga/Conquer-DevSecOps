# 🔐 The Ultimate Guide to Managing Secrets in IaC: `sensitive = true` & External Secret Managers



If you write Infrastructure as Code (IaC) using tools like Terraform or OpenTofu, you are constantly handling sensitive data — database passwords, API keys, TLS certificates, and OAuth tokens. Mismanaging these secrets in version control or CI/CD pipelines is one of the fastest vectors for severe enterprise data breaches.

Below is an engineering guide and hands-on runbook demonstrating how to combine **external secret managers** with the **`sensitive = true`** attribute — along with the operational edge cases that standard tutorials frequently omit.

---

## 🛡️ Part 1: Understanding the Overlapping Defenses

True DevSecOps requires defense-in-depth across multiple operational layers:

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │ 1. Storage Layer: AWS Secrets Manager / HashiCorp Vault                │
 │    Secrets reside outside Git; fetched dynamically at runtime          │
 └───────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼ Fetched via Data Source
 ┌────────────────────────────────────────────────────────────────────────┐
 │ 2. Presentation Layer: sensitive = true / sensitive()                  │
 │    Redacts plaintext secrets from CLI stdout, plan diffs & CI logs     │
 └───────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼ Persisted to State
 ┌────────────────────────────────────────────────────────────────────────┐
 │ 3. State Storage Layer: Encrypted Remote Backend (S3 + KMS)            │
 │    Protects unredacted state JSON with encryption-at-rest & IAM RBAC   │
 └────────────────────────────────────────────────────────────────────────┘
```

---

### 1️⃣ Why We Must Use External Secret Managers

Storing secrets in AWS Secrets Manager, HashiCorp Vault, or Azure Key Vault provides core security controls:

* **Zero Secrets in Git:** Eliminates plaintext passwords from `.tf` or `.tfvars` files, ensuring secrets are never preserved in Git commit history.
* **Separation of Duties:** Developers author infrastructure code without needing access to production database passwords. Security teams administer the secret vault; Terraform simply queries it.
* **Centralized Audit Trails:** Cloud secret managers log every read event via AWS CloudTrail. You cannot audit who inspected a plaintext credential committed to a Git repository.
* **Dynamic Lifecycle & Rotation:** When an external secret manager rotates credentials automatically, Terraform queries the newest version on the next execution without requiring code commits.
* **Regulatory Compliance:** Satisfies encryption-at-rest, encryption-in-transit, and RBAC requirements for SOC 2, PCI-DSS, and HIPAA frameworks.

---

### 2️⃣ Why We Must Use the `sensitive = true` Flag

Querying the secret securely is only the first step. Once Terraform ingests the value, it can leak across execution logs without proper controls:

* **Masking CLI Terminal Output:** Without `sensitive = true`, `terraform plan` and `terraform apply` display plaintext passwords directly in terminal stdout. The flag replaces the value with `(sensitive value)`.
* **Sanitizing CI/CD Pipeline Logs:** Prevents credentials from leaking into GitHub Actions, GitLab CI, or Jenkins job logs, which are often readable by broad engineering teams.
* **Downstream Masking Inheritance:** Marking a module output as sensitive guarantees that any parent module or resource consuming that attribute also inherits log redaction automatically.

---

### ⚠️ What `sensitive = true` Does NOT Protect Against

> [!WARNING]
> **The State File Trap:**  
> The `sensitive = true` attribute is purely a **display-layer mask for terminal output and logs**. It does **not** encrypt secrets within `terraform.tfstate`. Terraform must persist the plaintext secret in its state JSON to track drift across API calls.

Other critical exposure surfaces include:

1. **Saved Execution Plan Files (`-out=tfplan`):** Running `terraform plan -out=tfplan` creates a binary representation containing **unredacted plaintext secrets**. If uploaded as an unencrypted CI/CD pipeline artifact, anyone with repository read access can extract the credentials.
2. **Crash Dump Logs (`crash.log`):** If Terraform panics or crashes mid-execution, it dumps the full state buffer — including plaintext secrets — directly into a local `crash.log` file.
3. **Provider API Error Dumps:** If an upstream cloud API returns a validation error that echoes back the request payload, Terraform may print the unmasked secret directly into the error log.
4. **Shell History and Process Inspection:** Supplying secrets via `-var="db_password=..."` exposes credentials to your local shell history (`~/.bash_history`) and to any process monitoring tools (`ps aux`) on shared CI runners.

> [!TIP]
> **OpenTofu Native State Encryption:**  
> If using OpenTofu (the open-source fork), native state and plan encryption has been generally available since v1.7. This allows clients to encrypt state and plan files client-side before they are pushed to backends. Standard Terraform users remain dependent on backend-side storage encryption (e.g., S3 KMS).

---

## 🛠️ Part 2: The Hands-On Runbook

**Scenario:** Deploy an Amazon RDS PostgreSQL instance using a master password fetched dynamically from AWS Secrets Manager, with end-to-end state and log protection.

---

### Phase 1: Prerequisites

* Terraform CLI ($\ge$ 1.5.0) and AWS CLI installed.
* Active AWS IAM credentials with permissions for Secrets Manager, RDS, and S3.

---

### Phase 2: Secure the Remote State Backend

Never manage secrets against local workstation state. Configure an encrypted S3 remote backend in `backend.tf`:

```hcl
# backend.tf

terraform {
  backend "s3" {
    bucket         = "my-secure-terraform-state-bucket"
    key            = "prod/database/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true # Enforces server-side encryption for state at rest
    dynamodb_table = "terraform-locks"
  }
}
```

---

### Phase 3: Provision the Secret Out-of-Band

Create the master credential externally via AWS CLI so it never touches local disk or Git tracking:

```bash
aws secretsmanager create-secret \
  --name "prod/db/master_password" \
  --description "Master password for production RDS database" \
  --secret-string "SuperSecureP@ssw0rd123!"
```

---

### Phase 4: Configure Least-Privilege IAM Policy

Restrict the Terraform execution role to read only the specific target secret:

```hcl
data "aws_iam_policy_document" "tf_secret_read" {
  statement {
    actions   = ["secretsmanager:GetSecretValue"]
    resources = [data.aws_secretsmanager_secret.db_password.arn]
  }
}
```

---

### Phase 5: Author the Terraform Configuration (`main.tf`)

#### Step A: Query the External Secret
```hcl
data "aws_secretsmanager_secret" "db_password" {
  name = "prod/db/master_password"
}

data "aws_secretsmanager_secret_version" "db_password_version" {
  secret_id = data.aws_secretsmanager_secret.db_password.id
}
```

#### Step B: Wrap in `sensitive()`
```hcl
locals {
  # Explicitly wrapping the string in sensitive() ensures Terraform masks it in all logs
  db_pass = sensitive(data.aws_secretsmanager_secret_version.db_password_version.secret_string)
}
```

#### Step C: Reference the Secret in the Workload
```hcl
resource "aws_db_instance" "production_db" {
  allocated_storage   = 20
  engine              = "postgres"
  instance_class      = "db.t3.micro"
  db_name             = "proddb"
  username            = var.db_username
  password            = local.db_pass
  skip_final_snapshot = true
}
```

#### Step D: Protect Inputs & Outputs (`variables.tf` & `outputs.tf`)
```hcl
# variables.tf
variable "db_username" {
  description = "Database administrator username"
  type        = string
  sensitive   = true
}

# outputs.tf
output "database_endpoint" {
  description = "Connection endpoint for the database"
  value       = aws_db_instance.production_db.endpoint
}

output "database_password" {
  description = "The database master password (masked)"
  value       = local.db_pass
  sensitive   = true # Prevents plaintext leak in stdout & downstream state references
}
```

---

### Phase 6: Execution & Verification

To prevent shell history leaks, inject non-vaulted variables via environment variables (`TF_VAR_*`) rather than passing inline CLI flags:

```bash
# 1. Initialize backend and plugins
terraform init

# 2. Inject inputs via environment variable (avoids shell history & ps aux visibility)
export TF_VAR_db_username="dbadmin"

# 3. Preview execution plan
terraform plan
```

#### Verified Terminal Plan Output:
```text
Terraform will perform the following actions:

  # aws_db_instance.production_db will be created
  + resource "aws_db_instance" "production_db" {
      + allocated_storage   = 20
      + db_name             = "proddb"
      + engine              = "postgres"
      + instance_class      = "db.t3.micro"
      + password            = (sensitive value) <── MASKED!
      + username            = (sensitive value) <── MASKED!
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

---

### Phase 7: Day-2 Operations (Rotation & Auditing)

1. **Verify Backend KMS Encryption:** Confirm via AWS CLI or Console that `prod/database/terraform.tfstate` in S3 has AWS KMS SSE enabled.
2. **Plan Artifact Protection:** If generating plan files in CI (`terraform plan -out=tfplan`), **never upload `tfplan` as an unencrypted, publicly accessible build artifact**. Store it in an encrypted ephemeral storage volume.
3. **Automated Secret Rotation:** When security teams rotate the master credential in AWS Secrets Manager, trigger your CI/CD pipeline. Terraform reads the new version during `plan`, detects the update, and applies the change without exposing secrets in logs.

---

### Phase 8: Teardown (Lab Sandbox Cleanup)

```bash
# Destroy managed cloud resources
terraform destroy -auto-approve

# Purge secret from Secrets Manager
aws secretsmanager delete-secret \
  --secret-id "prod/db/master_password" \
  --force-delete-without-recovery
```

---

## ✅ The Ultimate Secrets Hygiene Checklist

- [ ] **No Hardcoded Secrets:** Zero credentials, API tokens, or keys in `.tf` or `*.tfvars` files.
- [ ] **External Secret Manager:** Secrets reside in AWS Secrets Manager, HashiCorp Vault, or Azure Key Vault.
- [ ] **Least-Privilege IAM:** Terraform execution roles are scoped strictly to `secretsmanager:GetSecretValue` on explicit ARNs.
- [ ] **Display Masking:** All variables and outputs handling credentials specify `sensitive = true`.
- [ ] **Locals Wrapping:** Dynamically computed credentials use `sensitive(...)` in `locals.tf`.
- [ ] **Zero-CLI History Leaks:** Variables are injected via `TF_VAR_<name>` environment variables rather than inline `-var` flags.
- [ ] **Plan Artifact Hygiene:** Execution plan files (`-out=tfplan`) are protected, encrypted, and excluded from public CI artifacts.
- [ ] **Encrypted State Storage:** Remote state buckets enforce encryption-at-rest (SSE-KMS) and strict bucket access policies.
- [ ] **Log Verification:** Validated that CI/CD console logs output `(sensitive value)` instead of plaintext strings.