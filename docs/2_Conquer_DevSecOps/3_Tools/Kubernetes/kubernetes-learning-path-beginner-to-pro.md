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

<!-- -->

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

1. [Deploy a real application on Kubernetes](kubernetes-deploy-real-application.md)

2. [Use ConfigMaps to manage application configuration](kubernetes-configmaps-application-configuration.md)

3. [Use Secrets to manage sensitive data](kubernetes-secrets-sensitive-data.md)

4. [Attach persistent storage with PersistentVolumes and PersistentVolumeClaims](kubernetes-persistent-volumes-persistent-volume-claims.md)

5. [Understand StorageClasses and the Container Storage Interface for dynamic provisioning](kubernetes-storageclasses-csi-dynamic-provisioning.md)

6. [Set resource requests and limits](kubernetes-resource-requests-limits.md)

7. [Configure liveness and readiness probes](kubernetes-liveness-readiness-probes.md)

8. [Configure Startup probes for slow-starting applications](kubernetes-startup-probes.md)

9. [Use a multi-container Pod — sidecar pattern](kubernetes-multi-container-pod-sidecar-pattern.md)

10. [Deploy a multi-tier app — frontend + backend + database](kubernetes-multi-tier-app-frontend-backend-database.md)

11. [Run batch workloads using Jobs and CronJobs](kubernetes-jobs-cronjobs.md)

12. [Understand graceful shutdown — SIGTERM and preStop hooks for zero-downtime deployments](kubernetes-graceful-shutdown-sigterm-prestop.md)

---

## Level 3 — Cluster Operations & Traffic Management
**Goal:** Manage traffic ingress, scale dynamically, and secure internal cluster communications.

1. [Install and use Helm to deploy applications](kubernetes-install-use-helm.md)

2. [Write your own Helm chart](kubernetes-write-helm-chart.md)

3. [Set up Ingress and route traffic to multiple services](kubernetes-ingress-multiple-services.md)

4. [Set up Cert-Manager for automatic TLS certificates via Let's Encrypt](kubernetes-cert-manager-lets-encrypt-tls.md) — for production, not locally

5. [Learn the Gateway API — the modern successor to Ingress](kubernetes-gateway-api.md)

6. [Configure Horizontal Pod Autoscaler](kubernetes-horizontal-pod-autoscaler.md)

7. [Explore Vertical Pod Autoscaler and KEDA for event-driven autoscaling](kubernetes-vertical-pod-autoscaler-keda.md)

8. [Understand and apply RBAC — Roles, RoleBindings, ClusterRoles, ServiceAccounts](kubernetes-rbac-roles-bindings-serviceaccounts.md)

9. [Apply Network Policies to restrict pod-to-pod traffic](kubernetes-network-policies.md)

10. [Understand and use Init Containers](kubernetes-init-containers.md)

11. [Understand DaemonSets and StatefulSets](kubernetes-daemonsets-statefulsets.md)

12. [Master advanced scheduling — Taints & Tolerations, Node Selectors, Node Affinity/Anti-affinity](kubernetes-advanced-scheduling-taints-tolerations-affinity.md)

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