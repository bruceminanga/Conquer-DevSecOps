# 🚀 Stop Wasting Money on Load Balancers: The Ultimate Guide to Kubernetes Ingress
### *(Plus a Complete Production Runbook with Path, Host, & TLS Routing)*



In Kubernetes, Pods are ephemeral—they crash, reschedule, and cycle IP addresses constantly. While internal Services allow microservices to communicate, exposing them externally presents a major cost and architecture challenge.

Kubernetes provides three external exposure mechanisms: **NodePort**, **LoadBalancer**, and **Ingress**. They are not competitors—they solve different problems at different network layers.

> 💰 **The Financial Reality:** If you have 50 HTTP microservices and assign each one a `type: LoadBalancer` Service, you are provisioning 50 separate cloud load balancers (AWS ALB/NLB, Azure LB, GCP Cloud LB). At ~$20/month per load balancer, **you are burning $1,000/month on networking before your apps process a single request**. 
>
> **Ingress solves this by fronting hundreds of services with a single load balancer.**

---

## 🗺️ Know Your Tools: NodePort vs. LoadBalancer vs. Ingress

| Feature                | NodePort                      | LoadBalancer                         | Ingress                                   |
| :--------------------- | :---------------------------- | :----------------------------------- | :---------------------------------------- |
| **OSI Layer**          | Layer 4 (TCP/UDP)             | Layer 4 / Layer 7 (Cloud dependent)  | **Layer 7 (HTTP / HTTPS)**                |
| **Port Range**         | High ports (`30000–32767`)    | Standard ports (`80`, `443`, custom) | Standard ports (`80`, `443`)              |
| **Cloud Provisioning** | ❌ None                        | ✅ One Cloud LB per Service ($$$)     | ✅ **One Cloud LB for the entire cluster** |
| **URL Routing**        | ❌ None                        | ❌ None                               | ✅ Path (`/api`) & Host (`api.domain.com`) |
| **TLS Termination**    | ❌ Manual                      | ⚠️ Handled at Cloud LB                | ✅ **Centralized at cluster edge**         |
| **Primary Use Case**   | On-prem/bare-metal, local dev | Databases, gRPC, raw TCP/UDP         | **HTTP/HTTPS microservice fleets**        |

---

## 🧠 Part 1: The 7 Superpowers of Ingress

An **Ingress** acts as an intelligent Layer 7 reverse proxy and traffic cop at the perimeter of your cluster:

1. **Massive Cost Consolidation:** You provision exactly one external Cloud Load Balancer. The Ingress Controller multiplexes traffic internally across your services.
2. **Path-Based Routing:** Route traffic by URI (`/billing` ➔ `billing-svc`, `/checkout` ➔ `checkout-svc`).
3. **Host-Based (Subdomain) Routing:** Route by domain name (`api.example.com` ➔ `api-svc`, `app.example.com` ➔ `frontend-svc`).
4. **Centralized TLS/SSL Termination:** Terminate HTTPS certificates at the Ingress edge and forward unencrypted HTTP internally over private cluster networks.
5. **Transparent URL Rewriting:** Rewrite request paths (e.g., stripping `/api/v1` prefixes) before requests reach the application.
6. **Advanced Traffic Shaping:** Native rate limiting, sticky sessions, and canary deployments (e.g., routing 10% of traffic to a new release).
7. **Perimeter Authentication:** Enforce OAuth, OpenID Connect (OIDC), or basic auth checks before traffic enters backend pods.

---

## 🛠️ Part 2: The Hands-On Runbook

> 💡 **Core Distinction:** An **Ingress Resource** is merely a manifest describing rules. To enforce them, you must have an **Ingress Controller** (e.g., NGINX Ingress, Traefik, HAProxy) running in the cluster to process requests.

---

### Phase 1: Install the Ingress Controller

Deploy the NGINX Ingress Controller:

```bash
# Production Cloud Cluster (EKS, GKE, AKS):
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# Local Minikube:
minikube addons enable ingress
```

Verify that the controller pods are operational:

```bash
kubectl get pods -n ingress-nginx -w
```

Check the external entry point:

```bash
kubectl get svc -n ingress-nginx
```

* **Cloud:** The `ingress-nginx-controller` Service shows `TYPE: LoadBalancer` with an `EXTERNAL-IP`. This is the single public IP you pay for.
* **Minikube:** Run `minikube tunnel` in a separate terminal to allocate a routable local IP.

---

### Phase 2: Deploy Sample Backend Microservices

We will deploy two separate demo apps (`apple` and `banana`) using `hashicorp/http-echo`.

Create `apps.yaml`:

```yaml
# --- APPLE DEPLOYMENT & SERVICE ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apple-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: apple
  template:
    metadata:
      labels:
        app: apple
    spec:
      containers:
      - name: apple-app
        image: hashicorp/http-echo:0.2.3
        args: ["-text=You have reached the APPLE application!"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: apple-service
spec:
  type: ClusterIP   # Internal cluster IP only
  ports:
  - port: 80
    targetPort: 5678
  selector:
    app: apple
---
# --- BANANA DEPLOYMENT & SERVICE ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: banana-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: banana
  template:
    metadata:
      labels:
        app: banana
    spec:
      containers:
      - name: banana-app
        image: hashicorp/http-echo:0.2.3
        args: ["-text=You have reached the BANANA application!"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: banana-service
spec:
  type: ClusterIP   # Internal cluster IP only
  ports:
  - port: 80
    targetPort: 5678
  selector:
    app: banana
```

Apply the services:

```bash
kubectl apply -f apps.yaml
```

---

### Phase 3: Path-Based Routing

Route traffic to different backends based on the URL path (`/apple` and `/banana`).

Create `ingress-path.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fruit-ingress-path
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx   # Replaces deprecated kubernetes.io/ingress.class annotation
  rules:
  - http:
      paths:
      - path: /apple
        pathType: Prefix
        backend:
          service:
            name: apple-service
            port:
              number: 80
      - path: /banana
        pathType: Prefix
        backend:
          service:
            name: banana-service
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f ingress-path.yaml
```

---

### Phase 4: Host-Based Routing (Subdomains)

Route traffic by hostname (`apple.example.com` vs. `banana.example.com`):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fruit-ingress-host
spec:
  ingressClassName: nginx
  rules:
  - host: apple.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: apple-service
            port:
              number: 80
  - host: banana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: banana-service
            port:
              number: 80
```

---

### Phase 5: Centralized TLS/HTTPS Termination

Terminate encryption at the Ingress layer so pods process unencrypted internal traffic:

```bash
# 1. Generate a self-signed certificate for testing
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=fruit.example.com"

# 2. Store the certificate as a Kubernetes TLS Secret
kubectl create secret tls fruit-tls-secret --key tls.key --cert tls.crt
```

Reference the secret in `ingress-tls.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fruit-ingress-tls
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - fruit.example.com
    secretName: fruit-tls-secret
  rules:
  - host: fruit.example.com
    http:
      paths:
      - path: /apple
        pathType: Prefix
        backend:
          service:
            name: apple-service
            port:
              number: 80
```

Apply:
```bash
kubectl apply -f ingress-tls.yaml
```

---

### Phase 6: Verifying Traffic Routing

Find the external IP assigned to your Ingress:

```bash
kubectl get ingress fruit-ingress-path
```

Test path-based endpoints:

```bash
# 1. Test Apple routing
curl http://<INGRESS_IP>/apple
# Output: You have reached the APPLE application!

# 2. Test Banana routing
curl http://<INGRESS_IP>/banana
# Output: You have reached the BANANA application!

# 3. Test Invalid Route
curl -i http://<INGRESS_IP>/orange
# Output: HTTP/1.1 404 Not Found
```

---

## 🔧 DevOps Troubleshooting Playbook

| Step                   | Command                                                                 | What to Check                                                                      |
| :--------------------- | :---------------------------------------------------------------------- | :--------------------------------------------------------------------------------- |
| **1. Ingress Status**  | `kubectl describe ingress <name>`                                       | Check `Events:` for class mismatches or syntax errors.                             |
| **2. Endpoint Wiring** | `kubectl get endpoints <service-name>`                                  | If `Endpoints` is `<none>`, the Service `spec.selector` does not match Pod labels. |
| **3. Controller Logs** | `kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx` | Inspect raw NGINX logs for upstream 502/504 errors and configuration reloads.      |

---

## 🔮 The Road Ahead: Ingress vs. Gateway API

While Ingress remains the established standard for HTTP routing, Kubernetes is transitioning to the **Gateway API** as its long-term successor. 

The Gateway API introduces role-oriented design (separating infrastructure provisioning by Platform Admins from routing rules by Developers) and native Layer 4/Layer 7 cross-namespace support. Mastering Ingress fundamentals provides the foundation needed to transition to the Gateway API seamlessly.

