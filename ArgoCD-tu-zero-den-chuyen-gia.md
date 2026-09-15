# 🚢 Tài liệu ArgoCD: Từ Zero đến Chuyên gia

> **Mục tiêu**: Triển khai GitOps chuyên nghiệp với ArgoCD trên Kubernetes: Application, AppSet, Image Updater, multi-cluster.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Chuẩn bị](#p0) | Cài ArgoCD, K8s cluster |
| [P1. Cơ bản](#p1) | Application, sync, status |
| [P2. Sync strategies](#p2) | Manual, auto, prune, self-heal |
| [P3. App of Apps](#p3) | Cấu trúc multi-app |
| [P4. ApplicationSet](#p4) | Multi-cluster/multi-env |
| [P5. Kustomize & Helm](#p5) | Tích hợp tools |
| [P6. Image Updater](#p6) | Auto update image |
| [P7. Notifications](#p7) | Slack, webhook |
| [P8. SSO & Security](#p8) | RBAC, OIDC, secrets |
| [P9. Production](#p9) | HA, backup, monitoring |

---

<a id="p0"></a>
## P0. Chuẩn bị

### Bước 1: Cài K8s cluster (lab)

```bash
# K3s (nhẹ, nhanh)
curl -sfL https://get.k3s.io | sh -

# Hoặc Minikube
minikube start --cpus=4 --memory=8g

# Hoặc Kind
kind create cluster --config kind-multi-node.yaml
```

### Bước 2: Cài ArgoCD

```bash
# Tạo namespace
kubectl create namespace argocd

# Cài (non-HA)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Cài với Helm (khuyến nghị)
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=NodePort \
  --set server.service.nodePortHttps=30443 \
  --set server.extraArgs="{--insecure}" \
  --set dex.enabled=false \
  --set configs.cm.create=true

# HA install
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set redis-ha.enabled=true \
  --set controller.replicas=2 \
  --set server.replicas=2 \
  --set repoServer.replicas=2
```

### Bước 3: Truy cập UI

```bash
# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Lấy admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Login
argocd login localhost:8080 --username admin --password <password>

# Cài CLI
brew install argocd
# hoặc
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

### Bước 4: ArgoCD architecture

```
┌────────────────────────────────────────────────────────┐
│                    ArgoCD Architecture                  │
│                                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │   API   │  │   UI    │  │   CLI   │  │  Webhook│  │
│  │ Server  │  │ Server  │  │         │  │         │  │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  │
│       │            │            │            │        │
│       └────────────┴─────┬──────┴────────────┘        │
│                          │                              │
│  ┌──────────────────────▼──────────────────────────┐  │
│  │          Application Controller                  │  │
│  │  (kiểm tra Git vs cluster state)                │  │
│  └──────────────────────┬──────────────────────────┘  │
│                          │                              │
│  ┌──────────────────────▼──────────────────────────┐  │
│  │            Repo Server (cache)                  │  │
│  │  (clone, render manifests)                      │  │
│  └──────────────────────┬──────────────────────────┘  │
│                          │                              │
│  ┌──────────────────────▼──────────────────────────┐  │
│  │            Redis (state cache)                  │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
                          │
                          ▼
                  ┌──────────────────┐
                  │  K8s API Server  │
                  └──────────────────┘
```

---

<a id="p1"></a>
## P1. Cơ bản

### Bước 1: Cấu trúc repo Git

```bash
myapp-config/
├── apps/
│   ├── myapp-prod.yaml       # ArgoCD Application
│   └── myapp-staging.yaml
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── production/
│   │   ├── kustomization.yaml
│   │   └── patch.yaml
│   └── staging/
│       └── kustomization.yaml
└── README.md
```

### Bước 2: Application đầu tiên

```yaml
# apps/myapp-prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/myapp-config
    targetRevision: main
    path: overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### BƯớc 3: CLI commands

```bash
# List
argocd app list
argocd app list -o wide

# Get
argocd app get myapp-production
argocd app get myapp-production -o yaml

# Sync manually
argocd app sync myapp-production
argocd app sync myapp-production --prune
argocd app sync myapp-production --async

# Status
argocd app status myapp-production
argocd app wait myapp-production --timeout 5m

# History
argocd app history myapp-production

# Diff
argocd app diff myapp-production

# Logs
argocd app logs myapp-production
argocd app logs myapp-production --container app

# Manifests
argocd app manifests myapp-production

# Delete
argocd app delete myapp-production
```

### Bước 4: Resources & tree

```bash
# Xem resources
argocd app resources myapp-production

# Tree view
argocd app tree myapp-production

# Diff cluster vs git
argocd app diff myapp-production

# Compare local vs cluster
argocd app manifests myapp-production > /tmp/local.yaml
kubectl -n production get deploy myapp -o yaml > /tmp/cluster.yaml
diff /tmp/local.yaml /tmp/cluster.yaml
```

### Bước 5: Health checks

```bash
# Health status
argocd app get myapp-production -o json | jq '.status.health.status'

# Custom health check (Lua)
```

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations.health.argoproj.io_Application: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.health.status == "Healthy" then
        hs.status = "Healthy"
        hs.message = obj.status.health.message
        return hs
      end
      if obj.status.health.status == "Progressing" then
        hs.status = "Progressing"
        hs.message = obj.status.health.message
        return hs
      end
    end
    hs.status = "Degraded"
    hs.message = "Waiting for Application status"
    return hs
```

---

<a id="p2"></a>
## P2. Sync strategies

### Bước 1: Manual sync

```bash
# Disable auto-sync
argocd app set myapp-production --sync-policy none

# Manual sync
argocd app sync myapp-production

# Force (replace resources)
argocd app sync myapp-production --force
```

### Bước 2: Auto sync

```yaml
spec:
  syncPolicy:
    automated:
      prune: true            # Xoá resources không có trong Git
      selfHeal: true         # Đồng bộ cluster về Git nếu drift
      allowEmpty: false      # Không cho sync khi Git trống
```

```bash
# CLI
argocd app set myapp-production \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### Bước 3: Sync windows

```yaml
# Allow sync chỉ trong khung giờ
spec:
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    syncWindows:
    - kind: allow
      schedule: '0 8 * * 1-5'   # 8h sáng T2-T6
      duration: 8h
      applications:
      - '*-production'
    - kind: deny
      schedule: '0 0 * * *'     # Cấm sync ban đêm
      duration: 24h
```

### Bước 4: PreSync, Sync, PostSync hooks

```yaml
# Database migration hook
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: myapp:latest
        command: ["/app/migrate"]
  backoffLimit: 3
```

```yaml
# Notification hook
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-deploy
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: notify
        image: curlimages/curl
        command:
        - /bin/sh
        - -c
        - |
          curl -X POST $SLACK_WEBHOOK \
            -H 'Content-Type: application/json' \
            -d '{"text":"Deployed myapp"}'
```

### Bước 5: Resource hooks (blue-green)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/hook: Sync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100
```

### Bước 6: Ignore differences

```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
  - group: apps
    kind: Deployment
    name: myapp
    jqPathExpressions:
    - .spec.template.spec.containers[].image
```

```bash
# CLI
argocd app set myapp-production \
  --ignore-differences='{
    "group": "apps",
    "kind": "Deployment",
    "jqPathExpressions": [".spec.replicas"]
  }'
```

---

<a id="p3"></a>
## P3. App of Apps pattern

### Bước 1: Project

```yaml
# apps/project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-org
  namespace: argocd
spec:
  description: My Organization Apps

  sourceRepos:
  - 'https://github.com/myorg/*'
  - 'https://charts.bitnami.com/*'

  destinations:
  - namespace: '*'
    server: '*'

  clusterResourceWhitelist:
  - group: ''
    kind: Namespace

  namespaceResourceWhitelist:
  - group: ''
    kind: '*'
  - group: apps
    kind: '*'
  - group: networking.k8s.io
    kind: '*'

  roles:
  - name: developer
    description: Developer access
    policies:
    - p, proj:my-org:developer, applications, get, my-org/*, allow
    - p, proj:my-org:developer, applications, sync, my-org/*, allow
    groups:
    - developers

  syncWindows:
  - kind: deny
    schedule: '0 0 * * 6'
    duration: 24h
    applications:
    - '*-prod'
```

### Bước 2: App of Apps

```yaml
# apps/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-config
    targetRevision: main
    path: apps/
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Bước 3: Apps trong apps/

```yaml
# apps/infrastructure.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
spec:
  project: infrastructure
  source:
    repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: v1.14.0
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager

---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
spec:
  project: infrastructure
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: 4.9.1
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
```

```yaml
# apps/applications.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  project: my-org
  source:
    repoURL: https://github.com/myorg/myapp-config
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

### Bước 4: Multi-tenancy

```yaml
# Tổ chức apps theo team/environment
apps/
├── infrastructure/
│   ├── cert-manager.yaml
│   ├── ingress-nginx.yaml
│   └── monitoring.yaml
├── team-a/
│   ├── app1-prod.yaml
│   ├── app1-staging.yaml
│   └── app2-prod.yaml
├── team-b/
│   └── ...
└── root-app.yaml
```

---

<a id="p4"></a>
## P4. ApplicationSet

### Bước 1: List generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: production
        url: https://kubernetes.default.svc
        values: prod
      - cluster: staging
        url: https://staging-cluster.example.com
        values: staging
      - cluster: dev
        url: https://dev-cluster.example.com
        values: dev

  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      project: my-org
      source:
        repoURL: https://github.com/myorg/myapp-config
        targetRevision: main
        path: overlays/{{values}}
      destination:
        server: '{{url}}'
        namespace: myapp
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Bước 2: Cluster generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-apps
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          type: production

  template:
    metadata:
      name: 'prometheus-{{nameNormalized}}'
    spec:
      project: monitoring
      source:
        repoURL: https://prometheus-community.github.io/helm-charts
        chart: kube-prometheus-stack
        targetRevision: 50.x.x
      destination:
        server: '{{server}}'
        namespace: monitoring
```

### Bước 3: Git generator (directory)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: git-directory
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/myorg/myapp-config
      revision: HEAD
      directories:
      - path: apps/*

  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/myapp-config
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

### BƯớc 4: Matrix generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: matrix
spec:
  generators:
  - matrix:
      generators:
      - list:
          elements:
          - cluster: prod
            url: https://prod-cluster
          - cluster: staging
            url: https://staging-cluster
      - list:
          elements:
          - app: frontend
          - app: backend

  template:
    metadata:
      name: '{{app}}-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/{{app}}
        targetRevision: main
        path: deploy/
      destination:
        server: '{{url}}'
        namespace: '{{app}}'
```

### Bước 5: Git file generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: git-files
spec:
  generators:
  - git:
      repoURL: https://github.com/myorg/multi-app-config
      revision: HEAD
      files:
      - path: clusters/*/config.json

  template:
    metadata:
      name: '{{cluster}}'
    spec:
      project: default
      source:
        repoURL: '{{repoURL}}'
        targetRevision: HEAD
        path: 'clusters/{{cluster}}/manifests'
      destination:
        server: '{{server}}'
        namespace: '{{cluster}}'
```

```json
// clusters/prod/config.json
{
  "cluster": "prod",
  "repoURL": "https://github.com/myorg/multi-app-config",
  "server": "https://prod-cluster.example.com"
}
```

---

<a id="p5"></a>
## P5. Kustomize & Helm

### Bước 1: Kustomize application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-kustomize
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-config
    targetRevision: main
    path: overlays/production
    # ArgoCD tự detect kustomization.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production

resources:
- ../../base

patches:
- patch.yaml

images:
- name: myapp
  newTag: v1.2.3
```

### Bước 2: Helm application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-ingress
spec:
  project: infrastructure
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: 4.9.1
    helm:
      valueFiles:
      - values-prod.yaml
      parameters:
      - name: controller.replicaCount
        value: "3"
      - name: controller.resources.limits.cpu
        value: 1000m
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
```

### Bước 3: Helm với private repo

```bash
# Add Helm repo credentials
argocd repo add https://charts.example.com \
  --type helm \
  --name my-charts \
  --username user \
  --password token
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-helm
spec:
  source:
    repoURL: https://charts.example.com
    chart: myapp
    targetRevision: 1.0.0
    helm:
      releaseName: myapp
      valueFiles:
      - $values/values-prod.yaml
```

### Bước 4: Plain YAML / Jsonnet

```yaml
# Plain YAML
spec:
  source:
    repoURL: https://github.com/myorg/myapp-config
    path: manifests/
    directory:
      recurse: true
      include: '{*.yaml,*.yml}'
      exclude: '{*.tmpl.yaml}'

# Jsonnet
spec:
  source:
    repoURL: https://github.com/myorg/myapp-config
    path: jsonnet/
    directory:
      jsonnet:
        extVar:
        - name: cluster
          value: prod
        tlaVar:
        - name: namespace
          value: production
```

### Bước 5: Multiple sources (Helm + Kustomize)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-multi
spec:
  sources:
  - repoURL: https://github.com/myorg/base-config
    targetRevision: main
    path: base/
    ref: base
  - repoURL: https://github.com/myorg/helm-charts
    targetRevision: main
    chart: myapp
    helm:
      valueFiles:
      - $base/values.yaml
    dependsOn:
    - base

  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

---

<a id="p6"></a>
## P6. ArgoCD Image Updater

### Bước 1: Cài Image Updater

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

### Bước 2: Đánh dấu applications

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=myregistry/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: latest
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/write-back-method: git
spec:
  # ...
```

### Bước 3: Git write-back

```yaml
# argocd-image-updater-config (Secret)
apiVersion: v1
kind: Secret
metadata:
  name: argocd-image-updater-config
  namespace: argocd
stringData:
  registries.conf: |
    registries:
    - name: Docker Hub
      url: docker.io
    - name: GHCR
      url: ghcr.io
      credentials: secret:ghcr-creds

  config.yaml: |
    credentials:
    - env:GHCR_TOKEN
      registries:
      - ghcr.io
```

### BƯớc 4: Kustomize integration

```yaml
# Image updater với Kustomize
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=myregistry/myapp
    argocd-image-updater.argoproj.io/myapp.kustomize.image-name: myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: newest-build
```

### Bước 5: Helm integration

```yaml
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=myregistry/myapp
    argocd-image-updater.argoproj.io/myapp.helm.image-name: image.repository
    argocd-image-updater.argoproj.io/myapp.helm.image-tag: image.tag
```

---

<a id="p7"></a>
## P7. Notifications

### Bước 1: Cấu hình

```yaml
# argocd-notifications-cm
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  trigger.sync-succeeded: |
    - description: Application synced
      send:
      - slack-success
      - email

  trigger.sync-failed: |
    - description: Application sync failed
      send:
      - slack-failure
      - pagerduty
      email

  trigger.health-degraded: |
    - description: Health degraded
      send:
      - slack-failure
      pagerduty:
        severity: critical

  trigger.out-of-sync: |
    - description: Application out of sync
      send:
      - slack-warning

  template.slack-success: |
    message: |
      :white_check_mark: Application *{{.app.metadata.name}}* synced.
      Revision: `{{.app.status.operationState.syncResult.revision}}`
      By: {{.app.status.operationState.syncResult.user}}

  template.slack-failure: |
    message: |
      :x: Application *{{.app.metadata.name}}* sync failed.
      Error: {{.app.status.operationState.message}}

  service.slack: |
    token: $slack-token

  subscriptions:
  - recipients:
    - slack:devops-alerts
    triggers:
    - sync-succeeded
    - sync-failed
    - health-degraded
```

### Bước 2: Templates & subscriptions

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  subscriptions:
  # Application-specific
  - selector: app=myapp
    recipients:
    - slack:dev-team
    triggers:
    - sync-succeeded
    - sync-failed
    - out-of-sync
```

### Bước 3: Webhook triggers

```bash
# Trigger sync từ CI/CD
curl -X POST \
  https://argocd.example.com/api/v1/projects/default/applications/myapp/sync \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"revision": "main"}'

# Hoặc CLI
argocd app sync myapp --revision main
```

### BƯớc 4: Custom webhook receivers

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.webhook.deploy-webhook: |
    url: https://my-app.com/webhook
    headers:
    - name: X-Auth-Token
      value: $webhook-token
```

---

<a id="p8"></a>
## P8. SSO & Security

### Bước 1: OIDC với Google

```yaml
# argocd-cm
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.example.com

  oidc.config: |
    name: Google
    issuer: https://accounts.google.com
    clientId: xxx.apps.googleusercontent.com
    clientSecret: $oidc.google.clientSecret
    requestedScopes:
    - openid
    - profile
    - email
    - groups

  policy.csv: |
    p, role:readonly, applications, get, *, allow
    p, role:readonly, projects, get, *, allow

    g, [email protected], role:readonly
    g, [email protected], role:admin
```

### Bước 2: Dex với LDAP

```yaml
# argocd-cm
data:
  dex.config: |
    connectors:
    - type: ldap
      name: OpenLDAP
      id: ldap
      config:
        host: ldap.example.com:636
        rootCAData: $(cat /etc/ssl/certs/ldap-ca.pem | base64 -w 0)
        bindDN: cn=admin,dc=example,dc=com
        bindPW: $dex.ldap.bindPW
        userSearch:
          baseDN: ou=users,dc=example,dc=com
          filter: (objectClass=inetOrgPerson)
          username: uid
          idAttr: uid
          emailAttr: mail
        groupSearch:
          baseDN: ou=groups,dc=example,dc=com
          filter: (objectClass=groupOfNames)
          userAttr: DN
          groupAttr: member
          nameAttr: cn
```

### Bước 3: RBAC Policy

```yaml
# argocd-rbac-cm
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly

  policy.csv: |
    # Built-in roles
    p, role:admin, *, *, *, allow
    p, role:readonly, applications, get, *, allow
    p, role:readonly, projects, get, *, allow
    p, role:readonly, repositories, get, *, allow

    # Custom roles
    p, role:developer, applications, get/*/dev-*, allow
    p, role:developer, applications, sync/*/dev-*, allow
    p, role:sre, applications, *, *-prod, allow

    # Group mappings
    g, myorg:admins, role:admin
    g, myorg:developers, role:developer
    g, myorg:sre, role:sre
    g, myorg:viewers, role:readonly

    # User-specific
    p, role:admin, applications, override, myapp-production, deny
```

### Bước 4: Secrets management

```bash
# Repository credentials
argocd repo add https://github.com/myorg/private-repo \
  --username myuser \
  --password $GITHUB_TOKEN

# Helm repo credentials
argocd repo add https://charts.example.com \
  --type helm \
  --username user \
  --password $PASS \
  --enable-oci

# Cluster credentials
argocd cluster add my-context \
  --name production-cluster \
  --kubeconfig-context production

# SSH private key
argocd repo add [email protected]:myorg/repo.git \
  --ssh-private-key-path ~/.ssh/id_rsa
```

### Bước 5: Sealed Secrets / SOPS

```bash
# Cài Sealed Secrets
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Encrypt
kubectl create secret generic db-password \
  --from-literal=password=secret123 \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# Commit vào Git (encrypted!)
git add sealed-secret.yaml
git commit -m "add db password"
```

```yaml
# application sẽ apply SealedSecret
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
  namespace: production
spec:
  encryptedData:
    password: AgBxxx...   # encrypted
  template:
    data:
      password: ""
    metadata:
      name: db-password
      namespace: production
```

---

<a id="p9"></a>
## P9. Production Best Practices

### Bước 1: HA ArgoCD

```bash
# Install với HA
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set redis-ha.enabled=true \
  --set redis-ha.haproxy.enabled=true \
  --set controller.replicas=3 \
  --set server.replicas=2 \
  --set server.autoscaling.enabled=true \
  --set server.autoscaling.minReplicas=2 \
  --set server.autoscaling.maxReplicas=5 \
  --set repoServer.replicas=2 \
  --set repoServer.autoscaling.enabled=true \
  --set repoServer.autoscaling.minReplicas=2 \
  --set repoServer.autoscaling.maxReplicas=5 \
  --set applicationSet.replicas=2
```

### Bước 2: Backup

```bash
# Backup ArgoCD config
kubectl get all,configmap,secret -n argocd -o yaml > argocd-backup.yaml

# Backup với argocd-backup tool
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-backup-tool/main/manifests/install.yaml

# Cronjob
cat > backup-cronjob.yaml <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: argocd-backup
  namespace: argocd
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: quay.io/argoprojlabs/argocd-backup-tool:latest
            args: ["backup", "--s3-bucket", "argocd-backups"]
          restartPolicy: OnFailure
EOF
```

### Bước 3: Monitoring

```yaml
# Prometheus scrape
- job_name: argocd-server
  static_configs:
  - targets: ['argocd-server-metrics:8083']

- job_name: argocd-controller
  static_configs:
  - targets: ['argocd-application-controller-metrics:8082']

- job_name: argocd-repo-server
  static_configs:
  - targets: ['argocd-repo-server:8084']
```

```yaml
# Grafana dashboard
# Import dashboard: https://grafana.com/grafana/dashboards/14584
```

### Bước 4: Metrics quan trọng

```promql
# Application health
sum(argocd_app_info{health_status="Healthy"}) by (namespace)
sum(argocd_app_info{health_status="Degraded"}) by (namespace)
sum(argocd_app_info{health_status="Progressing"}) by (namespace)

# Sync status
sum(argocd_app_info{sync_status="Synced"})
sum(argocd_app_info{sync_status="OutOfSync"})

# Controller metrics
rate(argocd_app_reconcile_count[5m])
histogram_quantile(0.95, rate(argocd_app_reconcile_duration_seconds_bucket[5m]))
```

### Bước 5: Disaster recovery

```bash
# Scenario: ArgoCD cluster bị crash

# 1. ArgoCD là GitOps - source of truth là Git
# 2. Cài lại ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Apply lại root-app
kubectl apply -n argocd -f root-app.yaml

# 4. ArgoCD tự động sync tất cả apps
```

### Bước 6: Best practices

```
1. Source of truth = Git
   - Mọi thay đổi phải qua Git, không kubectl apply trực tiếp
   - Bật self-heal để revert manual changes

2. Project isolation
   - Mỗi team một Project riêng
   - Destinations/SourceRepos whitelist chặt

3. Sync policy phù hợp
   - Dev: automated, self-heal
   - Staging: automated
   - Prod: automated với sync windows + manual approval cho breaking changes

4. Secrets management
   - Sealed Secrets hoặc External Secrets
   - Không commit plain secrets

5. Health checks
   - Định nghĩa Lua health checks cho custom CRDs
   - Readiness/Liveness probes cho Pods

6. Observability
   - Metrics, logs, traces cho ArgoCD
   - Alert khi app Degraded/OutOfSync quá lâu

7. HA trong production
   - Multi-replica cho mọi component
   - Redis HA
   - External Redis cho scale lớn
```

---

## 🎯 Bài tập P0-P9

1. Cài ArgoCD lên K3s/Minikube, truy cập UI thành công
2. Deploy 1 app đơn giản qua Application resource
3. Setup App of Apps cho 5 services
4. Cấu hình ApplicationSet cho 3 environments
5. Tích hợp Helm chart + Kustomize overlays
6. Cài Image Updater, auto update từ registry
7. Setup Slack notification khi sync fail
8. Backup & restore ArgoCD

---

> **💡 Tip cuối**: ArgoCD là GitOps - Git là source of truth. Không bao giờ kubectl apply trực tiếp khi có ArgoCD. Self-heal = Git luôn thắng. Cấu hình RBAC chặt để multi-team an toàn.

---

*Tạo bởi tài liệu học ArgoCD - Chúc bạn thành công! 🚀*
