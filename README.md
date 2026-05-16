# DevSecOps Pipeline

A production-grade DevSecOps pipeline that automates building, securing, deploying, and monitoring a containerized Go microservice — with security scanning at every stage to block vulnerable code before it reaches production.

## Architecture

Code Push → GitHub Actions → Security Scanning → Docker Build → GHCR → Kubernetes → Monitoring

## Tech Stack

| Category | Tools |
|----------|-------|
| Application | Go |
| Containerization | Docker (multi-stage build) |
| Registry | GitHub Container Registry (GHCR) |
| CI/CD | GitHub Actions |
| Security Scanning | Trivy, Checkov, tfsec |
| Deployment | Kubernetes (Minikube) |
| Config Management | ConfigMaps, Secrets |
| Monitoring | Prometheus, Grafana |

## Features

- **Multi-stage Docker build** — final image size ~10MB vs 200MB+ baseline
- **3-layer security scanning** — Docker images (Trivy), Kubernetes manifests (Checkov), Terraform code (tfsec)
- **Automated pipeline** — triggers on every push to main
- **SHA-based image tagging** — every image traceable to exact commit
- **Zero downtime deployments** — rolling update strategy with `maxUnavailable: 0`
- **One-command rollback** — `kubectl rollout undo` to instantly revert bad deployments
- **Security hardened pods** — non-root user, read-only filesystem, dropped capabilities, seccomp profile
- **Auto-generated scan reports** — Trivy report saved as pipeline artifact on every run
- **Real-time monitoring** — Prometheus scraping custom app metrics, visualized in Grafana

## Project Structure
devsecops-pipeline/
├── app/
│   ├── main.go          # Go REST API with Prometheus metrics
│   ├── go.mod
│   ├── go.sum
│   └── Dockerfile       # Multi-stage build
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml  # Rolling update + security context
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── servicemonitor.yaml
├── terraform/
│   └── main.tf
└── .github/
└── workflows/
└── build.yml    # CI/CD pipeline

## API Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Returns service health status |
| `GET /version` | Returns application version |
| `GET /metrics` | Prometheus metrics endpoint |

## Pipeline Stages
1. Checkout code
2. Build Docker image (multi-stage, ~10MB)
3. Trivy scan — Docker image vulnerability scan
4. Checkov scan — Kubernetes manifest security scan
5. tfsec scan — Terraform code security scan
6. Push to GHCR with SHA + latest tags
7. Upload scan report as artifact

## Getting Started

### Prerequisites
- Docker
- Minikube
- kubectl
- Helm

### Run Locally

```bash
# Clone the repo
git clone https://github.com/dot05/devsecops-pipeline.git
cd devsecops-pipeline

# Run the app
cd app
go mod tidy
go run main.go

# Test endpoints
curl http://localhost:8080/health
curl http://localhost:8080/version
curl http://localhost:8080/metrics
```

### Deploy to Kubernetes

```bash
# Start Minikube
minikube start

# Apply manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/servicemonitor.yaml

# Get service URL
minikube service devsecops-service -n devsecops --url
```

### Set Up Monitoring

```bash
# Install Prometheus + Grafana
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123

# Access Grafana
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
# Open http://localhost:3000 (admin/admin123)

# Access Prometheus
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
# Open http://localhost:9090
```

### Rolling Update + Rollback

```bash
# Trigger rolling update
kubectl set image deployment/devsecops-app \
  devsecops-app=ghcr.io/dot05/devsecops-pipeline:sha-0e595d5 \
  -n devsecops

# Watch rollout
kubectl rollout status deployment/devsecops-app -n devsecops

# Rollback if needed
kubectl rollout undo deployment/devsecops-app -n devsecops
```

## Security

- Container runs as non-root user (UID 1000)
- Read-only root filesystem
- All Linux capabilities dropped
- Seccomp profile set to RuntimeDefault
- Privilege escalation disabled
- Service account token automounting disabled
- Secrets mounted as environment variables via K8s Secrets

## Key Decisions & Learnings

- **Go over Python** — produces a statically compiled binary, enabling a scratch-based Docker image (~10MB vs 200MB+)
- **Multi-stage build** — separates build environment from runtime, drastically reducing attack surface
- **`maxUnavailable: 0`** — ensures zero downtime during rolling updates by never terminating a pod before its replacement is ready
- **Liveness vs Readiness probes** — liveness restarts unhealthy pods, readiness prevents traffic from reaching pods that aren't ready yet
- **Checkov findings** — scanner identified 10 real security misconfigurations in K8s manifests which were subsequently fixed
- **Base64 ≠ encryption** — Kubernetes Secrets are base64 encoded, not encrypted; production systems should use HashiCorp Vault or cloud-native secret managers
