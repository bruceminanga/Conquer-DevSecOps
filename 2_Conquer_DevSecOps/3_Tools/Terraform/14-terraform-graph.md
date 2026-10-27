# 🚀 Mastering Terraform Dependencies: The Complete `terraform graph` Runbook



If you manage Infrastructure as Code (IaC), you know that as your environments grow, your Terraform architecture can easily turn into an invisible, tangled web of dependencies.

To execute predictable changes, Terraform converts your code into a **Directed Acyclic Graph (DAG)**. Visualizing this graph is one of the most effective ways to understand deployment sequences, assess blast radius, and resolve circular dependency errors.

Below is an engineering breakdown of why graph visualization is critical, its limitations, and a step-by-step runbook.

---

## 🧠 Part 1: Why Do We Need `terraform graph`?

Terraform constructs a Directed Acyclic Graph before creating, updating, or destroying any resource. Visualizing this execution graph provides seven distinct advantages:

1. **Understanding Order of Operations (Provisioning Sequence):**  
   Terraform parallelizes operations whenever possible. If Resource A does not depend on Resource B, they are provisioned concurrently. The graph displays which resources are built first, which wait, and which run in parallel.
2. **Debugging Chained Dependencies:**  
   Resources fail to deploy or tear down when chained incorrectly:
   * **Implicit dependencies:** Inferred automatically when one resource references an attribute of another (e.g., `subnet_id = aws_subnet.sub1.id`).
   * **Explicit dependencies:** Declared manually using the `depends_on` meta-argument.  
   The graph exposes unnecessary bottlenecks and validates that explicit dependencies are correctly applied.
3. **Assessing Blast Radius:**  
   Planning to alter a core VPC, route table, or database? The graph maps every downstream resource connected to that object, helping you evaluate operational risk before typing `terraform apply`.
4. **Untangling Dependency Cycles:**  
   A circular dependency (Resource A depends on B, while B depends on A) blocks execution. Adding the `-draw-cycles` flag isolates and marks the cyclical loop visually.
5. **Architectural Documentation & Onboarding:**  
   Handing a visual dependency diagram to a new engineer is far more effective than asking them to parse thousands of lines of raw HCL across nested directories.
6. **Optimizing Pipeline Execution Speed:**  
   If your deployment takes 30+ minutes, you likely have long, single-file sequential chains. Visualizing the DAG highlights bottlenecks so you can refactor for greater concurrency.
7. **CI/CD Review Artifacts:**  
   In mature platform engineering pipelines, graph SVGs are generated automatically as pre-apply artifacts, giving pull request reviewers a visual representation of the planned topology.

---

## ⚠️ Part 2: Know the Limitations First

Before running commands, keep these architectural constraints in mind:

* **It graphs code configuration, not live cloud state:** If infrastructure has drifted out-of-band in the cloud console, `terraform graph` will not show that drift.
* **It does not replace `terraform plan`:** Always review the plan artifact. The graph shows structural relationships; the plan shows attribute-level deltas.
* **Pair with state queries for drift analysis:** Cross-reference `terraform graph` with `terraform state list` to confirm what Terraform is actively tracking vs. what is declared in `.tf` files.

---

## 🛠️ Part 3: The Step-by-Step Runbook

---

### Phase 1: Prerequisites & Setup

#### 1. Verify Terraform CLI
```bash
terraform version
```

#### 2. Install Graphviz
`terraform graph` outputs raw graph data in **DOT** format. To convert DOT syntax into visual diagrams (PNG or SVG), install **Graphviz**:

* **macOS (Homebrew):**
  ```bash
  brew install graphviz
  ```
* **Linux (Ubuntu/Debian):**
  ```bash
  sudo apt-get update && sudo apt-get install -y graphviz
  ```
* **Windows (Chocolatey):**
  ```powershell
  choco install graphviz
  ```

#### 3. Test Configuration (Optional Sandbox)
Create a `main.tf` file to test chained dependencies:

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

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "sub1" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.sub1.id
}
```

---

### Phase 2: Generating the Graph

#### 4. Initialize Working Directory
You must run `init` so Terraform can register provider schemas; otherwise, provider dependency edges cannot be computed:

```bash
terraform init
```

#### 5. Generate and Convert to Visual Formats
Pipe the raw DOT output directly into Graphviz. **SVG** is recommended for zooming into complex enterprise diagrams without raster pixelation:

```bash
# Export to vector SVG (Recommended)
terraform graph | dot -Tsvg > graph.svg

# Export to raster PNG
terraform graph | dot -Tpng > graph.png
```

---

### Phase 3: Advanced Filtering & Scoping

#### 6. Graph Specific Operational Lifecycles (`-type`)
By default, Terraform graphs the plan lifecycle. You can scope the DAG to specific operational phases using the `-type` flag (`plan`, `apply`, `plan-destroy`, `destroy`):

```bash
# Graph for the apply phase
terraform graph -type=apply | dot -Tsvg > apply_graph.svg

# Graph for the destruction phase
terraform graph -type=plan-destroy | dot -Tsvg > destroy_graph.svg
```

#### 7. Debug Circular Loops (`-draw-cycles`)
When a circular dependency error occurs, highlight the problematic cycle directly in red:

```bash
terraform graph -draw-cycles | dot -Tsvg > cycle_debug.svg
```

#### 8. Scope Modules in Large Repositories (`-module-depth`)
In multi-tier codebases, default graphs can become unreadable. Use `-module-depth` to collapse nested modules:

```bash
# Top-level view (treats child modules as single nodes)
terraform graph -module-depth=1 | dot -Tsvg > module_overview.svg

# Expand two levels deep
terraform graph -module-depth=2 | dot -Tsvg > module_detail.svg
```

#### 9. Filter Internal Metadata with `grep`
Strip out provider meta-nodes and variables to focus strictly on physical cloud resources:

```bash
terraform graph | grep -v 'var\.' | grep -v 'provider\.' | dot -Tsvg > clean_graph.svg
```

---

### Phase 4: How to Read the Graph Correctly

When inspecting `graph.svg` in a browser or image viewer:

* **Nodes (Boxes / Ovals):** Represent resources, variables, data sources, and modules (e.g., `aws_vpc.main`, `[root]`).
* **Edges (Arrows) — THE GOLDEN RULE:**
  $$\text{Node A} \longrightarrow \text{Node B} \implies \text{Node A depends on Node B}$$
  *Node B must be created before Node A can begin.*
* **Example:**  
  `aws_instance.web` $\rightarrow$ `aws_subnet.sub1` $\rightarrow$ `aws_vpc.main`  
  *Terraform creates the VPC first, then the Subnet, and finally the EC2 instance.*
* **Red Edges (with `-draw-cycles`):** Highlight the exact circular dependency deadlock that must be refactored.

---

### Phase 5: Drift Reconciliation Workflow

Because the graph models declared configuration rather than live state, use this cross-referencing process when investigating drift:

1. **Query State:**
   ```bash
   terraform state list
   ```
2. **Generate Configuration Graph:**
   ```bash
   terraform graph | dot -Tsvg > graph.svg
   ```
3. **Compare Delta:** Resources present in `state list` but absent from `graph.svg` represent orphaned or manually removed resources still tracked in state.
4. **Plan Remediation:**
   ```bash
   terraform plan
   ```

---

### Phase 6: CI/CD Pipeline Automation

Integrate dependency graph rendering as an automated check in GitHub Actions or GitLab CI to produce visual PR review artifacts:

```yaml
name: Generate Terraform Architecture Graph

on:
  pull_request:
    branches: [main]

jobs:
  graph:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Install Graphviz
        run: sudo apt-get update && sudo apt-get install -y graphviz

      - name: Initialize Terraform
        run: terraform init

      - name: Generate Graph Artifact
        run: terraform graph -type=plan | dot -Tsvg > architecture.svg

      - name: Upload Graph Artifact
        uses: actions/upload-artifact@v4
        with:
          name: terraform-architecture-graph
          path: architecture.svg
```

---

### Phase 7: Modern Ecosystem Visualization Tools

For complex multi-cloud enterprise codebases, native DOT graphs can be complemented with modern visualizers:

* **[Terraform Visual](https://github.com/hieuvp/terraform-visual):** Generates interactive web dashboards from JSON plan files (`terraform show -json tfplan > plan.json`).
* **[Rover](https://github.com/im2nguyen/rover):** A lightweight containerized explorer that renders color-coded, zoomable live topology graphs at `localhost:9000`.
* **[Infracost](https://www.infracost.io/) / [Pluralith](https://www.pluralith.com/):** Integrates cost breakdowns and compliance checks directly over the dependency topology.

---

## 📌 Summary Checklist for Daily Use

- [ ] **1. Write or Update HCL:** Maintain clear attribute references for implicit dependencies.
- [ ] **2. Initialize Workspace:** Run `terraform init` to pull provider graph logic.
- [ ] **3. Render SVG:** Run `terraform graph -type=plan | dot -Tsvg > architecture.svg`.
- [ ] **4. Trace Dependencies:** Follow edges ($A \rightarrow B$ means $B$ deploys before $A$).
- [ ] **5. Resolve Cycles:** Use `terraform graph -draw-cycles` to isolate circular loops.
- [ ] **6. Scope Modules:** Use `-module-depth=1` to simplify complex multi-module repositories.
- [ ] **7. Verify State Alignment:** Cross-reference against `terraform state list` to spot drift.
- [ ] **8. Confirm Blast Radius:** Validate all downstream connections before executing `terraform apply`.