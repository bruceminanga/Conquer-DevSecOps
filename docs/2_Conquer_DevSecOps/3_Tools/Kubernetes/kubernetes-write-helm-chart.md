# 🚀 The Ultimate Guide to Writing Your Own Helm Chart
### *(From Boilerplate Scaffolding to Production Packaging)*



If you work with Kubernetes, you already know the operational friction: running a single modern application requires juggling multiple YAML files (Deployments, Services, Ingresses, ConfigMaps, Secrets, ServiceAccounts).

As your application ecosystem grows, maintaining static YAML files across multiple environments becomes unmanageable.

To understand why and how to build your own Helm chart, we break this down into two distinct parts:
* **Part 1: The "Why"** — The engineering benefits of custom charts.
* **Part 2: The "How"** — An end-to-end hands-on runbook.

---

## 💡 Part 1: Why Write Your Own Helm Chart?

While public Helm charts exist for common off-the-shelf software (like Nginx, Redis, or PostgreSQL), you must author custom charts for proprietary in-house applications.

1. **♻️ Reusability & DRY Principles:** Without Helm, deploying across environments (Dev, Staging, QA, Prod) involves copy-pasting manifests and hand-editing environment variables, replica counts, and image tags. Helm lets you maintain a single parameterized template set driven by environment-specific values files (`values-dev.yaml`, `values-prod.yaml`).
2. **⏪ Atomic Revision Tracking & Rollbacks:** Helm tracks the release lifecycle with explicit versions (e.g., `v1.0.0`). If a deployment fails, you can roll back the entire resource collection atomically with a single command.
3. **⚙️ Dynamic Go Templating:** Raw Kubernetes YAML is static. With Helm, you can conditionally toggle resources (e.g., `{{ if .Values.ingress.enabled }}`), loop through arrays of environment variables, and compute values dynamically.
4. **☂️ Umbrella Charts for Complex Systems:** If an application relies on a backend API, frontend web client, and background worker, an "Umbrella Chart" allows you to manage them as sub-charts deployed concurrently in the correct dependency order.
5. **🛡️ Enforcing Organizational Guardrails:** Platform teams can maintain an internal "starter" chart that mandates security best practices (e.g., `readOnlyRootFilesystem: true`, non-root execution, mandatory resource limits, and standard telemetry sidecars).
6. **🔄 Standardized CI/CD Integration:** Instead of brittle `sed` or `envsubst` scripts in deployment pipelines, CI/CD tools simply invoke:
   ```bash
   helm upgrade --install my-app ./my-chart --set image.tag=$COMMIT_HASH
   ```

---

## 🛠️ Part 2: The Step-by-Step Runbook

### Pre-flight Prerequisites
* A running Kubernetes cluster (`minikube`, `kind`, or cloud provider).
* `kubectl` and `helm` CLIs installed.
* A containerized application image accessible by the cluster.

---

### 🏗️ Phase 1: Scaffolding the Chart Structure

#### Step 1: Create the Boilerplate
Generate the standard directory hierarchy:

```bash
helm create my-custom-api
cd my-custom-api
```

#### Step 2: Understand the Generated Structure
```text
my-custom-api/
├── Chart.yaml          # Metadata: chart version, app version, name
├── values.yaml         # Default configuration values
├── .helmignore         # Files to ignore during packaging
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl    # Reusable Go template snippets
│   └── NOTES.txt       # Post-installation usage instructions
```

#### Step 2b: The DRY Engine (`_helpers.tpl`)
This file defines named, reusable template functions that can be invoked across your manifests to prevent duplication (such as label blocks and standardized names).

To define a custom shared annotation block, append to `templates/_helpers.tpl`:

```yaml
{{/*
Common organizational annotations
*/}}
{{- define "my-custom-api.commonAnnotations" -}}
app.kubernetes.io/managed-by: Helm
team: platform-engineering
{{- end }}
```

You can now call it from any template (e.g., inside `deployment.yaml` or `service.yaml`):

```yaml
metadata:
  annotations:
    {{- include "my-custom-api.commonAnnotations" . | nindent 4 }}
```

---

### 🛠️ Phase 2: Configuring Metadata & Values

#### Step 3: Update `Chart.yaml`

```yaml
apiVersion: v2
name: my-custom-api
description: Production-grade Helm chart for custom backend API
type: application
version: 0.1.0         # Increment on chart template changes
appVersion: "1.0.0"    # Matches application container image tag
```

#### Step 4: Define Defaults in `values.yaml`

```yaml
replicaCount: 3

image:
  repository: mydockerhub/my-custom-api
  pullPolicy: IfNotPresent
  tag: "1.0.0"

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: api.mycompany.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```

#### Step 4b: Safe Secrets Handling
> ⚠️ **Never commit unencrypted passwords into `values.yaml`!**

1. Create a local, uncommitted values file:
   ```yaml
   # values-secret.yaml (Never commit to Git)
   database:
     password: "SuperSecretPassword123!"
   ```
2. Add it to `.helmignore`:
   ```text
   values-secret.yaml
   ```
3. Pass it during local execution:
   ```bash
   helm install my-backend-api ./my-custom-api \
     -f values.yaml \
     -f values-secret.yaml \
     -n my-namespace
   ```
*(For production, inject credentials using **HashiCorp Vault**, **External Secrets Operator**, or **Sealed Secrets**).*

---

### ✨ Phase 3: Manifest Templating

#### Step 5: Adapt `templates/deployment.yaml`
Notice how values are resolved dynamically and piped to `nindent` for proper indentation:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-custom-api.fullname" . }}
  labels:
    {{- include "my-custom-api.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-custom-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-custom-api.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

#### Step 6: Prune Unused Templates
Delete default boilerplate templates your application doesn't currently use (e.g., `templates/hpa.yaml`, `templates/serviceaccount.yaml`) to keep the chart maintainable.

---

### 🧪 Phase 4: Pre-Flight Testing & Validation

#### Step 7: Linting
Run syntax and schema checks:

```bash
helm lint ./my-custom-api
```
*Expected output:* `1 chart(s) linted, 0 chart(s) failed`

#### Step 8: Client-Side Template Rendering vs. Server-Side Dry-Run

```bash
# Option A: Client-side local rendering (Fast, offline, validates Go templates)
helm template test-release ./my-custom-api --debug

# Option B: Server-side dry-run (Connects to cluster to validate against admission controllers & API schemas)
helm install my-backend-api ./my-custom-api --dry-run -n my-namespace
```

---

### 🚀 Phase 5: Deployment & Lifecycle Management

#### Step 9: Install the Release

```bash
# Multi-environment deployment: layer production overrides over baseline defaults
helm install my-backend-api ./my-custom-api \
  -f values.yaml \
  -f values-prod.yaml \
  -n production \
  --create-namespace
```

`values-prod.yaml` only needs to contain overrides:
```yaml
replicaCount: 5
image:
  tag: "1.2.0"
resources:
  requests:
    cpu: 500m
    memory: 512Mi
```

#### Step 10: Verify the Deployment

```bash
helm list -n production
kubectl get pods -n production
```

#### Step 11: Upgrade (CI/CD Pipeline Flow)

```bash
helm upgrade my-backend-api ./my-custom-api \
  --set image.tag="1.0.1" \
  -n production
```

#### Step 12: Rollback

```bash
# Inspect release revisions
helm history my-backend-api -n production

# Revert to a previous revision
helm rollback my-backend-api 1 -n production
```

---

### 📦 Phase 6: Distribution & OCI Packaging

#### Step 13: Package the Chart

```bash
helm package ./my-custom-api
# Output: Successfully packaged chart and saved it to: my-custom-api-0.1.0.tgz
```

#### Step 14: Push to an OCI Registry

```bash
helm push my-custom-api-0.1.0.tgz oci://ghcr.io/my-organization/helm-charts
```

#### Step 15: Clean Up

```bash
helm uninstall my-backend-api -n production
```

---

## 📋 Best Practices Summary

| Best Practice                      | Description                                                                         |
| :--------------------------------- | :---------------------------------------------------------------------------------- |
| **Pin `appVersion` and `version`** | Adhere strictly to Semantic Versioning (`SemVer`).                                  |
| **Keep `_helpers.tpl` DRY**        | Move repeated naming conventions and label blocks into template helpers.            |
| **Use `nindent` over `indent`**    | Avoid formatting surprises by ensuring consistent indentation across nested blocks. |
| **Always include probes**          | Zero-downtime rolling upgrades depend on configured `readinessProbe` definitions.   |
| **Lint in CI pipelines**           | Enforce `helm lint --strict` on pull requests modifying chart directories.          |

