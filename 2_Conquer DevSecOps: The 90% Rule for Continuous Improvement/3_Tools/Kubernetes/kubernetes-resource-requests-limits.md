# 🛑 Flying Blind in Kubernetes? The Ultimate Guide to Resource Requests & Limits



If you aren't setting **Resource Requests and Limits** on your Kubernetes workloads, you are flying blind—and eventually, your cluster will fail under production load.

Here is a comprehensive breakdown of the architectural "Why" and an end-to-end runbook for implementing production-ready resource controls.

---

## 🧠 Part 1: The "Why"

In Kubernetes:
* **Requests:** The resources a container is **guaranteed** to have (used by the scheduler for node placement).
* **Limits:** The **hard ceiling** a container is never allowed to exceed.

### Critical Reasons Requests & Limits are Non-Negotiable:

1. **Intelligent Scheduling:** The `kube-scheduler` makes placement decisions based on pod **Requests**, not current usage. Without requests, it assumes an app needs 0 resources and can schedule 50 compute-heavy pods onto a small worker node, causing an instant node crash.
2. **Preventing "Noisy Neighbors":** Limits place a strict cap on runaway containers (such as memory leaks), preventing a rogue application from starving adjacent workloads on the same physical node.
3. **Preventing Node-Level System OOMs:** If containers exhaust 100% of a node's physical memory, the Linux kernel's `oom-killer` will activate. It may terminate system-critical processes like `kubelet`, `containerd`, or your database pod.
4. **Quality of Service (QoS) Tiering:** Kubernetes assigns every pod a QoS class based on requests and limits, determining eviction priority during node memory pressure:
   * **`Guaranteed`** (`Requests == Limits` for CPU & Memory): Evicted last.
   * **`Burstable`** (`Requests < Limits`): Evicted when resource pressure is high.
   * **`BestEffort`** (No requests or limits configured): Evicted first.
5. **Cluster Autoscaling (Cluster Autoscaler / Karpenter):** Node autoscalers trigger when pods are stuck in `Pending` due to insufficient requested capacity. No requests = no pending capacity signal = no node scale-up.
6. **Cost Allocation & Right-Sizing:** Accurate requests allow platform teams to monitor reserved vs. utilized capacity and budget cloud spend across namespaces.

---

## ⚠️ CPU vs. Memory: A Critical Distinction

| Resource   | Nature             | When Exceeded...                                                                                                            | Metric to Monitor                           |
| :--------- | :----------------- | :-------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------ |
| **CPU**    | **Compressible**   | Workload is throttled by the Linux CFS scheduler (execution slows down, but the pod does not die).                          | `container_cpu_cfs_throttled_seconds_total` |
| **Memory** | **Incompressible** | Linux cannot compress RAM. Kubernetes immediately kills the container with **`OOMKilled` (Exit Code 137)** and restarts it. | `container_memory_working_set_bytes`        |

> 💡 **CFS Throttling Gotcha:** Even when average CPU usage looks low in `kubectl top pods`, bursty workloads (e.g., JVM garbage collection, incoming HTTP spikes) can hit the Completely Fair Scheduler (CFS) quota period limit, causing hidden latency. Always monitor Prometheus for throttling metrics before clamping down CPU limits.

---

## 🛠️ Part 2: Implementation Runbook

### Phase 1: Discovery (Never Guess Your Baselines)

1. Ensure `metrics-server` is deployed in your cluster.
2. Observe standard runtime metrics:
   ```bash
   kubectl top pods -n <namespace>
   ```
3. **Use the Vertical Pod Autoscaler (VPA) in Recommendation Mode:**  
   Deploy VPA with `updateMode: "Off"`. VPA will observe actual consumption patterns over time (accounting for diurnal cycles) and output right-sized request/limit recommendations without restarting your pods.

---

### Phase 2: Formulating the Allocation Strategy

* **Memory:** Set `Request == Limit` for mission-critical services. Because memory cannot be compressed, allowing pods to burst far beyond requests creates unpredictable OOM risks for neighbor pods.
* **CPU:** Set `Request` to your 95th percentile steady-state baseline. Set `Limit` higher (or omit limits entirely if using a cluster-wide Fair-Share strategy) to accommodate rapid short-lived execution bursts without CFS throttling.

---

### Phase 3: Application Manifest Implementation

> ⚠️ **Init Container Warning:** Remember that `initContainers` run before the application starts, and their requests directly impact scheduling calculations. An over-allocated init container can prevent a pod from scheduling even if the main container is lightweight.

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-backend-app
  namespace: my-dev-namespace
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      initContainers:
      - name: init-db-check
        image: busybox:1.36
        command: ['sh', '-c', 'sleep 2']
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "64Mi"
      containers:
      - name: app-container
        image: my-app:v1.0.0
        resources:
          requests:
            cpu: "200m"       # Reserves 0.2 CPU cores
            memory: "256Mi"   # Reserves 256 MiB RAM
          limits:
            cpu: "500m"       # Hard cap at 0.5 CPU cores
            memory: "256Mi"   # Hard cap (Guarantees predictable OOM behavior)
```

---

### Phase 4: Cluster-Level Guardrails (Platform Admin)

Developers occasionally omit resource stanzas. Protect cluster stability with layered perimeter controls.

#### 1. LimitRange (Default Fallbacks)
Injects baseline defaults into any pod manifest submitted without explicit resources:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: my-dev-namespace
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "256Mi"
```

#### 2. ResourceQuota (Namespace Budget Ceiling)
Enforces a hard boundary on aggregate consumption across the entire namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: my-dev-namespace
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
```

Apply and verify:

```bash
kubectl apply -f limitrange.yaml
kubectl apply -f resourcequota.yaml

# Check quota utilization
kubectl describe resourcequota compute-resources -n my-dev-namespace
```

Expected output:
```text
Name:            compute-resources
Namespace:       my-dev-namespace
Resource         Used    Hard
--------         ----    ----
limits.cpu       600m    8
limits.memory    768Mi   16Gi
requests.cpu     300m    4
requests.memory  384Mi   8Gi
```

#### 3. Policy Engines (The Hard Gate)
`LimitRange` merely injects defaults. To strictly prevent un-configured or misconfigured manifests from ever being admitted to the cluster, enforce an Admission Controller policy using **Kyverno** or **OPA Gatekeeper**.

---

## 🔧 Day-2 Troubleshooting Cheatsheet

| Symptom                           | Probable Cause                                    | Action                                                                                           |
| :-------------------------------- | :------------------------------------------------ | :----------------------------------------------------------------------------------------------- |
| **`OOMKilled` (Exit Code 137)**   | Container exceeded `limits.memory`.               | Check for application memory leaks or increase `limits.memory`.                                  |
| **Latency Spikes / Sluggishness** | CPU throttling by Linux CFS.                      | Inspect `container_cpu_cfs_throttled_seconds_total` in Prometheus; raise CPU limits.             |
| **Pod Evictions (`Evicted`)**     | Node experienced overall compute/disk starvation. | Upgrade to `Guaranteed` QoS by aligning `Requests == Limits` on critical apps.                   |
| **Pod stuck in `Pending`**        | Namespace budget or LimitRange blocking schedule. | Inspect with `kubectl describe resourcequota -n <ns>` and `kubectl describe limitrange -n <ns>`. |

---

## 📋 Production Readiness Checklist

- [ ] All containers declare explicit `requests.cpu` and `requests.memory`.
- [ ] Critical production workloads use aligned memory requests and limits (`Guaranteed` QoS).
- [ ] `LimitRange` configured in every developer namespace as a safety fallback.
- [ ] `ResourceQuota` enforced per tenant namespace to protect total cluster capacity.
- [ ] Prometheus alerting configured for CFS throttling and `OOMKilled` pod restarts.

