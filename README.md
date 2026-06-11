# test-k8s

# Kubernetes Manifests — DevSecOps Project

GitOps repository containing Helm charts and Kubernetes 
manifests for automated deployments to GKE via ArgoCD.

> This repository is automatically watched by ArgoCD.
> Any change to `helm/values.yaml` triggers an automatic
> redeployment to the GKE cluster.

## How it Works

```
Jenkins CI pipeline (in test-repo) completes build
              ↓
Automatically updates image tag in helm/values.yaml
              ↓
ArgoCD detects the change in this repo
              ↓
Pulls Helm chart and deploys updated app to GKE
              ↓
App deployed and accessible via GKE LoadBalancer
with SSL/TLS enabled through cert-manager and
NGINX Ingress Controller
```

## Repository Structure

```
test-k8s/
    ├── helm/
    │     ├── Chart.yaml              → Helm chart metadata
    │     ├── values.yaml             → App configuration (auto-updated by Jenkins)
    │     └── templates/
    │               ├── deployment.yaml   → Kubernetes Deployment definition
    │               ├── service.yaml      → Kubernetes Service definition
    │               └── ingress.yaml      → NGINX Ingress with SSL configuration
    │
    └── manifests/
              └── clusterissuer.yaml  → cert-manager ClusterIssuer for SSL
```

## Helm Chart

### values.yaml
Contains all configurable values for the deployment:

```yaml
replicaCount: 2
image: us-central1-docker.pkg.dev/PROJECT_ID/docker-repo/hello:latest
service:
  type: LoadBalancer
  port: 80
```

The `image` field is automatically updated by Jenkins CI pipeline 
on every successful build with the new version tag.

### Templates

**deployment.yaml**
- Defines how many pod replicas to run
- Specifies which Docker image to pull from GCP Artifact Registry
- Configures container port and resource limits

**service.yaml**
- Exposes the application pods via a stable internal address
- Uses label selectors to route traffic to correct pods
- Configured as LoadBalancer for external access

**ingress.yaml**
- Routes external traffic to the application service
- Configured with NGINX Ingress Controller
- SSL/TLS termination via cert-manager and Let's Encrypt

## SSL Configuration

SSL certificates are automatically managed by cert-manager:

```
ClusterIssuer (manifests/clusterissuer.yaml)
    → Configured with Let's Encrypt production server
    → Uses HTTP01 challenge via NGINX Ingress
    → Auto-renews certificates before expiry
```

Apply ClusterIssuer manually before first deployment:
```bash
kubectl apply -f manifests/clusterissuer.yaml
```

## ArgoCD Setup

Application configured in ArgoCD with:

```
Repository URL  : https://github.com/theengineer-pb/test-k8s
Branch          : main
Path            : helm
Sync Policy     : Automatic
Cluster URL     : https://kubernetes.default.svc
Namespace       : default
Values File     : values.yaml
```

## Prerequisites

- GKE cluster running (provisioned via test-repo Terraform)
- ArgoCD installed on GKE cluster
- cert-manager installed on GKE cluster
- NGINX Ingress Controller installed on GKE cluster
- GoDaddy domain DNS pointing to NGINX Ingress public IP

## Deployment Order

```
1. kubectl apply -f manifests/clusterissuer.yaml
2. Configure ArgoCD application pointing to this repo
3. ArgoCD automatically deploys helm/templates/
4. cert-manager issues SSL certificate automatically
5. App accessible via LoadBalancer external IP
   with HTTPS enabled through cert-manager
```

## Related Repository

CI pipeline and infrastructure code:
**[test-repo](https://github.com/theengineer-pb/test-repo)**
