---
layout: default
title:  "GitOps with ArgoCD for Kubernetes"
date:   2025-11-18 08:00:00
categories: DevOps Kubernetes GitOps
---

GitOps uses Git as the single source of truth for declarative infrastructure and applications. ArgoCD is the most popular GitOps tool for Kubernetes. Here's how to implement it effectively.

## What is GitOps?

Core principles:
- **Declarative**: System state defined in Git
- **Versioned**: All changes tracked
- **Automated**: Changes applied automatically
- **Self-healing**: Drift detection and correction

## ArgoCD Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Git Repo  │────▶│   ArgoCD    │────▶│  Kubernetes │
│  (Source)   │     │  (Sync)     │     │  (Target)   │
└─────────────┘     └─────────────┘     └─────────────┘
```

## Installation

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Repository Structure

### Recommended: Separate Config Repo

```
app-config/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── prod/
│       └── kustomization.yaml
└── argocd/
    └── application.yaml
```

### Application Definition

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/app-config.git
    targetRevision: main
    path: overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Multi-Environment Setup

### Using ApplicationSets

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            cluster: https://dev-cluster
          - env: staging
            cluster: https://staging-cluster
          - env: prod
            cluster: https://prod-cluster
  template:
    metadata:
      name: 'my-app-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/app-config.git
        targetRevision: main
        path: 'overlays/{{env}}'
      destination:
        server: '{{cluster}}'
        namespace: my-app
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## Sync Strategies

### Automatic Sync

```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources removed from Git
    selfHeal: true   # Revert manual changes
    allowEmpty: false
```

### Manual with Hooks

```yaml
# Pre-sync job (e.g., database migration)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: my-app:latest
          command: ["python", "manage.py", "migrate"]
      restartPolicy: Never
```

## Progressive Delivery

### With Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {duration: 10m}
        - setWeight: 40
        - pause: {duration: 10m}
        - setWeight: 60
        - pause: {duration: 10m}
        - setWeight: 80
        - pause: {duration: 10m}
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:v2
```

## Secrets Management

### With Sealed Secrets

```bash
# Install kubeseal
brew install kubeseal

# Seal secret
kubectl create secret generic my-secret \
  --from-literal=password=secret123 \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-secret.yaml
```

### With External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: my-secret
  data:
    - secretKey: password
      remoteRef:
        key: my-app/prod
        property: password
```

## CI/CD Integration

### GitHub Actions Workflow

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        run: |
          docker build -t myregistry/myapp:${{ github.sha }} .
          docker push myregistry/myapp:${{ github.sha }}

      - name: Update manifest
        run: |
          git clone https://github.com/org/app-config.git
          cd app-config
          kustomize edit set image myapp=myregistry/myapp:${{ github.sha }}
          git commit -am "Update image to ${{ github.sha }}"
          git push
```

## Monitoring and Notifications

### Slack Notifications

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.slack: |
    token: $slack-token
  template.app-deployed: |
    message: |
      {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application {{.app.metadata.name}} is now running new version.
  trigger.on-deployed: |
    - description: Application is synced and healthy
      send:
        - app-deployed
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
```

## Best Practices

1. **Separate repos**: Keep config separate from application code
2. **Use folders, not branches**: Model environments with folders
3. **Avoid mixing resources**: Keep Kubernetes and ArgoCD manifests separate
4. **Enable auto-sync carefully**: Use for non-prod first
5. **Implement RBAC**: Restrict who can sync to production
6. **Use ApplicationSets**: For managing multiple clusters/environments

## Common Patterns

### App of Apps

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
spec:
  source:
    repoURL: https://github.com/org/argocd-apps.git
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
```

### Helm with Values from Git

```yaml
spec:
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: postgresql
    targetRevision: 12.1.0
    helm:
      valueFiles:
        - $values/overlays/prod/values.yaml
  sources:
    - repoURL: https://github.com/org/app-config.git
      targetRevision: main
      ref: values
```

## Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [Codefresh ArgoCD Guide](https://codefresh.io/learn/argo-cd/)
- [DigitalOcean GitOps Tutorial](https://www.digitalocean.com/community/tutorials/how-to-deploy-to-kubernetes-using-argo-cd-and-gitops)

---

*Questions about GitOps with ArgoCD? [Let me know](mailto:jordan@jordananderson.us).*
