# 🚀 Mastering Advanced Kubernetes Scheduling
### *(Node Affinity, Pod Spreading, Topology Constraints, Taints & Tolerations)*



Out of the box, the default Kubernetes scheduler (`kube-scheduler`) is simplistic: it inspects a pod's CPU and memory requests and places it onto any node with adequate available allocatable capacity.

In production environments, clusters are **heterogeneous** (mixed node sizes, GPU accelerators, dedicated SSD pools) and run diverse workloads. Relying strictly on default scheduling can cause expensive GPU nodes to host basic Nginx containers, or compliant database workloads to land on unhardened nodes.

Here is the architectural guide to advanced scheduling, followed by an 8-step practical runbook.

---

## 🧠 Part 1: The 5 Advanced Scheduling Mechanisms

### 1. Node Selectors (The Basic Matcher) 🗂️
* **What it is:** A strict key-value matching system (`spec.nodeSelector`). You label a node (e.g., `disktype=ssd`), and the pod only schedules onto nodes carrying that exact label.
* **Limitations:** Only supports exact equality matches. You cannot define logical `OR` conditions, ranges, or soft preferences.

### 2. Node Affinity & Anti-Affinity (Advanced Attractor / Repeller) 🧲
The expressive evolution of Node Selectors supporting logical operators (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`) and two enforcement tiers:
* **Hard Constraints (`requiredDuringSchedulingIgnoredDuringExecution`):** Mandatory requirements. If no node matches the rule, the pod remains in `Pending`.
* **Soft Constraints (`preferredDuringSchedulingIgnoredDuringExecution`):** Weighted preferences (1–100). Kubernetes attempts to satisfy the rule, but places the pod on an alternate node if preferred nodes are full.

### 3. Pod Affinity & Anti-Affinity (The Neighbor Rules) 🏘️
While Node Affinity attracts Pods to *Nodes*, Pod Affinity schedules Pods relative to *other Pods*:
* **`podAffinity` (Co-location for Low Latency):** Forces an application container onto the exact same node as its local Redis cache pod to eliminate network hops.
* **`podAntiAffinity` (High Availability Spreading):** Prevents two replicas of the same service from landing on the same node, ensuring a single host failure cannot cause a total outage.

### 4. Topology Spread Constraints (The Multi-Zone Distributor) 🗺️
Enforces balanced distribution across failure domains (Nodes, Availability Zones, Cloud Regions) using a mathematical `maxSkew` parameter:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: frontend
```
*Guarantees your replicas are distributed evenly across zones (e.g., `2` in `us-east-1a`, `2` in `us-east-1b`, `2` in `us-east-1c`).*

### 5. Taints & Tolerations (The Bouncer & The VIP Pass) 🛑🎟️
While Affinity attracts Pods to Nodes, **Taints allow a Node to repel a set of Pods**. A taint says: *"Do not schedule here unless you carry an explicit matching Toleration."*

#### The Three Taint Effects:

| Effect                 | Behavior on New Pods                                          | Behavior on Already Running Pods                          |
| :--------------------- | :------------------------------------------------------------ | :-------------------------------------------------------- |
| **`NoSchedule`**       | Rejects pods lacking a toleration.                            | Pods already running are **not** affected.                |
| **`PreferNoSchedule`** | Soft warning: avoids placing pods here if alternatives exist. | Pods already running are **not** affected.                |
| **`NoExecute`**        | Rejects pods lacking a toleration.                            | **Immediately evicts** running pods lacking a toleration. |

> 💡 **Graceful Maintenance with `tolerationSeconds`:** When a node experiences issues or is tainted with `NoExecute`, you can give critical workloads a grace period before eviction:
> ```yaml
> tolerations:
> - key: "node.kubernetes.io/unreachable"
>   operator: "Exists"
>   effect: "NoExecute"
>   tolerationSeconds: 300  # Pod survives 5 minutes before eviction
> ```

---

## 🌟 The Golden Rule of Synergy: Dedicating Nodes

* **Node Affinity** pulls Pods toward specific Nodes. *(But non-matching, unconfigured pods can still land there).*
* **Taints** prevent unauthorized Pods from landing on specific Nodes. *(But tolerated pods can still schedule anywhere else in the cluster).*

> 🔒 **To truly dedicate a node to a single workload:**  
> **Taint the node** (to repel standard workloads) **AND configure Node Affinity on the Pod** (to attract it specifically to that tainted node).

---

## 🛠️ Part 2: The Master Runbook

**Prerequisites:** A Kubernetes cluster with at least 3 nodes (`node-1`, `node-2`, `node-3`) and `kubectl`.

---

### Step 1: Label the Nodes

Assign simulated hardware and environment identities:

```bash
kubectl label nodes node-1 storage=ssd env=production
kubectl label nodes node-2 hardware=gpu
kubectl label nodes node-3 storage=hdd

# Verify assigned labels
kubectl get nodes --show-labels | grep -E "storage|hardware|env"
```

---

### Step 2: Implement Node Selectors

Force an Nginx pod to land strictly on the SSD node (`node-1`).

Create `pod-selector.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ssd-pod
spec:
  nodeSelector:
    storage: ssd
  containers:
  - name: nginx
    image: nginx:1.27
```

Apply and verify:

```bash
kubectl apply -f pod-selector.yaml
kubectl get pod nginx-ssd-pod -o wide
# Scheduled onto node-1
```

---

### Step 3: Implement Node Affinity (Hard & Soft Rules)

Create `pod-affinity.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-affinity-pod
spec:
  affinity:
    nodeAffinity:
      # HARD REQUIREMENT: Must have storage=ssd
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: storage
            operator: In
            values: ["ssd"]
      # SOFT PREFERENCE: Prefers env=production (Weight 1-100)
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: env
            operator: In
            values: ["production"]
  containers:
  - name: nginx
    image: nginx:1.27
```

Apply and verify:

```bash
kubectl apply -f pod-affinity.yaml
kubectl get pod nginx-affinity-pod -o wide
```

---

### Step 4: Implement Pod Anti-Affinity & Co-Location

#### 1. Spread application replicas across different nodes (`app-deployment.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: ["my-app"]
            topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: nginx:1.27
```

```bash
kubectl apply -f app-deployment.yaml
kubectl get pods -o wide -l app=my-app
# The two replicas schedule onto DIFFERENT nodes
```

#### 2. Co-locate a Redis cache pod onto the same node as the app (`cache-pod.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-cache
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["my-app"]
        topologyKey: kubernetes.io/hostname
  containers:
  - name: cache
    image: redis:7.2-alpine
```

```bash
kubectl apply -f cache-pod.yaml
kubectl get pod my-cache -o wide
# Matches the host of one of the my-app pods
```

---

### Step 5: Taints & Tolerations (Dedicated Hardware)

Taint `node-2` so standard pods cannot schedule there:

```bash
kubectl taint nodes node-2 dedicated=ml-team:NoSchedule
```

#### 1. Attempt deploying without a toleration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: regular-pod
spec:
  nodeSelector:
    hardware: gpu
  containers:
  - name: nginx
    image: nginx:1.27
```

```bash
kubectl apply -f regular-pod.yaml
kubectl get pod regular-pod
# Status: Pending
```

`kubectl describe pod regular-pod` will output:
```text
Warning FailedScheduling: 1 node(s) had untolerated taint {dedicated: ml-team}
```

#### 2. Deploy with the matching toleration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-pod
spec:
  nodeSelector:
    hardware: gpu
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "ml-team"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx:1.27
```

```bash
kubectl apply -f ml-pod.yaml
kubectl get pod ml-pod -o wide
# Status: Running on node-2!
```

---

### Step 6: The `NoExecute` Eviction Test

Demonstrating immediate eviction vs. timed grace periods (`tolerationSeconds`).

#### 1. Deploy test pods to `node-3`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: doomed-pod
spec:
  nodeName: node-3
  containers:
  - name: nginx
    image: nginx:1.27
---
apiVersion: v1
kind: Pod
metadata:
  name: graceful-pod
spec:
  nodeName: node-3
  tolerations:
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 60  # Survives 60s before eviction
  containers:
  - name: nginx
    image: nginx:1.27
```

```bash
kubectl apply -f pod-node3.yaml
kubectl get pods -o wide
```

#### 2. Apply the `NoExecute` taint to `node-3`:

```bash
kubectl taint nodes node-3 maintenance=true:NoExecute
kubectl get pods -w
```

* **`doomed-pod`:** Evicted and terminated immediately.
* **`graceful-pod`:** Stays in `Running` state for 60 seconds before terminating.

---

### Step 7: Troubleshooting Checklist

```bash
# 1. Check pod scheduling failure events
kubectl describe pod <pod-name> | grep -A 10 Events

# 2. View all active taints across cluster nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# 3. View node labels
kubectl get nodes --show-labels
```

---

### Step 8: Clean Up

Remove the created pods and use the `-` (minus) suffix to strip labels and taints:

```bash
# Delete Pods & Deployments
kubectl delete pod nginx-ssd-pod nginx-affinity-pod regular-pod ml-pod doomed-pod graceful-pod my-cache --ignore-not-found
kubectl delete deployment my-app --ignore-not-found

# Remove Labels
kubectl label nodes node-1 storage- env-
kubectl label nodes node-2 hardware-
kubectl label nodes node-3 storage-

# Remove Taints
kubectl taint nodes node-2 dedicated-
kubectl taint nodes node-3 maintenance-
```

---

## 📋 Scheduling Cheat Sheet

| Mechanism             | Placed On | Purpose                             | Example Syntax                                       |
| :-------------------- | :-------- | :---------------------------------- | :--------------------------------------------------- |
| **`nodeSelector`**    | Pod Spec  | Exact match node label              | `storage: ssd`                                       |
| **`nodeAffinity`**    | Pod Spec  | Flexible & weighted node placement  | `requiredDuringScheduling...`                        |
| **`podAffinity`**     | Pod Spec  | Co-locate with other pods           | `topologyKey: kubernetes.io/hostname`                |
| **`podAntiAffinity`** | Pod Spec  | Spread replicas across hosts        | `topologyKey: kubernetes.io/hostname`                |
| **`Taints`**          | Node      | Repels pods without matching pass   | `kubectl taint nodes node-1 dedicated=db:NoSchedule` |
| **`Tolerations`**     | Pod Spec  | Permits pod to land on tainted node | `operator: "Equal", effect: "NoSchedule"`            |

