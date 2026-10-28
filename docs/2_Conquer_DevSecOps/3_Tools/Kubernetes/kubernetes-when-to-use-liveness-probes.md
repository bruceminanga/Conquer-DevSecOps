# ⚠️ Do You Really Need a Liveness Probe in Your Project?



Here is an uncomfortable truth about Kubernetes: **if you don't know exactly why you need a liveness probe, you probably shouldn't have one.**

The *"liveness probes are dangerous"* argument isn't fringe speculation—it has been heavily documented for years, notably in Henning Jacobs' widely cited 2019 Zalando post, which ultimately led to an open issue on the Kubernetes documentation repository requesting explicit warnings against cascading probe failures.

Yet if you inspect almost any Helm chart or Deployment YAML across the industry, what do you see?

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 8080
readinessProbe:
  httpGet:
    path: /
    port: 8080
```

Copy-pasted. Hitting the exact same endpoint.

Here is why this pattern triggers 3:00 AM outages—and why you should rethink your probes.

---

## 🛑 Myth: *"Without a liveness probe, Kubernetes won't restart my crashed app."*

**False.** 

If your application process crashes, exits with an unhandled exception (non-zero exit code), or gets killed by Linux via `OOMKilled` (Exit Code 137), Kubernetes detects it immediately because `PID 1` inside the container died. 

Kubernetes automatically restarts the container according to your `restartPolicy: Always` (the default for Deployments)—**no health probe required**.

> **You do not need a liveness probe for crash recovery.**

---

## 🧟 The Trap: The Cascading Death Spiral

To understand why liveness probes backfire, compare what they actually ask:

* **Readiness probe asks:** *"Can you serve incoming user traffic right now?"*  
  ➔ *Action on failure:* Kubernetes removes the pod from the Service load balancer until it recovers.
* **Liveness probe asks:** *"Is this process deadlocked beyond all hope?"*  
  ➔ *Action on failure:* Kubernetes **kills the container immediately and restarts it**.

### The Failure Sequence Under Load:

1. High traffic surges into your application, or a downstream dependency (database/cache) slows down.
2. Application response times spike beyond your configured 2-second probe timeout.
3. **The Readiness probe fails:** *Good!* Traffic is paused, giving the container breathing room to clear its internal connection queues.
4. **The Liveness probe also fails:** Because the liveness probe points to the same route (or hits a route coupled to that same slow database), it times out too.
5. **Kubernetes kills the container.**
6. The remaining surviving pods absorb 100% of the traffic, saturate, time out their own liveness probes, and get killed in sequence.

Within minutes, your cluster is locked in an infinite restart loop—**a liveness probe attempted to "fix" high network load by rebooting healthy containers.**

> 💡 **The Root Cause:** The bug isn't just using the exact same path. **It's a shared dependency.** Two different paths (`/healthz` and `/ready`) that both touch a database will trigger the exact same death spiral.

---

## 📜 The 3 Golden Rules for Health Probes

### 1️⃣ Readiness Probe? Almost Always YES.
Users should never hit cold, uninitialized, or saturated containers.  
* *Rule:* This is the **only** probe permitted to verify external/downstream dependencies (databases, Redis, remote APIs).

### 2️⃣ Liveness Probe? Almost Always NO.
Particularly for frontends, static websites, worker queues, and stateless microservices.  
* *Rule:* Only add a liveness probe if your application runtime can enter an unrecoverable **deadlock/zombie state** where the process remains alive (`PID 1` running) but stops serving permanently.  
* *If you must use one:* Keep it 100% dependency-free (strictly checking internal process responsiveness, like an in-memory boolean flag) and tune it loose (high `failureThreshold` and longer `periodSeconds` so transient spikes don't trigger reboots).

### 3️⃣ Slow Startup? Use Startup Probes, Not Liveness Hacks.
Never inflate `initialDelaySeconds: 180` on a liveness probe to accommodate a slow 3-minute Java boot sequence. If the app deadlocks later, you add a 3-minute blind spot to your detection.  
* *Rule:* Use a dedicated **`startupProbe`** instead. It temporarily suspends liveness checks until initial boot is complete.

---

## 🎯 The Litmus Test

Before adding your next liveness probe, ask yourself one question:

> *"Will killing and restarting this process actually resolve the underlying problem—or will it turn a transient blip into a cluster-wide outage?"*

