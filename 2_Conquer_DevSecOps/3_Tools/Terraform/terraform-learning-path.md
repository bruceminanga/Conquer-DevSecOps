# Terraform Learning Path: Beginner to Pro 🏗️


### First things first:

* Complex systems are an illusion. They’re just simple components working together. ---Terraform architecture
* 𝗧𝗲𝗿𝗿𝗮𝗳𝗼𝗿𝗺 𝗰𝗼𝗱𝗲 𝗶𝘀𝗻'𝘁 𝗮 𝗽𝗿𝗼𝗰𝗲𝗱𝘂𝗿𝗮𝗹 𝘀𝗰𝗿𝗶𝗽𝘁. 𝗜𝘁'𝘀 𝗮 𝘀𝘆𝘀𝘁𝗲𝗺 𝘁𝗼𝗽𝗼𝗹𝗼𝗴𝘆.
* From Screwdrivers to Software: What Your Terraform Code Is Actually Doing
* One mental model to make Terraform instantly click for you
* Most people learn Terraform backwards.
* Rule #1 of Infrastructure as Code: Your HCL should never know whether it’s targeting AWS or LocalStack.

---

### Networking concepts in terraform: 

* **VPC / VNet (Virtual Private Cloud / Virtual Network):** Creating isolated software-defined networks in cloud providers (AWS, Azure, GCP).
* **Public vs. Private Subnet Topology:** Splitting networks into public subnets (internet-facing) and private subnets (isolated databases, application servers).
* **Internet Gateways (IGW) & NAT Gateways:** Giving private subnets one-way outbound internet access (for software updates) without allowing incoming internet traffic.
* **Route Tables & Custom Routing:** Directing traffic between subnets, gateways, and virtual appliances.
* **Cloud Security Groups & NACLs (Network Access Control Lists):** Provisioning cloud-level firewalls.
* **Cloud Load Balancers:** Provisioning AWS ALBs/NLBs, Azure Load Balancers, or GCP Cloud Load Balancing via code.
* **DNS Management (e.g., AWS Route53, Cloudflare):** Provisioning DNS zones, A-records, CNAMEs, and automated SSL/TLS certificates.

---

### Now:

> Welcome to the complete Terraform journey. Five levels. Real infrastructure. No shortcuts.

---

## ➊ 𝗟𝗘𝗩𝗘𝗟 𝟭 — 𝗧𝗲𝗿𝗿𝗮𝗳𝗼𝗿𝗺 𝗣𝗿𝗶𝗺𝗶𝘁𝗶𝘃𝗲𝘀 & 𝗔𝗿𝗰𝗵𝗶𝘁𝗲𝗰𝘁𝘂𝗿𝗲

> 𝘎𝘰𝘢𝘭: Understand how Terraform works under the hood, learn HCL syntax, and interact with core building blocks.

1. Understand the architecture — Terraform Core, Provider Plugins, HCL, and the Execution Graph
2. [How Terraform maps API calls to real infrastructure](01-terraform-api-calls.md)
3. [The State File — what nobody tells you about how it actually works](02-terraform-state-file.md)
4. [Infrastructure as Code: Terraform vs Ansible vs Pulumi](03-iac-terraform-ansible-pulumi.md)
5. [Terraform is essentially a State Machine for cloud APIs](04-terraform-state-machine.md)
6. [Master the core workflow: init → plan → apply → destroy](05-terraform-workflow.md)
7. [Understand resource blocks and provision your first cloud instance](06-resource-block-first-instance.md)
8. [Imperative vs declarative infrastructure management](07-imperative-vs-declarative.md)
9. [Data sources — fetch information about existing infrastructure without modifying it](08-terraform-data-sources.md)
10. [Input Variables (variable) and Output Values (output) — make your code dynamic](09-input-variables-and-outputs.md)
11. [Local Values (locals) — reduce repetition and simplify expressions](10-terraform-locals.md)
12. [Version constraints — required_providers, required_version, and provider pinning for reproducible builds](11-version-constraints-and-lock-file.md)
13. [Variable files — terraform.tfvars, *.auto.tfvars, and environment-specific var files](12-terraform-variable-files.md)
14. [Inspect and debug deployments with TF_LOG=DEBUG and terraform console](13-terraform-debugging.md)
15. [Visualise the dependency graph with terraform graph](14-terraform-graph.md)
16. [Track drift and trigger resource recreation — observe how Terraform heals state](15-terraform-drift-and-recreation.md)
17. [Code hygiene: terraform fmt and terraform validate](16-terraform-fmt-validate.md)

---

## ➋ 𝗟𝗘𝗩𝗘𝗟 𝟮 — 𝗥𝗲𝗮𝗹 𝗜𝗻𝗳𝗿𝗮𝘀𝘁𝗿𝘂𝗰𝘁𝘂𝗿𝗲, 𝗦𝘁𝗮𝘁𝗲 & 𝗠𝗼𝗱𝘂𝗹𝗮𝗿𝗶𝘁𝘆

> 𝘎𝘰𝘢𝘭: Manage real-world infrastructure reliably, handle state securely, and write reusable code.

1. [Migrate from local state to Remote State (S3, GCS, or Terraform Cloud)](01-terraform-remote-state.md)
2. [State Locking with traditional DynamoDB or modern S3 Native locking — prevent concurrent execution corruption](02-terraform-state-locking.md)
3. [Write and consume your first Terraform Module](03-terraform-first-module.md)
4. [Pass variables into modules and extract data via module outputs](04-terraform-module-inputs-outputs.md)
5. [Manage sensitive data with the sensitive = true flag and external secret managers](05-terraform-sensitive-data-and-secret-managers.md)
6. [The count meta-argument — scale identical resources](06-terraform-count-meta-argument.md)
7. [The for_each meta-argument — iterate over maps and sets](07-terraform-for-each-meta-argument.md)
8. [Provider configurations and aliases for multi-region deployments](08-terraform-provider-aliases-multi-region.md)
9. [Terraform built-in functions: merge, lookup, cidrsubnet, flatten, and more](09-terraform-built-in-functions.md)
10. [Provision a multi-tier VPC: public/private subnets, NAT gateways, routing tables](10-terraform-multi-tier-vpc.md)
11. [Deploy a classic 3-tier architecture: Web, App, and Managed Database layers](11-terraform-three-tier-architecture.md)

---

## ➌ 𝗟𝗘𝗩𝗘𝗟 𝟯 — 𝗔𝗱𝘃𝗮𝗻𝗰𝗲𝗱 𝗛𝗖𝗟, 𝗦𝘁𝗮𝘁𝗲 𝗦𝘂𝗿𝗴𝗲𝗿𝘆 & 𝗥𝗲𝗳𝗮𝗰𝘁𝗼𝗿𝗶𝗻𝗴

> 𝘎𝘰𝘢𝘭: Master dynamic configurations, complex logic, and safely modify existing infrastructure.

- [ ] Dynamic blocks — construct repeatable nested configurations (e.g., security group rules)
- [ ] Bring unmanaged infrastructure under Terraform control with terraform import
- [ ] The moved block — the modern, safe way to rename or relocate resources without CLI state surgery (Terraform 1.1+)
- [ ] State surgery when you need it: terraform state mv, rm, and replace-provider
- [ ] Workspaces vs Directory Separation — managing Dev, Staging, and Prod environments (Source: Terraform Workspace Guide)
- [ ] Lifecycle rules: create_before_destroy, prevent_destroy, and ignore_changes
- [ ] terraform_data (formerly null_resource) — trigger workflows based on state changes
- [ ] Provisioners (local-exec, remote-exec) — understand them, and why Cloud-Init or Packer is usually better
- [ ] Explicit and implicit dependencies with depends_on
- [ ] Data-driven configuration — reading YAML/JSON files via yamldecode / jsondecode into locals

---

## ➍ 𝗟𝗘𝗩𝗘𝗟 𝟰 — 𝗣𝗿𝗼𝗱𝘂𝗰𝘁𝗶𝗼𝗻 𝗥𝗲𝗮𝗱𝗶𝗻𝗲𝘀𝘀, 𝗧𝗲𝘀𝘁𝗶𝗻𝗴 & 𝗖𝗜/𝗖𝗗

> 𝘎𝘰𝘢𝘭: Automate Terraform, enforce security, test your modules, and integrate into enterprise pipelines.

- [ ] Automated CI/CD pipelines (GitHub Actions / GitLab CI) for plan and apply
- [ ] OIDC authentication — ditch long-lived static cloud credentials in your pipelines
- [ ] Policy as Code — OPA (Rego) or HashiCorp Sentinel for governance and compliance
- [ ] Security scanning — tfsec or checkov inside the CI pipeline
- [ ] Linting with tflint — catch provider-specific errors before they reach cloud APIs
- [ ] Module documentation automation with terraform-docs
- [ ] Shift-left cost estimation — Infracost integrated into pull requests
- [ ] Drift detection pipelines — alert on manual cloud changes before they cause incidents
- [ ] Securely inject secrets dynamically using HashiCorp Vault
- [ ] Module testing with Terratest — write Go-based integration tests for your Terraform modules
- [ ] Ephemeral resources — short-lived, non-persisted values for secrets and tokens (Terraform 1.10+)

---

## ➎ 𝗟𝗘𝗩𝗘𝗟 𝟱 — 𝗣𝗹𝗮𝘁𝗳𝗼𝗿𝗺 𝗘𝗻𝗴𝗶𝗻𝗲𝗲𝗿𝗶𝗻𝗴 & 𝗘𝗰𝗼𝘀𝘆𝘀𝘁𝗲𝗺 𝗘𝘅𝘁𝗲𝗻𝘀𝗶𝗯𝗶𝗹𝗶𝘁𝘆

> 𝘎𝘰𝘢𝘭: Build scalable IaC platforms, manage multi-account setups, and extend Terraform's capabilities.

- [ ] DRY multi-environment configurations with Terragrunt or Terraspace
- [ ] Build an Internal Developer Platform (IDP) — expose Terraform modules via Backstage
- [ ] GitOps for IaC — compare standard pipelines with Atlantis or Flux's TF-Controller
- [ ] Write your own Custom Terraform Provider in Go to manage internal APIs
- [ ] Continuous reconciliation vs traditional apply-and-forget
- [ ] Terraform + Kubernetes natively — migration paths and comparison with Crossplane
- [ ] Terraform Cloud/Enterprise features — Run Tasks, Sentinel policies, Private Module Registry
- [ ] Ephemeral/preview environments — spin up full environments per Pull Request, tear them down on merge
- [ ] Multi-account Landing Zone deployments — AWS Control Tower / Azure Landing Zones via Terraform

---

### Extra:

* Starting terraform project: Most engineers over-engineer their Terraform before they've even provisioned a single VM.
* 11 Terraform Frictions Every Engineer Hits Right After "Hello World"
* Stop Terraform from downloading providers on every single init (even when you thought they were cached)