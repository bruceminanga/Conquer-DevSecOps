# 🔐 Stop Hardcoding Passwords in Kubernetes! The Ultimate Guide to K8s Secrets



If you are hardcoding database passwords or API keys into your application code or container images, you are creating a massive security risk. But even if you've moved them out of your code, are you managing them correctly inside Kubernetes?

In this guide, we break down why Kubernetes Secrets are essential, compare injection strategies, and walk through a production-ready runbook including RBAC isolation.

---

## 🛡️ Why Do We Need Kubernetes Secrets?

Why not just use a standard `ConfigMap`? Secrets provide distinct runtime and security protections:

1. **Decoupled Portability:** Keeps your container images environment-agnostic. The identical application image runs across Dev, Staging, and Production, with Kubernetes injecting the appropriate database credentials at runtime.
2. **In-Memory Storage (`tmpfs`):** When mounted into a Pod as a file volume, Kubernetes stores the secret in a memory-backed temporary filesystem (`tmpfs`). The secret exists strictly in RAM and is never written to the host node's physical disk.
3. **Strict RBAC Separation:** You can grant developers permission to view and edit ConfigMaps while completely locking down access to Secrets via Role-Based Access Control.
4. **Encryption at Rest:** Standard cluster state lives in `etcd`. While Secrets are only Base64-encoded by default, cluster administrators can enable **KMS Encryption at Rest** to ensure data is encrypted on disk.

---

## 🛠️ The Runbook: Building & Injecting Secrets

**Scenario:** A Django web application connecting to a PostgreSQL database. The credentials must be stored securely in Kubernetes and injected at runtime.

---

### Step 1: Create the Secret

You can declare secrets using `stringData` (human-readable plaintext that Kubernetes encodes on apply) or `data` (manually Base64-encoded strings).

Create `postgres-credentials-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials-secret
  namespace: default
type: Opaque
immutable: true                       # Best practice: locks secret to prevent drift and reduces API polling
stringData:
  postgres-password: "SuperSecret123!"
  postgres-user: "django_user"
  postgres-db: "django_db"
```

> 💡 **Manual Base64 Note:** If using the `data` field instead of `stringData`, encode values without newlines:
> ```bash
> echo -n 'SuperSecret123!' | base64
> # Output: U3VwZXJTZWNyZXQxMjMh
> ```
> *(Remember: Base64 is an encoding format for binary transfer, **not** encryption).*

Apply the secret:

```bash
kubectl apply -f postgres-credentials-secret.yaml
```

#### Common Built-in Secret Types:
* `Opaque`: Arbitrary user-defined key-value credentials (default).
* `kubernetes.io/dockerconfigjson`: Credentials for pulling images from private container registries.
* `kubernetes.io/tls`: Public certificates and private keys for TLS termination.

---

### Step 2: Inject the Secret into a Deployment

Create `django-deployment.yaml` demonstrating both injection methods (Environment Variable vs. Volume Mount):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-deployment
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django-app
  template:
    metadata:
      labels:
        app: django-app
    spec:
      containers:
      - name: django-container
        image: django-app:1.4.2
        ports:
        - containerPort: 8000

        # METHOD 1: As an Environment Variable
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials-secret
              key: postgres-password

        # METHOD 2: As a Mounted File (tmpfs in RAM)
        volumeMounts:
        - name: postgres-credentials-volume
          mountPath: "/etc/postgres-secrets"
          readOnly: true

      volumes:
      - name: postgres-credentials-volume
        secret:
          secretName: postgres-credentials-secret
```

Apply and verify:

```bash
kubectl apply -f django-deployment.yaml

# Read the in-memory secret file from the container:
kubectl exec -it deployment/django-deployment -- cat /etc/postgres-secrets/postgres-password
```

---

## 🤯 Env Vars vs. Mounted Volumes: Security Breakdown

| Feature              | Environment Variables (`env`)                     | Mounted Volumes (`tmpfs`)                 |
| :------------------- | :------------------------------------------------ | :---------------------------------------- |
| **Exposure Surface** | ❌ Leaks to `/proc/1/environ` and all subprocesses | ✅ Scoped only to explicit file reads      |
| **Crash Reports**    | ❌ Often dumped in app crash stack traces          | ✅ Never dumped in process stack traces    |
| **Disk Security**    | ❌ Stored in container metadata                    | ✅ Stored only in RAM; purged on pod death |
| **Live Updates**     | ❌ Never update without full pod restart           | ✅ Kubelet auto-refreshes file in ~1 min   |

> 🚨 **The Dynamic Update Trap:** While mounted volume files update automatically, most runtimes (Django, Spring Boot, Node.js) only read database credentials once on startup. To pick up rotated secrets automatically, deploy an operator like **[Stakater Reloader](https://github.com/stakater/Reloader)** to trigger rolling restarts on secret changes.

---

## 🔑 RBAC: Locking Down Secret Access

Never grant blanket read permissions. Below, we grant developer `alice` access to view `ConfigMaps` while strictly denying access to `Secrets`.

### 1. Define the Role (`configmap-reader-role.yaml`):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader-role
  namespace: default
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
# Note: "secrets" is omitted — access is denied by default
```

### 2. Bind the Role (`configmap-reader-binding.yaml`):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-reader-binding
  namespace: default
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: configmap-reader-role
  apiGroup: rbac.authorization.k8s.io
```

Apply the RBAC rules:

```bash
kubectl apply -f configmap-reader-role.yaml
kubectl apply -f configmap-reader-binding.yaml
```

### 3. Verify Permissions with `auth can-i`:

```bash
# Verify permissions logically:
kubectl auth can-i get configmaps --as=alice
# Output: yes

kubectl auth can-i get secrets --as=alice
# Output: no

# Verify execution behavior:
kubectl get secrets --as=alice
# Output: Error from server (Forbidden): secrets is forbidden...
```

---

## 🗺️ The Secrets Maturity Ladder

As architectures scale, secret management typically evolves through five stages:

```text
Level 1: Native K8s Secrets (Base layer in etcd, Base64 encoded)
    ↓
Level 2: Sealed Secrets (Encrypted YAML safe to commit to Git/GitOps)
    ↓
Level 3: External Secrets Operator (Syncs from AWS Secrets Manager / Azure Key Vault / GCP)
    ↓
Level 4: Secrets Store CSI Driver (Mounts cloud secrets directly into RAM; never touches etcd)
    ↓
Level 5: HashiCorp Vault (Dynamic on-demand credentials with short TTLs and audit logs)
```

---

## 📋 Production Readiness Checklist

- [ ] **Set `immutable: true`:** Prevents accidental modification and removes kubelet API polling overhead.
- [ ] **Never commit plaintext secrets to Git:** Use Sealed Secrets or External Secrets Operator for GitOps.
- [ ] **Enable KMS Encryption at Rest:** Verify with your cloud provider (EKS, GKE, AKS) that `etcd` disk encryption is active.
- [ ] **Prefer Mounted Volumes over Env Vars:** Prevents leaking credentials into `/proc/1/environ` and error logs.
- [ ] **Sanitize stdout logs:** Ensure applications do not dump runtime configs or environment variables into logging aggregators (Datadog, CloudWatch).
- [ ] **Automate secret rotation:** Implement controllers like Reloader or native HashiCorp Vault agent sidecars.

