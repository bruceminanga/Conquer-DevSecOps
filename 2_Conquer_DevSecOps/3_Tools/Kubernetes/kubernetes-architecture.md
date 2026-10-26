# Kubernetes Architecture: Control Plane, Worker Nodes & Core Internals

> *"Complex systems are an illusion. They’re just simple components working together."*  
> — [Complex Systems Breakdown](kubernetes-complex-system-illusion.md)

For a long time, many of us have treated Kubernetes like a magic black box: we type `kubectl apply`, cross our fingers, and assume the cluster will figure it out. If you are strictly writing application code, that might be fine.

But if you are a DevOps Engineer, SRE, or Platform Engineer, treating Kubernetes like magic is dangerous. When things break, error messages rarely tell you how to fix them. You have to deduce the problem from the architecture—the same way a mechanic diagnoses an engine by understanding what each part does.

I learned this the hard way. Early on, I spent nearly two hours debugging a pod stuck in `Pending`. I restarted it repeatedly, checked the image, and checked the manifest. Nothing. The actual problem? The **Scheduler** couldn't find a node with enough memory to place it. Once I understood what the Scheduler actually does, that class of problem became a 2-minute diagnosis.

---

## The Core Concept: The Reconciliation Loop

A Kubernetes cluster is simply a collection of nodes (machines) divided into two roles:
* **Control Plane node(s):** The brain/management layer. Makes global decisions (like scheduling) and responds to cluster events.
* **Worker node(s):** The muscle. The compute instances where your containerized workloads (Pods) actually run.

Under the hood, **Kubernetes is a reconciliation engine**. It continuously compares two states:
1. **Desired State:** What you declared in your YAML manifests.
2. **Actual State:** What is currently running across the hardware.

Every component exists to close the gap between these two states. When your cluster misbehaves, your job as the mechanic is to ask: **Why is it failing to reconcile?**

---

## 🏛️ Control Plane Components

### 🚪 1. The API Server (`kube-apiserver`)
* **What it is:** The front door and the traffic cop.
* **Why the mechanic needs to know it:** It is the **only** component that communicates directly with the cluster database (`etcd`). Every `kubectl` command, internal service call, and controller check-in goes through here. It is horizontally scalable. If cluster commands time out, your network might not be down—your API Server might be buckling under request volume. It also acts as the authentication and authorization gatekeeper; if CI/CD pipelines suddenly lose access, investigate here first.

### 💾 2. The Database (`etcd`)
* **What it is:** The absolute source of truth—a consistent, highly available distributed key-value store.
* **Why the mechanic needs to know it:** `etcd` stores the desired state of your cluster (deployments, secrets, configs) and the actual state reported back by components. If your cluster crashes completely, you don't need to back up individual controllers—restoring a valid `etcd` snapshot restores the entire cluster state. Whoever controls `etcd` controls the cluster, making strict access control and encrypted backups mandatory.

### ⚙️ 3. The Controller Manager (`kube-controller-manager`)
* **What it is:** The reconciliation engine in action—the heartbeat of the cluster.
* **Why the mechanic needs to know it:** This single binary runs multiple controller processes in continuous loops. Each controller handles one specific concern: the **Node Controller** responds when a node goes offline; the **ReplicaSet Controller** ensures pod counts match your replica target. When actual state drifts from desired state, the relevant controller takes corrective action.

### ☁️ 4. The Cloud Controller Manager (`cloud-controller-manager`)
* **What it is:** The cloud provider liaison.
* **Why the mechanic needs to know it:** This isolates cloud-specific logic (AWS, GCP, Azure) from core Kubernetes code. On bare metal or local setups (Minikube/Kind), this component doesn't run. In cloud environments, it manages provider dependencies: checking if a cloud VM was deleted, setting up VPC subnet routing, or spinning up managed Load Balancers when a `LoadBalancer` Service is created.

### 🤝 5. The Scheduler (`kube-scheduler`)
* **What it is:** The matchmaker.
* **Why the mechanic needs to know it:** The Scheduler **never runs pods**; it only decides *where* they should run. It watches for newly created Pods with no assigned node, filters out incompatible nodes, and scores the remaining ones based on resource requests, taints/tolerations, affinity rules, and topology spreads. If a pod is stuck in `Pending`, restarting it won't help. The Scheduler is telling you no node satisfies its placement requirements.

---

## 🏗️ Worker Node Components

### 👨‍✈️ 1. The Kubelet
* **What it is:** The captain of the worker node.
* **Why the mechanic needs to know it:** The Kubelet is the node-level agent that takes `PodSpecs` from the API Server and ensures the requested containers are running and healthy. If a pod is scheduled to a node but stuck in `ContainerCreating`, the Kubelet on that specific node is where you inspect logs (`journalctl -u kubelet`).  
* **TL;DR:** Communicates with the control plane, triggers the runtime, and reports node health.

### 🔀 2. The Network Proxy (`kube-proxy`)
* **What it is:** The local switchboard operator.
* **Why the mechanic needs to know it:** `kube-proxy` runs on every node to maintain network rules (using `iptables` or `IPVS`) allowing communication to Pods across the cluster or through Services. If pods are running fine locally but cannot talk across nodes or resolve Service endpoints, proxy routing rules are likely broken. *(Note: Advanced eBPF-based CNI plugins like Cilium can bypass or replace `kube-proxy` entirely).*  
* **TL;DR:** Programs local OS networking rules so traffic correctly reaches target pods.

### 📦 3. The Container Runtime
* **What it is:** The engine room executing the workload.
* **Why the mechanic needs to know it:** Kubernetes does not execute containers directly; it delegates execution via the Container Runtime Interface (CRI) to engines like `containerd` or `CRI-O`. If the Kubelet is the captain giving orders, the runtime is the mechanic pulling images, mounting storage, and running the processes. If the runtime hangs or crashes, the node stops responding to container lifecycle commands.  
* **TL;DR:** The low-level software engine that pulls images and manages actual container processes.

---

## 🧩 Essential Addons

While the core components run the cluster, cluster addons (deployed as standard Deployments or DaemonSets in `kube-system`) provide critical operational functionality:

* **Cluster DNS (CoreDNS):** Resolves internal Kubernetes service names (e.g., `db.default.svc.cluster.local`). If apps can connect via raw IP but fail via service name, CoreDNS is the primary culprit.
* **Container Network Interface (CNI):** Software plugins (Calico, Flannel, Cilium) that assign IP addresses to Pods and handle cross-node packet routing.
* **Metrics Server:** Collects lightweight resource metrics (CPU/RAM) from Kubelets to power the Horizontal Pod Autoscaler (HPA) and `kubectl top`.
* **Centralized Logging & Ingress Controllers:** Ingress engines (like NGINX or Envoy) to route external HTTP traffic, and logging agents (like Fluentbit or Promtail) to ship node logs.

---

## Summary: Become the Mechanic

Understanding Kubernetes architecture transitions you from a **Kubernetes User** to a **Kubernetes Administrator**. 

* The **driver** turns the key and hopes the car starts.
* The **mechanic** pops the hood, understands every part of the drivetrain, and diagnoses issues step-by-step.

Stop treating error messages like mysterious puzzles. Read them as clues, trace them back to the component responsible, and let the reconciliation loop work for you. 🛠️

---

## 📚 Official Docs & Deep Dives

* [Kubernetes Architectural Concepts](https://kubernetes.io/docs/concepts/architecture/) — Official overview of nodes and control plane communication.
* [Kubernetes Components Reference](https://kubernetes.io/docs/concepts/overview/components/) — Concise reference sheet for every core daemon.
* [etcd Documentation](https://etcd.io/docs/) — Operational guidance, snapshots, and consensus mechanics.
* [kube-scheduler Internals](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/) — How node filtering, scoring, and placement work.
* [Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/) — How the Kubelet interacts with container runtimes.