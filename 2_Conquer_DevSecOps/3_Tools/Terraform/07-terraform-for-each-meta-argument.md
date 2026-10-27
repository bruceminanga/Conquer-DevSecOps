# Stop Using `count` Everywhere: The Real Guide to `for_each` (With a Hands-On Runbook) 🔁🛡️



Have you ever accidentally triggered the destruction of a perfectly healthy production server or database simply because you removed a single item from a Terraform list?

If you use the `count` meta-argument to iterate over collections of dynamic, named resources, you are playing with fire.

In Terraform, when you need to provision multiple similar resources, you have three design choices:

1. **Hardcode each resource block** (Violates the DRY principle).
2. **Use `count`** (Index-based scaling).
3. **Use `for_each`** (Key-based declarative management).

Below is an engineering deep dive into `for_each` — why `count` introduces destructive failure modes for dynamic data, why `for_each` is the industry standard, and a hands-on local runbook you can run on your workstation today.

---

## 🚨 The Fatal Flaw of `count` (The Index Shifting Problem)

`count` tracks resources in `terraform.tfstate` as a zero-indexed integer array (`[0]`, `[1]`, `[2]`, ...).

Imagine provisioning three servers from a list: `["web", "api", "db"]`:

* `aws_instance.server[0]` $\rightarrow$ `web`
* `aws_instance.server[1]` $\rightarrow$ `api`
* `aws_instance.server[2]` $\rightarrow$ `db`

Now, remove `"api"`. The list becomes `["web", "db"]`.

Because `count` maps strictly to index order:

```text
Index 0: "web" ──► Unchanged
Index 1: "db"  ──► Was previously "api"! (DESTROY "api" & CREATE "db")
Index 2: (Gone)──► Was previously "db"!  (DESTROY original "db")
```

Removing an item from the middle of a `count` list shifts every subsequent index downward. Terraform detects this as a state mutation and **destroys and recreates resources that were otherwise completely healthy**. In production environments, this results in unintended downtime and potential data loss.

---

## 🛡️ How `for_each` Solves This

`for_each` tracks resources by a **stable string key**, iterating over a **Set** or a **Map**:

```text
aws_instance.server["web"]
aws_instance.server["api"]
aws_instance.server["db"]
```

When you remove `"api"`, Terraform evaluates the state keys:
* `"web"` and `"db"` are left completely untouched.
* Only `aws_instance.server["api"]` is scheduled for deletion.

---

### Maps vs. Sets

| Collection Type          | Iteration Context                                                                                     | `each.key`                           | `each.value`                        |
| :----------------------- | :---------------------------------------------------------------------------------------------------- | :----------------------------------- | :---------------------------------- |
| **Set (`toset([...])`)** | Homogeneous items with identical configurations (e.g., IAM usernames).                                | The string element itself.           | Identical to `each.key`.            |
| **Map (`map(object)`)**  | Heterogeneous resources requiring distinct settings per item (e.g., custom roles, CIDRs, disk sizes). | The map entry key (e.g., `"alice"`). | The underlying object or value map. |

---

## ⚖️ When Is `count` Still Fine?

`count` is not obsolete; it is simply the wrong tool for dynamic, individually addressable entities.

`count` remains the preferred, idiomatic approach for:

1. **Conditional Resource Creation (The 1/0 Toggle):**
   ```hcl
   count = var.enable_monitoring ? 1 : 0
   ```
2. **Truly Anonymous Identical Pools:**  
   Spinning up $N$ completely identical, stateless worker nodes in an autoscaling group where individual nodes are never addressed by name and have no distinct state.

> **The Golden Rule:**  
> If you are iterating over dynamic, named, or individually meaningful infrastructure, **use `for_each`**.

---

## 💻 The Hands-On Runbook

This runbook uses the `local_file` provider. It requires zero cloud credentials and executes entirely on your local machine.

---

### Step 1: Initialize Project Directory

```bash
mkdir terraform-foreach-lab && cd terraform-foreach-lab
touch main.tf
```

---

### Step 2: Implement `for_each` with a SET

Open `main.tf`:

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

resource "local_file" "department_files" {
  # toset() converts a list of strings into a distinct set
  for_each = toset(["hr", "finance", "engineering"])

  filename = "${path.module}/${each.key}_folder.txt"
  content  = "This is the secure folder for the ${each.key} department."
}
```

---

### Step 3: Initialize and Apply

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

Inspect your directory: three files are generated (`hr_folder.txt`, `finance_folder.txt`, `engineering_folder.txt`).

Notice how Terraform tracks them in the terminal output:
```text
local_file.department_files["engineering"]
local_file.department_files["finance"]
local_file.department_files["hr"]
```

---

### Step 4: Implement `for_each` with a MAP (Dynamic Attributes)

Replace the contents of `main.tf` with a map structure that supplies distinct metadata per user:

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

variable "user_data" {
  type = map(object({
    title = string
    age   = number
  }))
  default = {
    "alice" = { title = "Staff Platform Engineer", age = 28 }
    "bob"   = { title = "Engineering Manager",      age = 45 }
    "carol" = { title = "VP of Infrastructure",     age = 52 }
  }
}

resource "local_file" "user_profiles" {
  for_each = var.user_data

  filename = "${path.module}/user_${each.key}.txt"
  content  = <<-EOT
  Name: ${each.key}
  Job Title: ${each.value.title}
  Age: ${each.value.age}
  EOT
}
```

---

### Step 5: Apply Map Configuration

```bash
terraform apply -auto-approve
```

Three files are generated (`user_alice.txt`, `user_bob.txt`, `user_carol.txt`), each populated with attributes derived from `each.value`.

---

### Step 6: Surgical Deletion in Action (The Safety Test)

Bob leaves the company. Remove his key from the map in `main.tf`:

```hcl
variable "user_data" {
  type = map(object({
    title = string
    age   = number
  }))
  default = {
    "alice" = { title = "Staff Platform Engineer", age = 28 }
    "carol" = { title = "VP of Infrastructure",     age = 52 }
  }
}
```

Preview the execution plan:

```bash
terraform plan
```

#### Terminal Output:
```text
Terraform will perform the following actions:

  # local_file.user_profiles["bob"] will be destroyed
  - resource "local_file" "user_profiles" "bob" {
      - content  = <<-EOT
            Name: bob
            Job Title: Engineering Manager
            Age: 45
        EOT -> null
      - filename = "./user_bob.txt" -> null
      - id       = "..." -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```

> [!NOTE]
> `alice` and `carol` are left completely untouched. Because Terraform tracks resources by map key rather than integer position, it surgically removes only the target resource.

---

### Step 7: Teardown

```bash
terraform destroy -auto-approve
```

---

## ⚠️ The Gotcha: Keys Must Be Known at Plan Time

`for_each` requires that the set or map keys be **deterministic and fully resolved during the planning phase**.

If you attempt to construct a map or set using an attribute that can only be computed after a resource is created:

```hcl
# ANTI-PATTERN: Fails if subnet IDs are generated during the same apply
resource "aws_route" "routes" {
  for_each = toset(aws_subnet.app[*].id) # Error!
  # ...
}
```

Terraform aborts with:
```text
Error: Invalid for_each argument
The "for_each" map includes keys derived from resource attributes that
cannot be determined until apply, and so Terraform cannot determine the
full set of keys that will be applied.
```

**Remediation:** Ensure the keys passed to `for_each` originate from static variables, locals, or pre-existing data sources, while referencing computed attributes inside the resource body.

---

## 🔁 Migrating from `count` to `for_each` Without Downtime

If you switch an existing resource from `count` to `for_each` in code without updating state, Terraform will attempt to destroy all indexed resources (`[0]`, `[1]`) and recreate them as keyed resources (`["hr"]`, `["finance"]`).

Use a declarative `moved` block (Terraform $\ge$ 1.1) to refactor the state addresses in place with zero downtime:

```hcl
moved {
  from = local_file.department_files[0]
  to   = local_file.department_files["hr"]
}

moved {
  from = local_file.department_files[1]
  to   = local_file.department_files["finance"]
}

moved {
  from = local_file.department_files[2]
  to   = local_file.department_files["engineering"]
}
```

When you run `terraform apply`, Terraform updates its internal state ledger with zero modifications to live infrastructure. Once applied across all environments, delete the `moved` blocks.

---

## ✅ The `for_each` Reference Checklist

- [ ] **Data Types:** Accepts only **Maps** or **Sets of strings**. Plain lists must be converted using `toset()`.
- [ ] **Context Variables:**  
  * `each.key`: The current set item or map key string.  
  * `each.value`: The associated data object (in maps) or string (in sets).
- [ ] **Address Syntax:** Resources are addressed in state and code with bracketed string quotes:  
  `<TYPE>.<NAME>["<KEY>"]` (e.g., `aws_instance.web["api"]`).
- [ ] **Deterministic Keys:** Ensure collection keys are known prior to apply; never derive keys from runtime-computed resource attributes.
- [ ] **Zero-Downtime Migration:** Always use declarative `moved` blocks when refactoring existing infrastructure from `count` to `for_each`.