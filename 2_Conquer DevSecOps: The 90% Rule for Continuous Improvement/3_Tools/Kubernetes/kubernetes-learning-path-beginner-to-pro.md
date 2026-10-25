# ☸️ Kubernetes Learning Path: Beginner to Pro

Welcome to the complete Kubernetes curriculum. This roadmap covers everything from low-level Linux/Networking primitives up to production platform engineering.

> 💡 **Tip:** Treat each level as a discrete milestone. Master the core primitives before moving on to advanced operational tooling.

---

## Level 0 — Foundations: Linux & Networking

Before touching a cluster, you must understand the operating system primitives and network models that Kubernetes automates.

### 🐧 Linux Foundations
* Process isolation: Linux Namespaces (`pid`, `net`, `mnt`, `ipc`, `uts`, `user`)
* Resource constraints: Control Groups (`cgroups v1 / v2`)
* System utilities: `systemd`, `journalctl`, and storage mounts

### 🌐 The Kubernetes Networking Model
Kubernetes enforces 4 fundamental networking rules:
1. Every Pod gets its own unique IP address (containers within the same Pod share this IP).
2. All Pods can communicate with all other Pods across nodes without NAT.
3. All Nodes can communicate with all Pods without NAT.
4. The IP a Pod sees as its own is the same IP every other Pod and Node sees.

* **Core Networking Primitives:** IPs, Subnets, CIDR blocks, DNS, and `iptables`/`IPVS`.
* **Container Network Interface (CNI):** Overlay (Flannel, Calico VXLAN) vs. Flat/Routed (AWS VPC CNI, Calico BGP).
* **Service Abstraction:** ClusterIP (internal VIP), NodePort, and LoadBalancer.
* **Cluster DNS:** How CoreDNS resolves `<service>.<namespace>.svc.cluster.local`.
* **Traffic Control:** Ingress Controllers, Gateway API, and Network Policies.
* **Data Plane Acceleration:** How `kube-proxy` programs `iptables`/`IPVS`, and how eBPF (Cilium) replaces it.

---

## Level 1 — Primitives & Core Architecture
**Goal:** Understand how Kubernetes works under the hood and interact with basic declarative objects.

* **Architecture Deep Dives:**
  * [Understand the Architecture: Control Plane, Worker Nodes & Internals](kubernetes-architecture.md)
  * [Complex Systems Are an Illusion: Breaking Down the Components](kubernetes-complex-system-illusion.md)
  * Abstraction Layers: How Pods wrap Containers, and Deployments wrap ReplicaSets
  * Writing Clean YAML: Structuring declarative manifests
1. [Understand what a Pod is and deploy one](kubernetes-standalone-pod.md) (Pod → Container)

2. [Understand what a Deployment is and create one](kubernetes-deployment-lifecycle.md) (Deployment → ReplicaSet → Pod → Container)

3. [Master imperative vs declarative management](kubernetes-imperative-vs-declarative-management.md)

4. [Understand Labels and Selectors](kubernetes-labels-selectors-deployments-pods-services.md) — how Deployments, Pods and Services connect

5. [Scale a Deployment up and down](kubernetes-deployment-scaling.md)

6. [Understand what a Service is and expose a Deployment](kubernetes-services-expose-deployment.md)

7. [Understand ClusterIP, NodePort and LoadBalancer](kubernetes-service-types-clusterip-nodeport-loadbalancer.md)

8. [Use kubectl port-forward for local testing and debugging](kubectl-port-forward-local-testing-debugging.md)

9. [Inspect and debug a running Pod — logs, exec, describe](kubernetes-pod-debugging-logs-exec-describe.md)

10. [Delete and recreate resources — observe self-healing](kubernetes-self-healing-delete-recreate-resources.md)

11. [Understand Kubernetes Namespaces and create your own](kubernetes-namespaces-create-manage.md)

---

## Level 2 — Workloads, Storage & Configuration
**Goal:** Deploy, configure, and manage real-world, multi-tier applications reliably.

* Decouple configuration from code using **ConfigMaps**
* Manage sensitive data with **Secrets** (base64 basics and access restrictions)
* Persist data with **PersistentVolumes (PV)**, **PersistentVolumeClaims (PVC)**, and **StorageClasses**
* Implement dynamic volume provisioning via the **Container Storage Interface (CSI)**
* Prevent noisy neighbors: configure resource `requests` and `limits` (CPU/Memory)
* Implement health checks: **Liveness**, **Readiness**, and **Startup** probes
* Multi-container patterns: using Sidecar and Adapter containers
* Deploy a full multi-tier app (Frontend + API Backend + Persistent Database)
* Run batch processing and scheduled tasks using **Jobs** and **CronJobs**
* Achieve zero-downtime rollouts: graceful shutdown, `SIGTERM`, and `preStop` lifecycle hooks

---

## Level 3 — Cluster Operations & Traffic Management
**Goal:** Manage traffic ingress, scale dynamically, and secure internal cluster communications.

* Package and release applications using **Helm** (templates, values, and releases)
* Implement Layer 7 HTTP/HTTPS routing using an **Ingress Controller** (NGINX / Traefik)
* Automate TLS certificate issuance and renewal with **cert-manager** and Let's Encrypt
* Modernize traffic routing using the **Gateway API**
* Scale workloads automatically based on CPU/RAM with the **Horizontal Pod Autoscaler (HPA)**
* Event-driven scaling with **KEDA** (Kafka queues, Redis metrics, cron schedules)
* Secure the control plane using **Role-Based Access Control (RBAC)**: Roles, ClusterRoles, and ServiceAccounts
* Harden cluster isolation using **Network Policies** (default-deny, namespace/pod isolation)
* Run system-level daemons on every node using **DaemonSets**
* Deploy stateful workloads safely using **StatefulSets** (ordered startups and stable network IDs)
* Advanced placement: **Taints & Tolerations**, **Node Selectors**, and **Node Affinity/Anti-Affinity**

---

## Level 4 — Production Readiness, Security & Observability
**Goal:** Harden, observe, and secure a Kubernetes environment to enterprise production standards.

* Cluster metrics collection: Prometheus operator, node-exporter, and Grafana dashboards
* Centralized logging: Shipping and querying container logs with Loki or the EFK/Elastic stack
* Distributed tracing: Tracing microservice latency with OpenTelemetry and Jaeger
* GitOps delivery: Automating cluster sync with **ArgoCD** or **Flux**
* Enterprise secrets management: **External Secrets Operator (ESO)** with HashiCorp Vault or AWS Secrets Manager
* Supply-chain security: Automated image vulnerability scanning with **Trivy** in CI pipelines
* Cluster policy enforcement and governance using **Kyverno** or **OPA Gatekeeper**
* Pod hardening: Enforcing **Pod Security Admission (PSA)** (restricted profile, non-root, read-only root filesystems)
* Disaster recovery: Backing up and restoring etcd/cluster state with **Velero**
* Cloud spend visibility: Tracking compute and storage costs with **Kubecost** or **OpenCost**

---

## Level 5 — Platform Engineering & Extensibility
**Goal:** Build, automate, and operate an extensible, multi-tenant developer platform on top of Kubernetes.

* Build multi-tenant environments using dedicated tenant namespaces, **ResourceQuotas**, and **LimitRanges**
* Template and patch multi-environment configurations (dev/staging/prod) with **Kustomize**
* High-density database management on Kubernetes (e.g., CloudNativePG, Zalando Postgres Operator)
* Resilience testing: Implement Chaos Engineering (LitmusChaos or Chaos Mesh)
* Dynamic cluster scaling: Compare traditional **Cluster Autoscaler** with just-in-time provisioning via **Karpenter**
* High availability: Balance pods across availability zones using **Topology Spread Constraints**
* Zero-downtime cluster upgrades: Node draining, cordoning, and **PodDisruptionBudgets (PDB)**
* Kubernetes extension: Building Custom Resource Definitions (**CRDs**) and custom **Operators** (Kubebuilder / Operator SDK)
* Enterprise Service Mesh: Implement **Istio**, **Linkerd**, or **Cilium Service Mesh** for cluster-wide mTLS
* Internal Developer Platforms (IDP): Expose self-service cloud infrastructure using **Backstage** and **Crossplane**

---

## 📚 Deep Dives & Practical Notes

* [Kubernetes Architecture & Core Daemons](kubernetes-architecture.md)
* [Mental Models: Deconstructing Complex Systems](kubernetes-complex-system-illusion.md)
* **The "Running" Status Trap:** Why a `STATUS: Running` pod can still fail health probes and black-hole traffic
* **The Production Gap:** Key architectural differences between local dev clusters (Minikube/Kind) and multi-region production clusters
* **Hands-on Labs:** Interactive cluster debugging scenarios on [Killercoda](https://killercoda.com)