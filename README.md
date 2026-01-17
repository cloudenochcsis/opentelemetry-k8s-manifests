# OpenTelemetry Demo - Kubernetes Manifests

Kubernetes manifests for deploying the [OpenTelemetry Demo](https://opentelemetry.io/docs/demo/) application on DigitalOcean Kubernetes (DOKS).

## About

This repository contains Kubernetes manifests extracted from the official OpenTelemetry Demo Helm chart, along with additional configurations for:

- **NGINX Ingress Controller** setup for DigitalOcean
- **TLS/SSL certificates** with cert-manager and Let's Encrypt
- **Production-ready ingress** configurations

## Quick Start

```bash
# Deploy the OpenTelemetry Demo
kubectl apply -f k8s-manifests/complete-deploy.yaml

# Set up ingress (update domain first)
kubectl apply -f k8s-manifests/ingress/otel-demo-ingress.yaml
```

## Documentation

- [DigitalOcean Ingress Setup Guide](docs/digitalocean-ingress-setup.md)

## Project Structure

```
├── docs/
│   └── digitalocean-ingress-setup.md    # Detailed ingress setup guide
├── k8s-manifests/
│   ├── complete-deploy.yaml             # All-in-one deployment manifest
│   ├── serviceaccount.yaml              # Service account for the demo
│   ├── ingress/                         # Ingress configurations
│   │   ├── otel-demo-ingress.yaml       # Basic HTTP ingress
│   │   ├── otel-demo-ingress-tls.yaml   # HTTPS with TLS
│   │   └── cluster-issuer.yaml          # Let's Encrypt issuers
│   └── [service-folders]/               # Individual service manifests
└── README.md
```

## Services Included

The OpenTelemetry Demo includes the following microservices:

| Service | Description |
|---------|-------------|
| Frontend | Web UI for the "Astronomy Shop" |
| Cart | Shopping cart service |
| Checkout | Order processing |
| Payment | Payment processing |
| Product Catalog | Product information |
| Recommendation | Product recommendations |
| Shipping | Shipping calculations |
| Currency | Currency conversion |
| Email | Email notifications |
| Ad | Advertisement service |
| Fraud Detection | Fraud analysis |
| Kafka | Message broker |
| Valkey | In-memory data store |

## Credits & Attribution

This project is based on the official **[OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo)** by the [OpenTelemetry](https://opentelemetry.io/) community.

- **Original Repository**: [open-telemetry/opentelemetry-demo](https://github.com/open-telemetry/opentelemetry-demo)
- **Documentation**: [OpenTelemetry Demo Docs](https://opentelemetry.io/docs/demo/)
- **License**: Apache License 2.0

> The OpenTelemetry Demo is a microservice-based distributed system intended to illustrate the implementation of OpenTelemetry in a near real-world environment.

## License

This repository follows the same [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0) as the original OpenTelemetry Demo project.

## Resources

- [OpenTelemetry](https://opentelemetry.io/)
- [OpenTelemetry Demo Architecture](https://opentelemetry.io/docs/demo/architecture/)
- [DigitalOcean Kubernetes Documentation](https://docs.digitalocean.com/products/kubernetes/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
