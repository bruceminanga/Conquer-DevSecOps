# 🔒 Stop Manually Managing TLS in Kubernetes
### *(The Complete Runbook for Cert-Manager & Let's Encrypt)*



We've all heard the horror stories: an SSL certificate quietly expires over the weekend, users are greeted with a *"Your connection is not private"* warning, and a production application goes down.

In modern infrastructure, manual certificate management is a massive operational liability. If you are running applications in Kubernetes, automated certificate lifecycle management is mandatory.

Here is the complete architectural breakdown of the **Cert-Manager + Let's Encrypt** stack, followed by an end-to-end production runbook.

---

## 🧠 Part 1: Why This Stack is the Industry Standard

### 1. Why Let's Encrypt?
Historically, TLS certificates cost hundreds of dollars annually per domain from commercial Certificate Authorities (CAs).
* **Cost:** Let's Encrypt is a non-profit CA providing trusted X.509 certificates at **$0**.
* **The 90-Day Lifespan (The Forcing Function):** Let's Encrypt certificates expire every **90 days**. This is intentional: it forces engineering teams to automate renewals rather than relying on manual calendar reminders. Across 50+ microservices, manual 90-day renewals guarantee eventual human error.

### 2. Why Cert-Manager?
Cert-Manager is a native Kubernetes **Operator**. It extends the Kubernetes API with Custom Resource Definitions (CRDs) to automate certificate issuance and renewal:
* **Declarative Issuance:** Cert-Manager watches cluster resources. When an Ingress requests a TLS certificate, it communicates with Let's Encrypt, completes the ACME domain ownership challenge, acquires the certificate, and stores it in a Kubernetes `Secret`.
* **Zero-Touch Renewals:** Cert-Manager monitors certificate expiration and automatically triggers renewal **30 days before expiration** (at the 60-day mark) with zero downtime.

---

## 🛠️ Part 2: Hands-On Production Runbook

### 📋 Prerequisites
* A running Kubernetes cluster.
* `kubectl` and `helm` installed.
* An active Ingress Controller (e.g., NGINX Ingress, Traefik).
* A registered public domain name (e.g., `app.yourdomain.com`).
* A **DNS A-Record** pointing `app.yourdomain.com` to your Ingress Controller's public IP address.

---

### Step 1: Install Cert-Manager via Helm

Deploy Cert-Manager into its own dedicated namespace with CRDs enabled:

```bash
# 1. Add Jetstack repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# 2. Install cert-manager with Custom Resource Definitions
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.4 \
  --set crds.enabled=true
```

Verify that all three controller pods are `Running`:

```bash
kubectl get pods -n cert-manager
```

#### ⚠️ CRD Safety Verification
Confirm that all 6 core CRDs registered cleanly:

```bash
kubectl get crds | grep cert-manager
```

*Expected output must include:* `clusterissuers.cert-manager.io`, `certificates.cert-manager.io`, `orders.acme.cert-manager.io`, and `challenges.acme.cert-manager.io`.

---

### Step 2: Configure the ClusterIssuer

An **Issuer** configures the connection between Cert-Manager and Let's Encrypt. A **ClusterIssuer** is cluster-scoped and can issue certificates across all namespaces.

> 🚨 **CRITICAL: Test with Staging First!**  
> Let's Encrypt Production enforces a strict rate limit of **5 duplicate certificates per week per domain**. If your configuration has an error and retries in a tight loop, your domain will be locked out for 7 days. Staging has significantly higher rate limits.

#### 1. Create Staging Issuer (`cluster-issuer-staging.yaml`):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your-email@yourdomain.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

#### 2. Create Production Issuer (`cluster-issuer-prod.yaml`):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@yourdomain.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

Apply the production issuer:

```bash
kubectl apply -f cluster-issuer-prod.yaml
kubectl get clusterissuer letsencrypt-prod
```
*Wait until `READY` displays `True`.*

---

### Step 3: Deploy Application Ingress with Automatic TLS

Annotate your Ingress resource to instruct Cert-Manager to generate a certificate automatically.

Create `app-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-secure-app-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.yourdomain.com
    secretName: my-app-tls-secret
  rules:
  - host: app.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f app-ingress.yaml
```

---

### Step 4: Verification & Inspection

```bash
# 1. Watch certificate issuance (transitions to READY=True in ~30-60s)
kubectl get certificate -w

# 2. Confirm TLS secret creation
kubectl get secret my-app-tls-secret

# 3. Test via cURL
curl -Iv https://app.yourdomain.com
```

Open `https://app.yourdomain.com` in your browser. Click the padlock icon to verify a valid, trusted certificate issued by Let's Encrypt.

---

## 📌 HTTP-01 vs. DNS-01 Challenges

| Challenge Type             | How It Works                                                                                              | When to Use It                                                                                                  |
| :------------------------- | :-------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------- |
| **HTTP-01** *(Used above)* | Let's Encrypt sends an HTTP request to `http://<domain>/.well-known/acme-challenge/` to verify ownership. | Standard public web applications reachable over port 80.                                                        |
| **DNS-01**                 | Cert-Manager creates a temporary `_acme-challenge` TXT record in your DNS provider (Route53, Cloudflare). | **Wildcard certificates** (`*.domain.com`) or **private/internal clusters** not exposed to the public internet. |

---

## 🔧 Part 3: Operational Debugging Runbook

When a certificate fails to issue (`READY: False`), inspect Cert-Manager's resource hierarchy in this exact order:

```text
Certificate  ──►  CertificateRequest  ──►  Order  ──►  Challenge
```

### 1. Inspect the Certificate Resource
```bash
kubectl describe certificate my-app-tls-secret
```
*Look at `Status.Conditions` for high-level errors.*

### 2. Inspect the CertificateRequest
```bash
kubectl get certificaterequest
kubectl describe certificaterequest <REQUEST_NAME>
```

### 3. Inspect the ACME Order
```bash
kubectl get orders
kubectl describe order <ORDER_NAME>
```

### 4. Inspect the Challenge (The Root Cause Layer)
```bash
kubectl get challenges
kubectl describe challenge <CHALLENGE_NAME>
```

### 🔍 Most Common HTTP-01 Failures:
* **Port 80 Blocked:** Let's Encrypt **requires port 80** to validate HTTP-01 challenges, even though the end goal is HTTPS port 443. Ensure cloud security groups allow inbound port 80.
* **DNS Propagation Delay:** The DNS A-record must point to your Ingress controller's external IP address before applying the Ingress manifest.
* **Rate Limits:** If you hit Let's Encrypt production rate limits, switch temporarily to `letsencrypt-staging` while debugging.

### 🔄 Forcing an Immediate Certificate Renewal
If credentials are compromised or you need to force an immediate rotation, delete the TLS secret:

```bash
kubectl delete secret my-app-tls-secret
```
*Cert-Manager will immediately detect the missing secret and request a fresh certificate from the CA.*

