# 🏠 Kubernetes Homelab DevOps Stack

Complete Kubernetes homelab with Grafana, Prometheus, Loki, Tempo, and ArgoCD - publicly accessible via Cloudflare Tunnel.

## 🎯 Overview

Production-ready DevOps platform for homelabs:
- **MicroK8s** on ARM64 (Raspberry Pi compatible)
- **Observability**: Grafana, Prometheus, Loki, Tempo
- **GitOps**: ArgoCD
- **Public Access**: Cloudflare Tunnel (no port forwarding!)
- **Auto SSL**: via Cloudflare

## 🏗️ Architecture

```
Internet → Cloudflare (SSL) → Tunnel → NGINX Ingress → K8s Services
```

## 💻 Hardware

- 4x Raspberry Pi (1 master + 3 workers)
- 4GB RAM minimum per node
- Ubuntu 22.04 ARM64

## 🚀 Quick Start

### 1. Install MicroK8s

```bash
sudo snap install microk8s --classic --channel=1.32/stable
sudo usermod -a -G microk8s $USER
newgrp microk8s
microk8s enable dns helm helm3 storage observability ingress cert-manager
```

### 2. Install ArgoCD

```bash
microk8s kubectl create namespace argocd
microk8s helm repo add argo https://argoproj.github.io/argo-helm
microk8s helm install argocd argo/argo-cd --namespace argocd
```

### 3. Setup Cloudflare Tunnel

```bash
# Install
sudo apt install cloudflared -y

# Authenticate
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create homelab

# Configure (see configs/)
sudo cloudflared service install
sudo systemctl start cloudflared
```

### 4. Configure Cloudflare

1. Dashboard → SSL/TLS → **"Flexible" mode**
2. Create DNS records with `cloudflared tunnel route dns`

⚠️ **Important:** Use first-level subdomains: `grafana-homelab.domain.com` 
NOT `grafana.homelab.domain.com` (wildcard SSL limitation!)

## 🔑 Access Services

- Grafana: `https://grafana-homelab.yourdomain.com`
- ArgoCD: `https://argocd-homelab.yourdomain.com`
- Prometheus: `https://prometheus-homelab.yourdomain.com`

Get passwords:
```bash
# ArgoCD
microk8s kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Grafana  
microk8s kubectl get secret -n observability kube-prom-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

## 🐛 Troubleshooting

### SSL Errors

**Problem:** ERR_SSL_VERSION_OR_CIPHER_MISMATCH

**Solution:**
1. Cloudflare SSL mode = "Flexible"
2. Use first-level subdomains (`service-name.domain.com`)
3. Purge Cloudflare cache

### Tunnel Issues

```bash
sudo systemctl status cloudflared
sudo journalctl -u cloudflared -f
```

## 📁 Repository Structure

```
homelab-setup/
├── manifests/        # Kubernetes manifests
├── configs/          # Configuration files
├── scripts/          # Helper scripts
└── docs/            # Detailed documentation
```

## 📚 Documentation

- [Detailed Setup Guide](docs/SETUP.md)
- [Troubleshooting Guide](docs/TROUBLESHOOTING.md)
- [Security Best Practices](docs/SECURITY.md)

## 📝 License

MIT License

## 📖 Detailed Documentation

### Deployment Guides

- **[Deploying Applications](docs/DEPLOYING-APPS.md)** - Complete guide to deploying apps (kubectl, ArgoCD, GitOps)
- **[CI/CD Pipeline](docs/CICD-PIPELINE.md)** - GitHub Actions workflows and automation
- **[Helm Deployments](docs/HELM-DEPLOYMENTS.md)** - Using Helm charts in your application repos

### Key Topics Covered

#### Application Deployment
- Manual deployment with kubectl
- GitOps with ArgoCD
- Helm chart management
- Multi-environment strategies

#### CI/CD Automation
- GitHub Actions workflows
- Docker image building
- Automated testing
- ArgoCD integration
- Multi-environment pipelines

#### Helm Best Practices
- Chart structure in app repos
- Values management per environment
- CI/CD with Helm
- ArgoCD + Helm integration
- Secrets management

## 🚢 Deployment Workflow Summary

### Option 1: Manual with kubectl
```bash
kubectl apply -f manifests/
```

### Option 2: GitOps with ArgoCD
1. Push manifests to Git
2. Create ArgoCD Application
3. ArgoCD auto-syncs changes

### Option 3: Helm + ArgoCD (Recommended)
1. Store Helm chart in app repo (`./helm/`)
2. GitHub Actions builds image & updates values
3. ArgoCD syncs from Helm chart
4. Automatic deployment per environment

## 🔄 Complete Deployment Flow

```
1. Developer pushes code
   ↓
2. GitHub Actions triggers
   ↓
3. Run tests & build Docker image
   ↓
4. Push image to registry (GHCR)
   ↓
5. Update Helm values or manifests
   ↓
6. ArgoCD detects changes
   ↓
7. ArgoCD syncs to Kubernetes
   ↓
8. Application deployed! 🎉
```

## 🎯 Quick Start Deployment

### Deploy Your First App

```bash
# 1. Clone template
git clone https://github.com/yourusername/my-app.git
cd my-app

# 2. Add Helm chart
cp -r /path/to/homelab-setup/manifests/examples/helm-example ./helm

# 3. Customize values
vim helm/values-production.yaml

# 4. Create ArgoCD application
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/my-app
    path: helm
    targetRevision: main
    helm:
      valueFiles:
        - values.yaml
        - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# 5. Watch deployment
argocd app get my-app --watch
```

