# 🛑 Stop Wasting Cloud Budget & Waking Up at 3 AM
### *(The Ultimate Kubernetes Horizontal Pod Autoscaler Guide & Runbook)*



If you are running Kubernetes in production without the **Horizontal Pod Autoscaler (HPA)**, you are either burning through your infrastructure budget by over-provisioning or risking application downtime during traffic surges.

HPA is the heartbeat of a dynamic, self-healing cluster. It continuously observes compute metrics and automatically scales the number of Pod replicas up or down to match real-time demand.

Here is the architectural case for HPA, followed by a complete end-to-end runbook from metrics installation to load testing.

---

## 🧠 Part 1: Why Modern Infrastructure Needs HPA

### 1. Dynamic Traffic Management (Surviving Spikes) 🚦
* **Without HPA:** Statically provisioned pods get overwhelmed during marketing campaigns or viral events. CPU and memory saturate, latency spikes, and applications drop requests.
* **With HPA:** The autoscaler senses the elevated metric and automatically provisions replica pods to distribute the incoming request load.

### 2. Cost Efficiency (No More Paying for Idle Headroom) 💰
* **Without HPA:** To survive traffic spikes safely, teams "over-provision." If peak hours require 10 pods, they run 10 pods 24/7. Overnight, 8 pods sit idle, burning cloud budget.
* **With HPA:** Set `minReplicas: 2` and `maxReplicas: 10`. HPA scales down during low-traffic periods, ensuring you only pay for compute when users are active.

> 💡 **Scale-to-Zero Note:** Standard HPA enforces a minimum floor of `minReplicas: 1`. If your workload requires true scale-to-zero (e.g., event-driven workers that should shut down when queues are empty), implement **[KEDA (Kubernetes Event-Driven Autoscaling)](https://keda.sh)**, which extends HPA with zero-replica scaling.

### 3. Zero-Touch Incident Prevention 😴
Instead of an on-call engineer waking up at 3:00 AM to manually run `kubectl scale`, HPA reacts within 30–90 seconds autonomously.

### 4. Node & Hardware Protection 🛡️
When memory saturates, the Linux kernel terminates containers with `OOMKilled` (Exit Code 137). When CPU saturates, processes are heavily throttled. HPA scales horizontally *before* existing pods breach their hard limits.

### 5. Synergy with the Cluster Autoscaler 🤝
HPA scales Pods. But what happens when the underlying worker nodes are full?
$$\text{Traffic Spikes} \longrightarrow \text{HPA creates Pods} \longrightarrow \text{Pods stuck in Pending} \longrightarrow \text{Cluster Autoscaler provisions Node VMs}$$

### 6. Custom & Business Metric Scaling 📈
HPA is not limited to CPU and memory. Using the Prometheus Adapter, it can scale on:
* RabbitMQ / SQS queue depth
* In-flight HTTP requests per second
* Active WebSocket connection counts

---

## 🔀 HPA vs. VPA: Scaling Out vs. Scaling Up

| Capability       | Horizontal Pod Autoscaler (HPA)              | Vertical Pod Autoscaler (VPA)                                    |
| :--------------- | :------------------------------------------- | :--------------------------------------------------------------- |
| **Action**       | Adds/removes **pod replicas** (Scale Out/In) | Adjusts container **CPU/Memory requests** (Scale Up/Down)        |
| **Best For**     | Stateless web apps, APIs, microservices      | Stateful apps, batch jobs, monolithic apps                       |
| **Downtime**     | Zero downtime                                | Typically requires **pod restarts** to apply new resource limits |
| **Metric Focus** | Reactive traffic and queue volume            | Historical consumption baselines                                 |

> ⚠️ **Golden Rule:** Do **not** run HPA and VPA simultaneously in auto-mode on the same metrics (CPU/Memory). They will enter a feedback loop and fight each other. Instead, use **VPA in Recommendation Mode (`updateMode: "Off"`)** to right-size baseline requests, and let **HPA manage dynamic replica counts**.

---

## 🛠️ Part 2: The Hands-On Runbook

---

### Step 1: Install the Kubernetes Metrics Server

HPA cannot compute utilization without the cluster Metrics Server API (`metrics.k8s.io`).

```bash
# 1. Deploy the official Metrics Server release
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 2. Wait 60 seconds, then verify cluster metrics
kubectl top nodes
kubectl top pods -A
```

#### ⚠️ Minikube TLS Gotcha:
If `kubectl top nodes` returns an error on local clusters due to self-signed kubelet certificates, patch the Metrics Server deployment:

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}
]'
```

---

### Step 2: Deploy the Target Workload

> 🚨 **MANDATORY RULE:** Your Deployment **MUST** declare `resources.requests.cpu`. Without requests, HPA cannot calculate percentage utilization and will fail with `<unknown>`.

Create `php-apache.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m     # Baseline: HPA calculates % utilization from this
          limits:
            cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```

Apply:

```bash
kubectl apply -f php-apache.yaml
```

---

### Step 3: Configure the HorizontalPodAutoscaler

We want to maintain **50% average CPU utilization** across all pods. If load increases, scale up to a maximum of 10 replicas; when load subsides, scale down to 1.

Create `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  # Anti-flapping stabilization window configuration
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min of low load before scaling down
      policies:
      - type: Percent
        value: 50                       # Remove at most 50% of pods at a time
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0    # React immediately to traffic spikes
      policies:
      - type: Percent
        value: 100                      # Double pod count per minute if needed
        periodSeconds: 60
```

Apply:

```bash
kubectl apply -f hpa.yaml
```

---

### Step 4: Verify HPA Status

```bash
kubectl get hpa
```

*Expected output after 1–2 minutes:*
```text
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          45s
```

*(If `TARGETS` shows `<unknown>/50%`, wait 60 seconds for metrics-server ingestion).*

---

### Step 5: Simulate Traffic Load

Spawn an ephemeral busybox pod to execute an infinite HTTP GET request loop:

```bash
kubectl run -i --tty load-generator \
  --rm \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

---

### Step 6: Watch HPA Auto-Scale in Real Time

In a second terminal window, monitor the HPA:

```bash
kubectl get hpa -w
```

Within 1–2 minutes, CPU usage exceeds 50%, and HPA triggers horizontal scaling:
```text
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS
php-apache   Deployment/php-apache   220%/50%   1         10        3
php-apache   Deployment/php-apache   160%/50%   1         10        6
php-apache   Deployment/php-apache   52%/50%    1         10        8
```

---

### Step 7: Observe Scale-Down & Cool-Down Windows

1. Terminate the load generator with `Ctrl + C`.
2. Monitor HPA. CPU target drops back to `0%/50%`.
3. Due to `stabilizationWindowSeconds: 300` (configured in Step 3), HPA waits **5 minutes** before scaling down. This prevents **flapping** (rapid pod creation/deletion cycles caused by transient traffic dips).

---

### Step 8: Custom Business Metrics (Prometheus Example)

To scale on custom business metrics (e.g., RabbitMQ queue backlog via the Prometheus Adapter):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker-deployment
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: rabbitmq_queue_messages_ready
        selector:
          matchLabels:
            queue: "video-processing"
      target:
        type: AverageValue
        averageValue: "30"  # Add a worker pod for every 30 queued messages
```

---

## 🔧 Operational Troubleshooting Matrix

| Problem                                    | Root Cause                                      | Solution                                                                |
| :----------------------------------------- | :---------------------------------------------- | :---------------------------------------------------------------------- |
| **`TARGETS: <unknown>/50%`**               | Missing `resources.requests.cpu` in Deployment. | Add `requests.cpu` to the container spec.                               |
| **`metrics-server` communication errors**  | Kubelet self-signed TLS cert rejected.          | Patch deployment with `--kubelet-insecure-tls` flag.                    |
| **Pods stuck in `Pending` after scale-up** | Cluster nodes are at 100% capacity.             | Configure **Cluster Autoscaler** or increase cloud node group maximums. |
| **Pod Flapping (rapid restarts/churn)**    | Missing scale-down stabilization window.        | Add `behavior.scaleDown.stabilizationWindowSeconds: 300` to HPA spec.   |

---

## 🧹 Cleanup

```bash
kubectl delete -f hpa.yaml
kubectl delete -f php-apache.yaml
```

