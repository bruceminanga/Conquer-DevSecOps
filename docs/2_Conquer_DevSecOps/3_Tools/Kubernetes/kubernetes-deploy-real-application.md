# 🚢 Crossing the Bridge from "Hello World" to Real Workloads in Kubernetes



Level 1 of learning Kubernetes is all about basic pods, imperative CLI commands, and running default Nginx containers. 

**How do you cross the bridge into Level 2? You deploy a real application.**

A lot of tutorials overwhelm beginners immediately with persistent databases, secret management, and complex volume mounts. But the first milestone of running real workloads is simple: **transitioning from terminal commands to declarative YAML, and deploying a functional, environment-aware web app.**

Here is the complete runbook to deploy a multi-replica Node.js application locally using Minikube.

---

## 🛠️ Step 1: Write the Deployment

Instead of running a basic sleep command or static server, we deploy `paulbouwer/hello-kubernetes`—a pre-built Node.js app that reads its environment variables and serves a dynamic webpage displaying the host Pod details.

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: real-web-app
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nodejs-app
        image: paulbouwer/hello-kubernetes:1.10
        ports:
        - containerPort: 8080
```

---

## 🔗 Step 2: Write the Service

Those 3 Pods are isolated inside the cluster's internal network. To route browser traffic to them, create a `NodePort` Service to open an entry point on the node.

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: real-web-service
spec:
  type: NodePort
  selector:
    app: web    # Connects directly to the deployment's pod label
  ports:
    - port: 80          # Port exposed by the Service internally
      targetPort: 8080  # Port where the Node.js app is listening inside the container
      nodePort: 30080   # Port exposed on the host node
```

---

## 🚀 Step 3: Apply and Access

Reconcile both declarative manifests:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Verify that all 3 replicas are running:

```bash
kubectl get pods -l app=web
```

### Accessing the App (Minikube):
Because Minikube runs inside a virtual network, open a direct tunnel/browser window with:

```bash
minikube service real-web-service
```

Your browser will open showing the Node.js application interface. If you refresh the page multiple times, you'll see the Pod hostname change as the Service load-balances requests across your 3 replicas.

---

## 🧹 Step 4: Clean Up

Tear down the deployment and service when finished:

```bash
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
```

---

## 💡 Why This Step Matters

By isolating this single step, the foundation is set. You now have a resilient, stateless application running declaratively. 

This sets the stage for the next milestones in production readiness:
* Passing dynamic configurations and credentials via **ConfigMaps & Secrets**
* Adding persistent storage via **PersistentVolumes (PVs)**
* Enforcing self-healing via **Readiness and Liveness Probes**

Mastering Kubernetes isn't about rushing to the end; it's about mastering each layer of the architecture.

