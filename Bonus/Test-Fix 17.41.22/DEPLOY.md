# Deployment & Verification Guide: `http-echo`

This guide outlines all the commands, configurations, and verification steps used to fix, deploy, and validate the `http-echo` Helm chart with Ingress and TLS (HTTPS) on Minikube.

---

## 1. Prerequisites & Cluster Setup

### Check Minikube Status
Ensure the local Kubernetes cluster is running:
```bash
minikube status
```

### Enable NGINX Ingress Controller Addon
Minikube's built-in NGINX Ingress controller is disabled by default. Enable it:
```bash
minikube addons enable ingress
```

Verify that the Ingress controller pods are running and ready:
```bash
kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```
*Expected status: `1/1 Running`.*

Retrieve the Minikube node IP:
```bash
minikube ip
```

---

## 2. Production TLS Setup: `cert-manager` + Let's Encrypt (DNS-01 Challenge)

In production, manual self-signed certificates are replaced by automated certificate lifecycle management using `cert-manager` and Let's Encrypt with the **ACME DNS-01 challenge solver**. 

DNS-01 requires **no open inbound ports or public IP port forwarding** because Let's Encrypt verifies domain ownership by checking TXT records directly on DuckDNS nameservers.

### Step 1: Install `cert-manager` (if not already installed)
```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

### Step 2: Install the DuckDNS Webhook for cert-manager
Install the `cert-manager-webhook-duckdns` plugin:
```bash
helm repo add mmontes https://mmontes11.github.io/charts
helm repo update mmontes
helm upgrade --install cert-manager-webhook-duckdns mmontes/cert-manager-webhook-duckdns \
  --namespace cert-manager \
  --set duckdns.token="6cc6ba38-ed18-4bde-9523-682f8b814235" \
  --set logLevel=2
```

### Step 3: Apply the DNS-01 `ClusterIssuer`
Apply `cert-manager-issuer.yaml` configured with `elad.suzy@hotmail.com` and DuckDNS DNS-01 solver:
```bash
kubectl apply -f cert-manager-issuer.yaml
```

Verify that the ACME account is registered and the issuer is **Ready**:
```bash
kubectl get clusterissuer letsencrypt-prod
```
*Expected status: `READY: True`.*

### Step 4: Clean Up Any Old Self-Signed Secret
```bash
kubectl delete secret http-echo-tls --ignore-not-found
```

---

## 3. Helm Chart Validation & Deployment

### Lint the Chart
Validate chart syntax and structure:
```bash
helm lint devops-test
```
*Expected output: `1 chart(s) linted, 0 chart(s) failed`.*

### Test Template Rendering
Verify rendered Kubernetes manifests (dry-run):
```bash
helm template http-echo devops-test
```

### Deploy / Upgrade the Helm Release
Deploy the package to the cluster:
```bash
helm upgrade --install http-echo devops-test
```

---

## 4. Workload & Ingress Verification

### Check Pod Status
Confirm the Pod has been scheduled, is `Running`, and passes readiness probes (`1/1`):
```bash
kubectl get pods -l app.kubernetes.io/name=http-echo -o wide
```

### Check Service & Endpoints
Ensure the service is using **TCP** and points to the running container endpoint on port **5678**:
```bash
kubectl get svc,endpoints -l app.kubernetes.io/name=http-echo
```

### Check Ingress Configuration
Verify that the Ingress resource is bound to the NGINX class and listening on ports **80** and **443**:
```bash
kubectl get ingress -l app.kubernetes.io/name=http-echo
```
*Expected output:*
```text
NAME        CLASS   HOSTS                  ADDRESS        PORTS     AGE
http-echo   nginx   http-echo.duckdns.org  <MINIKUBE_IP>  80, 443   ...
```

### Check Certificate Status
Verify that `cert-manager` has processed the Ingress annotation and issued the Certificate via DNS-01:
```bash
kubectl get certificate http-echo-tls
```
*Expected status: `READY: True`.*

---

## 5. Automated Test Suite

Run the built-in Helm test to verify internal service connectivity:
```bash
helm test http-echo
```
*Expected output:*
```text
TEST SUITE:     http-echo-test-connection
Phase:          Succeeded
```

---

## 6. End-to-End HTTP & HTTPS Traffic Tests

### Option A: Testing via `minikube ssh` (Docker Driver on macOS)
Inside the Minikube VM:

#### Test HTTP (Port 80) -> Redirects to HTTPS:
```bash
minikube ssh 'curl -s -H "Host: http-echo.duckdns.org" http://localhost/'
```

#### Test HTTPS / TLS (Port 443):
```bash
minikube ssh 'curl -s -H "Host: http-echo.duckdns.org" https://localhost/'
```

*Expected JSON response:*
```json
{
  "host": {
    "hostname": "http-echo.duckdns.org"
  },
  "http": {
    "method": "GET",
    "protocol": "http"
  },
  "request": {
    "headers": {
      "host": "http-echo.duckdns.org",
      "x-forwarded-port": "443",
      "x-forwarded-proto": "https"
    }
  },
  "environment": {
    "PORT": "5678"
  }
}
```

---

### Option B: Testing from Local Host Machine

1. **Verify DuckDNS resolution:**
   ```bash
   dig +short http-echo.duckdns.org
   ```

2. **If testing locally via Minikube tunnel (Docker driver on macOS):**
   ```bash
   minikube tunnel
   ```

3. **Send requests:**
   ```bash
   # HTTP Test (308 redirect)
   curl -i http://http-echo.duckdns.org/

   # HTTPS Test (trusted Let's Encrypt production certificate)
   curl -i https://http-echo.duckdns.org/
   ```

---

## 7. Rollback / Cleanup (Optional)

To delete the deployment, cert-manager resources, and secret:
```bash
helm uninstall http-echo
kubectl delete secret http-echo-tls --ignore-not-found
```
