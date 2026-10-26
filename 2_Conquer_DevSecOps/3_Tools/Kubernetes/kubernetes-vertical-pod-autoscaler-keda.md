# 🚀 Stop Guessing Kubernetes Limits: The Complete Guide to 2D Autoscaling
### *(Horizontal Scaling with KEDA + Vertical Sizing with VPA)*



Are your Kubernetes pods randomly getting `OOMKilled`? Or worse—are you vastly over-provisioning resource requests, leaving nodes half-empty and wasting thousands of dollars on idle cloud compute?

If you rely solely on manual guessing and standard Horizontal Pod Autoscalers (HPA), you are stuck in a cycle of reactive scaling and resource waste.

The solution is **Two-Dimensional Autoscaling**: pairing the **Vertical Pod Autoscaler (VPA)** for pod sizing with **KEDA (Kubernetes Event-driven Autoscaling)** for event-driven replica scaling.

Here is the architectural blueprint and an end-to-end production runbook to implement 2D autoscaling safely.

---

## 🧠 Part 1: Why Traditional Autoscaling Falls Short

### ❌ The Flaw of Manual Static Sizing
By default, engineers guess CPU and Memory requests and limits:
* **Guess too low:** The pod is heavily throttled by the Linux CFS scheduler or terminated by the kernel with `OOMKilled` (Exit Code 137).
* **Guess too high:** Cloud compute sits idle 90% of the day, inflating your cloud invoice.
* **Standard HPA reacts too late:** HPA reacts to internal CPU/Memory usage. By the time CPU saturates, requests are already queuing, latency has spiked, and user experience is degraded.

---

### 📈 Enter VPA: Solving the *"How Big?"* Problem
The **Vertical Pod Autoscaler (VPA)** continuously observes actual runtime consumption and right-sizes your container requests and limits automatically:
* **Eliminates OOMKills:** If a service experiences steady-state memory growth, VPA adjusts memory requests upward.
* **Cost Right-Sizing:** If you requested 4GiB of RAM but the container only ever uses 500MiB, VPA lowers the requests, freeing up schedulable node capacity.

---

### ⚡ Enter KEDA: Solving the *"How Many & When?"* Problem
**KEDA** transforms your cluster from reactive internal metric scaling to **proactive event-driven scaling** (e.g., RabbitMQ queues, Kafka consumer lag, AWS SQS, or Prometheus queries):
* **Proactive Horizon:** KEDA adds worker pods the instant messages arrive in a queue—*before* CPU saturation occurs.
* **Scale-to-Zero:** Native Kubernetes HPA cannot scale below 1 replica. KEDA enables scaling down to **0 replicas** during idle periods, dropping compute costs for that workload to zero.

---

## 🤝 The Magic Synergy: 2D Autoscaling

Combining both tools provides complete operational coverage:
* **KEDA** manages the **Horizontal Axis** (*"There are 50,000 tasks in the queue; scale to 15 pods!"*).
* **VPA** manages the **Vertical Axis** (*"These batch tasks require 400MiB RAM instead of 200MiB; adjust container sizes!"*).

> ⚠️ **THE GOLDEN RULE OF 2D AUTOSCALING:**  
> **Never allow KEDA and VPA to scale on the same metric.**  
> KEDA must control replica counts based strictly on **external events** (queue depth, lag), while VPA controls container sizing based on **CPU/Memory history**. If both controllers react to CPU, they will enter a race condition and destabilize your cluster.

---

## 🛠️ Part 2: The Production Runbook

---

### Phase 1: Prerequisites
* A running Kubernetes cluster (EKS, GKE, AKS, or local Minikube/kind).
* `kubectl` and `helm` installed.

---

### Phase 2: Install Foundation Components

#### 1. Deploy Metrics Server (Required by VPA)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### 2. Install the Vertical Pod Autoscaler (VPA)
```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
cd ../..
```

#### 3. Install KEDA via Helm
```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

---

### Phase 3: Deploy the Target Workload

Deploy a worker service that processes batch tasks.

Create `app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-app
  namespace: default
spec:
  replicas: 1  # KEDA manages this field dynamically
  selector:
    matchLabels:
      app: worker-app
  template:
    metadata:
      labels:
        app: worker-app
    spec:
      containers:
      - name: worker
        image: ubuntu:22.04
        command: ["/bin/sh", "-c", "while true; do sleep 5; done"]
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
```

Apply:
```bash
kubectl apply -f app.yaml
```

---

### Phase 4: Configure KEDA (Horizontal Scaling)

We configure a `ScaledObject` using a cron trigger to simulate periodic batch spikes. In production, swap `type: cron` for `rabbitmq`, `aws-sqs-queue`, or `kafka`.

Create `keda-scaler.yaml`:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-app-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: worker-app
  minReplicaCount: 2     # Maintain at least 2 for High Availability
  maxReplicaCount: 10
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          # Prevents KEDA from scaling down while VPA is evicting pods for resize
          stabilizationWindowSeconds: 60
  triggers:
  - type: cron
    metadata:
      timezone: UTC
      start: 0 * * * *       # Top of every hour
      end: 15 * * * *        # Run for 15 minutes
      desiredReplicas: "5"
```

Apply:
```bash
kubectl apply -f keda-scaler.yaml
```

---

### Phase 5: Configure VPA (Vertical Sizing)

#### VPA Update Modes:
| Mode          | Action on Recommendations                                | Live Pod Evictions? | Production Recommendation                          |
| :------------ | :------------------------------------------------------- | :------------------ | :------------------------------------------------- |
| **`Off`**     | Computes recommendations only; no changes applied        | ❌ None              | **Start here** for 1–2 weeks to establish baseline |
| **`Initial`** | Applies sizing only when new pods are scheduled          | ❌ None              | Safe mode; applies on rollouts without disruption  |
| **`Auto`**    | Dynamically evicts and restarts live pods with new sizes | ✅ Yes               | Production-ready **only** when protected by a PDB  |

#### Create `vpa-config.yaml`:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: worker-app-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: worker-app
  updatePolicy:
    updateMode: "Off"    # Start with 'Off'. Promote to 'Initial', then 'Auto' after review.
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 50m
          memory: 50Mi
        maxAllowed:
          cpu: 1000m
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits
```

Apply:
```bash
kubectl apply -f vpa-config.yaml
```

---

### Phase 6: Protect Availability with a PodDisruptionBudget (PDB)

> 🚨 **CRITICAL PRODUCTION GUARDRAIL:**  
> When VPA is promoted to `updateMode: Auto`, it resizes containers by **evicting running pods** so the mutating webhook can inject updated requests on recreation. Without a `PodDisruptionBudget`, VPA could evict all replicas simultaneously, triggering a complete outage.

Create `pdb.yaml`:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: worker-app-pdb
  namespace: default
spec:
  minAvailable: 1        # Guarantees at least 1 pod serves traffic during VPA resizes
  selector:
    matchLabels:
      app: worker-app
```

Apply:
```bash
kubectl apply -f pdb.yaml
```

---

### Phase 7: Validation & Observation

#### 1. Monitor KEDA Horizontal Scaling:
```bash
kubectl get pods -l app=worker-app -w
```
*Observe KEDA dynamically scaling between `2` and `5` replicas based on the event schedule.*

#### 2. Inspect VPA Sizing Recommendations:
```bash
kubectl describe vpa worker-app-vpa
```

Look at the `Status.Recommendation` block:
```text
  Recommendation:
    Container Recommendations:
      Container Name:  worker
      Lower Bound:
        Cpu:     50m
        Memory:  50Mi
      Target:
        Cpu:     150m
        Memory:  128Mi
      Upper Bound:
        Cpu:     800m
        Memory:  400Mi
```

Once recommendations match expected traffic profiles, promote `vpa-config.yaml` to `updateMode: "Auto"`.

---

## 🔧 Troubleshooting & Architectural Traps

* **The Scale-to-Zero vs. PDB Deadlock:** If using KEDA's `minReplicaCount: 0` alongside a PDB configured with `minAvailable: 1`, the PDB will block VPA from evicting the final pod when scaling down. For scale-to-zero workloads, remove the PDB or set `minAvailable: 0`. For high-availability workloads, keep `minReplicaCount: 2` in KEDA and `minAvailable: 1` in the PDB.
* **OOMKill Restart Loops:** Ensure `maxAllowed` in your VPA manifest accommodates application startup bursts (e.g., JVM heap allocation). If `maxAllowed.memory` is lower than startup memory, the container will enter a continuous `CrashLoopBackOff`.
* **VPA Requires Observation Time:** VPA needs at least 10–30 minutes of representative cluster traffic before its mathematical recommendations stabilize. Do not promote to `Auto` immediately after deploying.

---

## 🧹 Cleanup

```bash
kubectl delete -f vpa-config.yaml
kubectl delete -f keda-scaler.yaml
kubectl delete -f pdb.yaml
kubectl delete -f app.yaml
```

