# 🛠️ The Ultimate Terraform Debugging Playbook: Mastering `TF_LOG` & `terraform console`



If you work with Terraform long enough, you will eventually hit a wall: `terraform apply` fails with a generic, unhelpful error message while your HCL code looks completely fine on the surface.

To master Terraform troubleshooting, you must realize that Terraform operates on two entirely distinct layers:

```text
 ┌────────────────────────────────────────────────────────┐
 │ 1. The Logic Layer (HCL Language Engine)               │
 │    Functions, loops, data types, string interpolations │
 │    TOOL: terraform console                             │
 └───────────────────────────┬────────────────────────────┘
                             │
                             ▼ Generates State Transitions
 ┌────────────────────────────────────────────────────────┐
 │ 2. The Execution Layer (Provider HTTP/REST API Engine) │
 │    HTTP headers, TLS handshakes, JSON payloads, IAM    │
 │    TOOL: TF_LOG / TF_LOG_PROVIDER                      │
 └────────────────────────────────────────────────────────┘
```

When an apply fails, it is virtually always a failure in one of these two layers. Below is an architectural breakdown of both tools, followed by an end-to-end debugging runbook.

---

## 🧠 PART 1: The "Why"

### 💻 Why `terraform console`? (The Logic Layer)

HCL is declarative, but modern modules feature complex function chains, conditional expressions, dynamic `for` loops, and nested object maps. Debugging syntax by repeatedly running `terraform plan` is slow and inefficient.

`terraform console` provides an interactive REPL (Read-Eval-Print Loop) environment for:

* **Rapid Function Prototyping:** Test complex built-in functions (`cidrsubnet()`, `merge()`, `regex()`, `flatten()`) live without triggering deployment workflows.
* **State Inspection:** Query resource addresses (`aws_vpc.main`) directly to view every attribute stored in the local or remote state file.
* **Variable & Local Resolution:** Evaluate `local.my_complex_map` or `var.subnet_cidrs` to inspect the exact resolved values before invoking API calls.
* **Syntax Troubleshooting:** Isolate HCL syntax or type-casting issues without guessing.

---

### 📡 Why `TF_LOG`? (The Execution Layer)

Standard terminal output is heavily filtered for readability. It confirms that a resource failed, but often obscures the underlying API error code.

Enabling `TF_LOG` exposes the raw wire calls:

* **Exposing Cloud API Interactions:** Inspect raw HTTP requests (`POST`, `PUT`, `DELETE`), custom headers, authentication tokens, and request/response JSON payloads.
* **Catching Silenced Errors:** Cloud providers frequently return HTTP `403 Forbidden` (fine-grained IAM permission boundaries) or HTTP `429 Too Many Requests` (API throttling) that standard error parsers omit.
* **Graph & Timing Race Conditions:** View real-time Directed Acyclic Graph (DAG) node evaluations to identify concurrency locks or timing bugs.
* **Diagnosing Provider Plugin Bugs:** Pinpoint whether an unexpected failure originates from Terraform Core or an upstream bug inside the provider plugin binary.

---

### 📌 Choosing the Right Log Level

`TF_LOG` supports five standard severity levels:

| Level       | Verbosity | Primary Use Case                                                                                 |
| :---------- | :-------- | :----------------------------------------------------------------------------------------------- |
| **`ERROR`** | Lowest    | Displays only fatal execution halts.                                                             |
| **`WARN`**  | Low       | Displays non-fatal warnings and deprecated schema arguments.                                     |
| **`INFO`**  | Medium    | High-level execution summaries and state actions.                                                |
| **`DEBUG`** | High      | **Standard troubleshooting choice.** Exposes HTTP endpoints, headers, and response payloads.     |
| **`TRACE`** | Extreme   | Comprehensive plugin internals, memory allocations, and RPC exchanges. Use for provider crashes. |

#### Granular Logging (Core vs. Provider)

Modern Terraform allows you to decouple log levels between the core binary and the provider plugins:

```bash
export TF_LOG_CORE=DEBUG
export TF_LOG_PROVIDER=TRACE
```

This isolates whether an issue stems from internal Terraform graph calculation (`TF_LOG_CORE`) or a specific cloud provider's API handler (`TF_LOG_PROVIDER`).

---

## 🚀 PART 2: The Ultimate Debugging Runbook

---

### 🔍 Phase 1: Triage and Isolation

**Objective:** Determine whether the failure is a logic issue or an execution-level API error.

#### Step 1: Run Static Validation First

Before executing network calls or evaluating state, run static validation:

```bash
terraform validate
```

*This parses syntax, verifies schema arguments, and checks variable types instantly without calling cloud APIs or locking remote state.*

#### Step 2: Analyze the Error Output

```text
 ┌────────────────────────────────────────────────────────┐
 │ Is it a syntax error, type mismatch, or invalid loop?  │──► Proceed to Phase 2
 └────────────────────────────────────────────────────────┘
 ┌────────────────────────────────────────────────────────┐
 │ Is it a 4xx/5xx HTTP code, timeout, or provider crash? │──► Proceed to Phase 3 & 4
 └────────────────────────────────────────────────────────┘
 ┌────────────────────────────────────────────────────────┐
 │ Is it an orphaned resource or state drift issue?       │──► Proceed to Phase 5
 └────────────────────────────────────────────────────────┘
```

---

### 💻 Phase 2: Debugging Logic with `terraform console`

**Objective:** Fix HCL coding errors, list comprehensions, and data transformations.

#### Step 1: Initialize Working Directory

```bash
terraform init
```

#### Step 2: Launch the Console REPL

```bash
terraform console
```

> [!WARNING]
> Launching the console acquires a read lock on your remote state backend. Ensure no active CI/CD pipelines are running `apply` concurrently to avoid state lock conflicts.

#### Step 3: Inspect Variables and Live State

At the `>` prompt, inspect expressions directly:

```hcl
> var.environment
"production"

> local.common_tags
{
  "Environment" = "production"
  "ManagedBy"   = "Terraform"
}

> aws_vpc.main.cidr_block
"10.0.0.0/16"
```

#### Step 4: Test Built-in Functions

```hcl
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"
```

#### Step 5: Test Complex Comprehensions & Filters

```hcl
> [for s in var.subnets : s.id if s.public == true]
[
  "subnet-01234567",
  "subnet-89abcdef",
]
```

#### Step 6: Exit the REPL

Type `exit` or press `Ctrl + C` to return to your shell, then update your `.tf` files with the validated logic.

---

### ⚙️ Phase 3: Isolating Failing Resources with `-target`

**Objective:** Narrow the blast radius when debugging a single failing resource inside a large infrastructure stack.

Instead of running a full apply that touches hundreds of resources, isolate the target:

```bash
terraform apply -target=aws_security_group.web_sg
```

> [!CAUTION]
> Avoid `-target` in day-to-day CI/CD automation because it creates partial state. However, during active incident debugging, it significantly speeds up feedback loops and reduces log noise.

---

### 📡 Phase 4: Debugging Execution with `TF_LOG`

**Objective:** Capture and analyze raw API communication between Terraform and the cloud control plane.

#### Step 1: Set the Log Level

```bash
export TF_LOG=DEBUG
```

*(For provider crashes or panic traces, set `export TF_LOG_PROVIDER=TRACE`).*

#### Step 2: Route Output to a Dedicated Log File

Never let thousands of lines of raw JSON flood your terminal:

```bash
export TF_LOG_PATH="terraform-debug.log"
```

#### Step 3: Reproduce the Failure

```bash
terraform apply -target=aws_security_group.web_sg
```

#### Step 4: Analyze the Captured Logs

Search the log file for HTTP status codes, headers, and response payloads:

```bash
# Search for HTTP error codes
grep -E "HTTP/[0-9\.]+ (4[0-9]{2}|5[0-9]{2})" terraform-debug.log

# Search for provider error blocks
grep -A 10 "\[ERROR\]" terraform-debug.log
```

Open the log in a pager (`less terraform-debug.log`) and search (`/`) for:
* `[ERROR]` or `[WARN]`
* Specific status codes: `400`, `403`, `409`, `429`, `500`
* `response:` to inspect the raw JSON error string returned by the cloud provider.

---

### 🗄️ Phase 5: Fixing Broken State

**Objective:** Reconcile state drift, remove orphaned entries, or import existing cloud resources.

#### List All Tracked Resources

```bash
terraform state list
```

#### Inspect State Attributes of a Specific Resource

```bash
terraform state show aws_instance.web_server
```

#### Remove Orphaned Resources from State

If a resource was deleted manually in the cloud console and Terraform state is failing:

```bash
terraform state rm aws_instance.old_server
```

*(Removes the resource from state tracking without attempting to delete it).*

#### Import Existing Infrastructure into State

```bash
terraform import aws_instance.web_server i-0abc123def456
```

---

### 🧹 Phase 6: Post-Debug Sanitization & Cleanup

**Objective:** Secure and restore your terminal environment after resolving the failure.

#### Step 1: Unset Logging Variables

If left enabled, future runs will run slower and generate massive log files:

```bash
unset TF_LOG
unset TF_LOG_PATH
unset TF_LOG_CORE
unset TF_LOG_PROVIDER
```

#### Step 2: Securely Delete the Debug Log File

> [!WARNING]
> **Critical Security Warning:**  
> `TF_LOG=DEBUG` captures raw HTTP request payloads, which may contain sensitive headers, auth tokens, plaintext passwords, or private keys. **Never commit `terraform-debug.log` to version control.**

```bash
rm -f terraform-debug.log
```

#### Step 3: Verify with Clean Apply

```bash
terraform plan
terraform apply
```

---

## 🎯 Summary Reference

| Tool / Flag                 | Target Layer    | Primary Operational Purpose                                                     |
| :-------------------------- | :-------------- | :------------------------------------------------------------------------------ |
| **`terraform validate`**    | Syntax          | Validates HCL configuration syntax and provider schema compliance statically.   |
| **`terraform console`**     | Logic Layer     | Interactive REPL for testing functions, loops, variables, and inspecting state. |
| **`-target=<ADDR>`**        | Scope Isolation | Restricts execution to a single resource and its dependencies during triage.    |
| **`TF_LOG=DEBUG`**          | Execution Layer | Exposes raw cloud provider HTTP/REST API request and response payloads.         |
| **`TF_LOG_PROVIDER=TRACE`** | Plugin Layer    | Generates maximum verbosity for debugging provider-level crashes and RPC calls. |
| **`terraform state *`**     | State Layer     | Inspects (`show`), prunes (`rm`), or adopts (`import`) resources in state.      |