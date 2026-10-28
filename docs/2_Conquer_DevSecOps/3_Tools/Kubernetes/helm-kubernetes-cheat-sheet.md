# 🚀 The Ultimate Helm Cheat Sheet: Master Package Management in Kubernetes



If Kubernetes is the engine, **Helm** is the gearbox. It is how you package, version, share, and deploy applications on Kubernetes at scale—without manually maintaining hundreds of raw YAML manifests.

Bookmark this cheatsheet for day-to-day cluster operations and CI/CD pipeline automation.

---

## 🗂️ 1. Repository Management

| Command                                                    | Description                                                             |
| :--------------------------------------------------------- | :---------------------------------------------------------------------- |
| `helm repo list`                                           | List all configured chart repositories.                                 |
| `helm repo add bitnami https://charts.bitnami.com/bitnami` | Add a new remote chart repository.                                      |
| `helm repo update`                                         | Refresh local cache index for all repositories *(run before installs)*. |
| `helm repo remove bitnami`                                 | Delete a repository from local configuration.                           |
| `helm search repo <keyword>`                               | Search charts in your configured local repositories.                    |
| `helm search repo <chart> --versions`                      | List all available historical versions of a chart.                      |
| `helm search hub <keyword>`                                | Search public charts directly on Artifact Hub.                          |

---

## 📦 2. Release Management (Install, Upgrade, Uninstall)

```bash
# Basic installation
helm install myrelease bitnami/nginx

# Install into a target namespace, creating it if it doesn't exist
helm install myrelease bitnami/nginx -n mynamespace --create-namespace

# Pin to a specific immutable chart version (Production Best Practice)
helm install myrelease bitnami/nginx --version 13.2.0

# Install from a local chart directory
helm install myrelease ./mychart

# Idempotent Deploy (Installs if missing, upgrades if present - CI/CD Essential)
helm upgrade --install myrelease bitnami/nginx

# Automatic rollback if upgrade fails
helm upgrade myrelease bitnami/nginx --atomic

# Delete new resources created during a failed rollout
helm upgrade myrelease bitnami/nginx --cleanup-on-fail

# Uninstall a release
helm uninstall myrelease

# Uninstall but retain release history (allows future rollback)
helm uninstall myrelease --keep-history
```

---

## 🔍 3. Inspecting Live Releases

| Command                           | Description                                                               |
| :-------------------------------- | :------------------------------------------------------------------------ |
| `helm list`                       | List releases in the current namespace.                                   |
| `helm list -A`                    | List all releases across all namespaces.                                  |
| `helm list --failed`              | Filter output to only failed releases.                                    |
| `helm status myrelease`           | Display operational status and notes of a release.                        |
| `helm history myrelease`          | Show revision history, timestamps, and deployment status.                 |
| `helm get all myrelease`          | Dump all release data (values, manifests, hooks, notes).                  |
| `helm get values myrelease`       | View user-supplied configuration overrides applied to the release.        |
| `helm get values myrelease --all` | View complete merged values (defaults + user overrides).                  |
| `helm get manifest myrelease`     | View the exact rendered Kubernetes YAML currently applied in the cluster. |
| `helm get notes myrelease`        | Retrieve the post-install instructions provided by chart authors.         |

---

## ⚙️ 4. Values & Configuration Overrides

### Values Precedence (Lowest ➔ Highest):
1. Chart default values (`values.yaml` in chart)
2. Custom values file (`-f custom-values.yaml`)
3. Command-line flags (`--set key=value`)

```bash
# Override values via CLI flags
helm install myrelease bitnami/nginx --set service.type=NodePort
helm install myrelease bitnami/nginx --set replicaCount=3,image.tag=1.21

# Override using an external values file (Production Standard)
helm install myrelease bitnami/nginx -f custom-values.yaml

# Layer multiple values files (later files override earlier ones)
helm install myrelease bitnami/nginx -f base-values.yaml -f env-prod.yaml

# Print all default values of a chart before installing
helm show values bitnami/nginx

# Dump chart defaults to a file to use as a starting template
helm show values bitnami/nginx > values.yaml
```

---

## 🧪 5. Linting, Templating, & Pre-Flight Checks

Catch syntax and schema issues before touching the API server:

```bash
# Validate chart structure and syntax locally
helm lint ./mychart

# Strict linting (treat warnings as errors - CI Standard)
helm lint ./mychart --strict

# Local template rendering (offline; no cluster required)
helm template myrelease ./mychart -f values.yaml

# Server-side dry-run (validates against real cluster admission controllers)
helm install myrelease ./mychart --dry-run --debug

# Client-side validation chained with kubectl schema checking
helm template myrelease ./mychart | kubectl apply --dry-run=client -f -
```

> 💡 **`helm template` vs `--dry-run`:**  
> * `helm template` executes 100% locally on your machine—ideal for CI linting and GitOps render stages.  
> * `--dry-run` connects to the Kubernetes API server to validate schemas, resource quotas, and admission webhooks without saving state.

---

## ⏪ 6. Atomic Rollbacks

Recover immediately when an upgrade introduces runtime failure:

```bash
# Roll back to the immediately preceding revision
helm rollback myrelease

# Roll back to a specific revision number (check `helm history` first)
helm rollback myrelease 3

# Wait for rollback pods to become ready before exiting CLI
helm rollback myrelease 3 --wait --timeout 5m
```

---

## 📊 7. Chart Development & Packaging

```bash
# Scaffold a new chart with standard directory structure
helm create mychart

# Package a chart directory into an archive (.tgz)
helm package mychart/

# Package and cryptographically sign the chart
helm package mychart/ --sign --key 'key-name' --keyring ~/.gnupg/secring.gpg

# Manage chart dependencies (Chart.yaml)
helm dependency list mychart/
helm dependency update mychart/
helm dependency build mychart/

# Read chart documentation
helm show chart bitnami/nginx
helm show readme bitnami/nginx
```

---

## 🗄️ 8. Modern OCI Registry Workflow

Helm 3.8+ natively supports OCI registries (e.g., GitHub Container Registry, AWS ECR, Harbor) for chart distribution without maintaining static HTTP repositories:

```bash
# Authenticate to an OCI registry
helm registry login registry.example.com

# Push a packaged chart archive
helm push mychart-0.1.0.tgz oci://registry.example.com/charts

# Pull a remote chart package
helm pull oci://registry.example.com/charts/mychart --version 0.1.0

# Install directly from an OCI URI
helm install myrelease oci://registry.example.com/charts/mychart --version 0.1.0

# Log out
helm registry logout registry.example.com
```

---

## 🔌 9. Essential Production Plugins

| Plugin              | Installation                                                         | Core Use Case                                                                                                                              |
| :------------------ | :------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------- |
| **`helm-diff`**     | `helm plugin install https://github.com/databus23/helm-diff`         | Previews unified diffs between current cluster state and proposed upgrades: <br>`helm diff upgrade myrelease bitnami/nginx -f values.yaml` |
| **`helm-secrets`**  | `helm plugin install https://github.com/jkroepke/helm-secrets`       | Decrypts secrets on the fly using Mozilla SOPS, AWS KMS, or HashiCorp Vault.                                                               |
| **`helm-unittest`** | `helm plugin install https://github.com/helm-unittest/helm-unittest` | Runs BDD-style unit tests against Go templates in CI pipelines without a cluster.                                                          |

---

## 🧹 10. Cache & Local Clean Up

```bash
# View locally cached chart packages
helm cache list

# Show Helm configuration paths and environment variables
helm env
```

---

## 💡 Top 6 Production Rules for Daily Use

1. **Always run `helm repo update`** before installs or upgrades to avoid stale chart index caching.
2. **Use `helm upgrade --install`** in CI/CD automation pipelines for idempotent execution.
3. **Always run `helm diff upgrade`** before modifying live infrastructure to review the exact manifest changes.
4. **Pin chart versions (`--version`)** in production pipelines; never let automation float on `latest`.
5. **Use `helm get values <release>`** to audit active values rather than assuming source control matches cluster state.
6. **Pair `--atomic` and `--cleanup-on-fail`** to prevent failed deployments from leaving orphan resources in a broken state.

