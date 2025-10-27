# 🔄 CI/CD Pipeline with GitHub Actions

Complete guide to automating deployments with GitHub Actions and ArgoCD.

## 📋 Table of Contents

1. [Overview](#overview)
2. [GitHub Actions Workflow Examples](#workflow-examples)
3. [ArgoCD Integration](#argocd-integration)
4. [Complete CI/CD Pipeline](#complete-pipeline)
5. [Best Practices](#best-practices)

---

## 🎯 Overview

### CI/CD Flow

```
Code Push → GitHub Actions → Build & Test → Push Image → Update Manifests → ArgoCD Sync → Deploy
```

### Components

- **GitHub Actions**: CI/CD automation
- **Docker Hub/GHCR**: Container registry
- **Git Repository**: Manifest storage
- **ArgoCD**: GitOps deployment
- **Kubernetes**: Target cluster

---

## 🚀 GitHub Actions Workflow Examples

### Example 1: Simple Build and Push

Create `.github/workflows/build-and-push.yml`:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main
      - master
    paths:
      - 'src/**'
      - 'Dockerfile'
  pull_request:
    branches:
      - main
      - master

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Example 2: Build, Test, and Deploy

Create `.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Run security scan
        run: npm audit --audit-level=moderate

  build:
    name: Build and Push Image
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix={{branch}}-,format=short
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    name: Update Kubernetes Manifests
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - name: Checkout manifests repo
        uses: actions/checkout@v4
        with:
          repository: yourusername/k8s-manifests
          token: ${{ secrets.PAT_TOKEN }}
          ref: main

      - name: Update image tag
        run: |
          cd overlays/production
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build.outputs.image-tag }}

      - name: Commit and push changes
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "Update image to ${{ needs.build.outputs.image-tag }}"
          git push
```

### Example 3: Multi-Environment Deployment

Create `.github/workflows/deploy-multi-env.yml`:

```yaml
name: Deploy to Multiple Environments

on:
  push:
    branches:
      - develop
      - staging
      - main

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  determine-environment:
    name: Determine Environment
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
    steps:
      - name: Set environment
        id: set-env
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [ "${{ github.ref }}" == "refs/heads/staging" ]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=development" >> $GITHUB_OUTPUT
          fi

  build-and-deploy:
    name: Build and Deploy
    needs: determine-environment
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-environment.outputs.environment }}
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=${{ needs.determine-environment.outputs.environment }}-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          build-args: |
            ENVIRONMENT=${{ needs.determine-environment.outputs.environment }}

      - name: Update manifests
        uses: actions/checkout@v4
        with:
          repository: yourusername/k8s-manifests
          token: ${{ secrets.PAT_TOKEN }}
          path: manifests

      - name: Update Kustomization
        working-directory: manifests/overlays/${{ needs.determine-environment.outputs.environment }}
        run: |
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}=${{ steps.meta.outputs.tags }}

      - name: Commit changes
        working-directory: manifests
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "Deploy ${{ steps.meta.outputs.tags }} to ${{ needs.determine-environment.outputs.environment }}"
          git push
```

---

## 🔗 ArgoCD Integration

### Method 1: Image Updater (Recommended)

Install ArgoCD Image Updater:

```bash
microk8s kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

Configure application with annotations:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=ghcr.io/yourusername/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: latest
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^main-.*
spec:
  # ... rest of application spec
```

### Method 2: Webhook Trigger

Configure ArgoCD webhook in GitHub:

1. Go to GitHub repo → Settings → Webhooks
2. Add webhook:
   - **Payload URL**: `https://argocd-homelab.yourdomain.com/api/webhook`
   - **Content type**: `application/json`
   - **Secret**: (Set in ArgoCD webhook secret)
   - **Events**: Push, Pull request

Create webhook secret in ArgoCD:

```bash
# Generate secret
WEBHOOK_SECRET=$(openssl rand -hex 32)

# Configure in ArgoCD
microk8s kubectl -n argocd create secret generic webhook-secret \
  --from-literal=webhook.github.secret=$WEBHOOK_SECRET
```

### Method 3: Manual GitOps Update

GitHub Actions updates manifest repo:

```yaml
- name: Update image in Git
  run: |
    # Clone manifest repo
    git clone https://${{ secrets.PAT_TOKEN }}@github.com/yourusername/k8s-manifests.git
    cd k8s-manifests
    
    # Update image tag
    sed -i "s|image:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}|" base/deployment.yaml
    
    # Commit and push
    git config user.name "Bot"
    git config user.email "bot@github.com"
    git add .
    git commit -m "Update image to ${{ github.sha }}"
    git push
```

---

## 🔄 Complete CI/CD Pipeline

### Repository Structure

```
my-app/                          # Application code
├── .github/
│   └── workflows/
│       ├── ci.yml              # Build and test
│       └── cd.yml              # Deploy
├── src/                        # Application source
├── tests/                      # Tests
├── Dockerfile                  # Container image
└── README.md

k8s-manifests/                   # Kubernetes manifests (separate repo)
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   ├── staging/
│   └── production/
└── argocd/
    └── application.yaml
```

### Step-by-Step Setup

#### 1. Create Application Repository

```bash
# Clone or create your app repo
git clone https://github.com/yourusername/my-app.git
cd my-app

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./
EXPOSE 3000
CMD ["node", "dist/index.js"]
EOF

# Create workflow
mkdir -p .github/workflows
# (Add workflow files from examples above)
```

#### 2. Create Manifests Repository

```bash
# Create separate manifests repo
mkdir k8s-manifests && cd k8s-manifests
git init

# Create base manifests
mkdir -p base overlays/{dev,staging,production}

# Create deployment.yaml (see examples)
# Create service.yaml
# Create ingress.yaml
# Create kustomization.yaml for each overlay

git add .
git commit -m "Initial manifests"
git remote add origin git@github.com:yourusername/k8s-manifests.git
git push -u origin main
```

#### 3. Configure Secrets

In GitHub repository settings (Application repo):

- **Settings** → **Secrets and variables** → **Actions**
- Add secrets:
  - `PAT_TOKEN`: Personal Access Token (to update manifests repo)
  - `DOCKERHUB_USERNAME`: (if using Docker Hub)
  - `DOCKERHUB_TOKEN`: (if using Docker Hub)

#### 4. Create ArgoCD Application

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
    repoURL: https://github.com/yourusername/k8s-manifests
    targetRevision: HEAD
    path: overlays/production
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

#### 5. Test Pipeline

```bash
# Make a code change
echo "// Update" >> src/index.js

# Commit and push
git add .
git commit -m "Test CI/CD pipeline"
git push

# Watch GitHub Actions
# Check: https://github.com/yourusername/my-app/actions

# Watch ArgoCD sync
argocd app get my-app --watch

# Verify deployment
microk8s kubectl get pods -n my-app
```

---

## 📚 Advanced Patterns

### Pattern 1: Blue-Green Deployment

```yaml
# In GitHub Actions
- name: Deploy to blue environment
  run: |
    # Update blue deployment
    kubectl set image deployment/my-app-blue \
      app=${{ env.IMAGE }} -n my-app
    
    # Wait for rollout
    kubectl rollout status deployment/my-app-blue -n my-app
    
    # Run smoke tests
    ./scripts/smoke-test.sh blue
    
    # Switch traffic
    kubectl patch service my-app-service \
      -p '{"spec":{"selector":{"version":"blue"}}}' -n my-app
```

### Pattern 2: Canary Deployment

Use Argo Rollouts:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100
```

### Pattern 3: Preview Environments

```yaml
name: Create Preview Environment

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy preview
        run: |
          # Create namespace
          kubectl create namespace pr-${{ github.event.pull_request.number }}
          
          # Deploy application
          kustomize build overlays/dev | \
            sed "s/namespace: .*/namespace: pr-${{ github.event.pull_request.number }}/" | \
            kubectl apply -f -
          
          # Comment on PR with preview URL
          echo "Preview: https://pr-${{ github.event.pull_request.number }}.yourdomain.com"
```

---

## ✅ Best Practices

### 1. Separate Repos

- **Application code**: `my-app`
- **Kubernetes manifests**: `k8s-manifests`
- Why: Clear separation, GitOps principles

### 2. Semantic Versioning

```yaml
# Use semantic-release or similar
- name: Semantic Release
  uses: cycjimmy/semantic-release-action@v3
  with:
    extra_plugins: |
      @semantic-release/changelog
      @semantic-release/git
```

### 3. Security Scanning

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload Trivy results to GitHub Security
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

### 4. Rollback Strategy

```bash
# Rollback via ArgoCD
argocd app rollback my-app

# Or via kubectl
microk8s kubectl rollout undo deployment/my-app -n my-app
```

### 5. Environment Variables

Use Kubernetes ConfigMaps and Secrets:

```yaml
# In deployment.yaml
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: app-secrets
```

### 6. Notifications

Add Slack/Discord notifications:

```yaml
- name: Slack Notification
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: 'Deployment to production completed!'
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
  if: always()
```

---

## 🔍 Monitoring CI/CD

### Track Deployments in Grafana

Add labels to deployments:

```yaml
metadata:
  labels:
    version: ${{ github.sha }}
    deployed-by: github-actions
    deployed-at: ${{ github.event.head_commit.timestamp }}
```

### View in ArgoCD

```bash
# List applications
argocd app list

# Get app status
argocd app get my-app

# View sync history
argocd app history my-app

# View logs
argocd app logs my-app
```

---

## 🐛 Troubleshooting

### Pipeline Fails at Build

```bash
# Check Actions logs in GitHub
# Common issues:
# - Incorrect Dockerfile
# - Missing dependencies
# - Build timeout
```

### Image Push Fails

```bash
# Verify registry authentication
# Check GITHUB_TOKEN permissions
# Ensure image name is correct
```

### ArgoCD Not Syncing

```bash
# Check app status
argocd app get my-app

# Force refresh
argocd app sync my-app --force

# Check repo access
argocd repo list
```

---

## 📖 Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [Docker Build Documentation](https://docs.docker.com/build/)

---

**Happy CI/CD! 🚀**
