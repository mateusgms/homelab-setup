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
