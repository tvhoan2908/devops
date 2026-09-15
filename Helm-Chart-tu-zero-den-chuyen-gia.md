# ⎈ Tài liệu Helm Chart: Từ Zero đến Chuyên gia

> **Mục tiêu**: Đóng gói, quản lý và triển khai ứng dụng Kubernetes chuyên nghiệp với Helm.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Cơ bản](#p0) | Cài Helm, chart structure |
| [P1. Chart đầu tiên](#p1) | helm create, values, template |
| [P2. Template nâng cao](#p2) | Functions, control structures |
| [P3. Values & Release](#p3) | Quản lý values, upgrade, rollback |
| [P4. Hooks](#p4) | Pre-install, post-upgrade, test |
| [P5. Chart dependencies](#p5) | Subcharts, requirements |
| [P6. Helmfile & Argo](#p6) | Multi-environment |
| [P7. Chart testing](#p7) | helm unittest, ct |
| [P8. Production](#p8) | Security, signing, OCI registry |

---

<a id="p0"></a>
## P0. Cơ bản

### Bước 1: Cài Helm

```bash
# Linux/macOS
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows (scoop)
scoop install helm

# Verify
helm version

# Thêm repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### Bước 2: Khái niệm

```
┌──────────────────────────────────────────────────────────┐
│  Helm Architecture                                       │
│                                                           │
│  Chart ──► Release ──► Kubernetes Resources              │
│                                                           │
│  Chart     = Package (templates + values)                │
│  Release   = Instance đã deploy                          │
│  Repository = Nơi lưu chart                              │
│  Values    = Config cho chart                            │
└──────────────────────────────────────────────────────────┘
```

### Bước 3: Lệnh Helm cơ bản

```bash
# Tìm chart
helm search repo nginx
helm search hub nginx

# Tải chart
helm pull bitnami/nginx --untar

# Cài (install)
helm install my-nginx bitnami/nginx --namespace web --create-namespace

# Xem status
helm list -A
helm list -n web
helm status my-nginx -n web

# Upgrade
helm upgrade my-nginx bitnami/nginx -n web

# Lấy values
helm get values my-nginx -n web
helm get manifest my-nginx -n web

# Rollback
helm history my-nginx -n web
helm rollback my-nginx 1 -n web

# Uninstall
helm uninstall my-nginx -n web

# Xem chart info
helm show values bitnami/nginx
helm show chart bitnami/nginx
helm show all bitnami/nginx
```

---

<a id="p1"></a>
## P1. Chart đầu tiên

### Bước 1: helm create

```bash
helm create mychart
```
```bash
# Cấu trúc
mychart/
├── .helmignore
├── Chart.yaml              # Metadata
├── values.yaml             # Default values
├── charts/                 # Subcharts
├── templates/              # K8s manifests
│   ├── _helpers.tpl        # Template helpers
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── NOTES.txt           # Hiển thị sau khi cài
│   └── tests/
│       └── test-connection.yaml
└── README.md
```

### Bước 2: Chart.yaml

```yaml
apiVersion: v2                # Helm 3 (chỉ support v2)
name: mychart
description: A simple web application
type: application             # application | library
version: 0.1.0                # Chart version (SemVer)
appVersion: "1.0.0"           # Version của app
home: https://example.com
sources:
  - https://github.com/example/mychart
maintainers:
  - name: Your Name
    email: [email protected]
keywords:
  - web
  - nginx
kubeVersion: ">=1.24.0-0"     # K8s version tối thiểu
dependencies:                 # Subcharts
  - name: postgresql
    version: 12.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
icon: https://example.com/logo.png
```

### Bước 3: values.yaml

```yaml
# values.yaml
replicaCount: 1

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

podAnnotations: {}
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 101
  fsGroup: 101

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]

env:
  - name: ENV
    value: production

envFrom: []    # ConfigMap, Secret refs
```

### Bước 4: Templates

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "mychart.serviceAccountName" . }}
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
          containerPort: 80
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: http
        readinessProbe:
          httpGet:
            path: /
            port: http
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        env:
          {{- toYaml .Values.env | nindent 12 }}
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

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "mychart.selectorLabels" . | nindent 4 }}
```

```yaml
# templates/_helpers.tpl
{{/*
Expand the name of the chart.
*/}}
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "mychart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
Common labels
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
{{ include "mychart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}

{{/*
Selector labels
*/}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

{{/*
Service account name
*/}}
{{- define "mychart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create -}}
{{- default (include "mychart.fullname" .) .Values.serviceAccount.name -}}
{{- else -}}
{{- default "default" .Values.serviceAccount.name -}}
{{- end -}}
{{- end -}}
```

### Bước 5: Test chart

```bash
# Lint
helm lint ./mychart

# Template (render ra YAML)
helm template myrelease ./mychart
helm template myrelease ./mychart --set replicaCount=3
helm template myrelease ./mychart -f custom-values.yaml
helm template myrelease ./mychart --debug  # Xem cả error

# Install
helm install myrelease ./mychart --dry-run
helm install myrelease ./mychart -n web --create-namespace

# Cài với values tùy chỉnh
helm install myrelease ./mychart \
  --set replicaCount=3 \
  --set image.tag=1.26 \
  --set ingress.enabled=true
```

---

<a id="p2"></a>
## P2. Template nâng cao

### Bước 1: Built-in functions

```yaml
# String functions
{{ "Hello" | upper }}                    # HELLO
{{ "HELLO" | lower }}                    # hello
{{ "hello" | quote }}                    # "hello"
{{ "  hello  " | trim }}                 # hello
{{ "hello" | repeat 3 }}                 # hellohellohello
{{ "a-b-c" | replace "-" "_" }}          # a_b_c
{{ "hello world" | split " " }}          # [hello world]

# Default
{{ .Values.image.tag | default "latest" }}
{{ .Values.image.tag | default .Chart.AppVersion }}

# Indent (format YAML)
{{ include "mychart.labels" . | nindent 4 }}

# ToYaml
{{- toYaml .Values.resources | nindent 12 }}

# Required
{{ required "image.repository is required" .Values.image.repository }}
```

### Bước 2: Control structures

```yaml
# if/else
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

# with (scope)
{{- with .Values.ingress }}
ingress:
  className: {{ .className }}
  annotations:
    {{- toYaml .annotations | nindent 4 }}
{{- end }}

# range
{{- range $key, $value := .Values.config }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}

# range với list
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}

# range with index
{{- range $i, $e := .Values.hosts }}
- host: {{ $e.host }}
  paths:
  {{- range $e.paths }}
  - path: {{ .path }}
    pathType: {{ .pathType }}
  {{- end }}
{{- end }}
```

### Bước 3: Variables

```yaml
# Đặt biến local
{{- $fullName := include "mychart.fullname" . -}}
{{- $component := "api" -}}

apiVersion: v1
kind: Service
metadata:
  name: {{ $fullName }}-{{ $component }}
spec:
  selector:
    app: {{ $fullName }}-{{ $component }}
```

### Bước 4: Pipeline & chaining

```yaml
# Chaining functions
{{ .Values.name | default "myapp" | lower | quote }}
# Tương đương:
{{ quote (lower (default "myapp" .Values.name)) }}
```

### Bước 5: Lookup function

```go-template
# Tra cứu resource có sẵn trong cluster
{{- $secret := lookup "v1" "Secret" .Release.Namespace "my-secret" -}}
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
data:
  {{- if $secret }}
  password: {{ $secret.data.password }}
  {{- else }}
  password: {{ randAlphaNum 16 | b64enc }}
  {{- end }}
```

### Bước 6: Sprig functions hữu ích

```yaml
# Random
{{ randAlphaNum 16 }}                     # Random string
{{ randNumeric 8 }}                       # Random số

# Encryption
{{ "mypass" | b64enc }}                   # Base64 encode
{{ "bXlwYXNz" | b64dec }}                 # Base64 decode

# Date
{{ now | date "2006-01-02" }}             # 2026-09-15
{{ now | unixEpoch }}                     # 1757923200

# List
{{ list "a" "b" "c" | first }}            # a
{{ list "a" "b" "c" | last }}             # c

# Dict
{{ dict "key1" "val1" "key2" "val2" }}

# Ternary
{{ eq .Values.env "prod" | ternary "production" "staging" }}
```

---

<a id="p3"></a>
## P3. Values & Release

### Bước 1: Values files cho nhiều môi trường

```bash
# Cấu trúc
mychart/
├── values.yaml                # Default
├── values-dev.yaml            # Dev
├── values-staging.yaml        # Staging
└── values-production.yaml     # Production
```
```yaml
# values-production.yaml
replicaCount: 5
image:
  tag: "1.26.0"
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts: [app.example.com]
      secretName: app-tls
resources:
  limits:
    cpu: 1
    memory: 1Gi
  requests:
    cpu: 200m
    memory: 256Mi
autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
env:
  - name: LOG_LEVEL
    value: info
  - name: ENVIRONMENT
    value: production
postgresql:
  enabled: true
  auth:
    password: changeme
```

### Bước 2: Install/Upgrade với values

```bash
# Cài với values file
helm install myapp ./mychart -f values-production.yaml -n prod

# Cài với --set
helm install myapp ./mychart \
  --set replicaCount=3 \
  --set image.tag=v2.0 \
  --set ingress.hosts[0].host=app.com

# Cài với values JSON
helm install myapp ./mychart --set-json 'env=[{"name":"FOO","value":"bar"}]'

# Multiple values files
helm install myapp ./mychart \
  -f values.yaml \
  -f values-production.yaml \
  -f secrets-production.yaml

# Upgrade
helm upgrade myapp ./mychart -f values-production.yaml -n prod
helm upgrade --install myapp ./mychart -f values.yaml  # Cài nếu chưa có
```

### Bước 3: Secrets với SOPS

```bash
# Cài sops
brew install sops

# Encrypt file
sops --encrypt --age $(cat ~/.sops/age.pub) values-secrets.yaml > values-secrets.enc.yaml
sops --decrypt values-secrets.enc.yaml > values-secrets.yaml

# Helm dùng với plugin
helm plugin install https://github.com/jkroepke/helm-secrets
```

### Bước 4: Diff trước khi upgrade

```bash
# Cài plugin helm-diff
helm plugin install https://github.com/databus23/helm-diff

# Xem diff
helm diff upgrade myapp ./mychart -f values-production.yaml
helm diff upgrade myapp ./mychart -f values-production.yaml --three-way-merge
```

### Bước 5: Rollback & history

```bash
# Lịch sử
helm history myapp -n prod

# Rollback
helm rollback myapp 1 -n prod
helm rollback myapp --wait -n prod

# Rollback với timeout
helm rollback myapp 2 --wait --timeout 5m
```

---

<a id="p4"></a>
## P4. Helm Hooks

### Bước 1: Hook annotations

```yaml
# templates/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-migration
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migration
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["/app/migrate"]
```

### Bước 2: Hook types

```
pre-install          - Trước install
post-install         - Sau install
pre-delete           - Trước uninstall
post-delete          - Sau uninstall
pre-upgrade          - Trước upgrade
post-upgrade         - Sau upgrade
pre-rollback         - Trước rollback
post-rollback        - Sau rollback
test                 - Helm test
```

### Bước 3: Hook weights (thứ tự)

```yaml
# Hook chạy theo weight tăng dần
annotations:
  "helm.sh/hook": pre-install
  "helm.sh/hook-weight": "0"      # Chạy đầu tiên

annotations:
  "helm.sh/hook": pre-install
  "helm.sh/hook-weight": "10"     # Chạy sau
```

### Bước 4: Hook delete policies

```
before-hook-creation   - Xoá resource trước khi tạo hook mới
hook-succeeded        - Xoá khi hook thành công
hook-failed           - Xoá khi hook fail
```

### Bước 5: Helm Test

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "mychart.fullname" . }}-test-connection
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "mychart.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```
```bash
helm test myapp -n prod
helm test myapp -n prod --logs
helm test myapp -n prod --timeout 5m
```

### Bước 6: NOTES.txt

```
# templates/NOTES.txt
Congratulations! {{ .Chart.Name }} v{{ .Chart.AppVersion }} has been installed.

Release name: {{ .Release.Name }}
Namespace:    {{ .Release.Namespace }}

{{- if .Values.ingress.enabled }}

Application URL:
{{- range .Values.ingress.hosts }}
  {{- $tlsEnabled := $.Values.ingress.tls }}
  {{- $protocol := "http" }}
  {{- if $tlsEnabled }}
    {{- $protocol = "https" }}
  {{- end }}
  {{- range .paths }}
  {{ $protocol }}://{{ $.host }}{{ .path }}
  {{- end }}
{{- end }}

{{- else if contains "NodePort" .Values.service.type }}

Get the application URL by running:
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "mychart.fullname" . }})
  echo http://$NODE_IP:$NODE_PORT

{{- end }}
```

---

<a id="p5"></a>
## P5. Chart dependencies

### Bước 1: Thêm dependency

```bash
# Thêm dependency
helm dependency add ./mychart https://charts.bitnami.com/bitnami/postgresql-12.x.x.tgz

# Hoặc sửa Chart.yaml rồi update
helm dependency update ./mychart
helm dependency build ./mychart
helm dependency list ./mychart
```

### Bước 2: Chart.yaml với dependencies

```yaml
apiVersion: v2
name: myapp
version: 1.0.0
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
    alias: db
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
  - name: common
    version: "1.x.x"
    repository: "https://charts.example.com/"
```

### Bước 3: Dùng subchart values

```yaml
# values.yaml
postgresql:
  enabled: true
  auth:
    database: mydb
    username: myuser
    existingSecret: my-postgres-secret
  primary:
    persistence:
      size: 20Gi

redis:
  enabled: true
  auth:
    password: redispass
  master:
    persistence:
      size: 5Gi
```

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}-config
data:
  DB_HOST: {{ .Release.Name }}-postgresql
  DB_PORT: "5432"
  DB_NAME: {{ .Values.postgresql.auth.database }}
  REDIS_HOST: {{ .Release.Name }}-redis-master
  REDIS_PORT: "6379"
```

### Bước 4: Condition để bật/tắt

```yaml
# Trong Chart.yaml
dependencies:
  - name: postgresql
    condition: postgresql.enabled

# Trong values.yaml
postgresql:
  enabled: false   # Tắt nếu dùng external DB
```

### Bước 5: Alias

```yaml
# Cho phép dùng 1 dependency nhiều lần
dependencies:
  - name: postgresql
    alias: primary-db
  - name: postgresql
    alias: analytics-db
```

---

<a id="p6"></a>
## P6. Helmfile & GitOps

### Bước 1: Cài Helmfile

```bash
# macOS
brew install helmfile

# Linux
wget -O /usr/local/bin/helmfile \
  https://github.com/helmfile/helmfile/releases/latest/download/helmfile_linux_amd64
chmod +x /usr/local/bin/helmfile
```

### Bước 2: helmfile.yaml

```yaml
# helmfile.yaml
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts
  - name: mycompany
    url: ghcr.io/mycompany/charts
    oci: true

# Default values cho tất cả releases
environments:
  default:
    values:
      - default.yaml
  production:
    values:
      - default.yaml
      - production.yaml
    secrets:
      - secrets-production.enc.yaml
  staging:
    values:
      - default.yaml
      - staging.yaml

releases:
  # Namespace
  - name: namespaces
    chart: charts/namespaces
    installed: true
    values:
      - namespaces.yaml

  # App core
  - name: myapp
    chart: charts/myapp
    version: 1.2.0
    installed: true
    wait: true
    timeout: 600
    needs:
      - namespaces
    values:
      - release: myapp-common
        values:
          - myapp/values.yaml
          - myapp/values-{{ .Environment.Name }}.yaml

  # PostgreSQL
  - name: postgres
    chart: bitnami/postgresql
    version: 12.x.x
    installed: true
    needs:
      - namespaces
    values:
      - postgresql.yaml

  # Monitoring
  - name: kube-prom
    chart: prometheus-community/kube-prometheus-stack
    version: 50.x.x
    installed: true
    needs:
      - namespaces
    values:
      - monitoring.yaml

  # Cert-manager
  - name: cert-manager
    chart: jetstack/cert-manager
    version: v1.14.0
    installed: true
    namespace: cert-manager
    values:
      - cert-manager.yaml
```

### Bước 3: Lệnh helmfile

```bash
# Sync (apply tất cả)
helmfile sync
helmfile sync --environment production
helmfile sync --selector name=myapp

# Diff
helmfile diff
helmfile --environment production diff

# Apply cho 1 release
helmfile apply --selector name=myapp

# Build (xem output)
helmfile build
helmfile build --environment production

# Destroy
helmfile destroy
helmfile destroy --selector name=postgres
```

### Bước 4: ArgoCD ApplicationSet

```yaml
# applicationset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
spec:
  generator:
    - list:
        elements:
          - cluster: production
            url: https://kubernetes.default.svc
            values: values-prod.yaml
          - cluster: staging
            url: https://staging.example.com
            values: values-staging.yaml
  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/myapp-config
        targetRevision: main
        path: charts/myapp
        helm:
          valueFiles:
            - '{{values}}'
      destination:
        server: '{{url}}'
        namespace: myapp
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

<a id="p7"></a>
## P7. Chart testing

### Bước 1: helm-unittest

```bash
# Cài plugin
helm plugin install https://github.com/helm-unittest/helm-unittest

# Chạy test
helm unittest ./mychart
helm unittest ./mychart --output junit
helm unittest ./mychart --values values-test.yaml
```

### Bước 2: Viết unit test

```yaml
# templates/deployment_test.yaml
suite: test deployment
templates:
  - deployment.yaml
release:
  name: myrelease
  namespace: default
values:
  - values.yaml
  - replicaCount: 2
    image:
      tag: v2.0
tests:
  - it: should render Deployment
    asserts:
      - isKind:
          of: Deployment
      - hasDocuments:
          count: 1
      - matchSnapshot:
          path: data

  - it: should have 2 replicas
    asserts:
      - equal:
          path: spec.replicas
          value: 2

  - it: should have correct image
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: "nginx:v2.0"

  - it: should have security context
    set:
      securityContext:
        runAsNonRoot: true
    asserts:
      - equal:
          path: spec.template.spec.securityContext.runAsNonRoot
          value: true

  - it: should not include ingress when disabled
    template: ingress.yaml
    asserts:
      - hasDocuments:
          count: 0

  - it: should include ingress when enabled
    template: ingress.yaml
    set:
      ingress:
        enabled: true
        hosts:
          - host: app.example.com
    asserts:
      - isKind:
          of: Ingress
      - equal:
          path: spec.rules[0].host
          value: app.example.com
```

### Bước 3: chart-testing (ct)

```bash
# Cài
brew install chart-testing

# Lint + test + install thật
ct lint --chart-dirs charts/
ct lint --chart-dirs charts/ --charts mychart
ct install --chart-dirs charts/
ct install --chart-dirs charts/ --charts mychart --helm-extra-args "--values values-test.yaml"

# Cấu hình
cat > ct.yaml <<EOF
chart-dirs:
  - charts
charts:
  - mychart
helm-extra-args: --timeout 600s
EOF
ct lint --config ct.yaml
```

### Bước 4: Kubeval / kubeconform

```bash
# Validate K8s manifests
kubeconform -strict -summary -ignore-missing-schemas templates/

# Trong CI
helm template myrelease ./mychart | kubeconform -strict -summary
```

---

<a id="p8"></a>
## P8. Production Best Practices

### Bước 1: OCI Registry

```bash
# Login OCI registry
helm registry login ghcr.io -u user -p token

# Lưu chart dưới dạng OCI artifact
helm chart save ./mychart ghcr.io/myorg/mychart:1.0.0
helm chart push ghcr.io/myorg/mychart:1.0.0

# Pull OCI chart
helm pull oci://ghcr.io/myorg/mychart --version 1.0.0
```

### Bước 2: Chart signing (provenance)

```bash
# Tạo keypair
gpg --gen-key
gpg --list-keys
export HELM_GPG_KEY=<key-id>

# Package với signing
helm package --sign --key $HELM_GPG_KEY --keyring .gnupg/secring.gpg ./mychart

# Verify
helm verify mychart-1.0.0.tgz

# Install với verify
helm install myapp mychart-1.0.0.tgz --verify
```

### Bước 3: Security hardening

```yaml
# values.yaml - secure defaults
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 3000
  fsGroup: 2000
  seccompProfile:
    type: RuntimeDefault

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]

resources:
  limits:
    cpu: 500m
    memory: 512Mi

serviceAccount:
  create: true
  automountServiceAccountToken: false

# Network policy
networkPolicy:
  enabled: true
  ingressNamespaceSelector: {}
  podSelector:
    matchLabels:
      app: myapp
```

### Bước 4: Helm với Vault

```bash
# Plugin helm-vault
helm plugin install https://github.com/EndlessGaming/helm-vault

# Values với vault://
replicaCount: 3
dbPassword: vault://secret/myapp/db#password

# Install
helm secrets install myapp ./mychart -f values.yaml
```

### Bước 5: CI/CD cho chart

```yaml
# .github/workflows/release.yml
name: Release Chart
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure Git
        run: |
          git config user.email "[email protected]"
          git config user.name "GitHub Actions"

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.6.0
        env:
          CR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          charts-dir: charts

      - name: Sign and package
        run: |
          gpg --import <(echo "${{ secrets.GPG_PRIVATE_KEY }}")
          helm package --sign --key $GPG_KEY ./charts/mychart
```

### Bước 6: Helm 4 (future)

```bash
# Sidecar hooks (đã có sẵn)
# Schema validation tự động
# Library chart support tốt hơn
# WebAssembly rendering

# Preview tính năng
helm install --dry-run --debug ./mychart --show-only templates/deployment.yaml
```

---

## 🎯 Bài tập P0-P8

1. Tạo chart cho 1 app của bạn với deployment, service, ingress
2. Tạo 3 values file: dev, staging, production
3. Thêm Helm hook chạy migration Job trước upgrade
4. Subchart postgresql + redis
5. Viết helm-unittest, đạt 80% coverage
6. Helmfile quản lý 5 releases
7. Push chart lên OCI registry, deploy lên K8s cluster

---

> **💡 Tip cuối**: Chart càng nhỏ càng tốt, dùng subchart cho phần optional. Luôn test với helm-unittest trước khi release. Dùng GitOps (Helmfile/ArgoCD) thay vì chạy helm thủ công.

---

*Tạo bởi tài liệu học Helm - Chúc bạn thành công! 🚀*
