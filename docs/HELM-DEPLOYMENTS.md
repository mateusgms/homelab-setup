# ⚓ Helm Deployments Guide

Complete guide to deploying applications using Helm charts with ArgoCD and GitHub Actions.

## 📋 Table of Contents

1. [Overview](#overview)
2. [Helm Chart Structure](#helm-chart-structure)
3. [Creating Helm Charts](#creating-helm-charts)
4. [Values Management](#values-management)
5. [CI/CD with Helm](#cicd-with-helm)
6. [ArgoCD with Helm](#argocd-with-helm)
7. [Best Practices](#best-practices)

---

## 🎯 Overview

### Why Helm in Your Application Repo?

- **Single Source of Truth**: Application code + deployment config together
- **Version Control**: Chart versions match application versions
- **Easy Rollback**: Rollback both code and infrastructure together
- **Reusability**: Same chart for dev/staging/prod with different values

### Repository Structure

```
my-app/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── src/                          # Application code
├── helm/                         # Helm chart (in app repo!)
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── values-dev.yaml
│   ├── values-staging.yaml
│   ├── values-production.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       ├── configmap.yaml
│       └── _helpers.tpl
├── Dockerfile
└── README.md
```

---

## 📦 Helm Chart Structure

### Complete Chart Example

#### 1. Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 1.0.0              # Chart version
appVersion: "1.0.0"         # Application version

keywords:
  - web
  - api
  - nodejs

maintainers:
  - name: Your Name
    email: your-email@example.com

dependencies: []
```

#### 2. values.yaml (Default Values)

```yaml
# Default values for my-app
replicaCount: 2

image:
  repository: ghcr.io/yourusername/my-app
  pullPolicy: IfNotPresent
  tag: ""  # Overridden by appVersion by default

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "3000"
  prometheus.io/path: "/metrics"

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true

service:
  type: ClusterIP
  port: 80
  targetPort: 3000
  annotations: {}

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

nodeSelector: {}

tolerations: []

affinity: {}

env: []
  # - name: NODE_ENV
  #   value: production

envFrom: []
  # - configMapRef:
  #     name: app-config
  # - secretRef:
  #     name: app-secrets

configMap:
  enabled: false
  data: {}

secrets:
  enabled: false
  data: {}
```

#### 3. values-dev.yaml

```yaml
# Development environment overrides
replicaCount: 1

image:
  tag: develop

ingress:
  hosts:
    - host: myapp-dev.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

env:
  - name: NODE_ENV
    value: development
  - name: LOG_LEVEL
    value: debug
```

#### 4. values-staging.yaml

```yaml
# Staging environment overrides
replicaCount: 2

image:
  tag: staging

ingress:
  hosts:
    - host: myapp-staging.example.com
      paths:
        - path: /
          pathType: Prefix

env:
  - name: NODE_ENV
    value: staging
  - name: LOG_LEVEL
    value: info
```

#### 5. values-production.yaml

```yaml
# Production environment overrides
replicaCount: 3

image:
  tag: latest

ingress:
  hosts:
    - host: myapp-homelab.yourdomain.com
      paths:
        - path: /
          pathType: Prefix

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

env:
  - name: NODE_ENV
    value: production
  - name: LOG_LEVEL
    value: warn

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "3000"
```

#### 6. templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
        version: {{ .Values.image.tag | default .Chart.AppVersion }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
      - name: {{ .Chart.Name }}
        securityContext:
          {{- toYaml .Values.securityContext | nindent 12 }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        {{- with .Values.env }}
        env:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        {{- with .Values.envFrom }}
        envFrom:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        livenessProbe:
          {{- toYaml .Values.livenessProbe | nindent 12 }}
        readinessProbe:
          {{- toYaml .Values.readinessProbe | nindent 12 }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

#### 7. templates/_helpers.tpl

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "my-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

---

## 🔄 CI/CD with Helm

### GitHub Actions Workflow

Create `.github/workflows/helm-deploy.yml`:

```yaml
name: Build and Deploy with Helm

on:
  push:
    branches: [main, develop, staging]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-test:
    name: Lint and Test Helm Chart
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.12.0

      - name: Lint Helm chart
        run: helm lint ./helm

      - name: Template Helm chart
        run: |
          helm template my-app ./helm \
            --values ./helm/values.yaml \
            --values ./helm/values-dev.yaml

  build:
    name: Build and Push Image
    needs: lint-and-test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
      
    steps:
      - name: Checkout
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
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    name: Deploy with ArgoCD
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Determine environment
        id: env
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "environment=production" >> $GITHUB_OUTPUT
            echo "values_file=values-production.yaml" >> $GITHUB_OUTPUT
          elif [ "${{ github.ref }}" == "refs/heads/staging" ]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
            echo "values_file=values-staging.yaml" >> $GITHUB_OUTPUT
          else
            echo "environment=development" >> $GITHUB_OUTPUT
            echo "values_file=values-dev.yaml" >> $GITHUB_OUTPUT
          fi

      - name: Update Helm values with new image tag
        run: |
          # Update image tag in environment-specific values file
          sed -i "s|tag:.*|tag: ${{ needs.build.outputs.image-tag }}|" helm/${{ steps.env.outputs.values_file }}

      - name: Commit updated values
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add helm/${{ steps.env.outputs.values_file }}
          git commit -m "Update ${{ steps.env.outputs.environment }} image to ${{ needs.build.outputs.image-tag }}" || echo "No changes"
          git push

      - name: Trigger ArgoCD sync (webhook)
        run: |
          curl -X POST https://argocd-homelab.yourdomain.com/api/webhook \
            -H "Content-Type: application/json" \
            -d '{"repository": {"url": "${{ github.repository_url }}"}}'
```

### Alternative: Package and Push Helm Chart

```yaml
  package-helm:
    name: Package and Push Helm Chart
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v3

      - name: Package Helm chart
        run: |
          # Update Chart.yaml with new app version
          sed -i "s/appVersion:.*/appVersion: \"${{ needs.build.outputs.image-tag }}\"/" helm/Chart.yaml
          
          # Package chart
          helm package helm/ --destination .helm-releases/

      - name: Push to Helm repository
        run: |
          # Option 1: GitHub Pages
          # Option 2: ChartMuseum
          # Option 3: OCI registry (GHCR)
          helm push .helm-releases/*.tgz oci://ghcr.io/${{ github.repository_owner }}/charts
```

---

## 🎯 ArgoCD with Helm

### Method 1: ArgoCD Application (Direct Helm)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/my-app
    targetRevision: main
    path: helm
    helm:
      valueFiles:
        - values.yaml
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "v1.2.3"  # Can be overridden
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Method 2: Multiple Environments

```yaml
---
# Development
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/my-app
    targetRevision: develop
    path: helm
    helm:
      valueFiles:
        - values.yaml
        - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
---
# Staging
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/my-app
    targetRevision: staging
    path: helm
    helm:
      valueFiles:
        - values.yaml
        - values-staging.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
---
# Production
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/my-app
    targetRevision: main
    path: helm
    helm:
      valueFiles:
        - values.yaml
        - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-prod
  syncPolicy:
    automated:
      prune: false  # Manual sync for production
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Method 3: Helm Chart from OCI Registry

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    chart: my-app
    repoURL: ghcr.io/yourusername/charts
    targetRevision: 1.0.0
    helm:
      values: |
        image:
          tag: v1.2.3
        ingress:
          hosts:
            - host: myapp-homelab.yourdomain.com
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 🛠️ Local Development & Testing

### Test Helm Chart Locally

```bash
# Lint chart
helm lint ./helm

# Dry run
helm install my-app ./helm \
  --values ./helm/values-dev.yaml \
  --dry-run --debug

# Template and view
helm template my-app ./helm \
  --values ./helm/values-dev.yaml

# Install locally (if you have microk8s)
helm install my-app ./helm \
  --values ./helm/values-dev.yaml \
  --namespace my-app-dev \
  --create-namespace

# Upgrade
helm upgrade my-app ./helm \
  --values ./helm/values-dev.yaml \
  --namespace my-app-dev

# Rollback
helm rollback my-app 1 --namespace my-app-dev

# Uninstall
helm uninstall my-app --namespace my-app-dev
```

### Validate with ArgoCD

```bash
# Create app in ArgoCD
argocd app create my-app-dev \
  --repo https://github.com/yourusername/my-app \
  --path helm \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace my-app-dev \
  --helm-set-file values=values-dev.yaml

# Diff before sync
argocd app diff my-app-dev

# Sync
argocd app sync my-app-dev

# Check status
argocd app get my-app-dev
```

---

## ✅ Best Practices

### 1. Version Strategy

```yaml
# Chart.yaml
version: 1.2.3      # Chart version (semantic versioning)
appVersion: "1.2.3" # Application version (matches your app)
```

### 2. Values Organization

```
helm/
├── values.yaml              # Base/common values
├── values-dev.yaml          # Dev overrides
├── values-staging.yaml      # Staging overrides
├── values-production.yaml   # Production overrides
└── values.schema.json       # JSON schema for validation
```

### 3. Secrets Management

**Option A: External Secrets Operator**

```yaml
# In your Helm chart
{{- if .Values.externalSecrets.enabled }}
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "my-app.fullname" . }}
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: {{ include "my-app.fullname" . }}-secrets
  data:
    - secretKey: database-password
      remoteRef:
        key: /my-app/database
        property: password
{{- end }}
```

**Option B: Sealed Secrets**

```bash
# Create sealed secret
kubectl create secret generic my-app-secrets \
  --from-literal=db-password=secretpass \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > helm/templates/sealed-secret.yaml
```

### 4. Resource Naming

Use helpers for consistent naming:

```yaml
# templates/_helpers.tpl
{{- define "my-app.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
```

### 5. Documentation

```yaml
# values.yaml - Add comments
replicaCount: 2  # Number of pod replicas

image:
  repository: ghcr.io/yourusername/my-app  # Container image repository
  pullPolicy: IfNotPresent                  # Image pull policy: Always, IfNotPresent, Never
  tag: ""                                   # Overrides the image tag (default is appVersion)
```

---

## 📚 Advanced Patterns

### Pattern 1: Helm Hooks

```yaml
# templates/job-migration.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-app.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      containers:
      - name: migration
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["npm", "run", "migrate"]
      restartPolicy: Never
```

### Pattern 2: Conditional Resources

```yaml
# Only create HPA if autoscaling is enabled
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
# ...
{{- end }}
```

### Pattern 3: Multi-Chart Dependencies

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: 12.1.9
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: 17.3.14
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

---

## 🔍 Monitoring & Debugging

### View Deployed Resources

```bash
# List Helm releases
helm list -A

# Get release info
helm get all my-app -n my-app-dev

# Get values
helm get values my-app -n my-app-dev

# Get manifest
helm get manifest my-app -n my-app-dev

# History
helm history my-app -n my-app-dev
```

### Debug in ArgoCD

```bash
# View application
argocd app get my-app-dev

# View resources
argocd app resources my-app-dev

# View logs
argocd app logs my-app-dev

# View events
argocd app events my-app-dev
```

---

## 📖 Resources

- [Helm Documentation](https://helm.sh/docs/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [ArgoCD Helm Support](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [Chart Development Guide](https://helm.sh/docs/chart_template_guide/)

---

**Happy Helming! ⚓**
