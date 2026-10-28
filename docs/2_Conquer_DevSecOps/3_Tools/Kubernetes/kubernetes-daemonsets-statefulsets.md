# 🛑 Stop Treating All Pods Like Cattle: The Ultimate Guide to DaemonSets & StatefulSets



If you've spent any time working with Kubernetes, you've likely mastered **Deployments**. 

Deployments treat Pods as "cattle"—they are stateless, interchangeable, and disposable. If a Pod dies, the Deployment spins up a replacement with a random hash name, places it onto any available node, and forgets past ephemeral state.

**Deployments fail when workloads are tightly coupled to physical hardware or require strict identity and persistent memory.**

To level up your cloud-native architecture, you must master **DaemonSets** and **StatefulSets**.

---

## 🏗️ Part 1: Core Concepts

### 1. The Case for DaemonSets (Infrastructure & Node Layer)

A **DaemonSet** ensures that an exact copy of a specific Pod runs on all (or an explicitly selected subset of) Nodes in your cluster.

* ❌ **Why Deployments Fail Here:** If you tell a Deployment to run 5 replicas across a 5-node cluster, the Kubernetes scheduler might place 3 pods on Node A, 2 on Node B, and leave Nodes C, D, and E empty.
* ✅ **Why We Need DaemonSets:**
  * **Cluster-Wide Logging:** Agents like Fluent Bit, Promtail, or Logstash must run on every physical host to tail local container log files.
  * **Hardware & Node Monitoring:** Exporters like Prometheus Node Exporter or Datadog agents must poll the specific host CPU, RAM, and disk metrics of every machine.
  * **CNI Networking Plugins:** Pod-to-pod network overlays (e.g., Calico, Cilium, Flannel) require a host-level daemon on every node to program network interfaces and routing tables.
  * **Dynamic Cluster Expansion:** When your cluster autoscaler spins up a new worker node, the DaemonSet controller detects the new node and deploys the agent pod automatically.

#### 🔑 Advanced: Targeting Nodes with `nodeSelector` & Tolerations
In production, you often need to target specific hardware pools or schedule onto control-plane nodes that have taints applied:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        node-role: monitoring
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: monitor
        image: prom/node-exporter:v1.8.0
```

---

### 2. The Case for StatefulSets (Databases & State)

A **StatefulSet** manages stateful workloads (PostgreSQL, Kafka, ZooKeeper, Elasticsearch, Redis clusters) where pod identity and persistent storage are essential.

* ❌ **Why Deployments Fail Here:** Deployments assign random hashes (`web-7b9c6d...`), launch replicas in parallel, and re-create failed pods with brand-new identities and disconnected storage. Clustered databases cannot maintain quorum under this model.
* ✅ **Why We Need StatefulSets:**
  * **Stable, Predictable Network Identities:** Pods receive ordered index names (`db-0`, `db-1`, `db-2`). If `db-0` crashes, Kubernetes re-creates it as `db-0`.
  * **Dedicated, Persistent Storage (`volumeClaimTemplates`):** Each replica receives an independent PVC (`data-db-0`, `data-db-1`). If a pod dies, Kubernetes reattaches the exact same volume to the recreated replica.
  * **Ordered Initialization & Teardown:** Pods are deployed sequentially (`0` ➔ `1` ➔ `2`). `db-1` is not created until `db-0` passes readiness. During scale-down, pods terminate in reverse order (`2` ➔ `1` ➔ `0`) to prevent split-brain scenarios.
  * **Headless Services (`clusterIP: None`):** Creates direct A-records in CoreDNS for each individual pod:
    ```text
    <pod-name>.<service-name>.<namespace>.svc.cluster.local
    # Example: db-0.nginx-db-svc.default.svc.cluster.local
    ```
    This enables database replicas to establish direct peer connections to the primary node.

#### 🔑 Advanced StatefulSet Knobs:
* **`podManagementPolicy: Parallel`:** Bypasses sequential startup and launches all replicas concurrently. Best for stateless caches or systems that handle their own cluster synchronization.
* **`updateStrategy.rollingUpdate.partition`:** Enables canary rollouts. Setting `partition: 2` in a 3-replica set ensures only `db-2` receives the new image version, while `db-0` and `db-1` remain on the existing version.

---

## 🛠️ Part 2: The Hands-On Runbook

---

### Phase A: The DaemonSet Runbook

**Goal:** Prove that a pod gets deployed to every single worker node automatically.

#### Step 1: Inspect Cluster Nodes
```bash
kubectl get nodes
```

#### Step 2: Create `daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: hardware-monitor
  labels:
    app: hardware-monitor
spec:
  selector:
    matchLabels:
      name: hardware-monitor
  template:
    metadata:
      labels:
        name: hardware-monitor
    spec:
      containers:
      - name: monitor
        image: prom/node-exporter:v1.8.0
        ports:
        - containerPort: 9100
```

#### Step 3: Apply and Verify Node Placement
```bash
kubectl apply -f daemonset.yaml
kubectl get pods -o wide -l app=hardware-monitor
```
*Notice the `NODE` column: Kubernetes schedules exactly one pod per eligible node.*

#### Step 4: Clean Up
```bash
kubectl delete -f daemonset.yaml
```

---

### Phase B: The StatefulSet Runbook

**Goal:** Verify ordered creation, persistent network identity, and volume binding.

#### Step 1: Create `statefulset.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-db-svc
  labels:
    app: nginx-db
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None    # Makes this a Headless Service
  selector:
    app: nginx-db
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-db
spec:
  serviceName: "nginx-db-svc"
  replicas: 3
  selector:
    matchLabels:
      app: nginx-db
  template:
    metadata:
      labels:
        app: nginx-db
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www-data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:   # Allocates a dedicated PVC per replica
  - metadata:
      name: www-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

#### Step 2: Observe Ordered Sequential Initialization
```bash
kubectl apply -f statefulset.yaml
kubectl get pods -w
```
*`web-db-0` starts first. Only after it is `Running` will `web-db-1` initialize, followed by `web-db-2`.*

#### Step 3: Verify Stable Network Identity
```bash
kubectl get pods -l app=nginx-db
```
*Pods have predictable names (`web-db-0`, `web-db-1`, `web-db-2`), not random hashes.*

#### Step 4: Verify Individual DNS Addressing (Headless Service)
Query the internal CoreDNS record of `web-db-0`:

```bash
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup web-db-0.nginx-db-svc.default.svc.cluster.local
```
*The DNS query resolves directly to the individual Pod IP.*

#### Step 5: Test Storage & Identity Permanence (Failure Simulation)
Write custom data to `web-db-1`:

```bash
kubectl exec -it web-db-1 -- sh -c "echo 'Stateful data preserved' > /usr/share/nginx/html/index.html"

# Delete the pod to simulate a failure
kubectl delete pod web-db-1
kubectl get pods -w
```

*The StatefulSet recreates a pod named `web-db-1` and reattaches its original PVC. Confirm data survived:*

```bash
kubectl exec -it web-db-1 -- cat /usr/share/nginx/html/index.html
# Output: Stateful data preserved
```

#### Step 6: Verify Ordered Scale-Down
```bash
kubectl scale statefulset web-db --replicas=1
kubectl get pods -w
```
*Kubernetes terminates `web-db-2` first, waits for it to exit cleanly, then terminates `web-db-1`, leaving `web-db-0` untouched.*

---

### Step 7: Teardown & The PVC Gotcha

```bash
kubectl delete -f statefulset.yaml
```

> ⚠️ **CRITICAL OPERATIONAL GOTCHA: PVCs Survive Deletion Intentionally!**  
> When you delete a StatefulSet, Kubernetes **does not delete the associated PersistentVolumeClaims (PVCs)**. This design prevents catastrophic data loss if a StatefulSet is dropped accidentally.  
> You must delete PVCs manually when tearing down test environments:
> ```bash
> kubectl delete pvc -l app=nginx-db
> ```

---

## 📋 Architectural Cheat Sheet

| Feature             | Deployment                  | DaemonSet                              | StatefulSet                                               |
| :------------------ | :-------------------------- | :------------------------------------- | :-------------------------------------------------------- |
| **Pod Identity**    | Random hash (`app-5d8f...`) | Random hash (`ds-82f...`)              | **Deterministic index (`db-0`, `db-1`)**                  |
| **Scheduling**      | Anywhere with capacity      | **Every node (or filtered node pool)** | Spread across available nodes                             |
| **Storage Binding** | Shared volume or stateless  | Local host mounts (`hostPath`)         | **Dedicated volume per replica (`volumeClaimTemplates`)** |
| **Rollout Order**   | Rolling surge/unavailable   | Rolling update per node                | **Ordered (`0` ➔ `1` ➔ `2`) or Partitioned**              |
| **Primary Use**     | Web APIs, stateless workers | Logging, metrics, CNI networking       | **Databases, message queues, distributed consensus**      |

