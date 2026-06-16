# ShopVerse - Kubernetes Configuration

This repository contains the Kubernetes deployment configuration for ShopVerse, a 3-tier e-commerce application built with Go (backend) and React (frontend).

## Repository Structure

```
├── argocd-app.yaml                    # ArgoCD Application manifest
└── helm/
    └── shopverse/
        ├── Chart.yaml                 # Helm chart definition
        ├── values.yaml                # Default values and configuration
        └── templates/
            ├── backend.yaml           # Backend Deployment, Service, HPA, PDB
            ├── frontend.yaml          # Frontend Deployment, Service, PDB
            ├── ingress.yaml           # ALB Ingress configuration
            ├── configmap.yaml         # Application configuration
            ├── external-secret.yaml   # ExternalSecret for AWS Secrets Manager
            ├── rbac.yaml              # Role-Based Access Control
            ├── namespace.yaml         # Namespace definition
            └── networkpolicy.yaml     # Network policy rules
```

## Prerequisites

- Kubernetes cluster (1.21+)
- Helm 3.x
- ArgoCD installed (for GitOps deployment)
- AWS Load Balancer Controller installed
- External Secrets Operator installed

## Components

| Component | Description |
|-----------|-------------|
| Backend   | Go-based API server (port 8080) |
| Frontend  | React-based web application (port 80) |
| ConfigMap | Non-sensitive configuration (DB host, ports, etc.) |
| ExternalSecret | Secrets synchronized from AWS Secrets Manager |

## Architecture

```
                    ┌─────────────────┐
                    │   ALB Ingress   │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                           │
              ▼                           ▼
    ┌───────────────────┐      ┌───────────────────┐
    │  Backend Service  │      │ Frontend Service  │
    │   (port 8080)     │      │   (port 80)       │
    └────────┬──────────┘      └───────────────────┘
             │
             ▼
    ┌───────────────────┐
    │  AWS RDS MySQL    │
    │   (db.t3.micro)   │
    └───────────────────┘
```

## Quick Start

### Deploy via ArgoCD

```bash
kubectl apply -f argocd-app.yaml
```

### Deploy via Helm

```bash
helm install shopverse ./helm/shopverse \
  --set global.namespace=shopverse \
  --set backend.image.tag=<your-backend-tag> \
  --set frontend.image.tag=<your-frontend-tag>
```

## Configuration

Edit `values.yaml` to customize:

- **Image tags** - Update `backend.image.tag` and `frontend.image.tag` for new versions
- **Replicas** - Adjust `backend.replicas` and `frontend.replicas` for scaling
- **Resources** - Modify CPU/memory requests and limits
- **Database** - Update connection details in `config` section
- **Ingress** - Configure ALB settings in `ingress` section

## Secret Management

Database credentials and JWT secret are stored in AWS Secrets Manager:

- `/shopverse/db-credentials` - Contains `password`
- `/shopverse/jwt-secret` - Contains `secret`

The ExternalSecret manifest syncs these to Kubernetes secrets automatically.

## Monitoring

Health endpoints:
- Backend: `/health` (port 8080)
- Frontend: `/` (port 80)

HorizontalPodAutoscalers are configured for both components with CPU and memory-based scaling.

## Security

- Non-root containers
- PodDisruptionBudgets for high availability
- NetworkPolicy template available for namespace isolation
- External secrets for sensitive data

## License

MIT