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

## 2. TLS (HTTPS) Certificate Generation

> [!NOTE]
> **Production Implementation vs. Local Development:**
> The `openssl` commands below generate a self-signed certificate suitable **only for local development and testing** on Minikube.
>
> In a **production environment**, manual self-signed certificates should never be used. Instead, production TLS is implemented using one of the following architectures:
>
> 1. **Automated K8s-Native PKI (`cert-manager` + Let's Encrypt / ACME):**
>    - Install `cert-manager` in the cluster to automate certificate lifecycle management.
>    - Create an ACME `ClusterIssuer` (using HTTP-01 or DNS-01 challenge solvers).
>    - Add the annotation `cert-manager.io/cluster-issuer: "letsencrypt-prod"` (or `kubernetes.io/tls-acme: "true"`) to the Ingress resource in `values.yaml`.
>    - `cert-manager` automatically requests, issues, validates, and rotates the certificates every 60–90 days without downtime or manual intervention.
>
> 2. **Cloud Load Balancer SSL Offloading (AWS ACM / GCP Managed SSL / Azure App Gateway):**
>    - Terminate TLS directly at the cloud external Load Balancer (e.g., AWS ALB via AWS Load Balancer Controller).
>    - Attach the cloud-managed certificate ARN via Ingress annotations (`alb.ingress.kubernetes.io/certificate-arn: <arn>`).
>    - The cloud provider manages automated renewals with 100% SLA and offloads cryptographic computation from cluster nodes.
>
> 3. **Enterprise Internal PKI (HashiCorp Vault / Private CA):**
>    - For internal microservices and private VPCs, `cert-manager` is configured with a `Vault` or `CA` issuer backed by the enterprise corporate root CA.

```bash
# 1. Generate RSA private key and self-signed certificate (Dev/Minikube only)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key \
  -out /tmp/tls.crt \
  -subj "/CN=http-echo.k8s-server.local/O=http-echo"

# 2. Store certificate and key in a Kubernetes TLS secret
kubectl create secret tls http-echo-tls \
  --cert=/tmp/tls.crt \
  --key=/tmp/tls.key
```

Verify the secret was created in the target namespace:
```bash
kubectl get secret http-echo-tls
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
NAME        CLASS   HOSTS                        ADDRESS        PORTS     AGE
http-echo   nginx   http-echo.k8s-server.local   <MINIKUBE_IP>  80, 443   ...
```

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
When Minikube runs with the Docker driver on macOS, the Minikube internal IP (`192.168.49.2`) is isolated within the Docker network. You can test directly through the Ingress controller from inside the Minikube VM:

#### Test HTTP (Port 80):
```bash
minikube ssh 'curl -s -H "Host: http-echo.k8s-server.local" http://localhost/'
```

#### Test HTTPS / TLS (Port 443):
```bash
minikube ssh 'curl -k -s -H "Host: http-echo.k8s-server.local" https://localhost/'
```

*Expected JSON response:*
```json
{
  "host": {
    "hostname": "http-echo.k8s-server.local"
  },
  "http": {
    "method": "GET",
    "protocol": "http"
  },
  "request": {
    "headers": {
      "host": "http-echo.k8s-server.local",
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

1. **Add DNS mapping to `/etc/hosts`:**
   ```bash
   # If using hyperkit/virtualbox driver:
   echo "$(minikube ip) http-echo.k8s-server.local" | sudo tee -a /etc/hosts

   # If using docker driver on macOS with `minikube tunnel`:
   echo "127.0.0.1 http-echo.k8s-server.local" | sudo tee -a /etc/hosts
   ```

2. **If using Docker driver on macOS, start minikube tunnel in a separate terminal:**
   ```bash
   minikube tunnel
   ```

3. **Send requests from your host terminal:**
   ```bash
   # HTTP Test
   curl -i http://http-echo.k8s-server.local/

   # HTTPS Test (self-signed cert)
   curl -k -i https://http-echo.k8s-server.local/
   
   curl -vk https://http-echo.k8s-server.local/
   ```

---

## 7. Rollback / Cleanup (Optional)

To delete the deployment and TLS secret:
```bash
helm uninstall http-echo
kubectl delete secret http-echo-tls
```
