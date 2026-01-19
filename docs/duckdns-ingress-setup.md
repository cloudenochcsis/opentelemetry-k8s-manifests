# DuckDNS Domain Setup for DigitalOcean Kubernetes Ingress

> **Last Updated**: January 2026  
> This guide covers setting up a free DuckDNS domain and configuring HTTP ingress for the OpenTelemetry Demo on DigitalOcean Kubernetes.

---

## Prerequisites

- DigitalOcean Kubernetes (DOKS) cluster with kubectl access
- Helm v3 installed
- OpenTelemetry Demo deployed to your cluster

---

## Step 1: Create a DuckDNS Account and Domain

1. Go to [DuckDNS](https://www.duckdns.org/)
2. Sign in with GitHub, Google, Twitter, or another provider
3. Create a new subdomain (e.g., `cloudenochoteldemo`)
4. Note your **token** from the DuckDNS dashboard - you'll need this later

After creation, you'll see:
```
success: domain cloudenochoteldemo.duckdns.org added to your account
```

---

## Step 2: Install NGINX Ingress Controller

The NGINX Ingress Controller handles routing external traffic to your services.

### Add the Helm repository

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### Install the controller

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.publishService.enabled=true
```

### Verify installation

```bash
# Check pods are running
kubectl get pods -n ingress-nginx

# Expected output:
# NAME                                        READY   STATUS    RESTARTS   AGE
# ingress-nginx-controller-xxxxx-xxxxx        1/1     Running   0          2m
```

---

## Step 3: Get the Load Balancer IP

When you install the NGINX Ingress Controller on DigitalOcean, it automatically provisions a Load Balancer.

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

**Output:**
```
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   10.116.54.36   137.184.242.139   80:30603/TCP,443:32550/TCP   78s
```

Copy the `EXTERNAL-IP` value (e.g., `137.184.242.139`).

> [!NOTE]
> If EXTERNAL-IP shows `<pending>`, wait 2-5 minutes for DigitalOcean to provision the Load Balancer.

---

## Step 4: Point DuckDNS to the Load Balancer

Update your DuckDNS domain to point to the Load Balancer IP.

### Option A: Using curl (command line)

```bash
curl "https://www.duckdns.org/update?domains=cloudenochoteldemo&token=YOUR_DUCKDNS_TOKEN&ip=137.184.242.139"
```

Replace:
- `cloudenochoteldemo` with your subdomain
- `YOUR_DUCKDNS_TOKEN` with your token from the DuckDNS dashboard
- `137.184.242.139` with your Load Balancer IP

**Expected response:**
```
OK
```

### Option B: Using the DuckDNS web interface

1. Go to [DuckDNS](https://www.duckdns.org/)
2. Find your domain
3. Enter the Load Balancer IP in the "current ip" field
4. Click "update ip"

### Verify DNS is working

```bash
dig +short cloudenochoteldemo.duckdns.org

# Expected output:
# 137.184.242.139
```

---

## Step 5: Update the Ingress Manifest

Edit the ingress file to use your DuckDNS domain.

### File: `k8s-manifests/ingress/otel-demo-ingress.yaml`

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
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/enable-opentelemetry: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: cloudenochoteldemo.duckdns.org  # Your DuckDNS domain
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

**Key fields explained:**

| Field | Description |
|-------|-------------|
| `ingressClassName: nginx` | Tells Kubernetes to use the NGINX Ingress Controller |
| `host: cloudenochoteldemo.duckdns.org` | Your DuckDNS domain |
| `path: /` | Route all paths to the backend |
| `pathType: Prefix` | Match any path starting with `/` |
| `service.name` | The Kubernetes service to route traffic to |
| `service.port.number` | The port the service listens on |

---

## Step 6: Apply the Ingress

```bash
kubectl apply -f k8s-manifests/ingress/otel-demo-ingress.yaml
```

**Output:**
```
ingress.networking.k8s.io/opentelemetry-demo-ingress created
```

### Verify the Ingress

```bash
kubectl get ingress
```

**Output:**
```
NAME                         CLASS   HOSTS                            ADDRESS           PORTS   AGE
opentelemetry-demo-ingress   nginx   cloudenochoteldemo.duckdns.org   137.184.242.139   80      17s
```

---

## Step 7: Test Access

### Test with curl

```bash
curl -I http://cloudenochoteldemo.duckdns.org
```

**Expected output:**
```
HTTP/1.1 200 OK
Date: Sat, 17 Jan 2026 21:18:44 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 76592
Connection: keep-alive
```

### Access in browser

Open your browser and navigate to:

**http://cloudenochoteldemo.duckdns.org**

You should see the OpenTelemetry Demo "Astronomy Shop" frontend.

---

## Summary

| Step | Action | Command/URL |
|------|--------|-------------|
| 1 | Create DuckDNS domain | https://www.duckdns.org |
| 2 | Install NGINX Ingress | `helm install ingress-nginx ...` |
| 3 | Get Load Balancer IP | `kubectl get svc -n ingress-nginx` |
| 4 | Update DuckDNS | `curl "https://www.duckdns.org/update?..."` |
| 5 | Update ingress manifest | Edit `otel-demo-ingress.yaml` |
| 6 | Apply ingress | `kubectl apply -f ...` |
| 7 | Test access | `curl -I http://your-domain.duckdns.org` |

---

## Troubleshooting

### DNS not resolving

```bash
# Check if DNS has propagated
dig +short cloudenochoteldemo.duckdns.org

# If empty, wait a few minutes and try again
```

### 502 Bad Gateway

```bash
# Check if the backend service exists
kubectl get svc opentelemetry-demo-frontendproxy

# Check if pods are running
kubectl get pods -l app.kubernetes.io/component=frontendproxy

# Check pod logs
kubectl logs -l app.kubernetes.io/component=frontendproxy
```

### Ingress not getting an address

```bash
# Check NGINX Ingress Controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Describe the ingress for events
kubectl describe ingress opentelemetry-demo-ingress
```

---

## Next Steps

- [Enable HTTPS with cert-manager](digitalocean-ingress-setup.md#step-4-optional-enable-tls-with-cert-manager) (requires a domain with reliable DNS)
- Add monitoring with Prometheus and Grafana
- Configure rate limiting on the ingress
