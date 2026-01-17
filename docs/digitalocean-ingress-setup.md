# DigitalOcean Kubernetes (DOKS) Ingress Controller Setup Guide

> **Last Updated**: January 2026  
> **Target Application**: OpenTelemetry Demo v1.12.0

This guide covers setting up NGINX Ingress Controller with optional TLS/SSL via cert-manager on DigitalOcean Kubernetes for the OpenTelemetry Demo application.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Step 1: Install NGINX Ingress Controller](#step-1-install-nginx-ingress-controller)
4. [Step 2: Configure DNS](#step-2-configure-dns)
5. [Step 3: Create Ingress for OpenTelemetry Demo](#step-3-create-ingress-for-opentelemetry-demo)
6. [Step 4: (Optional) Enable TLS with cert-manager](#step-4-optional-enable-tls-with-cert-manager)
7. [Step 5: Verification](#step-5-verification)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before you begin, ensure you have:

| Requirement | Description |
|-------------|-------------|
| **DOKS Cluster** | A running DigitalOcean Kubernetes cluster |
| **kubectl** | Installed and configured to connect to your cluster |
| **Helm v3** | Installed for package management |
| **doctl** | DigitalOcean CLI (optional but recommended) |
| **Domain Name** | A registered domain for configuring DNS records |
| **OpenTelemetry Demo** | Already deployed to your cluster |

### Verify Cluster Access

```bash
# Verify kubectl is connected to your DOKS cluster
kubectl cluster-info

# Check nodes are ready
kubectl get nodes
```

---

## Architecture Overview

For the complete OpenTelemetry Demo architecture, see the official documentation:

🔗 **[OpenTelemetry Demo Architecture](https://opentelemetry.io/docs/demo/architecture/)**

**Traffic Flow Summary:**
1. Users → DigitalOcean Load Balancer → NGINX Ingress Controller
2. Ingress Controller → `frontendproxy` service (port 8080)
3. Frontend Proxy → Backend microservices (cart, checkout, payment, etc.)

---

## Step 1: Install NGINX Ingress Controller

### Option A: Using DigitalOcean 1-Click App (Recommended)

The easiest method is using DigitalOcean's Marketplace:

1. Log in to the [DigitalOcean Control Panel](https://cloud.digitalocean.com)
2. Navigate to **Kubernetes** → Select your cluster
3. Go to the **Marketplace** tab
4. Find **NGINX Ingress Controller** and click **Install**

This automatically:
- Deploys a pre-configured Helm chart
- Provisions a DigitalOcean Load Balancer (billed separately)

### Option B: Manual Installation via Helm

```bash
# Add the ingress-nginx Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Create namespace
kubectl create namespace ingress-nginx

# Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.publishService.enabled=true
```

### Verify Installation

```bash
# Check pods are running
kubectl get pods -n ingress-nginx

# Expected output:
# NAME                                        READY   STATUS    RESTARTS   AGE
# ingress-nginx-controller-xxxxx-xxxxx        1/1     Running   0          2m

# Check the LoadBalancer service and get external IP
kubectl get svc -n ingress-nginx

# Wait for EXTERNAL-IP to be assigned (may take 2-5 minutes)
kubectl get svc ingress-nginx-controller -n ingress-nginx -w
```

> [!NOTE]
> The EXTERNAL-IP will be a DigitalOcean Load Balancer IP. This Load Balancer will be billed at the standard rate (~$12/month for basic).

---

## Step 2: Configure DNS

Once the Load Balancer has an external IP, configure your DNS:

### Get the External IP

```bash
export LB_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Load Balancer IP: $LB_IP"
```

### Configure DNS Records

Create an **A record** in your DNS provider pointing to the Load Balancer IP:

| Record Type | Host | Value | TTL |
|-------------|------|-------|-----|
| A | `otel-demo` | `<LOAD_BALANCER_IP>` | 300 |
| A | `@` or `*` | `<LOAD_BALANCER_IP>` | 300 |

**Example using DigitalOcean DNS:**

```bash
# Using doctl (if your domain is managed by DigitalOcean)
doctl compute domain records create yourdomain.com \
  --record-type A \
  --record-name otel-demo \
  --record-data $LB_IP \
  --record-ttl 300
```

---

## Step 3: Create Ingress for OpenTelemetry Demo

Create the Ingress resource to route traffic to the OpenTelemetry Demo frontend proxy.

### Create Ingress Manifest

Create a new file at `k8s-manifests/ingress/otel-demo-ingress.yaml`:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: opentelemetry-demo-ingress
  labels:
    opentelemetry.io/name: opentelemetry-demo
    app.kubernetes.io/instance: opentelemetry-demo
    app.kubernetes.io/part-of: opentelemetry-demo
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    # Enable OpenTelemetry tracing on ingress
    nginx.ingress.kubernetes.io/enable-opentelemetry: "true"
spec:
  ingressClassName: nginx
  rules:
    # Main application
    - host: otel-demo.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: opentelemetry-demo-frontendproxy
                port:
                  number: 8080
```

### Apply the Ingress

```bash
# Create the ingress directory if it doesn't exist
mkdir -p k8s-manifests/ingress

# Apply the ingress manifest
kubectl apply -f k8s-manifests/ingress/otel-demo-ingress.yaml

# Verify the ingress was created
kubectl get ingress

# Check ingress details
kubectl describe ingress opentelemetry-demo-ingress
```

---

## Step 4: (Optional) Enable TLS with cert-manager

For production environments, enable HTTPS using cert-manager and Let's Encrypt.

### 4.1 Install cert-manager

```bash
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager with CRDs
kubectl create namespace cert-manager

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.0 \
  --set installCRDs=true

# Verify installation
kubectl get pods -n cert-manager
```

### 4.2 Create ClusterIssuer for Let's Encrypt

Create `k8s-manifests/ingress/cluster-issuer.yaml`:

```yaml
---
# Staging issuer (for testing - avoids rate limits)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your-email@yourdomain.com  # CHANGE THIS
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
      - http01:
          ingress:
            class: nginx
---
# Production issuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@yourdomain.com  # CHANGE THIS
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

```bash
# Apply the ClusterIssuers
kubectl apply -f k8s-manifests/ingress/cluster-issuer.yaml

# Verify ClusterIssuers are ready
kubectl get clusterissuers
```

### 4.3 Update Ingress with TLS

Update `k8s-manifests/ingress/otel-demo-ingress.yaml` to include TLS:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: opentelemetry-demo-ingress
  labels:
    opentelemetry.io/name: opentelemetry-demo
    app.kubernetes.io/instance: opentelemetry-demo
    app.kubernetes.io/part-of: opentelemetry-demo
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/enable-opentelemetry: "true"
    # cert-manager annotations
    cert-manager.io/cluster-issuer: letsencrypt-prod  # Use 'letsencrypt-staging' for testing
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - otel-demo.yourdomain.com
      secretName: otel-demo-tls
  rules:
    - host: otel-demo.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: opentelemetry-demo-frontendproxy
                port:
                  number: 8080
```

```bash
# Apply the updated ingress
kubectl apply -f k8s-manifests/ingress/otel-demo-ingress.yaml

# Check certificate status
kubectl get certificates
kubectl describe certificate otel-demo-tls
```

> [!TIP]
> Start with `letsencrypt-staging` for testing to avoid Let's Encrypt rate limits. Switch to `letsencrypt-prod` once everything works.

---

## Step 5: Verification

### Test HTTP Access

```bash
# Test with curl (replace with your domain)
curl -I http://otel-demo.yourdomain.com

# Expected: HTTP 200 response
```

### Test HTTPS Access (if TLS enabled)

```bash
# Test HTTPS
curl -I https://otel-demo.yourdomain.com

# Check certificate
echo | openssl s_client -servername otel-demo.yourdomain.com -connect otel-demo.yourdomain.com:443 2>/dev/null | openssl x509 -noout -issuer -dates
```

### Access the Application

Open your browser and navigate to:
- **HTTP**: `http://otel-demo.yourdomain.com`
- **HTTPS**: `https://otel-demo.yourdomain.com` (if TLS enabled)

You should see the OpenTelemetry Demo "Astronomy Shop" frontend.

---

## Troubleshooting

### Load Balancer IP Not Assigned

```bash
# Check if the ingress controller service is pending
kubectl get svc -n ingress-nginx

# If EXTERNAL-IP shows <pending>, check events
kubectl describe svc ingress-nginx-controller -n ingress-nginx
```

**Common causes:**
- DigitalOcean account quota limits
- Billing issues with DigitalOcean account

### 502 Bad Gateway

```bash
# Check if backend pods are running
kubectl get pods | grep frontendproxy

# Check pod logs
kubectl logs -l app.kubernetes.io/component=frontendproxy

# Verify the service exists and has endpoints
kubectl get endpoints opentelemetry-demo-frontendproxy
```

### Certificate Not Issuing

```bash
# Check certificate status
kubectl describe certificate otel-demo-tls

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager

# Check challenges
kubectl get challenges
kubectl describe challenge <challenge-name>
```

**Common causes:**
- DNS not propagated yet (wait 5-10 minutes)
- Firewall blocking HTTP-01 challenge
- Incorrect email in ClusterIssuer

### Ingress Not Working

```bash
# Check ingress status
kubectl describe ingress opentelemetry-demo-ingress

# Check NGINX controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

---

## Quick Reference: Useful Commands

```bash
# Get Load Balancer IP
kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Watch ingress status
kubectl get ingress -w

# Check all certificates
kubectl get certificates -A

# View NGINX configuration
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- cat /etc/nginx/nginx.conf

# Test connectivity from inside cluster
kubectl run curl --image=curlimages/curl -it --rm -- curl http://opentelemetry-demo-frontendproxy:8080
```

---

## Additional Resources

- [NGINX Ingress Controller Documentation](https://kubernetes.github.io/ingress-nginx/)
- [DigitalOcean Kubernetes Documentation](https://docs.digitalocean.com/products/kubernetes/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [OpenTelemetry Demo Documentation](https://opentelemetry.io/docs/demo/)
- [DigitalOcean Load Balancer Pricing](https://www.digitalocean.com/pricing/load-balancer)

---

## Next Steps

After completing this setup, you may want to:

1. **Enable monitoring** - Configure Prometheus and Grafana for observability
2. **Set up CI/CD** - Automate deployments with ArgoCD or GitHub Actions
3. **Configure rate limiting** - Add NGINX rate limiting annotations
4. **Enable WAF** - Add ModSecurity or similar for web application firewall
