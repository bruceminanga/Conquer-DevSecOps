# Mastering Terraform Variable Files: The Complete Guide (Runbook Included) 📁⚙️



If you work with Infrastructure as Code (IaC), mastering Terraform variable definition files (`.tfvars`) is a non-negotiable operational skill.

In Terraform, you declare variables in your schema (typically inside a `variables.tf` file) to make your code dynamic. However, declaring an input variable only defines its schema — it doesn't give it a concrete value. This is where `.tfvars` files come in.

Below is an architectural breakdown of how variable files work, followed by a hands-on, zero-cost runbook.

---

## 🧠 PART 1: Understanding `.tfvars`

Keeping configuration logic (`.tf` files) decoupled from environmental values (`.tfvars` files) provides five critical advantages:

### 1️⃣ Reusability and the DRY Principle (Don't Repeat Yourself)

Without `.tfvars` files, you would have to hardcode parameters (such as instance types, CIDR blocks, or region names) directly inside `main.tf`. By separating code from data, you write a single golden Terraform module and instantiate it across multiple projects or tiers without duplicating logic.

---

### 2️⃣ Multi-Environment Management

This is the primary architectural driver for environment-specific files (such as `dev.tfvars`, `qa.tfvars`, and `prod.tfvars`). Instead of maintaining three separate directory structures for Development, QA, and Production, you maintain one:

```bash
# Provision production with large, resilient instances
terraform apply -var-file="prod.tfvars"

# Provision development with small, low-cost instances
terraform apply -var-file="dev.tfvars"
```

---

### 3️⃣ Security and Secret Management Hygiene

Never hardcode database credentials, private tokens, or TLS keys directly inside `.tf` files. Passing sensitive values via an ignored variable file ensures credentials are never committed to version control:

```gitignore
# .gitignore — track environment configs, ignore secret files
secrets.tfvars
*.secrets.auto.tfvars
```

> [!WARNING]
> **Critical Security Note on State:**  
> `.gitignore` protects your Git history, **not your Terraform state**. Every resolved input variable — sensitive or not — is committed to `terraform.tfstate` in plain JSON text by default. Marking a variable `sensitive = true` only masks it from CLI stdout and pipeline logs; the raw value remains in the state file. When handling real secrets, combine `.tfvars` hygiene with an encrypted remote backend (e.g., AWS S3 + KMS), or pull secrets at runtime from an enterprise secret store like HashiCorp Vault or AWS Secrets Manager.

---

### 4️⃣ Automated CI/CD Pipelines (`*.auto.tfvars`)

In automated CI/CD pipelines (such as GitHub Actions or GitLab CI), runners often need to inject dynamic parameters (e.g., commit SHA, pipeline build ID, or dynamic runner tags). Pipelines can dynamically generate a file named `pipeline.auto.tfvars` immediately before execution. Because of Terraform's precedence hierarchy, this automatically overrides base defaults.

> [!CAUTION]
> **Pipeline Failure Gotcha:**  
> This automation pattern works only if every required input variable receives a value. If a variable lacks a default value and no file or flag supplies one, Terraform prompts for input interactively. In a non-interactive CI runner, Terraform cannot prompt and immediately exits with a fatal `"No value for required variable"` error.

---

### 5️⃣ The Terraform Variable Precedence Hierarchy

When a variable is defined across multiple files, CLI flags, and environment variables, Terraform evaluates values using a strict order of precedence.

**Lowest precedence to highest (bottom overrides top):**

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │ 1. Environment Variables (TF_VAR_<name>)                               │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 2. terraform.tfvars                                                    │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 3. terraform.tfvars.json                                               │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 4. *.auto.tfvars or *.auto.tfvars.json (Evaluated in lexical order)    │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 5. CLI Flags (-var and -var-file) (HIGHEST PRECEDENCE — Overrides All) │
 └────────────────────────────────────────────────────────────────────────┘
```

#### Critical Gotchas:

* **Alphabetical Evaluation of `*.auto.tfvars`:** When multiple `*.auto.tfvars` files exist, Terraform loads them in **alphabetical order by filename**. Whichever file loads last wins on any overlapping variables. For example, a stray `zz-debug.auto.tfvars` left behind from a debugging session will silently overwrite values in `app.auto.tfvars`.
* **Multiple `-var-file` Flags:** If you pass multiple `-var-file` flags (e.g., `-var-file="base.tfvars" -var-file="prod.tfvars"`), the last flag passed overrides any earlier conflicting declarations.
* **Inline Overrides:** The `-var` flag operates at the same top-tier precedence as `-var-file`, making it ideal for temporary CLI overrides:  
  `terraform plan -var="server_size=t2.nano"`

---

## 🛠️ PART 2: The Hands-On Runbook

This runbook uses the official `local` provider so you can test variable precedence rules directly on your local filesystem without cloud credentials.

### Prerequisites

* Terraform CLI installed (`terraform -version`).
* Terminal / command prompt open.

---

### Step 1: Create the Workspace

```bash
mkdir terraform-vars-runbook && cd terraform-vars-runbook
```

---

### Step 2: Declare Variable Schemas (`variables.tf`)

Create `variables.tf`. This declares the accepted schema and types without hardcoding values:

```hcl
variable "environment_name" {
  description = "The target deployment environment tier"
  type        = string
}

variable "server_size" {
  description = "The instance sizing category"
  type        = string
}
```

---

### Step 3: Author the Configuration (`main.tf`)

Create `main.tf`. This outputs variables to a rendered file so you can inspect which values win:

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

resource "local_file" "output_file" {
  filename = "${path.module}/server_info.txt"
  content  = <<-EOT
  Deployment Environment: ${var.environment_name}
  Provisioned Server Size: ${var.server_size}
  EOT
}
```

> [!TIP]
> **Heredoc Formatting:** While `<<EOT` requires its closing delimiter to sit flush against the left margin, using `<<-EOT` strips leading whitespace, allowing you to indent the closing tag cleanly within your resource block.

---

### Step 4: Create Default Base File (`terraform.tfvars`)

Create `terraform.tfvars`. Terraform loads this exact filename automatically without requiring CLI flags:

```hcl
environment_name = "Base-Environment"
server_size      = "small-server"
```

Initialize and preview:

```bash
terraform init
terraform plan
```

**Expected Result:** Terraform reads `terraform.tfvars` automatically and plans to generate the file with `Base-Environment` and `small-server`.

---

### Step 5: Override via Auto-Loading File (`custom.auto.tfvars`)

Create `custom.auto.tfvars` to simulate an automated CI injection or team override:

```hcl
server_size = "medium-server"
```

Run plan:

```bash
terraform plan
```

**Expected Result:** `environment_name` remains `Base-Environment` (inherited from `terraform.tfvars`), but `server_size` is updated to `medium-server`. The `*.auto.tfvars` file successfully overrode `terraform.tfvars`.

---

### Step 6: Deploy with Environment-Specific Files (`prod.tfvars`)

Create `prod.tfvars`:

```hcl
environment_name = "Production-Environment"
server_size      = "massive-server-xlarge"
```

Run standard plan first:

```bash
terraform plan
```

*Terraform ignores `prod.tfvars` because it does not match the default naming patterns (`terraform.tfvars` or `*.auto.tfvars`).*

Now explicitly pass the file flag:

```bash
terraform plan -var-file="prod.tfvars"
```

**Expected Result:** Complete override. The plan updates `environment_name` to `Production-Environment` and `server_size` to `massive-server-xlarge`. CLI `-var-file` flags defeat all base and auto files.

---

### Step 7: Apply and Verify

Apply the production configuration:

```bash
terraform apply -var-file="prod.tfvars" -auto-approve
```

Inspect the rendered file:

```bash
cat server_info.txt
```

**Output:**
```text
Deployment Environment: Production-Environment
Provisioned Server Size: massive-server-xlarge
```

---

### Step 8: Clean Up

Tear down the test infrastructure:

```bash
terraform destroy -var-file="prod.tfvars" -auto-approve
```

---

## 📌 Summary Checklist

* [ ] **`terraform.tfvars`:** Default base definition file; loaded automatically.
* [ ] **`*.auto.tfvars`:** Automatically loaded overrides. Evaluated in **alphabetical order by filename**; last file wins.
* [ ] **`<env>.tfvars` (e.g., `prod.tfvars`):** Isolated environment files. Must be passed explicitly via `-var-file="<env>.tfvars"`.
* [ ] **CLI Precedence:** `-var` and `-var-file` hold highest priority and override all other sources.
* [ ] **State Warning:** Remember that `.gitignore` only protects Git history; secrets passed through `.tfvars` still reside unencrypted in `terraform.tfstate`.