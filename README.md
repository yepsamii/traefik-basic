
## Prerequisites

- Minikube installed and running
- kubectl configured
- Helm installed

Start cluster:

```bash
minikube start
kubectl get nodes
```

---

## Step 1 — Install Traefik (Helm)

Add repo:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

Install:

```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set service.type=NodePort \
  --set ports.web.nodePort=30000 \
  --set ports.websecure.nodePort=30443 \
  --set ingressRoute.dashboard.enabled=true
```

Verify:

```bash
kubectl get pods -n traefik
kubectl get svc -n traefik
```

---

## Step 2 — Configure Local Domain

Get Minikube IP:

```bash
minikube ip
```

Edit `/etc/hosts`:

```bash
sudo vim /etc/hosts
```

Add:

```text
<MINIKUBE_IP> app.local traefik.local
```

---

## Step 3 — Deploy UI Application

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami:v1.10.1
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f whoami-deployment.yaml
```

---

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  selector:
    app: whoami
  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f whoami-service.yaml
```

---

## Step 4 — Create TLS Secret

Generate certificate:

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=app.local/O=app.local"
```

Create secret:

```bash
kubectl create secret tls app-tls \
  --cert=tls.crt \
  --key=tls.key
```

---

## Step 5 — Configure IngressRoute

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: whoami
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.local`)
      kind: Rule
      services:
        - name: whoami
          port: 80
  tls:
    secretName: app-tls
```

Apply:

```bash
kubectl apply -f whoami-ingressroute.yaml
```

---

## Step 6 — Enable Dashboard

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`traefik.local`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
```

Apply:

```bash
kubectl apply -f traefik-dashboard.yaml
```

---

## Step 7 — Access Application
App:

```text
https://app.local
```

Dashboard:

```text
http://traefik.local
```

---

## Issue Faced
### Problem
- Could not access:
    - `https://app.local:30443`
    - `http://traefik.local:30000`
- Traefik logs showed no active errors
- Kubernetes resources were correctly configured

---

### Root Cause

Minikube was running using Docker driver on macOS.
In this setup:

```text
Host (Mac) → Docker VM → Kubernetes Cluster
```

NodePort services are exposed inside the Docker VM, not directly on the host network.
Even though:

- Service was correctly configured
- Ports (30000, 30443) were mapped
- `/etc/hosts` was correct

The host machine could not reach those ports directly.

---

### Verification

Direct test:

```bash
curl -k https://<MINIKUBE_IP>:30443
```

This failed, confirming network accessibility issue.

---

### Solution

Use LoadBalancer with tunnel:

```bash
minikube tunnel
```

Reinstall Traefik:

```bash
helm uninstall traefik -n traefik

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set service.type=LoadBalancer \
  --set ingressRoute.dashboard.enabled=true
```

Then:

```bash
kubectl get svc -n traefik
```

Use EXTERNAL-IP in `/etc/hosts`.

---