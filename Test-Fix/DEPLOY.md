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

Create a self-signed TLS certificate for the domain `http-echo.k8s-server.local`:

```bash
# 1. Generate RSA private key and self-signed certificate
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
