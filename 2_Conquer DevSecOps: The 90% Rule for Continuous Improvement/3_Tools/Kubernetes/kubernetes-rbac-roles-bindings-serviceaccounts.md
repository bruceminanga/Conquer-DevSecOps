# 🔐 Mastering Kubernetes RBAC: The Ultimate Guide & Step-by-Step Runbook



Have you ever granted a Kubernetes Pod or CI/CD runner `cluster-admin` ("God Mode") permissions just to make a pipeline pass?

We've all seen it happen. But in production, running without strict **Role-Based Access Control (RBAC)** creates a severe blast radius for security breaches and accidental deletions.

Understanding **Roles**, **RoleBindings**, **ClusterRoles**, and **ServiceAccounts** is non-negotiable for securing Kubernetes clusters. 

Here is why RBAC is mandatory, followed by an exhaustive, 11-step hands-on runbook.

---

## 🧠 Part 1: Why Do We Need RBAC?

At its core, Kubernetes RBAC answers three vital questions:
1. **Who** is taking the action? *(Subject)*
2. **What** are they allowed to do? *(Verbs: `get`, `list`, `create`, `delete`)*
3. **Where** are they allowed to do it? *(Scope: Specific Namespace vs. Cluster-Wide)*

### 5 Critical Security Drivers:

1. **The Principle of Least Privilege:** A compromised web-tier pod should never have permissions to read secrets from other namespaces or modify node configurations.
2. **Differentiating Humans from Machines:**
   * **Users (Humans):** Managed outside the cluster via identity providers (AWS IAM, Okta, Azure AD, OIDC). Kubernetes does not maintain user database objects.
   * **ServiceAccounts (Machines):** Native Kubernetes identities assigned to Pods, GitHub Actions runners, or external automation pipelines.
3. **Limiting the Blast Radius:** A `Role` locked to the `frontend` namespace ensures an errant `kubectl delete pods --all` cannot impact backend or database services.
4. **Auditing & Regulatory Compliance (PCI-DSS / HIPAA):** RBAC ties identities to API server access logs, generating an auditable trail of every create, update, and delete action.
5. **Clean Multi-Tenancy:** Prevents distinct development teams sharing the same physical cluster from modifying or inspecting each other's configurations.

---

## 🏗️ The 4 Core Primitives Summarized

| Primitive                | Real-World Analogy   | Scope        | Description                                                                              |
| :----------------------- | :------------------- | :----------- | :--------------------------------------------------------------------------------------- |
| **`ServiceAccount`**     | The **ID Card**      | Namespaced   | Native machine identity for workloads and automation tools.                              |
| **`Role`**               | The **Room Key**     | Namespaced   | Defines allowed actions (`verbs`) on specific resources within **one** namespace.        |
| **`ClusterRole`**        | The **Master Key**   | Cluster-Wide | Defines permissions across **all** namespaces and cluster-scoped resources (Nodes, PVs). |
| **`RoleBinding`**        | The **Access Grant** | Namespaced   | Attaches an identity (`Subject`) to a `Role` within a specific namespace.                |
| **`ClusterRoleBinding`** | The **Global Pass**  | Cluster-Wide | Attaches an identity (`Subject`) to a `ClusterRole` across the entire cluster.           |

---

## ⚠️ The Hidden Footgun: Default ServiceAccounts

Every namespace automatically includes a ServiceAccount named `default`. If you omit `serviceAccountName` in your Pod specification, Kubernetes silently assigns this default identity and **auto-mounts its API token into the container filesystem** at `/var/run/secrets/kubernetes.io/serviceaccount/token`.

This means pods that have no business communicating with the Kubernetes API server still receive valid cluster credentials, creating an unnecessary attack surface.

### The Remediation:
Disable auto-mounting explicitly on workloads that do not interact with the API:

```yaml
# Disable on the Pod directly:
spec:
  automountServiceAccountToken: false
```

Or disable it globally at the `ServiceAccount` level:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-app
automountServiceAccountToken: false
```

---

## 🛠️ Part 2: Hands-On RBAC Runbook

---

### Step 1: Create an Isolated Sandbox Namespace

```bash
kubectl create namespace rbac-demo
```

---

### Step 2: Provision a ServiceAccount (The Machine Identity)

Create an identity for an automated backup runner:

```bash
kubectl create serviceaccount backup-sa --namespace rbac-demo
```

---

### Step 3: Test Access Before Applying Permissions (Baseline)

Verify default permissions using `kubectl auth can-i` impersonation:

```bash
kubectl auth can-i list pods \
  --namespace rbac-demo \
  --as=system:serviceaccount:rbac-demo:backup-sa
# Output: no
```

*By default, ServiceAccounts have zero permissions.*

---

### Step 4: Define Local Permissions (Role)

Our backup tool needs read-only access to Pods inside `rbac-demo`, with no permission to delete or modify them.

Create `role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-role
  namespace: rbac-demo
rules:
- apiGroups: [""] # Core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

Apply:
```bash
kubectl apply -f role.yaml
```

---

### Step 5: Bind the Identity to the Role (RoleBinding)

Attach `backup-sa` to `pod-reader-role`.

Create `rolebinding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: rbac-demo
subjects:
- kind: ServiceAccount
  name: backup-sa       
  namespace: rbac-demo
roleRef:
  kind: Role
  name: pod-reader-role 
  apiGroup: rbac.authorization.k8s.io
```

Apply:
```bash
kubectl apply -f rolebinding.yaml
```

---

### Step 6: Validate the Scope

```bash
# 1. Can it list pods in rbac-demo?
kubectl auth can-i list pods --namespace rbac-demo --as=system:serviceaccount:rbac-demo:backup-sa
# Output: yes ✅

# 2. Can it delete pods in rbac-demo?
kubectl auth can-i delete pods --namespace rbac-demo --as=system:serviceaccount:rbac-demo:backup-sa
# Output: no ❌

# 3. Can it list pods in another namespace (e.g. default)?
kubectl auth can-i list pods --namespace default --as=system:serviceaccount:rbac-demo:backup-sa
# Output: no ❌ (Roles are strictly namespaced)
```

---

### Step 7: Define Cluster-Wide Permissions (ClusterRole)

Non-namespaced resources (such as `Node` or `PersistentVolume`) require a `ClusterRole`.

Create `clusterrole.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader-clusterrole
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
```

Apply:
```bash
kubectl apply -f clusterrole.yaml
```

---

### 💡 Step 7.5: The Pro Pattern (ClusterRole + RoleBinding)

A `ClusterRole` does not always have to be bound cluster-wide! 

You can bind a `ClusterRole` using a namespaced **`RoleBinding`**. This allows you to define a standardized permission set centrally once, and reuse it across individual namespaces without granting global cluster access:

```text
ClusterRole: pod-reader  (Defined once centrally)
     │
     ├── RoleBinding in 'frontend' ──► Scoped strictly to frontend pods
     └── RoleBinding in 'payments' ──► Scoped strictly to payments pods
```

#### Binding Decision Matrix:

| Combination                          | Scope Granted        | Best Use Case                                                       |
| :----------------------------------- | :------------------- | :------------------------------------------------------------------ |
| **Role + RoleBinding**               | Single Namespace     | Custom, one-off local permissions.                                  |
| **ClusterRole + RoleBinding**        | **Single Namespace** | Reusing a standard role across teams without cluster-wide exposure. |
| **ClusterRole + ClusterRoleBinding** | **Entire Cluster**   | Infrastructure daemons, cluster monitoring, global admin tools.     |

---

### Step 8: Grant Global Node Access (ClusterRoleBinding)

Because nodes are non-namespaced resources, we use a `ClusterRoleBinding` here.

Create `clusterrolebinding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global-binding
subjects:
- kind: ServiceAccount
  name: backup-sa       
  namespace: rbac-demo
roleRef:
  kind: ClusterRole
  name: node-reader-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

Apply:
```bash
kubectl apply -f clusterrolebinding.yaml
```

Verify node query access:
```bash
kubectl auth can-i list nodes --as=system:serviceaccount:rbac-demo:backup-sa
# Output: yes ✅
```

---

### Step 9: Inject the ServiceAccount into a Live Pod

Create `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backup-app
  namespace: rbac-demo
spec:
  serviceAccountName: backup-sa          # Assigns our custom identity
  automountServiceAccountToken: true    # Explicit: this pod needs API access
  containers:
  - name: main
    image: alpine:3.19
    command: ["sleep", "3600"]
```

Apply:
```bash
kubectl apply -f pod.yaml
```

---

### Step 10: Test Permissions from Inside the Container

Exec into the running pod and query the Kubernetes API server using the injected token:

```bash
# 1. Open shell inside container
kubectl exec -it backup-app -n rbac-demo -- sh

# 2. Install curl
apk add --no-cache curl

# 3. Read the mounted API token
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# 4. Query pods in rbac-demo (Permitted)
curl -k -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/rbac-demo/pods

# 5. Query pods in kube-system (Forbidden)
curl -k -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/kube-system/pods
# Returns: 403 Forbidden!

exit
```

---

### Step 11: Cleanup

```bash
kubectl delete namespace rbac-demo
kubectl delete clusterrole node-reader-clusterrole
kubectl delete clusterrolebinding read-nodes-global-binding
```

---

## 📋 Production Security Checklist

- [ ] Disable `automountServiceAccountToken: false` on pods that do not interact with the API server.
- [ ] Never grant `cluster-admin` to application workloads or CI/CD pipelines.
- [ ] Use `ClusterRole` + `RoleBinding` to template roles across multiple tenant namespaces.
- [ ] Restrict `verbs` to read-only (`get`, `list`, `watch`) unless write actions are explicitly required.
- [ ] Validate permissions before applying changes using `kubectl auth can-i --as=...`.

