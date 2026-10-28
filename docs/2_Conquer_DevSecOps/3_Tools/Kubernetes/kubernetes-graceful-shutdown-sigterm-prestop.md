# 🛑 Stop Dropping Requests During Kubernetes Deployments
### *(The Magic of preStop Hooks, SIGTERM, & Graceful Shutdowns)*



You've set up Kubernetes. You've configured Rolling Updates. You hit deploy, feeling confident your users won't notice a thing.

Then the alerts fire:
* Users report sudden spikes of `HTTP 502 Bad Gateway` and `504 Gateway Timeout`.
* In-flight database transactions are terminated mid-query.
* Checkout requests are severed abruptly.

**Doesn't Kubernetes guarantee zero-downtime deployments out of the box?**

No, it doesn't. Rolling updates simply orchestrate spinning up new pods and killing old ones. If you haven't explicitly configured your application and cluster for **Graceful Shutdown**, you are pulling the power cord on live user traffic.

Here is the complete engineering breakdown of why this happens and how to achieve true zero-downtime rollouts using `preStop` hooks and `SIGTERM` signal propagation.

---

## 1️⃣ The Root Cause: The Asynchronous Termination Race Condition

When Kubernetes decides to terminate a Pod during a deployment rollout, two things occur simultaneously:

```text
               Kubernetes Initiates Termination
                              │
            ┌─────────────────┴─────────────────┐
            ▼                                   ▼
    [API Controller]                    [Kubelet on Node]
Removes Pod from Endpoints              Sends SIGTERM to Pod
            │                                   │
            ▼                                   ▼
Asynchronous network propagation        Container terminates immediately!
(kube-proxy, Ingress, iptables)                 │
Takes ~3 to 10 seconds                          ▼
            │                            [ POD IS DEAD ]
            ▼                                   │
Traffic STILL arrives at dead Pod <─────────────┘
  ❌ HTTP 502 / 504 Errors!
```

Removing an IP from the cluster network is an **asynchronous process** that must propagate across the API server, CoreDNS, Ingress controllers, and every worker node's `kube-proxy` (updating `iptables`/`ipvs` rules).

If your application receives `SIGTERM` and shuts its web process down immediately, **it will continue receiving routed user requests for several seconds**. Those requests hit a closed socket and fail.

---

## 2️⃣ The Application-Level Fix: Handling SIGTERM

`SIGTERM` is the operating system's signal asking a process to wrap up execution and terminate cleanly.

By default, many runtimes ignore this signal or exit abruptly. Your application code must explicitly listen for `SIGTERM`:

1. Stop accepting new incoming requests.
2. Complete processing all in-flight requests.
3. Safely flush write buffers and close database connection pools.
4. Exit cleanly with status code `0`.

### ⚠️ Dockerfile Trap: PID 1 Signal Swallowing
Make sure you use the **exec form** in your Dockerfile, not the shell form:

* ✅ **Exec form:** `CMD ["node", "app.js"]` (Node runs as PID 1 and receives OS signals directly).
* ❌ **Shell form:** `CMD npm start` (The shell `/bin/sh` runs as PID 1, swallows `SIGTERM`, and never forwards it to your application).

> 💡 **PID 1 Signal Forwarding:** If you use entrypoint wrappers or process supervisors (e.g., `supervisord`), PID 1 must explicitly forward signals to child processes. The cleanest solution is using **[tini](https://github.com/krallin/tini)** as your entrypoint:
> ```dockerfile
> ENTRYPOINT ["/tini", "--"]
> CMD ["node", "server.js"]
> ```

---

## 3️⃣ The Infrastructure-Level Fix: The `preStop` Hook

Even with perfect `SIGTERM` handling in code, the network propagation race condition still exists. We need to tell the kubelet: *"Wait for the network to finish updating before signaling the application."*

We accomplish this using a **`preStop` lifecycle hook**:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 15"]
```

While the `preStop` hook sleeps for 15 seconds:
1. The Pod continues handling incoming traffic normally.
2. The network asynchronously removes the Pod IP from Ingress, Service, and `kube-proxy` routing tables.
3. Once the sleep finishes, no new traffic is routed to the Pod.
4. Kubernetes finally sends `SIGTERM`, and the container safely drains remaining requests.

---

## 4️⃣ The Grace Period Calculation

Kubernetes uses a deadline timer called `terminationGracePeriodSeconds` (default: 30s). If your pod has not exited before this timer expires, Kubernetes issues `SIGKILL` (immediate, non-negotiable termination).

$$\text{terminationGracePeriodSeconds} > \text{preStop sleep time} + \text{App drain time}$$

> ⚠️ **The Timer Starts Immediately:**  
> The `terminationGracePeriodSeconds` clock starts counting the moment termination is initiated—**it includes the time spent inside the `preStop` hook**.  
> If your `preStop` sleeps for 15s and your app requires up to 20s to drain requests, your grace period must be at least 35s. Setting it to `45s–60s` provides safe operational headroom.

---

## 5️⃣ The RollingUpdate Strategy Settings

To prevent temporary capacity deficits during deployments, configure your deployment rollout bounds:

* **`maxUnavailable: 0`:** Guarantees that no running pod is killed until a replacement pod is completely healthy and passing its readiness probe.
* **`maxSurge: 1`:** Allows Kubernetes to spin up one extra pod above your desired replica count during the rollout to absorb incoming traffic.

---

## 🛠️ The Production Manifest

Here is the complete, production-ready manifest implementing all zero-downtime mechanisms:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zero-downtime-app
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0   # Never drop capacity during a rollout
      maxSurge: 1         # Provision temporary extra capacity
  template:
    metadata:
      labels:
        app: zero-downtime-app
    spec:
      containers:
      - name: my-app
        image: my-app:v1.2.0
        ports:
        - containerPort: 8080

        # Readiness Probe: controls when pod is added/removed from Endpoints
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

        # preStop Hook: delays SIGTERM to allow network routing tables to drain
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 15"]

      # Budget headroom: preStop sleep (15s) + app drain (20s) + buffer (10s) = 45s
      terminationGracePeriodSeconds: 45
```

---

## 📋 Zero-Downtime Verification Checklist

- [ ] **Application Code:** Catches `SIGTERM`, rejects new connections, and drains in-flight requests.
- [ ] **Dockerfile:** Application runs as PID 1 (using exec form `CMD ["app"]` or `/tini`).
- [ ] **Readiness Probe:** Configured so Kubernetes gates traffic until the app is fully initialized.
- [ ] **`preStop` Lifecycle Hook:** Executes `sleep 15` to accommodate network endpoint propagation latency.
- [ ] **`terminationGracePeriodSeconds`:** Tuned higher than `preStop sleep` + worst-case in-flight request duration.
- [ ] **Rollout Strategy:** `maxUnavailable: 0` and `maxSurge: 1` applied in the deployment specification.

