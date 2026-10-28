# 🚀 Bulletproofing Your Terraform State with S3 Versioning



In Infrastructure as Code (IaC) with Terraform, your `terraform.tfstate` file is the absolute source of truth for your cloud environment. If you lose or corrupt this file, Terraform loses its memory — leading to duplicate resources, orphaned infrastructure, or catastrophic accidental deletions.

To protect against state loss, implementing **S3 Bucket Versioning (`aws_s3_bucket_versioning`)** is strictly mandatory for production environments.

---

## 🛑 Part 1: Why S3 Versioning Is Strictly Necessary

Before writing infrastructure code, understand the failure modes that bucket versioning prevents:

1. **Protection Against Human Error (Accidental Overwrites):**  
   If an engineer runs `terraform apply` with an incorrect variable set or forces an outdated state file, the valid remote state is overwritten. Versioning ensures the previous, healthy state file is preserved. S3 places the new state object on top of the old one, allowing rollbacks to the exact timestamp prior to the error.
2. **Protection Against Accidental Deletion:**  
   If a misconfigured CI/CD pipeline, rogue automation script, or operator deletes the state file, a versioned bucket does not permanently purge the underlying object. S3 simply places a **Delete Marker** on top of the object stack. Removing this Delete Marker restores the state file immediately.
3. **State Corruption & Network Failures:**  
   If a CI/CD runner crashes or drops its network connection while streaming an updated state file to AWS, versioning ensures the prior, uncorrupted state remains intact beneath the partial upload.
4. **Disaster Recovery & Ransomware Mitigation:**  
   If an unauthorized party compromises developer AWS credentials and attempts to purge state files, a versioned bucket combined with **MFA Delete** prevents permanent destruction of historical versions without access to a physical hardware or virtual MFA device.
5. **Auditing and Historical Forensics:**  
   When tracking down infrastructure drift, versioning allows you to download historical state files from weeks or months prior. Running a JSON diff between historical states pinpoints the exact time an attribute diverged.

---

## 🛠️ Part 2: The Step-by-Step Runbook

---

### Phase 1: Implementation (The Bootstrap Setup)

#### ⚠️ Directory Architecture: The Bootstrap Pattern
The S3 bucket, versioning configuration, and lifecycle policies provision the backend infrastructure that your application environments reference in their `backend.tf` blocks. 

Because the backend bucket cannot exist inside the state file that it hosts, provision it in a standalone `bootstrap/` directory using local state:

```text
my-terraform-project/
├── bootstrap/             # Run ONCE locally to build the remote backend infrastructure
│   ├── versions.tf        # AWS Provider & CLI constraints (no backend block)
│   ├── providers.tf       # Provider definition & global tags
│   ├── main.tf            # State bucket, versioning, lifecycle rules, public access block
│   ├── outputs.tf         # Bucket name & ARN exports
│   └── terraform.tfstate  # Managed locally (Backed up securely outside Git)
└── environments/
    ├── dev/
    │   └── backend.tf     # References the bucket created by bootstrap/
    └── prod/
        └── backend.tf     # References the bucket created by bootstrap/
```

> [!WARNING]
> 1. Run `terraform apply` inside `bootstrap/` **first** before deploying any environment stacks.
> 2. Securely back up `bootstrap/terraform.tfstate` in an encrypted location (e.g., an encrypted vault or password manager). Losing this local state file impairs management of the backend bucket itself.

---

#### Step 0: Bootstrap Provider Configuration

##### `bootstrap/versions.tf`
```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Intentionally omitted: no backend block is declared here (Local State Bootstrap)
}
```

##### `bootstrap/providers.tf`
```hcl
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      ManagedBy = "terraform-bootstrap"
      Project   = "my-company-prod"
    }
  }
}
```

---

#### Step 1: Base Bucket Declaration

##### `bootstrap/main.tf`
```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-company-prod-terraform-state"

  # Guardrail: Prevent accidental destruction via terraform destroy
  lifecycle {
    prevent_destroy = true
  }
}
```

> [!NOTE]
> S3 bucket versioning can be set to `Enabled` or `Suspended`. Once enabled on a bucket, it cannot be fully disabled — suspending versioning only stops new versions from being generated; existing object versions remain intact.

---

#### Step 2: Enable Bucket Versioning

##### `bootstrap/main.tf` (Continued)
```hcl
resource "aws_s3_bucket_versioning" "terraform_state_versioning" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

---

#### Step 3: Lifecycle Rules (Cost & Storage Governance)

State files update repeatedly across active CI/CD pipelines. Unmanaged version accumulation can quietly increase storage costs. Configure automated lifecycle expiration for non-current versions:

##### `bootstrap/main.tf` (Continued)
```hcl
resource "aws_s3_bucket_lifecycle_configuration" "state_lifecycle" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    id     = "expire-old-state-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 90 # Retain historical versions for 90 days, then purge
    }
  }
}
```

---

#### Step 4: Configure Native S3 State Locking (Terraform $\ge$ 1.10)

In Terraform 1.10+, the S3 backend supports native locking via S3 conditional writes, eliminating the requirement for a dedicated DynamoDB table:

##### `environments/dev/backend.tf`
```hcl
terraform {
  backend "s3" {
    bucket       = "my-company-prod-terraform-state"
    key          = "dev/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true # Uses native S3 locking (.tflock file); replaces DynamoDB
  }
}
```

> [!TIP]
> Legacy DynamoDB locking (`dynamodb_table = "..."`) is maintained for backwards compatibility, but native S3 locking (`use_lockfile = true`) is the modern standard for new configurations.

---

### Phase 2: Testing & Verification

Validate recovery procedures prior to an actual production incident.

#### 1. Upload Initial State and Overwrite (Simulate Error)

```bash
# Upload Version 1 (Healthy)
echo '{"state": "v1-healthy"}' > terraform.tfstate
aws s3 cp terraform.tfstate s3://my-company-prod-terraform-state/terraform.tfstate

# Overwrite with Version 2 (Corrupted)
echo '{"state": "v2-corrupted"}' > terraform.tfstate
aws s3 cp terraform.tfstate s3://my-company-prod-terraform-state/terraform.tfstate
```

#### 2. Verify Historical Object Versions

```bash
aws s3api list-object-versions \
  --bucket my-company-prod-terraform-state \
  --prefix terraform.tfstate
```

The output returns two records:
* The newest version (`IsLatest: true`) corresponds to the corrupted V2.
* The previous version corresponds to healthy V1, each identified by a unique `VersionId`.

---

### Phase 3: Disaster Recovery Procedures

---

#### 🔥 Scenario A: State Was Overwritten or Corrupted

**Objective:** Roll back to the last known healthy state version.

1. List object versions and locate the `VersionId` of the healthy state:
   ```bash
   aws s3api list-object-versions \
     --bucket my-company-prod-terraform-state \
     --prefix terraform.tfstate
   ```

2. Download the healthy version locally:
   ```bash
   aws s3api get-object \
     --bucket my-company-prod-terraform-state \
     --key terraform.tfstate \
     --version-id "<HEALTHY_VERSION_ID>" \
     recovered_state.tfstate
   ```

3. Inspect `recovered_state.tfstate` in an editor to confirm data integrity.

4. Push the validated state file back to S3 as the new `Latest` version:
   ```bash
   aws s3 cp recovered_state.tfstate s3://my-company-prod-terraform-state/terraform.tfstate
   ```

---

#### 🔥 Scenario B: State File Was Accidentally Deleted

**Objective:** Remove the S3 Delete Marker to restore the object.

1. List object versions and locate the active Delete Marker:
   ```bash
   aws s3api list-object-versions \
     --bucket my-company-prod-terraform-state \
     --prefix terraform.tfstate
   ```

2. Under the `"DeleteMarkers"` array in the output, find the entry where `"IsLatest": true` and copy its `"VersionId"`.

3. Delete the Delete Marker:
   ```bash
   aws s3api delete-object \
     --bucket my-company-prod-terraform-state \
     --key terraform.tfstate \
     --version-id "<DELETE_MARKER_VERSION_ID>"
   ```

4. Verify that the state file is restored:
   ```bash
   aws s3 ls s3://my-company-prod-terraform-state/
   ```

---

#### 🔥 Scenario C: State Lock Is Stuck

**Objective:** Clear a stuck state lock left behind by an interrupted apply or terminated CI/CD container.

If a process terminates unexpectedly mid-apply, Terraform may leave the state locked:

```text
Error: Error acquiring the state lock
Lock Info:
  ID: 5f3c1a2e-8b4d-4a1f-9c3e-2d7a1b0f9c88
  Path: my-company-prod-terraform-state/prod/terraform.tfstate.tflock
```

1. Confirm with your team and active CI/CD pipelines that no apply operation is currently executing.
2. Force-release the stale lock using the lock ID:
   ```bash
   terraform force-unlock 5f3c1a2e-8b4d-4a1f-9c3e-2d7a1b0f9c88
   ```

> [!CAUTION]
> Never run `force-unlock` while an apply is active. Force-unlocking a live deployment can result in concurrent state writes and corruption.

---

## ✅ Part 4: Production Maintenance Checklist

- [ ] **Versioning Enabled:** Declared via `aws_s3_bucket_versioning` with `status = "Enabled"`.
- [ ] **Non-Current Version Expiration:** Configured in `aws_s3_bucket_lifecycle_configuration` (e.g., expire after 90 days).
- [ ] **Native S3 Locking:** Configured with `use_lockfile = true` on Terraform $\ge$ 1.10.
- [ ] **Public Access Block:** Configured via `aws_s3_bucket_public_access_block` to prevent public internet access.
- [ ] **Transport Security Policy:** Attached an S3 bucket policy enforcing TLS (`aws:SecureTransport = "true"`).
- [ ] **IAM Least Privilege:** Restricted `s3:DeleteObject*` and `s3:PutObject` privileges to CI/CD service roles and platform leads.
- [ ] **MFA Delete:** Enabled on the bucket for elevated protection against malicious version purging.
- [ ] **Encryption-at-Rest:** Server-side encryption enabled (SSE-S3 or AWS KMS CMK) to protect secrets stored in the state file.