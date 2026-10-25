# 🚨 The Ultimate Guide to Kubernetes Network Policies
### *(Zero-Trust Pod-to-Pod Segmentation & A Complete Implementation Runbook)*



By default, Kubernetes implements an **open, flat network**.

When you spin up a cluster, **every single pod can communicate with every other pod across all namespaces**—this is a "Default Allow" architecture.

While this makes early development convenient, it is an operational liability in production. If a public-facing web container is compromised, a flat network gives an attacker unhindered lateral access to backend databases, Redis caches, and internal services.

Here is why network policies are non-negotiable for Zero-Trust architectures, followed by a complete step-by-step production runbook.

---

## 🛑 Part 1: Why Restrict Pod-to-Pod Traffic?

1. **Halting Lateral Movement (Limiting the Blast Radius) 💥:** If an attacker exploits an RCE vulnerability in an external-facing frontend pod, a flat network allows them to port-scan your entire cluster. Network policies ensure that a compromised pod can only reach its explicitly whitelisted backend endpoints.
2. **Zero-Trust Workload Identity 🔒:** Modern security assumes the internal network is already compromised. Network policies enforce firewall boundaries based on dynamic Kubernetes metadata (**Labels & Selectors**), not brittle physical IP addresses.
3. **Regulatory Compliance (PCI-DSS, HIPAA, SOC 2) 📋:** Compliance frameworks mandate strict network segmentation between payment processing systems, databases storing sensitive personal data, and public services.
4. **Multi-Tenancy & Environment Isolation 🏢:** Prevents staging or dev namespaces sharing the same physical cluster from accidentally or maliciously communicating with production services.
5. **Mitigating Human Configuration Errors ⚠️:** Prevents a developer running a load test in a staging namespace from accidentally pointing traffic at a production database service URL.
6. **Blocking Egress Data Exfiltration 📤:** Network policies control outbound traffic as well as inbound. If a pod is compromised, egress restrictions prevent it from downloading crypto-miners or exfiltrating stolen credentials to command-and-control servers.

---

## 🛠️ Part 2: Hands-On Implementation Runbook

---

### Phase 1: Prerequisites & Traffic Discovery

#### Step 1: Verify your CNI (Container Network Interface)
> ⚠️ **CRITICAL GOTCHA:** Kubernetes stores NetworkPolicy objects in `etcd`, but the underlying **CNI plugin** enforces them at the packet level.  
> * **Supported CNIs:** Calico, Cilium, Antrea, Weave Net, AWS VPC CNI (with Network Policy support enabled).  
> * **Unsupported:** **Flannel (by itself)**. If you run Flannel alone, Kubernetes will accept your YAML without error, but **silently ignore all rules**.

#### Step 2: Map Application Traffic Dependencies
Before applying any deny rules, map the communication matrix:
* Does `frontend` need to talk to `backend` on port `8080`?
* Does `backend` need to reach PostgreSQL on port `5432`?
* Does `backend` need outbound internet access to payment processors (e.g., Stripe API on port 443)?

---

### Phase 2: Establish the Baseline (Default Deny)

Applying a policy to a namespace immediately isolates the selected pods.

#### Step 3: Apply "Default Deny All" (Ingress & Egress)
This locks down the namespace completely: all inbound and outbound traffic is severed.

Create `01-default-deny-all.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: my-app-namespace
spec:
  podSelector: {} # Empty selector matches ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress
```

Apply:

```bash
kubectl apply -f 01-default-deny-all.yaml
```

*(Workloads inside `my-app-namespace` are now isolated. Proceed immediately to Step 4).*

---

### Phase 3: Whitelisting Legitimate Traffic

#### Step 4: Allow CoreDNS Resolution (The Most Forgotten Step!)
Because we blocked all Egress in Step 3, pods can no longer resolve domain names via CoreDNS. We must explicitly open outbound port `53` to the `kube-system` namespace.

Create `02-allow-dns.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-dns
  namespace: my-app-namespace
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

Apply:

```bash
kubectl apply -f 02-allow-dns.yaml
```

---

#### Step 5: Allow Frontend-to-Backend Pod Traffic
Allow pods labeled `app: frontend` to access pods labeled `app: backend` on port `8080`.

Create `03-allow-front-to-back.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: my-app-namespace
spec:
  podSelector:
    matchLabels:
      app: backend   # Target: Applies ONLY to backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend # Source: Permits traffic only from frontend pods
    ports:
    - protocol: TCP
      port: 8080
```

Apply:

```bash
kubectl apply -f 03-allow-front-to-back.yaml
```

---

#### Step 6: Allow External Inbound Traffic to the Frontend
Allow external ingress traffic from Ingress Controllers or Load Balancers to the frontend web tier.

> ⚠️ **Syntax Warning:** In NetworkPolicy specs, `from: []` (empty array) means **"Allow from anywhere"**. Do not confuse this with omitting the `from` field entirely, which means **"Deny all traffic"**.

Create `04-allow-external-to-front.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-allow-external
  namespace: my-app-namespace
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from: [] # Explicit: Allow ingress from any external source
    ports:
    - protocol: TCP
      port: 80
```

Apply:

```bash
kubectl apply -f 04-allow-external-to-front.yaml
```

---

#### Step 7: Allow Egress to External Public APIs (Restricting Internal Subnets)
If backend pods need to query third-party public APIs (e.g., payment gateways over HTTPS port 443), allow external internet IPs while blocking internal RFC 1918 private subnets.

Create `05-backend-external-egress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-external-egress
  namespace: my-app-namespace
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0          # Allow public internet...
        except:
        - 10.0.0.0/8             # ...but block private VPC / Cluster IP ranges
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
```

Apply:

```bash
kubectl apply -f 05-backend-external-egress.yaml
```

---

#### Step 8: Strict Multi-Tenant Namespace Isolation
Prevent any workload in `staging`, `dev`, or other namespaces from communicating with `production`.

Create `06-deny-cross-namespace.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace-ingress
  namespace: production
spec:
  podSelector: {} # Protects all pods in production
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector: # Permits ingress ONLY from pods in production
        matchLabels:
          kubernetes.io/metadata.name: production
```

Apply:

```bash
kubectl apply -f 06-deny-cross-namespace.yaml
```

---

### Phase 4: Testing & Verification

#### Step 9: Verify Authorized Traffic Succeeds
```bash
# Exec into frontend pod and query backend
kubectl exec -it <frontend-pod-name> -n my-app-namespace -- \
  wget -qO- http://backend-service:8080
# Output: HTTP 200 OK (Traffic permitted)
```

#### Step 10: Verify Unauthorized Traffic is Blocked
Spawn an un-labeled test pod inside the same namespace:

```bash
kubectl run rogue-pod --image=busybox:1.36 -n my-app-namespace -- sleep 3600

# Attempt to query backend
kubectl exec -it rogue-pod -n my-app-namespace -- \
  wget -T 3 -qO- http://backend-service:8080
# Output: wget: download timed out (Connection silently dropped)
```

---

## 🔍 Phase 5: Observability & Monitoring Dropped Packets

Operating network policies without logging dropped traffic creates an operational blind spot.

* **Cilium (The Gold Standard):** Use **Hubble** to observe packet verdicts in real time:
  ```bash
  hubble observe --namespace my-app-namespace --verdict DROPPED
  ```
* **Calico:** Enable Felix flow logging to stream dropped flow events to syslog or Elasticsearch.
* **Troubleshooting DNS Failures:** If traffic fails, test connection using a raw IP instead of a Service DNS hostname. If the raw IP connects, **Step 4 (DNS Egress on port 53)** is misconfigured.

---

## 🚨 Phase 6: Emergency Rollback ("Break-Glass" Procedure)

If a policy causes an unintended outage, deleting the policy manifests restores the cluster to **Default Allow** immediately without restarting application pods:

```bash
kubectl delete networkpolicy default-deny-all -n my-app-namespace
kubectl delete networkpolicy allow-egress-dns -n my-app-namespace
kubectl delete networkpolicy backend-allow-frontend -n my-app-namespace
kubectl delete networkpolicy frontend-allow-external -n my-app-namespace
kubectl delete networkpolicy backend-allow-external-egress -n my-app-namespace
kubectl delete networkpolicy deny-cross-namespace-ingress -n production
```

---

## 📋 Network Policy Rule of Thumb

* [ ] Verify that your cluster uses a policy-enforcing CNI (Calico, Cilium, Antrea).
* [ ] Always whitelist CoreDNS outbound port 53 when applying Egress rules.
* [ ] Double check `from: []` (allow all) vs omitting `from` (deny all).
* [ ] Isolate multi-tenant environments using `namespaceSelector`.
* [ ] Deploy packet-level observability (e.g., Cilium Hubble) before enforcing policies in production.

