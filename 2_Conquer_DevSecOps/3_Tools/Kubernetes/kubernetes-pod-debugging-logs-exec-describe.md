# 🚀 The "Holy Trinity" of Kubernetes Debugging: Describe, Logs, Exec
### *(And Why Your Manifest is the Final Boss)*



Let me paint a familiar picture: An alert goes off. A critical service in your Kubernetes cluster is down. You pull up your terminal, type `kubectl get pods`, and stare at the dreaded `CrashLoopBackOff` or `Pending` status.

Kubernetes is incredible for scaling and orchestrating applications, but its layers of abstraction can sometimes feel like a black box. When things go wrong, you can't just SSH into a server and check a local file.

To find the root cause, you need the **"Holy Trinity" of Kubernetes debugging**: `describe`, `logs`, and `exec`.

Missing even one leaves a massive blind spot in your troubleshooting workflow. Here is the ultimate playbook for using them—and how they tie back to your ultimate source of truth: **your Manifest**.

---

## 1️⃣ `kubectl describe`: The Orchestrator's Perspective

Before your application code even runs, Kubernetes has to pull the image, allocate resources, and schedule the Pod. If the Pod fails to start, this is where you look first.

* **What it answers:** *"What does Kubernetes think of my Pod?"*
* **When to use it:** When a pod is stuck in `Pending`, `ImagePullBackOff`, or is constantly being killed.

> 💡 **Pro-Tip:** Scroll to the very bottom of the output to the **`Events`** section. This is the goldmine. It will tell you if your cluster is out of memory/CPU, if you have a typo in your image tag, or if a Liveness/Readiness probe is failing.

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### ⚠️ The Namespace Trap
If `kubectl get pods` returns nothing, you may simply be looking in the wrong namespace. A huge percentage of *"my pod isn't there"* confusion starts and ends here:

```bash
kubectl get pods -n <namespace>
# Or check across the entire cluster:
kubectl get pods -A
```

### 🕒 Bonus: When the Pod is Already Deleted
If a pod terminated before you could run `describe`, grab the cluster-wide, time-sorted event stream:

```bash
kubectl get events --sort-by=.lastTimestamp -n <namespace>
```

---

## 2️⃣ `kubectl logs`: The Application's Perspective

Once Kubernetes successfully starts the container, your application code takes the wheel. If the infrastructure is healthy but the app is throwing 500 errors, you need to hear what the application is outputting.

* **What it answers:** *"What is my code complaining about?"*
* **When to use it:** When you suspect code-level exceptions, database timeouts, or runtime crashes.

```bash
kubectl logs <pod-name> -n <namespace>
```

### 💡 Pro-Tip: Fast CrashLooping Containers
If your pod is crashing repeatedly in seconds, the current container might be dead before you can read the buffer. Use `--previous` to read the log buffer from the instance that just crashed:

```bash
kubectl logs <pod-name> --previous -n <namespace>
```

### 📦 Multi-Container Pods (Sidecars / Proxies)
When running sidecars (Envoy, log shippers, vault agents, init containers), you must specify the container name:

```bash
kubectl logs <pod-name> -c <container-name> -n <namespace>
```

*(Run `kubectl describe pod <pod-name>` first if you need to check container names).*

---

## 3️⃣ `kubectl exec`: The Detective's Perspective

Sometimes Kubernetes says the Pod is healthy (`Running`), the logs look clean, but the app still fails. It's time to step inside the container itself. `exec` opens an interactive terminal inside the running container.

* **What it answers:** *"What does the internal runtime environment actually look like?"*
* **When to use it:** To test internal network connectivity, confirm volume mounts, or check live environment variables.

```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
```

### Multi-Container Pods:
```bash
kubectl exec -it <pod-name> -c <container-name> -n <namespace> -- /bin/sh
```

---

## 📜 Tying It All Together: The Manifest is the Real Fix

Debugging is only half the battle. Once `describe`, `logs`, and `exec` point you to the root cause—**what do you actually do with that information?**

> ⚠️ **You NEVER patch the live container directly.**

Anything you edit inside a container via `exec` disappears the moment the Pod restarts. Container updates are not fixes—they are temporary diagnostics.

Your **Manifest** (the declarative YAML defining your Deployment, ConfigMap, Secrets, etc.) is your source of truth. That is where the permanent fix lives:

* `describe` revealed insufficient memory? ➔ Update `resources.requests` / `limits` in the manifest.
* `logs` revealed an unhandled exception in code? ➔ Build/push a new image, update the `image:` tag in the manifest.
* `exec` revealed a missing environment variable? ➔ Add it to the `env:` block in the manifest.

### 🔍 Compare Desired vs. Actual State
To inspect the live resolved state of your pod (including cluster defaults populated by Kubernetes):

```bash
kubectl get pod <pod-name> -o yaml
```

Apply your permanent fix:

```bash
kubectl apply -f manifest.yaml
```

Kubernetes reads the new desired state and rolls out the fix cleanly via your deployment controller.

---

## 📌 Debugging Decision Matrix

| Symptom                                              | Primary Tool       | Command / Action                                |
| :--------------------------------------------------- | :----------------- | :---------------------------------------------- |
| **Pod won't start (`Pending` / `ImagePullBackOff`)** | `describe`         | `kubectl describe pod <name>` (Check `Events:`) |
| **Pod missing entirely**                             | Namespace Check    | `kubectl get pods -A`                           |
| **Pod was evicted/deleted before inspection**        | Event Stream       | `kubectl get events --sort-by=.lastTimestamp`   |
| **Pod restarts repeatedly (`CrashLoopBackOff`)**     | `logs --previous`  | `kubectl logs <name> --previous`                |
| **Pod running, but app returning errors**            | `logs`             | `kubectl logs <name> -f`                        |
| **Multiple containers in one pod**                   | Flag `-c`          | `kubectl logs <name> -c <container-name>`       |
| **Network/ConfigMap sanity check**                   | `exec`             | `kubectl exec -it <name> -- /bin/sh`            |
| **Permanent Remediation**                            | **Git / Manifest** | Update YAML manifest ➔ `kubectl apply -f`       |



