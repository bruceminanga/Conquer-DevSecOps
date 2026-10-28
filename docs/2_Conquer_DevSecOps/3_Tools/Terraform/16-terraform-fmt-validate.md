# 🏗️ Terraform Code Hygiene: Why `fmt` and `validate` Are Non-Negotiable



If you work with Infrastructure as Code (IaC), a fundamental mindset shift must happen: **Terraform is not just a deployment tool; it is software code.**

Just as software engineers rely on linters, compilers, and strict formatting standards to maintain application reliability, Platform and DevOps engineers need the exact same discipline for infrastructure.

Below is an engineering breakdown of two of the most critical — yet frequently skipped — commands in the Terraform ecosystem: `terraform fmt` and `terraform validate`.

---

## 🧠 Part 1: Understanding the Purpose

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      The Three Validation Gates                        │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 1. terraform fmt      ──► Style & Readability (Canonical HCL standard) │
 │ 2. terraform validate ──► Syntax & Schema Consistency (Offline check)  │
 │ 3. terraform plan     ──► Cloud Reality & Provider APIs (Online check) │
 └────────────────────────────────────────────────────────────────────────┘
```

---

### 1️⃣ Why Do We Need `terraform fmt`? (Formatting)

The `terraform fmt` command rewrites Terraform configuration files into the canonical HashiCorp HCL style. It is mandatory for several reasons:

* **Eliminates Subjective PR Debates:** Ends team arguments over tabs vs. spaces, brace placement, or alignment of `=` signs. HashiCorp provides a single canonical style, and `fmt` enforces it deterministically.
* **Cleaner Git Diffs & Code Reviews:** If Developer A uses 2 spaces and Developer B uses 4 spaces, Git reports massive whitespace diffs even if only one line of logic changed. Running `fmt` ensures pull requests display only genuine infrastructure modifications.
* **Reduces Cognitive Load:** Uniformly formatted code accelerates comprehension. When modules, arguments, and locals share identical visual alignment, syntax anomalies stand out to human reviewers immediately.
* **Deterministic CI Exit Codes:** When executed as `terraform fmt -check`, the tool outputs exit code `1` if any file requires formatting and exit code `0` if all files adhere to standard. This is the mechanism that allows CI/CD pipelines to fail fast on poorly formatted code.

---

### 2️⃣ Why Do We Need `terraform validate`? (Structural Validation)

The `terraform validate` command verifies that configuration files are syntactically correct and internally consistent, executing completely offline without querying cloud provider APIs or remote state.

* **Catches Human Error Early:** Instantly surfaces missing brackets, invalid argument names, missing required attributes, or deprecated syntax before triggering network calls.
* **Zero-Latency Feedback Loop:** Running `terraform plan` is slow because it authenticates against cloud APIs and refreshes resources. In contrast, `terraform validate` runs locally in milliseconds.
* **Validates Type Schemas & Module Contracts:** Confirms that passed variable types match declared signatures (e.g., passing a `string` where a `list(string)` is required) and verifies that child module source references exist.
* **Saves Pipeline Resources:** Catching logical bugs via `validate` early in a CI runner saves compute minutes and avoids unnecessary cloud control plane API calls.

---

### ⚠️ Critical Distinction: What `terraform validate` Does NOT Catch

`validate` only evaluates local syntax and provider schemas offline. It will **not** catch:

* **Invalid or Non-Existent Cloud Identifiers:** It cannot verify if an AMI ID (`ami-0abc123`) exists in your target AWS region.
* **Cloud Quotas & Capacity Limits:** It cannot verify whether an instance family is available in a specific Availability Zone.
* **IAM Permission Boundaries:** It cannot determine if your deployment role has permission to provision the declared resources.
* **Dynamic Network Realities:** It cannot ping subnets or verify route connectivity.

> **The Synergy:**  
> `fmt` ensures code is **readable**.  
> `validate` ensures code is **structurally sound**.  
> `plan` verifies that the **cloud will accept it**.

---

## 🛠️ Part 2: The Ultimate Runbook

---

### Phase 1: Prerequisites & Initialization

#### Step 1: Verify Installation
```bash
terraform version
```

#### Step 2: Navigate to Configuration Directory
```bash
cd /path/to/terraform/project
```

#### Step 3: Initialize Provider Schemas
```bash
terraform init -backend=false
```

> [!IMPORTANT]
> `terraform validate` cannot run without provider schemas. Running `terraform init -backend=false` downloads the required provider schema definitions without attempting to connect to or lock your remote state backend (e.g., AWS S3, Terraform Cloud).

---

### Phase 2: Local Execution Workflow

#### Step 4: Dry-Run the Formatter (Check Mode)
To inspect files that need formatting without modifying them on disk:

```bash
terraform fmt -check -diff -recursive
```
* `-check`: Returns exit code `1` if formatting is required.
* `-diff`: Displays inline patch diffs of non-compliant lines.
* `-recursive`: Formats all subdirectories and child modules.

#### Step 5: Automatically Format Code
Rewrite all files to the canonical standard:

```bash
terraform fmt -recursive
```

#### Step 6: Validate Logic and Schemas
```bash
terraform validate
```

**Expected Success Output:**
```text
Success! The configuration is valid.
```

---

### ⚠️ Multi-Module Monorepo Loop

> [!CAUTION]
> Unlike `terraform fmt`, **`terraform validate` does not support a `-recursive` flag.** It validates only the single module in the current working directory. 

For multi-module architectures or monorepos, use this loop to validate each directory independently:

```bash
for dir in modules/*/; do
  echo "==> Validating module: $dir"
  terraform -chdir="$dir" init -backend=false
  terraform -chdir="$dir" validate
done
```

---

### Phase 3: Automation via Pre-Commit Hooks

Enforce hygiene checks before developers can commit code locally.

#### Step 7: Install `pre-commit`
```bash
# macOS (Homebrew)
brew install pre-commit

# Python / Pip
pip install pre-commit
```

#### Step 8: Configure `.pre-commit-config.yaml`
In the root of your Git repository, create `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.1 # Check repository for latest stable release
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
```

#### Step 9: Install the Git Hook
```bash
pre-commit install
```

*Any attempt to run `git commit` on improperly formatted or syntactically invalid Terraform code will be rejected automatically.*

---

### Phase 4: CI/CD Pipeline Integration (The Safety Net)

If an engineer bypasses local pre-commit hooks (e.g., using `git commit --no-verify`), your automated pipeline must catch and report the failure.

#### `.github/workflows/terraform-hygiene.yml`

```yaml
name: Terraform Hygiene Checks

on:
  pull_request:
    branches: [main]

jobs:
  hygiene:
    name: Lint & Validate
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.7"

      - name: Terraform Format Check
        id: fmt
        # -check returns exit code 1 if unformatted files exist
        run: terraform fmt -check -recursive
        continue-on-error: true

      - name: Terraform Init (Offline Schema Only)
        id: init
        run: terraform init -backend=false

      - name: Terraform Validate
        id: validate
        run: terraform validate
        continue-on-error: true

      - name: Post PR Comment on Failure
        if: github.event_name == 'pull_request' && (steps.fmt.outcome == 'failure' || steps.validate.outcome == 'failure')
        uses: actions/github-script@v7
        with:
          script: |
            const fmtStatus = '${{ steps.fmt.outcome }}';
            const valStatus = '${{ steps.validate.outcome }}';

            const comment = `### Terraform Hygiene Failure ❌
            
| Check                                   | Status                                               | Remediation                                                  |
| :-------------------------------------- | :--------------------------------------------------- | :----------------------------------------------------------- |
| **Format (\`terraform fmt\`)**          | ${fmtStatus === 'failure' ? '❌ Failed' : '✅ Passed'} | Run \`terraform fmt -recursive\` locally and commit changes. |
| **Validation (\`terraform validate\`)** | ${valStatus === 'failure' ? '❌ Failed' : '✅ Passed'} | Review syntax errors and run \`terraform validate\` locally. |

*Please resolve these errors before requesting review.*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
            core.setFailed('Terraform hygiene checks failed. See PR comment for details.');
```

---

## 🚨 Phase 5: Common Traps & Troubleshooting

| Error / Symptom                                                     | Root Cause                                                                                    | Remediation                                                                            |
| :------------------------------------------------------------------ | :-------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------- |
| **`validate` fails with "Provider uninitialized"**                  | Missing provider schema binaries.                                                             | Run `terraform init` (or `terraform init -backend=false`) first.                       |
| **`fmt -check` fails in CI, but looks fine locally**                | Uncommitted whitespace changes or unformatted nested modules.                                 | Run `terraform fmt -recursive` across the entire repo, commit the result, and push.    |
| **Code passes `validate`, but fails on `plan`**                     | Logical limits or nonexistent values (e.g., bad AMI ID, invalid CIDR block, IAM restriction). | Expected behavior. `validate` tests offline structure; `plan` validates cloud reality. |
| **`validate` only checks root module, misses errors in `modules/`** | `terraform validate` does not recurse into nested directories.                                | Use the shell loop from Phase 2 to iterate across each child module folder.            |

---

## ✅ The Master Checklist

- [ ] **1. Write / Update HCL Code.**
- [ ] **2. Initialize Providers:** `terraform init -backend=false`
- [ ] **3. Format Syntax:** `terraform fmt -recursive`
- [ ] **4. Validate Schemas:** `terraform validate`
- [ ] **5. Commit Code:** Pre-commit hooks run automated local checks.
- [ ] **6. Open Pull Request:** CI pipeline enforces compliance via automated PR status checks.
- [ ] **7. Execute Plan:** Run `terraform plan` to confirm cloud-side API compatibility.