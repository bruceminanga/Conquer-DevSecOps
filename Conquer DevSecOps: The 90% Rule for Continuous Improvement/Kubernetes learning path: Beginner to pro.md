# ☸️ Kubernetes Learning Path: Beginner to Pro

Welcome to the complete Kubernetes journey. The list below acts as your roadmap. Click on any topic to jump straight into the Runbook, featuring step-by-step commands and practical Minikube exercises.

---

# Level 0 

## Understanding linux

* So you've mastered Docker's Linux foundations (Namespaces and cgroups). You're ready for Kubernetes, right?

## Understanding Networking for Kubernetes

All these docker baseline networking knowledge transfers directly:

* IPs, Subnets, CIDR blocks, TCP/UDP, and DNS.
* Linux Network Namespaces, veth pairs, and routing tables.
* HTTP/HTTPS, Reverse Proxies, TLS, and iptables.

### The Big Shift: The Kubernetes Networking Model

You must understand Kubernetes’s 4 Core Networking Rules:

* Every Pod gets its own unique IP address (not individual containers).
* All Pods can communicate with all other Pods without NAT, across any node in the cluster.
* All Nodes can communicate with all Pods (and vice-versa) without NAT.
* The IP that a Pod sees as its own IP is the same IP that every other Pod sees it as.

### Additional Networking Concepts Introduced by Kubernetes

#### 1. Container Network Interface (CNI) Plugins

Kubernetes does not have a built-in network driver for cross-node Pod communication. Instead, it uses an open interface called CNI. You must pick a CNI plugin to manage cluster networking:

* **Overlay CNIs (e.g., Flannel, Calico):** Wrap Pod packets inside host packets (VXLAN/Geneve) to route across machines.
* **Flat/Routed CNIs (e.g., Calico BGP, AWS VPC CNI):** Assign real cloud or datacenter IPs directly to Pods without encapsulation.

#### 2. Pod-Level Networking (Containers Sharing Localhost)

In Docker, every container gets its own IP by default. In Kubernetes:

* A Pod is a group of one or more containers.
* All containers within the same Pod share the exact same network namespace and IP address.
* Containers in the same Pod communicate with each other via localhost and shared ports.

#### 3. Kubernetes Services (East-West / Internal Traffic)

Because Pods are ephemeral (they die and get new IP addresses constantly), you never target Pod IPs directly. Instead, Kubernetes introduces Services:

* **ClusterIP (Default):** Provides a single, permanent virtual IP (VIP) that automatically load-balances traffic across a dynamic group of Pods.
* **NodePort:** Exposes a Service on a static port across every physical worker node’s IP address.
* **LoadBalancer:** Asks your cloud provider (AWS, GCP, Azure) to provision an external hardware/cloud Load Balancer that routes to your cluster.
* **Headless Service:** Bypasses load balancing and returns the raw list of individual Pod IPs via DNS (used for stateful databases like PostgreSQL or Kafka).

#### 4. kube-proxy and the Cluster Data Plane

How do Virtual IPs actually route traffic inside Kubernetes? Through a component running on every node called kube-proxy:

* **iptables mode:** Generates thousands of iptables rules to route service traffic.
* **IPVS mode:** Uses Linux IPVS for higher performance in massive clusters.
* **eBPF mode (e.g., Cilium):** Bypasses iptables entirely using Linux eBPF in the kernel for extreme performance and security observability.

#### 5. Cluster-Internal DNS (CoreDNS)

Kubernetes runs an internal DNS service (CoreDNS) that uses structured Fully Qualified Domain Names (FQDNs):

* Format: `<service-name>.<namespace>.svc.cluster.local`
* Allows Pods in any namespace to locate other services cleanly without hardcoding IPs.

#### 6. Ingress Controllers & Gateway API (North-South / External Traffic)

Exposing dozens of services via individual Cloud Load Balancers gets expensive. Instead, Kubernetes uses Layer 7 entry points:

* **Ingress Controller (e.g., Nginx, Traefik, HAProxy):** A single entry point that manages external HTTP/HTTPS routing, SSL/TLS termination, and path-based routing (/api vs /app).
* **Gateway API:** The modern successor to Ingress, providing advanced routing, multi-tenant traffic management, and header matching.

#### 7. Network Policies (Cluster Firewalls)

By default, all Pods in Kubernetes can talk to all other Pods. Network Policies act as Kubernetes-native firewalls:

* Define Ingress (incoming) and Egress (outgoing) rules using Pod labels and Namespaces.
* Example rule: "Only Pods labeled app=frontend can talk to Pods labeled app=database on port 5432."

#### 8. Service Mesh (Advanced Kubernetes Networking)

For enterprise-grade applications, tools like Istio or Linkerd sit on top of Kubernetes networking to provide:

* **mTLS (Mutual TLS):** Automatically encrypts all Pod-to-Pod communication.
* **Traffic Shifting / Canary Deployments:** Send 10% of traffic to version 2, 90% to version 1.
* **Distributed Tracing & Observability:** Track latency and request flows between dozens of microservices.

---

# Level 1 — Getting familiar with Kubernetes primitives & architecture 
Goal: Understand how Kubernetes works under the hood and interact with its core building blocks.

* Architecture: (i)🛠️ Troubleshooting: Understand the architecture — control plane, worker nodes, API server, etcd, kubelet, scheduler. (ii) From Docker Layers to Kubernetes Hierarchies: The "Nesting Doll" Architecture 🪆 (iii) 🧵 𝗡𝗼𝗯𝗼𝗱𝘆 𝘁𝗲𝗹𝗹𝘀 𝘆𝗼𝘂 𝘁𝗵𝗶𝘀 𝗮𝗯𝗼𝘂𝘁 𝗬𝗔𝗠𝗟 (iv) From Cluster to Code: Understanding Kubernetes Abstraction Layers (v) Kubernetes Is Basically a Wrapper Within a Wrapper 📦
* Understand what a Pod is and deploy one(Pod → Container)
* Understand what a Deployment is and create one (Deployment → ReplicaSet → Pod → Container)
* Master imperative vs declarative management
* Understand Labels and Selectors — how Deployments, Pods and Services connect
* Scale a Deployment up and down
* Understand what a Service is and expose a Deployment
* Understand ClusterIP, NodePort and LoadBalancer
* Use kubectl port-forward for local testing and debugging
* Inspect and debug a running Pod — logs, exec, describe
* Delete and recreate resources — observe self-healing
* Understand Kubernetes Namespaces and create your own

---

# Level 2 — Running real workloads & configuration 
Goal: Deploy, configure and manage a real application on Kubernetes reliably.

* Deploy a real application on Kubernetes
* Use ConfigMaps to manage application configuration
* Use Secrets to manage sensitive data
* Attach persistent storage with PersistentVolumes and PersistentVolumeClaims
* Understand StorageClasses and the Container Storage Interface for dynamic provisioning
* Set resource requests and limits
* Configure liveness and readiness probes
* Configure Startup probes for slow-starting applications
* Use a multi-container Pod — sidecar pattern
* Deploy a multi-tier app — frontend + backend + database
* Run batch workloads using Jobs and CronJobs
* Understand graceful shutdown — SIGTERM and preStop hooks for zero-downtime deployments

---

# Level 3 — Operating a cluster & traffic management 
Goal: Manage how traffic flows and how the cluster behaves under load.

* Install and use Helm to deploy applications
* Write your own Helm chart
* Set up Ingress and route traffic to multiple services
* Set up Cert-Manager for automatic TLS certificates via Let's Encrypt --for production...not locally
* Learn the Gateway API — the modern successor to Ingress
* Configure Horizontal Pod Autoscaler
* Explore Vertical Pod Autoscaler and KEDA for event-driven autoscaling
* Understand and apply RBAC — Roles, RoleBindings, ClusterRoles, ServiceAccounts
* Apply Network Policies to restrict pod-to-pod traffic
* Understand and use Init Containers
* Understand DaemonSets and StatefulSets
* Master advanced scheduling — Taints & Tolerations, Node Selectors, Node Affinity/Anti-affinity

---

# Level 4 — Production readiness, security & observability 
Goal: Harden, observe and secure a Kubernetes environment to production standard.

* Set up Prometheus and Grafana for cluster monitoring
* Set up centralized logging — Loki or EFK stack
* Introduce distributed tracing with Jaeger and OpenTelemetry
* Implement GitOps with ArgoCD or Flux
* Manage secrets properly — Sealed Secrets, External Secrets Operator or HashiCorp Vault
* Scan container images with Trivy inside a CI pipeline
* Enforce cluster governance with OPA Gatekeeper or Kyverno
* Implement Pod Security Admission — non-root, read-only filesystem, dropped capabilities
* Set up a full CI/CD pipeline that deploys to Kubernetes
* Perform a cluster backup and restore with Velero
* Monitor and optimize cluster costs using Kubecost or OpenCost

---

# Level 5 — Platform engineering & advanced extensibility 
Goal: Build, automate and operate a multi-tenant platform on top of Kubernetes.

* Build a multi-environment setup — dev, staging and prod namespaces
* Use Kustomize to manage environment-specific configs
* Implement Resource Quotas and LimitRanges for multi-tenant resource management
* Deploy and manage a StatefulSet database — PostgreSQL with proper storage
* Implement Chaos Engineering — kill pods and nodes deliberately and observe
* Set up Cluster Autoscaler and compare with Karpenter
* Ensure high availability using Topology Spread Constraints
* Upgrade a Kubernetes cluster without downtime — draining nodes and PodDisruptionBudgets
* Understand Custom Resource Definitions and the Operator Pattern
* Implement a Service Mesh — Istio, Linkerd or Cilium for mTLS and advanced traffic splitting
* Build an Internal Developer Platform using Backstage or Crossplane

---

# Extra:

* Killercoda learnig
* The Ultimate Kubernetes Commands Cheat Sheet: Master K8s, Master DevOps! 🚀⛵
* Troubleshooting: The biggest trap in Kubernetes is believing that STATUS: Running means everything is fine
* KUBERNETES — Every Concept, Nothing Missing
* I used to think the only real difference between my Kubernetes lab and production was "traffic and users."