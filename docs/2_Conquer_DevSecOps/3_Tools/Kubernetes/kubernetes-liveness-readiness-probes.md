# 🚀 Stop Flying Blind in Kubernetes: The Ultimate Guide to Liveness, Readiness, & Startup Probes



If you are running applications in Kubernetes without correctly configured health probes, you are navigating a minefield blindfolded.

Here is a fundamental truth that catches many engineers off guard: **By default, Kubernetes is completely blind to your application's internal state.**

Out of the box, Kubernetes only checks if your container's main process (`PID 1`) is running. But in production, applications fail without crashing the process:
* What happens when your web server deadlocks on thread pool exhaustion?
* What happens when a database connection drops and your app returns `500 Internal Server Error` on every route?
* What happens when a JVM or Node.js application takes 60 seconds to warm up its local cache?

To Kubernetes, that container is still "Running." It will happily route live user traffic to your broken container.

To achieve self-healing and zero-downtime rollouts, you must give Kubernetes "eyes" using **Probes**.

---

## 🚦 The Three Probe Types

### 1. Readiness Probes *(Traffic Control)*
Just because a container process started does not mean it is ready to receive HTTP traffic. It might need to run database migrations, load configurations, or establish connection pools.

* **The Question:** *"Am I ready to receive live user requests?"*
* **The Action on Failure:** Kubernetes **does NOT restart the Pod**. Instead, it immediately strips the Pod's IP address from the matching Service Endpoints. Incoming traffic is diverted away to healthy replicas until this Pod passes again.
* **Without it:** During rolling updates, Kubernetes sends user traffic to newly started pods instantly, causing a spike of `502 Bad Gateway` errors while the app boots.

### 2. Liveness Probes *(Process Control)*
Applications suffer from "zombie" states—memory leaks, process deadlocks, or frozen event loops. The process is alive according to the OS, but permanently unresponsive.

* **The Question:** *"Am I healthy and operational, or am I stuck in an unrecoverable state?"*
* **The Action on Failure:** Kubernetes **terminates the container and restarts it** according to its `restartPolicy`.
* **Without it:** A deadlocked container will sit frozen indefinitely, requiring human intervention at 3:00 AM to manually delete the pod.

### 3. Startup Probes *(The Safety Net)*
Legacy or complex applications often have slow, variable startup sequences (e.g., 2–5 minutes).

* **The Question:** *"Has the application finished its initial boot sequence?"*
* **The Action on Failure:** Disables both Liveness and Readiness checks until the app has successfully completed its initial boot. This prevents premature Liveness failures from creating infinite restart loops.

---

## 🛠️ The 4 Probe Mechanisms

Kubernetes can execute health checks using four different protocols:

| Mechanism       | Description                                               | Success Criteria                            |
| :-------------- | :-------------------------------------------------------- | :------------------------------------------ |
| **`httpGet`**   | Performs an HTTP GET request to a specific path/port.     | HTTP status code between `200` and `399`.   |
| **`tcpSocket`** | Attempts to establish a TCP socket connection to a port.  | TCP connection handshake succeeds.          |
| **`exec`**      | Executes a shell command inside the container filesystem. | Process command exits with status code `0`. |
| **`grpc`**      | Invokes standard gRPC Health Checking Protocol v1.        | gRPC server returns `SERVING` status.       |

---

## ⚙️ Key Configuration Tuning Knobs

All three probe types share identical configuration parameters:

* **`initialDelaySeconds`:** Number of seconds to wait after container start before executing the first probe.
* **`periodSeconds`:** How often (in seconds) to execute the check.
* **`timeoutSeconds`:** Number of seconds before a non-responsive check is marked as a failure.
  > ⚠️ **Timeout Gotcha:** If a probe endpoint touches an external network resource (like a database check), never leave `timeoutSeconds` at the default of `1s`. Set it higher than your endpoint's p99 latency to prevent false-positive failures during high traffic load.
* **`failureThreshold`:** Number of consecutive failures required before taking action (pulling traffic or restarting).
* **`successThreshold`:** Consecutive passes required before marking an unhealthy probe as recovered. *(Must always be `1` for Liveness and Startup probes).*

---

## 📜 Complete Manifest Examples

### 1. Multi-Probe Exec Manifest

This manifest demonstrates all three probes working together with an `exec` check:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
  labels:
    app: probe-demo
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: my-app
    image: busybox:1.36
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; touch /tmp/ready; sleep 3600

    # 1. Startup Probe: Disables liveness/readiness until boot completes
    startupProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      failureThreshold: 30               # Allows up to 30 * 10s = 5 minutes to boot
      periodSeconds: 10

    # 2. Readiness Probe: Controls service endpoint inclusion
    readinessProbe:
      exec:
        command: ["cat", "/tmp/ready"]
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 2                # Must fail twice to pull from endpoints
      successThreshold: 2                # Must pass twice to resume traffic

    # 3. Liveness Probe: Triggers container restarts on deadlock
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3                # Must fail 3 times to restart container
```

> 💡 **Graceful Shutdown & Liveness:** When a Liveness probe fails, Kubernetes sends `SIGTERM` and waits `terminationGracePeriodSeconds` (default: 30s) before issuing `SIGKILL`. If your app needs time to drain connections, flush logs, or close transactions, ensure this value accommodates that process.

---

### 2. HTTP GET Probe (Standard Production Pattern)

The most common, lightweight, and language-agnostic production setup:

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: health-check
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /healthz      # Intentionally lightweight — just returns HTTP 200 OK
    port: 8080
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 3
```

---

### 3. Native gRPC Probe

For microservices communicating natively over gRPC:

```yaml
livenessProbe:
  grpc:
    port: 50051
    service: ""        # An empty string checks overall server health
  periodSeconds: 10
  failureThreshold: 3
```

---

## 💡 4 Golden Rules for Production Probes

1. **Never Point Liveness and Readiness to the Same Health Check Route:**  
   If your readiness probe checks downstream dependencies (like an external database), that's fine. But if your **Liveness** probe also checks the database, a momentary database network blip will cause Kubernetes to **restart every web container in your cluster simultaneously**, turning a minor degradation into a catastrophic total outage. Liveness should be cheap, internal, and local.
2. **Prevent Cascading Failures:**  
   If a readiness probe pulls pods when downstream systems slow down, the remaining pods absorb the entire traffic load, saturate, fail their own probes, and collapse the entire fleet. Readiness should reflect the container's ability to process requests, not external service health.
3. **Tune Failure Thresholds Appropriately:**  
   Network jitter is normal in cloud infrastructure. Setting `failureThreshold: 1` means a single dropped packet can restart a container. Always require at least 2–3 consecutive failures.
4. **Be Mindful of `exec` Probe Overhead:**  
   An `exec` probe forks a new process inside the container on every check. At aggressive intervals across hundreds of pods, this introduces measurable CPU and memory overhead. Prefer `httpGet` or `tcpSocket` whenever possible.

---

## 📚 Related Reading

* [Kubernetes Pod Lifecycles & Termination Grace Periods](#)
* [Debugging CrashLoopBackOff: The Ultimate Guide](#)
* [Do you really need a Liveness Probe in your project?](kubernetes-when-to-use-liveness-probes.md)

