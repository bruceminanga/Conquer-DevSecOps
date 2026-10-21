# Kubernetes Architecture — Control Plane, Worker Nodes, API Server, etcd, Kubelet, & Scheduler

# First things first:

> [Complex systems are an illusion](kubernetes-complex-system-illusion.md). They’re just simple components working together ----Kubernetes architecture

For a long time, many of us have treated Kubernetes like a magic black box. We type kubectl apply, cross our fingers, and assume the cluster will figure it out. If you are a software developer strictly writing code, that might be fine.

But if you are a DevOps Engineer, SRE, or Platform Engineer, treating Kubernetes like magic is dangerous. When things break, error messages rarely tell you how to fix them. You have to deduce the problem from the architecture — the same way a mechanic diagnoses an engine, not by guessing, but by understanding what each part is supposed to do.

I learned this the hard way. Early on, I spent nearly two hours debugging a pod stuck in Pending. I restarted it repeatedly, checked the image, checked the manifest. Nothing. The actual problem? The Scheduler couldn't find a node with enough memory to place it. Once I understood what the Scheduler actually does, that class of problem became a 2-minute diagnosis.

Here is my practical, no-nonsense breakdown of how Kubernetes actually works under the hood — and why every component matters.

A Kubernetes Cluster is a collection of nodes (machines) working together. Inside that cluster, the nodes are divided into two roles:

* Control Plane node(s) — The brain / management. Makes global decisions about the cluster (like scheduling) and responds to cluster events.
* Worker node(s) — The muscle / where your apps run. The actual servers/machines where your containerized applications (Pods) run. Every cluster needs at least one worker node.

Before we meet the players, there is one concept that ties everything together. Kubernetes is a reconciliation engine. It is always comparing two things: the Desired State (what you declared you want) and the Actual State (what is currently running). Every component exists to close the gap between those two states. When your cluster misbehaves, your job as the mechanic is to ask: why is it failing to reconcile?

Now let's meet the key players.

## 🏛️ THE CONTROL PLANE COMPONENTS

### 🚪 1. The API Server (kube-apiserver)

What it is: The front door and the traffic cop.

Why the mechanic needs to know it: It is the only component that communicates directly with the cluster database. Every kubectl command you run goes through here. It is designed to scale horizontally (deploying more instances to balance traffic). When commands time out, your network isn't necessarily down — your API Server might be buckling under the volume of requests. It is also the authentication and authorization gatekeeper. If your CI/CD pipeline suddenly loses cluster access, this is the first place you look.

### 💾 2. The Database (etcd)

What it is: The absolute source of truth. A consistent and highly-available key-value store.

Why the mechanic needs to know it: etcd stores the desired state of your cluster — every deployment, every secret, every config — as well as the status that components report back through the API Server. It is the crown jewels. If your cluster completely burns down, you do not need to back up the API Server or Scheduler individually. You only need to restore your etcd snapshot to bring the entire cluster back. Whoever controls etcd controls the cluster — which is why a robust backup plan, encryption, and strict access control are non-negotiable.

### ⚙️ 3. The Controller Manager (kube-controller-manager)

What it is: The reconciliation engine in action — the heartbeat of the cluster.

Why the mechanic needs to know it: This component runs multiple controller processes compiled into a single binary. They continuously watch the state stored in etcd (via the API Server) and take corrective action whenever the actual state drifts from the desired state. A Node Controller notices when a node goes down. A ReplicaSet Controller ensures the right number of pods are always running. Each controller is a dedicated reconciliation loop for one specific concern.

### ☁️ The Cloud Controller Manager (cloud-controller-manager)

What it is: The cloud provider liaison.

Why the mechanic needs to know it: This separates the components that interact with your cloud platform (AWS, GCP, Azure) from those that only interact with your cluster. If you run Kubernetes on bare metal or on your local PC, this component doesn't exist. But in the cloud, it manages cloud-specific dependencies — checking if a cloud VM was deleted, setting up cloud infrastructure routing, or spinning up cloud Load Balancers for your Services.

### 🤝 4. The Scheduler (kube-scheduler)

What it is: The matchmaker.

Why the mechanic needs to know it: The Scheduler does not run pods — it only decides where they should run. It continuously watches the API Server for newly created Pods with no assigned node, then selects the best-fit node based on resource requirements, hardware/software constraints, affinity rules, and deadlines. If a pod stays Pending, do not restart it. That is the Scheduler telling you it cannot find a suitable node. The fix is addressing the specific blocker — not restarting the pod.

## 🏗️ THE Worker Node COMPONENTS

### 👨✈️ 1. The Kubelet

What it is: The captain of the Worker Node.

Why the mechanic needs to know it: The Kubelet is the on-the-ground agent on every worker node. It receives PodSpecs from the API Server and ensures the containers described in them are running and healthy. If a pod is assigned to a node but is stuck in ContainerCreating, the Kubelet on that specific node is likely failing. The failure is local, not cluster-wide — and knowing that tells you exactly where to SSH and what logs to pull. (Note: Kubelet only manages containers created by Kubernetes.). So, it Talks to the control plane, checks on containers, and reports node health.

### 🔀 2. The Network Proxy (kube-proxy) (Optional)

What it is: The local switchboard operator.

Why the mechanic needs to know it: kube-proxy runs on every node and maintains the network rules that allow communication to your Pods from inside or outside the cluster (often using the OS packet filtering layer — iptables or ipvs). If pods are running but cannot talk to each other or be reached via a Service, your proxy rules might be broken. (Note: If you use an advanced network plugin that handles packet forwarding itself, you might not even need kube-proxy.). So, it Configures network rules so network traffic can reach your pods.

### 📦 3. The Container Runtime

What it is: The actual engine turning the gears.

Why the mechanic needs to know it: Kubernetes doesn't actually run containers; it delegates that to a Container Runtime Interface (CRI) like containerd or CRI-O. If the Kubelet is the node's captain, the Container Runtime is the engine room executing the lifecycle of the containers. If the runtime crashes, the node goes deaf to container commands. So, It's the software engine that actually pulls images and runs the containers.

## 🧩 THE ADDONS (The Aftermarket Upgrades)

While the core components run the cluster, Addons use standard Kubernetes resources (DaemonSets, Deployments) in the kube-system namespace to implement critical cluster-level features:

* DNS: Not strictly required by the spec, but practically mandatory. Containers automatically use this DNS server to resolve Kubernetes services. If apps can't find each other by name, check the DNS addon.
* Network Plugins (CNI): Software components responsible for allocating IP addresses to pods and enabling cross-node pod communication.
* Web UI (Dashboard): A general-purpose graphical interface for managing and troubleshooting the cluster.
* Monitoring & Logging: Tools for recording central time-series metrics about containers and saving container logs to a central, searchable store.

## Conclusion

Understanding Kubernetes architecture transitions you from a Kubernetes User to a Kubernetes Administrator. It turns panic-driven Googling into a logical, step-by-step debugging process. You stop fearing the error message and start reading it as a clue — because you know the system well enough to follow the trail.

The driver gets back in the car and hopes it starts. The mechanic opens the hood.

Learn the components. Understand the reconciliation loop. Become the mechanic. 🛠️

What was the Kubernetes component or concept that confused you most when you started? Drop it in the comments — I'll try to break it down.

## 📚 Go Deeper

* Kubernetes Architecture — The full deep-dive: nodes, control plane communication, controllers, CRI, garbage collection, and more. Start here if you want to go beyond the surface.
* Kubernetes Components Overview — A concise single-page reference listing every control plane and node component. Good for quick orientation or revisiting the basics.
* etcd Documentation — Architecture and operational guidance for the cluster database
* kube-scheduler — How the Scheduler selects nodes and what Pending pods actually mean
* kube-proxy — Network proxy configuration and modes (iptables vs. ipvs)
* Container Runtime Interface (CRI) — How Kubernetes delegates container execution
* Kubernetes Architecture Explained — DevOpsCube — A practitioner-level deep dive covering scheduling internals, etcd consensus mechanics, CNI plugin workflows, and more. Best read after you have the big picture.