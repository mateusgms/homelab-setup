# 🚀 Deploying Applications Guide

Complete guide to deploying applications on your homelab Kubernetes cluster.

## 📋 Table of Contents

1. [Manual Deployment with kubectl](#manual-deployment)
2. [GitOps Deployment with ArgoCD](#gitops-deployment)
3. [Exposing Apps Publicly](#exposing-apps-publicly)
4. [Complete Deployment Workflow](#complete-workflow)
5. [Real-World Examples](#examples)

---

## 🔧 Manual Deployment with kubectl

### Basic Application Deployment

Create a simple application:

```bash
# Create namespace
microk8s kubectl create namespace my-app

# Create deployment
microk8s kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
EOF

# Create service
microk8s kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: my-app
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
EOF

# Verify deployment
microk8s kubectl get pods -n my-app
```

---

## 🔄 GitOps Deployment with ArgoCD

### Step 1: Create Git Repository

Create a repository structure:

```
my-app-manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml
│   └── prod/
│       └── kustomization.yaml
└── README.md
```

### Step 2: Create Application Manifests

**base/deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0.0
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        env:
        - name: ENVIRONMENT
          value: production
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

**base/service.yaml:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

**base/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  app.kubernetes.io/name: my-app
  app.kubernetes.io/managed-by: argocd
```

### Step 3: Connect ArgoCD to Git Repository

#### Option A: Via ArgoCD UI

1. Login to ArgoCD: `https://argocd-homelab.yourdomain.com`
2. Click **"Settings"** → **"Repositories"**
3. Click **"Connect Repo"**
4. Enter repository details:
   - Repository URL: `https://github.com/yourusername/my-app-manifests`
   - For private repos, add credentials
5. Click **"Connect"**

#### Option B: Via CLI

```bash
# Install ArgoCD CLI
curl -sSL -o argocd-linux-arm64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-arm64
chmod +x argocd-linux-arm64
sudo mv argocd-linux-arm64 /usr/local/bin/argocd

# Login
argocd login argocd-homelab.yourdomain.com --username admin

# Add repository
argocd repo add https://github.com/yourusername/my-app-manifests \
  --name my-app-repo
```

### Step 4: Create ArgoCD Application

#### Option A: Via UI

1. Click **"Applications"** → **"+ New App"**
2. Fill in details:
   - **Application Name**: `my-app`
   - **Project**: `default`
   - **Sync Policy**: `Automatic`
   - **Repository URL**: `https://github.com/yourusername/my-app-manifests`
   - **Revision**: `HEAD` or specific branch
   - **Path**: `base` or `overlays/prod`
   - **Cluster**: `https://kubernetes.default.svc`
   - **Namespace**: `my-app`
3. Check **"Auto-Create Namespace"**
4. Click **"Create"**

#### Option B: Via CLI

```bash
argocd app create my-app \
  --repo https://github.com/yourusername/my-app-manifests \
  --path base \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace my-app \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

#### Option C: Via Kubernetes Manifest

```bash
microk8s kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/my-app-manifests
    targetRevision: HEAD
    path: base
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

### Step 5: Monitor Deployment

```bash
# Watch ArgoCD sync
argocd app get my-app --watch

# Check application status
argocd app list

# View sync history
argocd app history my-app

# Check pods
microk8s kubectl get pods -n my-app
```

---

## 🌐 Exposing Apps Publicly

### Step 1: Create Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp-homelab.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

Apply ingress:

```bash
microk8s kubectl apply -f ingress.yaml
```

### Step 2: Update Cloudflare Tunnel Config

Edit `/etc/cloudflared/config.yml`:

```yaml
tunnel: YOUR_TUNNEL_ID
credentials-file: /etc/cloudflared/YOUR_TUNNEL_ID.json

ingress:
  # Existing services...
  - hostname: grafana-homelab.yourdomain.com
    service: http://MASTER_IP:80
    originRequest:
      httpHostHeader: grafana.homelab.yourdomain.com
  
  # Add your new app
  - hostname: myapp-homelab.yourdomain.com
    service: http://MASTER_IP:80
    originRequest:
      httpHostHeader: myapp-homelab.yourdomain.com
  
  # Catch-all
  - service: http_status:404
```

Restart tunnel:

```bash
sudo systemctl restart cloudflared
sudo systemctl status cloudflared
```

### Step 3: Create DNS Record

```bash
cloudflared tunnel route dns YOUR_TUNNEL_NAME myapp-homelab.yourdomain.com
```

### Step 4: Test Access

Wait 1-2 minutes, then access:

```bash
curl https://myapp-homelab.yourdomain.com
```

Or open in browser!

---

## 🔄 Complete Workflow

### Workflow: Deploy New Application with GitOps

```bash
# 1. Create manifests repository
mkdir -p my-app-manifests/base
cd my-app-manifests

# 2. Create manifests (see examples above)
cat > base/deployment.yaml << EOF
# ... deployment yaml ...
EOF

cat > base/service.yaml << EOF
# ... service yaml ...
EOF

cat > base/ingress.yaml << EOF
# ... ingress yaml ...
EOF

# 3. Initialize git
git init
git add .
git commit -m "Initial commit: My App manifests"
git remote add origin git@github.com:yourusername/my-app-manifests.git
git push -u origin master

# 4. Create ArgoCD application
argocd app create my-app \
  --repo https://github.com/yourusername/my-app-manifests \
  --path base \
  --dest-namespace my-app \
  --sync-policy automated

# 5. Update Cloudflare Tunnel
sudo nano /etc/cloudflared/config.yml
# Add hostname entry

sudo systemctl restart cloudflared

# 6. Create DNS record
cloudflared tunnel route dns YOUR_TUNNEL_NAME myapp-homelab.yourdomain.com

# 7. Verify deployment
argocd app get my-app
microk8s kubectl get pods -n my-app
curl https://myapp-homelab.yourdomain.com
```

### Workflow: Update Existing Application

```bash
# 1. Update manifests in Git
cd my-app-manifests
# Edit deployment.yaml (e.g., change image tag)
git add .
git commit -m "Update: Bump version to v1.1.0"
git push

# 2. ArgoCD auto-syncs (if enabled)
# Or manually sync:
argocd app sync my-app

# 3. Monitor rollout
microk8s kubectl rollout status deployment/my-app -n my-app

# 4. Verify
curl https://myapp-homelab.yourdomain.com
```

---

## 📚 Real-World Examples

### Example 1: WordPress with MySQL

See `manifests/examples/wordpress/`

### Example 2: Node.js Application

See `manifests/examples/nodejs-app/`

### Example 3: Python Flask API

See `manifests/examples/flask-api/`

### Example 4: Static Website

See `manifests/examples/static-site/`

---

## 🔍 Monitoring Your Applications

### Add Prometheus Monitoring

Add annotations to your service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  # ... rest of service config
```

### View Logs in Loki

Access Grafana → Explore → Loki:

```logql
{namespace="my-app"}
```

### Distributed Tracing with Tempo

Configure your app to send traces:

```yaml
env:
- name: OTEL_EXPORTER_OTLP_ENDPOINT
  value: "http://tempo.observability.svc:4317"
```

---

## 🛠️ Troubleshooting Deployments

### Pod Not Starting

```bash
# Describe pod
microk8s kubectl describe pod POD_NAME -n NAMESPACE

# Check events
microk8s kubectl get events -n NAMESPACE --sort-by='.lastTimestamp'

# View logs
microk8s kubectl logs POD_NAME -n NAMESPACE
microk8s kubectl logs POD_NAME -n NAMESPACE --previous
```

### ArgoCD Sync Failed

```bash
# Check sync status
argocd app get APP_NAME

# View detailed sync result
argocd app sync APP_NAME --dry-run

# Force sync
argocd app sync APP_NAME --force

# Refresh app
argocd app sync APP_NAME --prune --force
```

### Ingress Not Working

```bash
# Check ingress
microk8s kubectl get ingress -n NAMESPACE
microk8s kubectl describe ingress INGRESS_NAME -n NAMESPACE

# Check ingress controller logs
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx

# Test internal access
microk8s kubectl port-forward -n NAMESPACE svc/SERVICE_NAME 8080:80
curl http://localhost:8080
```

### Cloudflare Tunnel Issues

```bash
# Check tunnel status
sudo systemctl status cloudflared

# View logs
sudo journalctl -u cloudflared -f

# Validate config
cloudflared tunnel ingress validate

# Test tunnel
cloudflared tunnel ingress rule https://myapp-homelab.yourdomain.com
```

---

## 📖 Best Practices

1. **Use GitOps** - Store all manifests in Git
2. **Version Everything** - Tag images with specific versions
3. **Resource Limits** - Always set CPU/memory requests/limits
4. **Health Checks** - Add liveness and readiness probes
5. **Namespaces** - Isolate applications in separate namespaces
6. **Secrets Management** - Use Kubernetes Secrets or external secret managers
7. **Rolling Updates** - Configure deployment strategy
8. **Backup** - Regular backups of PersistentVolumes
9. **Monitoring** - Add Prometheus metrics to all apps
10. **Documentation** - Document deployment process in Git repo

---

## 🚀 Next Steps

1. Create your first application repository
2. Deploy with ArgoCD
3. Expose via Cloudflare Tunnel
4. Add monitoring and logging
5. Set up automated CI/CD pipeline
6. Explore advanced features (Helm, Kustomize overlays)

---

**Happy Deploying! 🎉**
