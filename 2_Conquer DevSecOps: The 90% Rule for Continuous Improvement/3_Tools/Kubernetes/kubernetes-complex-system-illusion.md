# Complex systems are an illusion. They’re just simple components working together

Take Kubernetes for example. Most engineers get overwhelmed by Kubernetes because it feels like a mysterious black box.

When an error happens, they panic.

But:  
You don’t need to be a wizard to master it. You just need to understand your own computer.

Now:  
Think about the laptop or PC on your desk:
* The CPU doesn’t store your photos forever; it just calculates instructions.
* The SSD doesn’t render video games; it just stores data when the power is off.
* The RAM doesn't manage the power switch; it just holds active memory for running programs.
* The GPU doesn't worry about network traffic; it crunches parallel math for visuals.

If your screen stays black but the cooling fans are spinning, you don’t say, "The whole computer is broken."  
You instinctively isolate the handoff: "The power supply works, but the display cable or GPU failed."  
Every component does one thing, does it well, and hands off the work.

So:  
Kubernetes works the exact same way:  
inside kubernetes cluster, we have 2 types of nodes:

### 1. 🧠 The Control Plane
* etcd (The SSD) - Safely stores the cluster state. It has no idea what a container is.
* kube-apiserver (The Motherboard) - The central hub where all signals and requests pass through.
* kube-scheduler (The Matchmaker) - Finds which machine has enough RAM and CPU. It never starts apps; it just assigns the address ie assigns Pods to nodes
* kube-controller-manager (The Thermostat) - Constantly compares your desired state with reality (e.g., "keep 3 copies running").
* cloud-controller-manager (The Cloud Bridge) - Talks to AWS/GCP/Azure to provision external disks, subnets, and load balancers.

### 2. ⚙️ The Worker Node
* kubelet - Takes orders from the Control Plane and tells the local runtime what to execute.
* container runtime - The engine (like containerd) that actually pulls the image and runs the isolated processes.
* kube-proxy (The Network Card) - Manages IP and port forwarding rules so traffic finds the right container.

Remember:
* When your Pod is stuck in Pending, the cluster hasn't collapsed.  
The API server took your request, but the scheduler couldn’t find a machine with enough memory.
* When your Pod is in ImagePullBackOff, the scheduler did its job, but the local worker couldn't find your image on Docker Hub.

See:  
Troubleshooting distributed systems isn’t guesswork. It's just asking:  
"Which component failed to do its job?"

Once you stop viewing complex systems as monoliths and start seeing them as handoffs between specialized parts, nothing is a black box anymore.

