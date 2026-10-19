# Root Cause Analysis & Fixes: `http-echo` Helm Package

## Executive Summary
This document provides a comprehensive analysis of why the `http-echo` Helm package is broken, detailing the root causes across scheduling, container configuration, health probing, and networking, accompanied by production-grade fixes and step-by-step verification instructions.

---

## 1. Summary of Identified Issues

| # | Component | Root Cause | Impact |
|---|-----------|------------|--------|
| **1** | [values.yaml](file:///Users/es/Desktop/thetaray/devops-test/values.yaml#L67-L69) | CPU request is set to `6` CPUs | Pod stays in `Pending` indefinitely (`0/1 nodes are available: 1 Insufficient cpu`). |
| **2** | [templates/deployment.yaml](file:///Users/es/Desktop/thetaray/devops-test/templates/deployment.yaml#L36-L38) | Container args pass `-port "{{ .Values.service.port }}"` | `ealen/echo-server` ignores `-port` CLI flag and listens on port `80`, while the Service and Probes target port `5678`. |
| **3** | [templates/deployment.yaml](file:///Users/es/Desktop/thetaray/devops-test/templates/deployment.yaml#L43-L48) | Liveness probe executes `/bin/bash` | Image is Alpine-based; `/bin/bash` does not exist (`stat /bin/bash: no such file or directory`), causing restart loops (`CrashLoopBackOff`). |
| **4** | [templates/service.yaml](file:///Users/es/Desktop/thetaray/devops-test/templates/service.yaml#L12) | Service protocol is set to `UDP` | Ingress and HTTP clients cannot route TCP/HTTP traffic to a UDP service (returns `502 Bad Gateway`). |
| **5** | Cluster Ingress | Ingress controller not enabled | Ingress resource with `ingressClassName: nginx` has no controller to provision routing. |

---

## 2. In-Depth Root Cause Analysis

### Issue 1: Pod Unscheduled / Insufficient CPU
* **Location:** [values.yaml](file:///Users/es/Desktop/thetaray/devops-test/values.yaml#L67-L69)
```yaml
resources: 
   requests:
     cpu: "6"
     memory: "1Mi"
```
* **Analysis:**
  Local Kubernetes environments (Minikube, Docker Desktop, Kind) typically allocate 2 to 4 vCPUs per node. Requesting 6 full CPU cores prevents the Kubernetes scheduler from placing the pod on any available node. The pod remains in `Pending` state with the event:
  ```text
  Warning  FailedScheduling  default-scheduler  0/1 nodes are available: 1 Insufficient cpu.
  ```
  Additionally, `memory: "1Mi"` is unrealistically low for a Node.js process (`ealen/echo-server`), which requires 30–50MB memory baseline to initialize.

### Issue 2: Container Listens on Port 80, Ignoring `-port` CLI Arg
* **Location:** [templates/deployment.yaml](file:///Users/es/Desktop/thetaray/devops-test/templates/deployment.yaml#L36-L42)
```yaml
          args:
            - -port
            - "{{ .Values.service.port }}"
          ports:
            - name: http
              containerPort: 5678
              protocol: TCP
```
* **Analysis:**
  The container image `ealen/echo-server:0.5.2` has the entrypoint `["node", "webserver"]`. Unlike HashiCorp's `http-echo`, `ealen/echo-server` **does not accept command-line flags like `-port`**. Any flags passed via `args:` are ignored, and it defaults to reading its internal `global.json` which specifies `"port": 80`.
  To change its listening port, `ealen/echo-server` expects the `PORT` environment variable (e.g. `PORT=5678`).
  Because `args` is ignored, the container listens on port 80, but:
  - `containerPort` is set to `5678`
  - `readinessProbe` queries port `http` (5678)
  - `service.yaml` points `targetPort` to `http` (5678)
  
  All probes and incoming traffic to port 5678 result in `Connection refused`.

### Issue 3: Liveness Probe Exec Failure (`/bin/bash` Missing)
* **Location:** [templates/deployment.yaml](file:///Users/es/Desktop/thetaray/devops-test/templates/deployment.yaml#L43-L48)
```yaml
          livenessProbe:
            exec:
              command:
                - /bin/bash
                - -c
                - "wget --spider -q http://localhost:{{ .Values.service.port }}"
```
* **Analysis:**
  `ealen/echo-server` is packaged using **Alpine Linux**, which provides `/bin/sh` but does not include `/bin/bash`. When Kubelet invokes the exec liveness probe, the container runtime errors out:
  ```text
  Warning  Unhealthy  Liveness probe errored: exec: "/bin/bash": stat /bin/bash: no such file or directory
  ```
  This causes Kubelet to continuously terminate and restart the container, resulting in `CrashLoopBackOff`.
  Furthermore, using an `exec` probe running `wget` when standard `httpGet` probes are supported adds unnecessary process fork overhead.

### Issue 4: Service Configured with Protocol UDP
* **Location:** [templates/service.yaml](file:///Users/es/Desktop/thetaray/devops-test/templates/service.yaml#L9-L13)
```yaml
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: UDP
      name: http
```
* **Analysis:**
  HTTP is a TCP application protocol. The service specifies `protocol: UDP` while the pod container listens on TCP. Ingress controllers (like NGINX Ingress) ignore UDP endpoints for HTTP ingress paths or return `502 Bad Gateway`.

### Issue 5: Cluster Ingress Controller Inactive
* **Location:** [values.yaml](file:///Users/es/Desktop/thetaray/devops-test/values.yaml#L44-L45)
```yaml
ingress:
  enabled: true
  className: "nginx"
```
* **Analysis:**
  The Helm chart defines an Ingress rule utilizing `ingressClassName: "nginx"`. In a standard Minikube installation, the NGINX Ingress addon is disabled by default. Without an active ingress controller watching Ingress objects, the ingress will never receive an IP address and routing will not function.

---

## 3. Detailed Fixes & Code Modifications

### 1. Fix `values.yaml`
Update resource requests and limits to reasonable values for a lightweight microservice:

```yaml
resources:
  limits:
    cpu: 200m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 64Mi
```

### 2. Fix `templates/deployment.yaml`
1. Remove `args: [- -port, ...]`.
2. Inject the `PORT` environment variable with value `{{ .Values.service.port }}`.
3. Replace the `exec` liveness probe with an `httpGet` probe matching `readinessProbe`.

```yaml
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            - name: PORT
              value: "{{ .Values.service.port }}"
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### 3. Fix `templates/service.yaml`
Change `protocol: UDP` to `protocol: TCP`:

```yaml
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "http-echo.selectorLabels" . | nindent 4 }}
```

---

## 4. Verification & Testing Procedure

### Step 1: Enable Ingress in Minikube
```bash
minikube addons enable ingress
```
Verify the ingress controller pods are running:
```bash
kubectl get pods -n ingress-nginx
```

### Step 2: Deploy / Upgrade the Chart
```bash
helm upgrade --install http-echo ./devops-test
```

### Step 3: Verify Pods and Service
Ensure pod status is `Running` and `READY 1/1`:
```bash
kubectl get pods -l app.kubernetes.io/name=http-echo
kubectl get svc -l app.kubernetes.io/name=http-echo
kubectl get ingress -l app.kubernetes.io/name=http-echo
```

### Step 4: Configure DNS for `http-echo.duckdns.org`
Point `http-echo.duckdns.org` to your cluster IP using DuckDNS API:
```bash
curl "https://www.duckdns.org/update?domains=http-echo&token=6cc6ba38-ed18-4bde-9523-682f8b814235&ip=$(minikube ip)"
```

### Step 5: Verify Application Response
Send an HTTP request:
```bash
curl -i http://http-echo.duckdns.org/
```
Expected output:
HTTP 308 Permanent Redirect (or HTTP 200 OK with JSON request and environment dump).

### Step 6: Run Helm Connection Test
```bash
helm test http-echo
```

---

## 5. Production TLS / HTTPS Support: `cert-manager` + Let's Encrypt (DNS-01 Challenge)

In production, manual self-signed certificates are replaced with automated, Kubernetes-native PKI via `cert-manager` and Let's Encrypt using the **ACME DNS-01 challenge solver**.

### 1. Install `cert-manager`
```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

### 2. Install the DuckDNS Webhook for cert-manager
```bash
helm repo add mmontes https://mmontes11.github.io/charts
helm repo update mmontes
helm upgrade --install cert-manager-webhook-duckdns mmontes/cert-manager-webhook-duckdns \
  --namespace cert-manager \
  --set duckdns.token="6cc6ba38-ed18-4bde-9523-682f8b814235" \
  --set logLevel=2
```

### 3. Apply ACME `ClusterIssuer` (DNS-01)
Apply `cert-manager-issuer.yaml` (configured with `elad.suzy@hotmail.com`):
```bash
kubectl apply -f cert-manager-issuer.yaml
```

### 4. Ingress Configuration in `values.yaml`
`values.yaml` is configured with `cert-manager` annotations and the DuckDNS domain:
```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: http-echo.duckdns.org
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: http-echo-tls
      hosts:
        - http-echo.duckdns.org
```

### 5. Upgrade Helm Release and Test HTTPS
```bash
# Clean up any previous self-signed secret if one existed
kubectl delete secret http-echo-tls --ignore-not-found

# Deploy/Upgrade release
helm upgrade --install http-echo ./devops-test

# Verify Certificate issuance by cert-manager
kubectl get certificate http-echo-tls

# Test HTTPS with trusted Let's Encrypt certificate
curl -i https://http-echo.duckdns.org/
```
