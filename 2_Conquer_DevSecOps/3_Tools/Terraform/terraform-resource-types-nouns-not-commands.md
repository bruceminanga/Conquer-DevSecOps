# Terraform Resource Types Are Nouns, Not Commands 🏷️📦



A resource type is the most fundamental unit in a Terraform configuration, and it is also the most commonly misread:

```hcl
resource "local_file" "my_server" {
  filename = "server_config.txt"
  content  = "Server is running perfectly."
}
```

`local_file` here means a **file** — the object itself, not an action performed on it. That distinction is the foundational premise of declarative infrastructure.

---

## 🏷️ The Block Header Declares the Noun

```hcl
resource "local_file" "my_server" {
#  ▲           ▲              ▲
#  │           │              └─ 2. Resource Name
#  │           └──────────────── 1. Resource Type
#  └──────────────────────────── Block Keyword
```

Two labels follow the `resource` keyword:

1. **`"local_file"` (Resource Type):** Defined by the provider implementing it. This represents the category or entity: *"a file"*.
2. **`"my_server"` (Resource Name):** Chosen by you. This is Terraform's internal handle for this specific instance of that category — used in the state file, in CLI output, and whenever referencing this resource elsewhere in code (`local_file.my_server.id`). It has no bearing on the actual filename on disk.

---

## 📋 What's Inside the Block: Attributes, Not Instructions

```hcl
filename = "server_config.txt"
content  = "Server is running perfectly."
```

These are **attributes** — they describe what the noun should look like once it exists, not the procedural steps needed to create it. 

It is conceptually identical to declaring a configuration object:

```json
{
  "file": {
    "filename": "server_config.txt",
    "content": "Server is running perfectly."
  }
}
```

No verb appears anywhere in that data structure, and none appears in HCL either:

* You specify the entity's properties (its path, its content), **not** an imperative command to write them.
* Whether Terraform performs an actual disk write or API call is evaluated separately by diffing these declared attributes against the current state.
* If nothing differs, nothing happens. The `content` line remains in your configuration on every run, but triggers zero side effects.

> [!NOTE]
> This is why a declarative configuration block can be applied repeatedly without unintended side effects (**idempotency**). It never described an action to repeat; it described a target state, and that state either holds or it doesn't.

---

## ⚡ The Verbs Live Entirely in the CLI

Because resource types express no action, the lifecycle verbs (`create`, `read`, `update`, `destroy`) are handled outside the `.tf` files entirely. They are driven by the CLI commands executed against the whole configuration:

```bash
terraform plan     # Compute the diff between declared and actual state
terraform apply    # Execute whichever CRUD operations that diff requires
terraform destroy  # Remove all managed infrastructure in dependency order
```

You will never see a resource type paired with an imperative verb — there is no `create_local_file` or `local_file.write()`. The type remains a pure noun:

| Imperative Thinking (Scripts) | Declarative Reality (Terraform)                                |
| :---------------------------- | :------------------------------------------------------------- |
| `create_vm(size="t3.micro")`  | `resource "aws_instance" "web" { instance_type = "t3.micro" }` |
| `put_file(name="app.conf")`   | `resource "local_file" "app_conf" { filename = "app.conf" }`   |
| `update_bucket_policy(id)`    | `resource "aws_s3_bucket_policy" "main" { ... }`               |

Terraform decides based on the delta whether `create`, `update`, or `destroy` applies on any given run — that operational decision never lives inside your `.tf` file.

---

## 🌐 Every Provider Defines Its Own Nouns

Terraform's core engine has **zero** built-in resource types. Every noun is owned by an external provider plugin, scoped to the objects exposed by that target platform:

| Provider         | Prefix        | Example Nouns (Resource Types)                      | Managed Entity       |
| :--------------- | :------------ | :-------------------------------------------------- | :------------------- |
| **AWS**          | `aws_`        | `aws_instance`, `aws_s3_bucket`, `aws_vpc`          | Cloud infrastructure |
| **Azure**        | `azurerm_`    | `azurerm_virtual_machine`, `azurerm_resource_group` | Azure resources      |
| **Google Cloud** | `google_`     | `google_compute_instance`, `google_storage_bucket`  | GCP assets           |
| **Kubernetes**   | `kubernetes_` | `kubernetes_deployment`, `kubernetes_namespace`     | Cluster objects      |
| **Local**        | `local_`      | `local_file`, `local_sensitive_file`                | Filesystem items     |

The standard naming convention — `<provider>_<object>` — remains consistent across the ecosystem. There is no static master list to memorize: the [Terraform Registry](https://registry.terraform.io/) for each provider serves as the authoritative source of truth, expanding as providers expose new platform capabilities.