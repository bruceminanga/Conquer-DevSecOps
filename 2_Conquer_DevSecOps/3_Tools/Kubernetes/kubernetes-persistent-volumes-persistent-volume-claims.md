# 💾 Mastering Kubernetes Storage: Why We Need PVs & PVCs + A Complete Runbook



If you've ever deployed a database or stateful application in Kubernetes, you know the golden rule: **Containers are ephemeral. They are born to die.**

If a Deployment pod crashes or restarts, all data created or modified inside that container is permanently lost. How do we run databases, store user uploads, and keep state safe across restarts?

Enter Kubernetes storage primitives: **PersistentVolumes (PV)** and **PersistentVolumeClaims (PVC)**.

Here is a breakdown of why this decoupled architecture exists, followed by a step-by-step hands-on runbook proving data survival across workload deletion.

---

## 💡 Core Concepts You Need to Know

### 1️⃣ Separation of Concerns (Cluster Admin vs. App Developer)
In enterprise infrastructure, the engineer building the app is rarely the administrator provisioning physical SAN storage or cloud block devices:
* **The PersistentVolume (PV):** Provisioned by the **Cluster Administrator** (or CSI driver). Defines an actual storage slice (e.g., AWS EBS, Azure Disk, NFS, local disk).
* **The PersistentVolumeClaim (PVC):** Requested by the **Application Developer**. Acts as a ticket/voucher stating: *"I need 5Gi of storage with ReadWriteOnce access."* Developers don't need to know the underlying cloud API.

### 2️⃣ The Matchmaking Rules (How Binding Works)
When a PVC is created, Kubernetes runs through a strict compatibility matrix before binding it to a PV:
1. **StorageClass:** Must match between the PVC and PV.
2. **Access Modes:** Must be compatible (e.g., an `RWX` PVC will reject an `RWO` PV).
3. **Capacity:** The PV's capacity must be **greater than or equal to** the PVC's requested size.

If any of these conditions fail, the PVC remains stuck in `Pending`. Always inspect failures with `kubectl describe pvc <name>`.

### 3️⃣ Access Modes Explained

| Mode                 | Short Name | Meaning                                  | Common Use Case                   |
| :------------------- | :--------- | :--------------------------------------- | :-------------------------------- |
| **ReadWriteOnce**    | `RWO`      | Mounted read-write by a **single node**  | Databases (Postgres, MySQL)       |
| **ReadOnlyMany**     | `ROX`      | Mounted read-only by **multiple nodes**  | Static web assets, ML models      |
| **ReadWriteMany**    | `RWX`      | Mounted read-write by **multiple nodes** | Shared filesystems (NFS, AWS EFS) |
| **ReadWriteOncePod** | `RWOP`     | Mounted read-write by a **single Pod**   | Strict multi-tenant isolation     |

### 4️⃣ Reclaim Policies & Data Protection
What happens to underlying disk data when a PVC is deleted?
* **`Retain` (Recommended for critical state):** Data stays intact. The PV transitions to a `Released` state (not `Available`) so no other claim can accidentally overwrite it until an administrator explicitly clears the claim.
* **`Delete`:** Kubernetes automatically invokes the cloud provider API to delete the underlying storage volume (e.g., deletes the AWS EBS volume).
* **`Recycle`:** *(Deprecated)* Scrubs data via basic `rm -rf` and makes the volume available again.

### 5️⃣ Dynamic Provisioning & StorageClasses
In production, administrators rarely provision individual PVs manually. Instead, they define a `StorageClass`. When a developer applies a PVC referencing that class, the Container Storage Interface (CSI) driver dynamically calls the cloud provider's API, allocates the volume, and registers the PV automatically.

### 6️⃣ The Availability Zone Trap (`WaitForFirstConsumer`) 🚨
If a cloud disk is dynamically provisioned the second a PVC is applied, the cloud provider might place the EBS volume in `us-east-1a`. If the pod is later scheduled on a node in `us-east-1b`, the pod will crash with `FailedMount` errors.

> 🔒 **The Fix:** Always configure your StorageClass with:
> ```yaml
> volumeBindingMode: WaitForFirstConsumer
> ```
> This delays cloud disk provisioning until the Pod is actually scheduled onto a specific node, guaranteeing that the storage and compute live in the same Availability Zone.

### 7️⃣ Volume Expansion (Day-2 Operations)
If a database disk fills up, you do not need to migrate data to a new volume. If your StorageClass sets `allowVolumeExpansion: true`, simply edit the PVC's `spec.resources.requests.storage` from `5Gi` to `10Gi`. The CSI driver expands the filesystem on the fly.

---

## 🛠️ The Hands-On Runbook

We will manually provision a PV, claim it with a PVC, mount it to an NGINX web server, write persistent data, completely destroy the Deployment, and confirm the data survives.

> ⚠️ **Local Testing Note:** We use `hostPath` for this lab so it runs on Minikube, kind, or Docker Desktop without cloud credentials. *Never use `hostPath` in production multi-node environments.*

---

### Step 1: Administrator Provisions the PV

Create `1-pv.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data"
```

Apply and verify:

```bash
kubectl apply -f 1-pv.yaml
kubectl get pv
```

*Expected Status:* `Available`

---

### Step 2: Developer Claims the Storage

Create `2-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply and verify:

```bash
kubectl apply -f 2-pvc.yaml
kubectl get pvc
```

*Expected Status:* `Bound` (matched to `task-pv-volume`).

---

### Step 3: Workload Mounts the Claimed Storage

Create `3-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-pv-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: task-pv-app
  template:
    metadata:
      labels:
        app: task-pv-app
    spec:
      containers:
        - name: task-pv-container
          image: nginx:1.27
          ports:
            - containerPort: 80
              name: "http-server"
          volumeMounts:
            - mountPath: "/usr/share/nginx/html"
              name: task-pv-storage
      volumes:
        - name: task-pv-storage
          persistentVolumeClaim:
            claimName: task-pv-claim
```

Apply and verify mount status:

```bash
kubectl apply -f 3-deployment.yaml

# 1. Confirm the pod is Running
kubectl get pods -l app=task-pv-app

# 2. Confirm the storage filesystem is mounted inside the container
kubectl exec -it deploy/task-pv-deployment -- df -h /usr/share/nginx/html
```

---

### Step 4: The Acid Test (Simulating a Failure) 💥

#### 1. Write persistent data to the mounted volume:
```bash
kubectl exec -it deploy/task-pv-deployment -- bash -c "echo 'Hello from Persistent Storage! This data survived the crash.' > /usr/share/nginx/html/index.html"
```

#### 2. Destroy the entire Deployment:
```bash
kubectl delete deployment task-pv-deployment
```

Verify that the Pod is completely gone:
```bash
kubectl get pods -l app=task-pv-app
```

#### 3. Recreate the Deployment:
```bash
kubectl apply -f 3-deployment.yaml
```

#### 4. Verify data integrity:
```bash
kubectl exec -it deploy/task-pv-deployment -- curl http://localhost
```

**Result:** The new Pod reads from the volume and outputs:
```text
Hello from Persistent Storage! This data survived the crash.
```

---

## 🧹 Cleanup & The Reclaim Policy in Action

Because our Reclaim Policy was set to `Retain`, deleting the PVC does **not** return the PV to `Available`. It enters a protected `Released` state:

```bash
kubectl delete pvc task-pv-claim
kubectl get pv
```

*Status:* `Released`

Kubernetes locks the `spec.claimRef` to prevent accidental overwrites. To re-use this volume for a new PVC, an administrator must manually clear the claim binding:

```bash
# Clear the claimRef so the PV transitions from Released -> Available
kubectl patch pv task-pv-volume -p '{"spec":{"claimRef": null}}'

# Verify status is Available again
kubectl get pv
```

Finally, remove the PV:
```bash
kubectl delete pv task-pv-volume
```

---

## 📋 Production Storage Checklist

- [ ] Use `WaitForFirstConsumer` in cloud StorageClasses to prevent cross-AZ scheduling mismatches.
- [ ] Set `persistentVolumeReclaimPolicy: Retain` on critical stateful data (databases, logs).
- [ ] Use **StatefulSets** (with `volumeClaimTemplates`) for distributed applications requiring individual stable disks per replica.
- [ ] Enable `allowVolumeExpansion: true` on StorageClasses for zero-migration volume resizing.

