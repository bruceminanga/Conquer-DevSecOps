# 🚀 Learn the Kubernetes Gateway API: The Modern Successor to Ingress
### *(Architecture Breakdown + Advanced Traffic Splitting Runbook)*



If you've ever managed Kubernetes traffic at scale, you know the pain of **"Annotations Hell"**:

```yaml
nginx.ingress.kubernetes.io/rewrite-target: /
nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
nginx.ingress.kubernetes.io/proxy-body-size: "20m"
```

The classic `Ingress` resource has served the ecosystem for years, but as cloud-native infrastructure has matured, its structural limitations have become major bottlenecks.

Enter the **Kubernetes Gateway API**—the richer, portable, role-oriented standard for routing traffic.

Here is why Ingress is being superseded, how the Gateway API solves its flaws, and a complete, hands-on technical runbook to deploy advanced routing from scratch.

---

## 🛑 Part 1: Why Ingress Had to Be Replaced

When Ingress was designed in the early days of Kubernetes, it was built for simple Layer 7 routing: matching a **Host** (`api.example.com`) and a **Path** (`/v1`). Modern microservices demand far more:

* **Annotations Hell:** Ingress lacked native support for common routing features (rate limiting, canary weighting, header rewrites). Controller vendors (NGINX, Traefik, HAProxy) were forced to implement features using non-standard annotations.
* **Zero Portability:** Migrating from NGINX Ingress to Traefik or Envoy required rewriting every manifest because vendor annotations were incompatible.
* **The Single-Persona Bottleneck:** Ingress is a single monolithic object. The **Platform Admin** (managing TLS certificates and public load balancer ports) and the **Application Developer** (managing internal paths) had to edit the exact same YAML file, causing permission conflicts.
* **HTTP/HTTPS Only:** Ingress had no native support for Layer 4 protocols (raw TCP/UDP) or gRPC.
* **Namespace Isolation Friction:** Ingress resources must live in the same namespace as the target Service. A central load balancer in an infrastructure namespace could not cleanly route to backend pods across different application namespaces.

---

## 🌟 Part 2: How the Gateway API Solves This

The Gateway API is an official Kubernetes SIG-Network project that establishes a standardized, expressive specification:

### 1. Role-Oriented Architecture (The Core Innovation)
It decouples traffic infrastructure into three distinct resources mapped to real-world engineering roles:

```text
  [ Cloud / Infra Provider ]  ──►  GatewayClass   (Defines LB implementation, e.g. Envoy, AWS, Cilium)
              │
  [ Platform / Cluster Admin ] ──►  Gateway        (Defines Listeners, Ports, IPs, & TLS termination)
              │
  [ Application Developer ]   ──►  HTTPRoute      (Defines Routing Rules, Paths, Filters, & Canary Splits)
```

### 2. Expressive & Annotation-Free
Canary traffic splitting, URL rewrites, and header modifications are **first-class native YAML fields**, not vendor-specific annotations.

### 3. True Workload Portability
An `HTTPRoute` manifest written for Envoy Gateway works on Istio, Cilium, or Traefik without changing a single line of YAML.

### 4. Protocol Agnostic
Natively supports `HTTPRoute`, `GRPCRoute`, `TCPRoute`, `UDPRoute`, and `TLSRoute`.

### 5. Secure Cross-Namespace Routing (`ReferenceGrant`)
Gateways can live in an infrastructure namespace (`infra`) while routing traffic securely to Services in developer namespaces (`app`). The API uses an explicit handshake resource called **`ReferenceGrant`** to prevent unauthorized routing.

---

## 🔄 Part 3: Migration Cheat Sheet (Ingress vs. Gateway API)

### Classic Ingress (Monolithic & Annotation-Heavy):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: app-v1
            port:
              number: 8080
```

### Gateway API Equivalent (Role-Separated & Declarative):

#### Cluster Admin owns `gateway.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: api.example.com
```

#### Application Developer owns `httproute.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
spec:
  parentRefs:
  - name: main-gateway
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: app-v1
      port: 8080
```

---

## 🛠️ Part 4: Hands-On Runbook

We will deploy **Envoy Gateway**, configure a Gateway entry point, deploy two microservice versions, and demonstrate native canary traffic splitting and header injection.

---

### Step 1: Install Gateway API Standard CRDs

Install the official v1 CRDs:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

---

### Step 2: Install Envoy Gateway

Deploy Envoy Gateway via Helm:

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.1.1 \
  -n envoy-gateway-system \
  --create-namespace

# Wait for controller readiness
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

---

### Step 3: Verify the GatewayClass

Verify that the Envoy Gateway controller registered its class:

```bash
kubectl get gatewayclass
```
*You should see `eg` (Envoy Gateway) with `CONTROLLER: gateway.envoyproxy.io/gatewayclass-controller`.*

---

### Step 4: Provision the Gateway (Cluster Admin Role)

Create `gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: default
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

Apply and capture the provisioned IP:

```bash
kubectl apply -f gateway.yaml

# Extract the assigned external IP
export GATEWAY_IP=$(kubectl get gateway main-gateway -o jsonpath='{.status.addresses[0].value}')
echo "Gateway IP: $GATEWAY_IP"
```

---

### Step 5: Deploy Sample Applications (V1 & V2)

Create `apps.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-v1
spec:
  selector:
    app: v1
  ports:
  - port: 8080
    targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: v1
  template:
    metadata:
      labels:
        app: v1
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo:0.2.3
        args: ["-text=Hello from Version 1"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app-v2
spec:
  selector:
    app: v2
  ports:
  - port: 8080
    targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: v2
  template:
    metadata:
      labels:
        app: v2
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo:0.2.3
        args: ["-text=Hello from Version 2"]
        ports:
        - containerPort: 5678
```

Apply the applications:

```bash
kubectl apply -f apps.yaml
```

---

### Step 6: Basic HTTP Routing (App Developer Role)

Attach an `HTTPRoute` to `main-gateway`.

Create `httproute-basic.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
  namespace: default
spec:
  parentRefs:
  - name: main-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app-v1
      port: 8080
```

Apply and test:

```bash
kubectl apply -f httproute-basic.yaml
curl http://$GATEWAY_IP/
# Output: Hello from Version 1
```

---

### Step 7: Native Canary Traffic Splitting (90/10)

Route 90% of requests to `app-v1` and 10% to `app-v2` using native `weight` fields:

Create `httproute-canary.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
  namespace: default
spec:
  parentRefs:
  - name: main-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app-v1
      port: 8080
      weight: 90
    - name: app-v2
      port: 8080
      weight: 10
```

Apply and verify the canary distribution:

```bash
kubectl apply -f httproute-canary.yaml

# Test with 10 requests:
for i in {1..10}; do curl -s http://$GATEWAY_IP/; done
```

---

### Step 8: Native Header Manipulation (Filters)

Inject custom response headers at the proxy layer without application code changes:

Create `httproute-filter.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
  namespace: default
spec:
  parentRefs:
  - name: main-gateway
  rules:
  - filters:
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        add:
        - name: X-Gateway-Success
          value: "true"
    backendRefs:
    - name: app-v1
      port: 8080
```

Apply and verify:

```bash
kubectl apply -f httproute-filter.yaml
curl -i http://$GATEWAY_IP/
# Inspect headers: X-Gateway-Success: true is returned!
```

---

### Step 9: TLS/HTTPS Termination at the Gateway

```bash
# 1. Generate test certificates
openssl req -x509 -newkey rsa:4096 -keyout tls.key -out tls.crt -days 365 -nodes \
  -subj "/CN=api.example.com"

# 2. Store as Kubernetes Secret
kubectl create secret tls gateway-tls-secret --cert=tls.crt --key=tls.key
```

Update `gateway.yaml` to terminate HTTPS on port 443:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: default
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
  - name: https
    protocol: HTTPS
    port: 443
    hostname: api.example.com
    tls:
      mode: Terminate
      certificateRefs:
      - name: gateway-tls-secret
```

> **Persona Separation Benefit:** The Application Developer's `HTTPRoute` does not need to change at all. TLS is handled entirely by the Cluster Administrator at the `Gateway` layer.

---

### Step 10: Cross-Namespace Routing & `ReferenceGrant`

By default, an `HTTPRoute` in namespace `app` cannot attach to a `Gateway` in namespace `infra` without an explicit security handshake.

```bash
kubectl create namespace infra
kubectl create namespace app
```

#### 1. Admin configures Gateway in `infra` to allow routes from specific namespaces:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: infra
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            shared-gateway-access: "true"
```

```bash
kubectl label namespace app shared-gateway-access=true
```

#### 2. Service owner in `app` permits the cross-namespace reference:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-infra-gateway
  namespace: app
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: app
  to:
  - group: ""
    kind: Service
```

#### 3. Developer attaches to the cross-namespace Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
  namespace: app
spec:
  parentRefs:
  - name: shared-gateway
    namespace: infra     # Explicitly references the infra namespace
  rules:
  - backendRefs:
    - name: app-v1
      port: 8080
```

---

## 🔧 Operational Troubleshooting Guide

Inspect the status subresource of your Gateway API objects:

```bash
# 1. Check Route Status
kubectl describe httproute my-app-route
```

Look for two primary conditions:
* **`Accepted: True`:** The Gateway recognized and processed the routing rules.
* **`ResolvedRefs: True`:** All backend services referenced in `backendRefs` exist and are accessible.

```bash
# 2. Check Gateway Data Plane Status
kubectl describe gateway main-gateway
```
* **`Programmed: True`:** The underlying data plane (Envoy) successfully compiled and applied the routing configuration.

---

## 🧹 Cleanup

```bash
kubectl delete httproute my-app-route
kubectl delete gateway main-gateway
kubectl delete -f apps.yaml
helm uninstall eg -n envoy-gateway-system
```

---

## 📋 Gateway API vs. Ingress Summary

| Capability              | Classic Ingress                   | Gateway API                                        |
| :---------------------- | :-------------------------------- | :------------------------------------------------- |
| **Configuration Model** | Monolithic (`Ingress`)            | Role-Oriented (`GatewayClass`, `Gateway`, `Route`) |
| **Canary Deployments**  | Vendor annotations / Service Mesh | **Native `weight` field**                          |
| **Header Modification** | Vendor annotations                | **Native `filters` block**                         |
| **Protocol Support**    | HTTP / HTTPS only                 | **HTTP, HTTPS, gRPC, TCP, UDP, TLS**               |
| **Multi-Tenancy**       | Same-namespace only               | **Secure Cross-Namespace with `ReferenceGrant`**   |
| **Portability**         | Low (vendor-lock on annotations)  | **100% specification portability**                 |

