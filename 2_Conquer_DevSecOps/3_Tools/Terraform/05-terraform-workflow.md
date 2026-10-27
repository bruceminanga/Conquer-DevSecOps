# 🚀 Mastering Infrastructure as Code: The Terraform Workflow — A Complete Engineering Runbook



To truly master Infrastructure as Code (IaC), you must understand Terraform's core workflow: **Init, Plan, Apply, and Destroy** — plus the architectural guardrails that make it production-safe.

Terraform operates on **Declarative Infrastructure**. You write code declaring what you want (the *Desired State*), and Terraform figures out how to reconcile that against reality (the *Actual State*). To bridge that gap safely, it enforces a strict execution lifecycle.

Here is an engineering breakdown of why this workflow is mandatory, followed by a hands-on, zero-assumptions runbook.

---

## 🛠️ PART 1: Why This Workflow Is Mandatory

### 1️⃣ `terraform init`

* **Why we need it:** Terraform ships as a lightweight, provider-agnostic core engine. It contains no built-in logic for AWS, Azure, GCP, Kubernetes, or local filesystems.
* **What it does:** Scans your configuration files for required providers, queries the Terraform Registry, downloads the provider plugin binaries into a local `.terraform/` directory, locks dependency versions in `.terraform.lock.hcl`, and downloads any referenced remote modules.
* **Without it:** Terraform cannot parse provider schemas or establish network connections to target APIs.

---

### 2️⃣ `terraform plan`

* **Why we need it:** Cloud infrastructure is expensive, interconnected, and fragile. Blindly applying changes without inspecting the diff risks catastrophic outages (e.g., unintentionally destroying production databases or route tables).
* **What it does:** Refreshes its awareness of current infrastructure via Cloud APIs, compares reality against your declarative code, calculates the execution graph delta, and outputs a detailed preview of all resource additions, modifications, and deletions.
* **Production Best Practice:** Always lock the plan file:
  ```bash
  terraform plan -out=tfplan
  ```
  This guarantees that the exact execution path reviewed and approved in your pull request is what runs during deployment—even if state or code changes moments later.

---

### 3️⃣ `terraform apply`

* **Why we need it:** This stage executes the transition from text declarations to live cloud infrastructure.
* **What it does:** Takes the calculated execution plan, translates declarative blocks into imperative HTTP API requests (`POST`, `PUT`, `PATCH`, `DELETE`), awaits provider confirmations, and commits the resulting IDs and attributes to `terraform.tfstate`.

> [!WARNING]
> **Never manually edit the state file.** Hand-editing raw JSON state risks corrupting checksums, serial numbers, and resource lineage, causing Terraform to make unpredictable API decisions.

---

### 4️⃣ `terraform destroy`

* **Why we need it:** Infrastructure has a finite lifecycle. Abandoned test clusters, staging instances, and orphaned disks waste budgets and widen attack surfaces.
* **What it does:** Evaluates the state file, reverses the Directed Acyclic Graph (DAG) dependency order, and systematically issues deletion API calls for every resource tracked in the workspace.
* **Without it:** Teams waste engineering hours tracking down orphaned "ghost resources" across cloud provider consoles.

---

## 🛡️ PART 2: Concepts That Make This Production-Safe

### 1. Remote State Backends

Storing `terraform.tfstate` on a local laptop only works for solo sandbox experiments. The moment two engineers or concurrent CI runners touch the same codebase, state conflicts arise.

In production environments, state must reside in a centralized remote backend (such as AWS S3, Google Cloud Storage, Azure Blob Storage, or Terraform Cloud).

> [!TIP]
> **Modern AWS S3 Backend Note:**  
> In Terraform 1.10+, the S3 backend supports native S3 conditional writes for state locking, removing the mandatory requirement for a separate DynamoDB table on new setups (though legacy setups using S3 + DynamoDB remain widely supported) [cite: 1, 2].

**Remote state ensures:**
* **State Locking:** Only one engineer or CI/CD job can apply changes at a time, eliminating write-race conditions.
* **Durability:** State isn't lost if a developer workstation fails.
* **Single Source of Truth:** Every contributor evaluates code against identical metadata.

---

### 2. Variables and Parameter Separation

Hardcoding static values directly inside `main.tf` prevents code reusability. Production patterns separate concerns across three distinct files:

* `variables.tf`: Defines input schema, types, and validation rules.
* `terraform.tfvars`: Assigns concrete values per deployment (never commit raw secrets here).
* `main.tf`: References parameters dynamically via `var.<variable_name>`.

```text
  ┌──────────────────┐       ┌──────────────────────┐
  │   variables.tf   │       │   terraform.tfvars   │
  │  (Schema & Type) │       │   (Concrete Values)  │
  └────────┬─────────┘       └──────────┬───────────┘
           │                            │
           └──────────────┬─────────────┘
                          ▼
               ┌──────────────────────┐
               │       main.tf        │
               │   (var.file_content) │
               └──────────────────────┘
```

---

### 3. Workspaces

Terraform Workspaces allow you to manage distinct environments (e.g., `dev`, `staging`, `prod`) from a single configuration by maintaining separate state files under `terraform.tfstate.d/<workspace-name>/`.

```bash
terraform workspace new staging
terraform workspace select staging
terraform apply
```

---

### 4. Drift and Import

Real-world operations introduce changes outside of standard pipelines:

* **Drift:** An administrator modifies firewall rules or instance sizes manually via the cloud console. Running `terraform plan` detects this out-of-band delta and schedules a corrective apply to bring reality back into compliance.
* **Import:** If infrastructure was created manually or via legacy scripts before adopting IaC, `terraform import` imports existing cloud IDs into the state file without triggering destructive recreations.

---

## 💻 PART 3: The Complete Hands-On Runbook

This exercise uses the official `local` provider so you can test the entire lifecycle on your laptop with no cloud credentials required.

### Prerequisites

* Terraform CLI installed (`terraform -version`).
* Terminal open in an empty directory.

---

### Step 0: Author the Configuration

Create an isolated directory:

```bash
mkdir terraform-training && cd terraform-training
```

Inside this directory, create the following three files:

#### `variables.tf`
```hcl
variable "file_content" {
  description = "The content to write into the output file"
  type        = string
}
```

#### `terraform.tfvars`
```hcl
file_content = "Terraform is easy to master once you understand the workflow!"
```

#### `main.tf`
```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

resource "local_file" "my_first_resource" {
  content  = var.file_content
  filename = "${path.module}/hello_world.txt"
}
```

---

### Step 1: Initialization

```bash
terraform init
```

Terraform downloads the `hashicorp/local` provider plugin and writes:
* `.terraform/`: Local provider cache directory.
* `.terraform.lock.hcl`: Cryptographic dependency lock file (**commit this to Git**).

---

### Step 2: Formatting & Validation

Format syntax and validate logic:

```bash
# Auto-formats code according to HCL conventions
terraform fmt

# Verifies syntax, provider arguments, and variable references
terraform validate
```

**Expected Output:**
```text
Success! The configuration is valid.
```

---

### Step 3: Planning (Save the Plan Artifact)

Generate and save the execution plan:

```bash
terraform plan -out=myplan.tfplan
```

**Understanding Plan Symbols:**
* `+` : Resource creation.
* `-` : Resource destruction.
* `~` : In-place modification.
* `-/\+` : Destroy and recreate (replacement).

You will see:
```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

---

### Step 4: Execution

Apply the saved plan file:

```bash
terraform apply myplan.tfplan
```

Terraform executes the exact plan you generated:
```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Verify that `hello_world.txt` now exists on disk and examine `terraform.tfstate`.

---

### Step 5: Day-2 Operations (Updating Infrastructure)

Modify `terraform.tfvars`:

```hcl
file_content = "I have updated my infrastructure!"
```

Generate a new plan to observe the replacement action:

```bash
terraform plan -out=myplan.tfplan
```

> [!NOTE]
> Notice the `-/\+` indicator: The `local_file` provider cannot update file contents in-place on disk; it must destroy the previous resource and create a new one. In production cloud environments, replacing a resource might mean a new IP address or instance recreation. Always review replacement warnings carefully.

Apply the verified plan:

```bash
terraform apply myplan.tfplan
```

Inspect `hello_world.txt` to confirm that the file contents have been updated.

---

### Step 6: Decommissioning (Teardown)

Destroy the managed infrastructure:

```bash
terraform destroy
```

Review the destructive plan summary and type `yes` when prompted.

```text
Destroy complete! Resources: 1 destroyed.
```

`hello_world.txt` is removed, and `terraform.tfstate` is cleared of managed resources.

---

## ✅ The Production Master Checklist

Commit this operational lifecycle loop to memory:

1. [ ] **`terraform init`** (Run upon project setup, or whenever adding new providers/modules).
2. [ ] **Separate inputs**: Always separate `variables.tf` and `terraform.tfvars` from `main.tf`.
3. [ ] **Configure Remote State**: Implement remote storage with state locking before sharing code across teams.
4. [ ] **Run Checks**: Execute `terraform fmt` and `terraform validate` in pre-commit hooks or CI.
5. [ ] **Lock the Plan**: Always run `terraform plan -out=tfplan` in your CI pipeline.
6. [ ] **Apply the Plan**: Run `terraform apply tfplan` to guarantee deterministic execution.
7. [ ] **Clean Up**: Run `terraform destroy` when decommissioning temporary or preview environments.

Extra:
1. [Terraform Lifecycle: Workflow and Lifecycle Meta-Argument](https://www.pynetlabs.com/terraform-lifecycle/)