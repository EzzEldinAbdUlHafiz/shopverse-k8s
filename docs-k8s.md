# ShopVerse Kubernetes Documentation

## Helm

### What is Helm?

Helm is the package manager for Kubernetes. It allows you to package, configure, and deploy Kubernetes applications and services through Helm charts - reusable, versioned collections of Kubernetes resource manifests.

### What does Helm do?

- **Package Management**: Bundles Kubernetes manifests into charts that can be versioned and shared
- **Templating**: Uses Go templates to parameterize configurations with values from `values.yaml`
- **Release Management**: Tracks deployments, enabling rollbacks, upgrades, and uninstallation
- **Dependency Management**: Manages dependencies between multiple charts

### How is Helm better?

Helm simplifies Kubernetes deployments by:
- Eliminating manual manifest duplication across environments
- Enabling consistent, reproducible deployments through templating
- Providing atomic upgrades and rollbacks
- Reducing complexity through chart composition
- Supporting value overrides without modifying templates

---

## Helm Chart: shopverse

The `shopverse` chart is a complete application package for the ShopVerse 3-tier e-commerce application. It encapsulates all Kubernetes resources needed to run the application in a single, deployable unit.

**Chart metadata** (Chart.yaml):
- API version: `v2` (Helm 3)
- Application version: `1.0.0`
- Keywords: ecommerce, shopverse, golang, react

---

## File Explanations

### `argocd-app.yaml`

ArgoCD Application manifest that enables GitOps deployment. ArgoCD continuously monitors this repository and synchronizes the cluster state with the Helm chart.

**Key fields**:
- `repoURL`: Source Git repository containing Helm charts
- `targetRevision: main`: Deploys from main branch
- `path: helm/shopverse`: Points to the Helm chart location
- `syncPolicy.automated`: Enables automatic sync with prune and self-heal capabilities
- `CreateNamespace=true`: Ensures the namespace is created if missing

---

### `helm/shopverse/Chart.yaml`

Helm chart definition file. Defines the chart name, version, description, and metadata. Helm uses this file to identify the chart and its version compatibility.

---

### `helm/shopverse/values.yaml`

Default configuration values for the Helm chart. All template variables are populated from this file unless overridden at install/upgrade time.

**Sections**:
- `global`: Namespace, environment, and createNamespace flag
- `backend`: Image repository/tag, replica count, resource limits, HPA settings
- `frontend`: Image repository/tag, replica count, resource limits
- `config`: Non-sensitive configuration (DB host, ports, environment)
- `ingress`: ALB-specific ingress configuration
- `resourceQuota`: Optional namespace resource limits
- `networkPolicy`: Optional network isolation toggle

---

### `helm/shopverse/templates/backend.yaml`

Deploys the Go-based API backend with the following resources:

1. **Deployment** (lines 1-114):
   - Runs 2 backend replicas by default
   - Pulls image from ECR: `676206911950.dkr.ecr.us-east-1.amazonaws.com/shopverse-backend`
   - Runs as non-root user (UID/GID 1000) for security
   - Pod anti-affinity ensures pods spread across nodes
   - Probes: startup (10s), liveness (15s), readiness (10s) health checks against `/health` endpoint

2. **Service** (lines 116-132):
   - ClusterIP service exposing port 8080
   - Routes traffic to backend pods

3. **HorizontalPodAutoscaler** (lines 134-161):
   - Scales between 2-5 replicas based on CPU (70%) and memory (80%) utilization

4. **PodDisruptionBudget** (lines 163-176):
   - Ensures minimum 1 backend pod available during voluntary disruptions

---

### `helm/shopverse/templates/frontend.yaml`

Deploys the React-based web frontend with the following resources:

1. **Deployment** (lines 1-72):
   - Runs 2 frontend replicas by default
   - Pulls image from ECR: `676206911950.dkr.ecr.us-east-1.amazonaws.com/shopverse-frontend`
   - Runs as non-root user (GID 101)
   - Pod anti-affinity for high availability
   - Probes: startup (5s), liveness (10s), readiness (5s) health checks against `/`

2. **Service** (lines 74-90):
   - ClusterIP service exposing port 80
   - Routes traffic to frontend pods

3. **PodDisruptionBudget** (lines 92-105):
   - Ensures minimum 1 frontend pod available during disruptions

---

### `helm/shopverse/templates/ingress.yaml`

AWS Application Load Balancer (ALB) ingress configuration that routes external traffic to services.

**Routing rules**:
- `/api` and `/health` → Backend service (port 8080)
- `/` → Frontend service (port 80)

**ALB annotations**:
- `alb.ingress.kubernetes.io/scheme: internet-facing`: Creates public-facing load balancer
- `alb.ingress.kubernetes.io/target-type: ip`: Registers pods directly as targets
- `alb.ingress.kubernetes.io/healthcheck-path: /health`: Configures health check endpoint

---

### `helm/shopverse/templates/configmap.yaml`

ConfigMap storing non-sensitive application configuration. Injects environment variables into pods at runtime.

**Keys**:
- `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`: Database connection parameters
- `APP_ENV`, `APP_PORT`, `FRONTEND_PORT`: Application settings

---

### `helm/shopverse/templates/external-secret.yaml`

ExternalSecret manifest that syncs secrets from AWS Secrets Manager into Kubernetes secrets.

**Syncs**:
- `DB_PASSWORD` from `shopverse/db-credentials` → Kubernetes secret
- `DB_ROOT_PASSWORD` from same secret
- `JWT_SECRET` from `shopverse/jwt-secret` → Kubernetes secret

The External Secrets Operator reconciles these secrets hourly (`refreshInterval: 1h`) and creates the target secret `shopverse-secret`.

---

### `helm/shopverse/templates/rbac.yaml`

Role-Based Access Control definitions providing minimal required permissions:

1. **ServiceAccounts**:
   - `shopverse-backend-sa`: Identity for backend pods
   - `shopverse-frontend-sa`: Identity for frontend pods

2. **Roles**:
   - `shopverse-backend-role`: Allows read access to ConfigMaps and Secrets (needed for database/JWT credentials)
   - `shopverse-frontend-role`: Allows read access to ConfigMaps only

3. **RoleBindings**: Bind ServiceAccounts to their respective Roles

---

### `helm/shopverse/templates/namespace.yaml`

Conditional Namespace creation template. Only creates the namespace when `global.createNamespace` is set to `true` in values.yaml. Otherwise, expects the namespace to exist.

---

### `helm/shopverse/templates/networkpolicy.yaml`

Placeholder for NetworkPolicy configuration. Currently disabled (contains only a comment). NetworkPolicies would enforce namespace-level network isolation when enabled.

## Project Workflow

### GitOps Deployment Flow

```
Developer Push → Git Repository → ArgoCD Webhook → Sync → Helm Template → Kubernetes
```

1. Developer pushes changes to `helm/shopverse/` directory
2. ArgoCD detects changes via webhook/polling
3. ArgoCD syncs the Application, pointing to the Helm chart path
4. Helm renders templates with values.yaml
5. Kubernetes applies all manifests to the cluster

### Secret Management Flow

```
AWS Secrets Manager → External Secrets Operator → Kubernetes Secret → Pod Env Vars
```

1. Secrets stored in AWS Secrets Manager under `/shopverse/db-credentials` and `/shopverse/jwt-secret`
2. ExternalSecret manifest triggers External Secrets Operator
3. Operator fetches secrets and creates `shopverse-secret` Kubernetes Secret
4. Pods reference secret values through `env.valueFrom.secretKeyRef`

### Traffic Flow

```
Internet → ALB → Ingress → Service → Pod
```

1. Internet traffic hits AWS ALB (created by AWS Load Balancer Controller)
2. ALB routes to services based on Ingress rules
3. Frontend service (`/`) serves React application
4. Frontend makes API calls to `/api` paths
5. Backend service (`/api`, `/health`) handles Go API requests
6. Backend connects to RDS MySQL database via injected credentials

### Scaling Flow

```
HPA → Metrics Server → CPU/Memory → Scale Up/Down
```

1. HorizontalPodAutoscaler monitors pod metrics via Metrics Server
2. When CPU exceeds 70% or memory exceeds 80%, HPA triggers scale-up
3. New pods are scheduled respecting pod anti-affinity rules
4. Maximum 5 replicas for backend, maintaining high availability via PDB

## Architecture Summary

```
┌─────────────────┐
│   ALB Ingress   │
└────────┬────────┘
         │
┌────────┴────────┬───────────────┐
│                 │               │
▼                 ▼               ▼
Frontend    Backend (/api)   Backend (/health)
Service     Service         Service
(ClusterIP) (ClusterIP)     (ClusterIP)
▲                 ▲               ▲
│                 │               │
Pod               Pod             Pod
(replicas: 2)     (replicas: 2-5) (replicas: 2-5)
PDB              PDB
▲
│
ConfigMap (non-sensitive config)
ExternalSecret → Kubernetes Secret
(sensitive: DB_PASSWORD, JWT_SECRET)
```