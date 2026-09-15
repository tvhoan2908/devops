# 🕸️ Tài liệu Istio Service Mesh: Từ Zero đến Chuyên gia

> **Mục tiêu**: Triển khai Istio production-ready: traffic management, security, observability, multi-cluster.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Chuẩn bị](#p0) | Cài Istio, sidecar injection |
| [P1. Traffic management](#p1) | VirtualService, DestinationRule |
| [P2. Gateway & Ingress](#p2) | Istio Gateway |
| [P3. Security](#p3) | mTLS, AuthorizationPolicy |
| [P4. Observability](#p4) | Kiali, Jaeger, Prometheus |
| [P5. Resilience](#p5) | Retry, timeout, circuit breaker |
| [P6. Multi-cluster](#p6) | Multi-primary, primary-remote |
| [P7. Production](#p7) | Performance, upgrade, troubleshooting |

---

<a id="p0"></a>
## P0. Chuẩn bị

### Bước 1: Cài Istio

```bash
# Tải istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.21.*
export PATH=$PWD/bin:$PATH

# Hoặc brew
brew install istioctl

# Verify
istioctl version

# Cài với profile demo
istioctl install --set profile=demo -y

# Production profile
istioctl install --set profile=default -y
```

### Các profile:

```
demo       - Có tất cả (Prometheus, Kiali, Jaeger...)
default    - Recommended cho production
minimal    - Minimal (chỉ pilot)
ambient    - Không sidecar (mesh mới)
preview    - Tính năng mới nhất
```

### Bước 2: Sidecar injection

```bash
# Label namespace để auto-inject
kubectl label namespace default istio-injection=enabled

# Verify
kubectl get namespace -L istio-injection

# Restart pods để inject
kubectl rollout restart deployment -n default

# Inject thủ công cho 1 pod
istioctl kube-inject -f deployment.yaml | kubectl apply -f -

# Kiểm tra
kubectl get pod -n default
# READY   2/2   (container app + istio-proxy)
```

### Bước 3: Verify installation

```bash
# Check control plane
kubectl get pods -n istio-system

# Components
# - istiod (control plane)
# - ingressgateway (ingress)
# - egressgateway (egress - optional)

# Check version
istioctl version

# Analyze cluster
istioctl analyze
istioctl analyze -A    # All namespaces

# Check proxy
istioctl proxy-status
istioctl proxy-config endpoints <pod-name>.<namespace>
```

### Bước 4: Sample app để test

```bash
# Bookinfo app
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# Expose
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# Truy cập
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway \
  -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
echo "http://$INGRESS_HOST:$INGRESS_PORT/productpage"
```

### Bước 5: Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     Istio Service Mesh                    │
│                                                           │
│  ┌────────────────┐                                       │
│  │   istiod       │ (control plane)                       │
│  │   - Pilot      │ (config distribution)                 │
│  │   - Citadel    │ (cert, mTLS)                          │
│  │   - Galley     │ (config validation)                   │
│  └────────┬───────┘                                       │
│           │ xDS API                                       │
│           ▼                                               │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Data Plane (Envoy sidecars)                         │  │
│  │                                                     │  │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    │  │
│  │  │ App Pod  │    │ App Pod  │    │ App Pod  │    │  │
│  │  │ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │    │  │
│  │  │ │ App  │ │    │ │ App  │ │    │ │ App  │ │    │  │
│  │  │ └──┬───┘ │    │ └──┬───┘ │    │ └──┬───┘ │    │  │
│  │  │    │     │    │    │     │    │    │     │    │  │
│  │  │ ┌──▼───┐ │    │ ┌──▼───┐ │    │ ┌──▼───┐ │    │  │
│  │  │ │Envoy │ │    │ │Envoy │ │    │ │Envoy │ │    │  │
│  │  │ │Proxy │ │    │ │Proxy │ │    │ │Proxy │ │    │  │
│  │  │ └──────┘ │    │ └──────┘ │    │ └──────┘ │    │  │
│  │  └──────────┘    └──────────┘    └──────────┘    │  │
│  └────────────────────────────────────────────────────┘  │
│                                                           │
│  Traffic: L7 routing, retries, circuit breaker, mTLS    │
│  Observability: metrics, traces, logs                    │
└──────────────────────────────────────────────────────────┘
```

---

<a id="p1"></a>
## P1. Traffic Management

### Bước 1: VirtualService - Routing cơ bản

```yaml
# reviews-virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: default
spec:
  hosts:
  - reviews        # DNS name trong cluster

  http:
  - match:
    - headers:
        x-user-type:
          exact: premium
    route:
    - destination:
        host: reviews
        subset: v2
      weight: 100

  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```

### Bước 2: DestinationRule - Subsets & policies

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews

  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50

  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

### Bước 3: Canary deployment

```yaml
# 95% v1, 5% v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 95
    - destination:
        host: myapp
        subset: v2
      weight: 5
```

### Bước 4: A/B Testing theo header

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  # Test group A
  - match:
    - headers:
        x-experiment:
          exact: "test-a"
    route:
    - destination:
        host: myapp
        subset: v1

  # Test group B
  - match:
    - headers:
        x-experiment:
          exact: "test-b"
    route:
    - destination:
        host: myapp
        subset: v2

  # Default
  - route:
    - destination:
        host: myapp
        subset: v1
```

### Bước 5: URL rewrite & redirect

```yaml
# Redirect /api/v1 -> /api/v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
  - api
  http:
  - match:
    - uri:
        prefix: /api/v1/
    redirect:
      uri: /api/v2/
      redirectCode: 301

  - match:
    - uri:
        prefix: /old
    rewrite:
      uri: /new
    route:
    - destination:
        host: api
        subset: v1
```

### Bước 6: Mirror (shadow traffic)

```yaml
# Gửi 100% traffic production + mirror sang v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 100
    mirror:
      host: myapp
      subset: v2
    mirrorPercentage:
      value: 100.0
```

### Bước 7: Fault injection

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 10
        fixedDelay: 5s
      abort:
        percentage:
          value: 5
        httpStatus: 500
    route:
    - destination:
        host: ratings
```

---

<a id="p2"></a>
## P2. Gateway & Ingress

### Bước 1: Istio Gateway

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: myapp-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - app.example.com
    - "*.example.com"
    tls:
      httpsRedirect: true

  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - app.example.com
    tls:
      mode: SIMPLE
      credentialName: app-tls-secret
      minProtocolVersion: TLSV1_2
```

### Bước 2: Gateway + VirtualService

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - app.example.com
  gateways:
  - myapp-gateway   # Phải khớp với Gateway name

  http:
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: api-service
        port:
          number: 8080

  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: web-service
        port:
          number: 80
```

### Bước 3: TLS với multiple hosts

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: multi-tls
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https-app
      protocol: HTTPS
    hosts:
    - app.example.com
    tls:
      mode: SIMPLE
      credentialName: app-tls

  - port:
      number: 443
      name: https-api
      protocol: HTTPS
    hosts:
    - api.example.com
    tls:
      mode: SIMPLE
      credentialName: api-tls

  - port:
      number: 15443
      name: https-passthrough
      protocol: HTTPS
    hosts:
    - direct.example.com
    tls:
      mode: PASSTHROUGH
```

### Bước 4: Cert rotation với cert-manager

```bash
# Cài cert-manager cho Istio
kubectl apply -f https://github.com/istio/cert-manager/releases/latest/download/cert-manager.yaml

# Tạo Certificate
```
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-cert
  namespace: istio-system
spec:
  secretName: app-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - app.example.com
```

### Bước 5: Backend mTLS

```yaml
# Gateway -> Service với mTLS
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-mtls
spec:
  host: api-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

### Bước 6: Egress Gateway

```yaml
# Egress cho external services
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: egress-gateway
  namespace: istio-system
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - api.external.com

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: egress-to-external
spec:
  host: api.external.com
  subsets:
  - name: external
    trafficPolicy:
      portLevelSettings:
      - port:
          number: 80
        connectionPool:
          http:
            h2UpgradePolicy: UPGRADE

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: direct-external
spec:
  hosts:
  - api.external.com
  gateways:
  - mesh
  - egress-gateway
  http:
  - match:
    - gateways:
      - mesh
    route:
    - destination:
        host: egress-gateway.istio-system.svc.cluster.local
        subset: external
      weight: 100
  - match:
    - gateways:
      - egress-gateway
    route:
    - destination:
        host: api.external.com
        port:
          number: 80
      weight: 100
```

---

<a id="p3"></a>
## P3. Security

### Bước 1: mTLS toàn cluster

```yaml
# Mesh policy - PeerAuthentication
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT     # Mọi traffic phải mTLS
```

### Bước 2: Per-namespace mTLS

```yaml
# Production namespace: strict
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

# Dev namespace: permissive (cho phép plaintext)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: dev
spec:
  mtls:
    mode: PERMISSIVE

# Disable cho 1 service
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-app
  namespace: production
spec:
  selector:
    matchLabels:
      app: legacy-app
  mtls:
    mode: PERMISSIVE
```

### Bước 3: AuthorizationPolicy - RBAC

```yaml
# Cho phép GET từ frontend
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-rbac
  namespace: production
spec:
  selector:
    matchLabels:
      app: api

  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/production/sa/frontend-sa"
    - source:
        principals:
        - "cluster.local/ns/production/sa/monitoring-sa"

    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/*"]

  - from:
    - source:
        principals:
        - "cluster.local/ns/production/sa/admin-sa"

    to:
    - operation:
        methods: ["GET", "POST", "PUT", "DELETE"]
        paths: ["*"]

  # Deny all other
```

### Bước 4: Authorization với JWT

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: api
  jwtRules:
  - issuer: "https://auth.example.com"
    jwksUri: "https://auth.example.com/.well-known/jwks.json"
    audiences:
    - "api.example.com"
    forwardOriginalToken: true

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
spec:
  selector:
    matchLabels:
      app: api
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals: ["*"]
```

### Bước 5: RequestAuthentication chi tiết

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: api-auth
spec:
  selector:
    matchLabels:
      app: api
  jwtRules:
  - issuer: "https://auth.example.com"
    jwksUri: "https://auth.example.com/.well-known/jwks.json"
    audiences:
    - "api.example.com"
    # Optional claims match
    forwardOriginalToken: true
    outputClaimToHeaders:
    - header: x-user-id
      claim: sub
    - header: x-user-roles
      claim: roles
```

### Bước 6: Network policies thay thế

```yaml
# Istio AuthorizationPolicy thay thế NetworkPolicy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ip-whitelist
spec:
  selector:
    matchLabels:
      app: admin-api
  action: ALLOW
  rules:
  - from:
    - source:
        ipBlocks:
        - 10.0.0.0/8
        - 192.168.0.0/16
```

---

<a id="p4"></a>
## P4. Observability

### Bước 1: Prometheus

```bash
# Istio đã có Prometheus tích hợp
# Scrape config: /etc/prometheus/prometheus.yml

# Metrics quan trọng
istio_requests_total{destination_service=~".*", response_code="500"}
istio_request_duration_milliseconds_bucket{...}
istio_active_connections
istio_tcp_connections_opened_total
istio_tcp_connections_closed_total
```

```yaml
# ServiceMonitor cho Istio
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-component-monitor
  namespace: istio-system
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: istiod
  endpoints:
  - port: http-monitoring
```

### Bước 2: Kiali

```bash
# Cài Kiali (đã có trong demo profile)
kubectl apply -f samples/addons/kiali.yaml

# Truy cập
istioctl dashboard kiali

# Hoặc port-forward
kubectl port-forward -n istio-system svc/kiali 20001:20001
```

```yaml
# ConfigMap cho Kiali
apiVersion: v1
kind: ConfigMap
metadata:
  name: kiali
  namespace: istio-system
data:
  config.yaml: |
    auth:
      strategy: openshift    # openid, openshift, ldap, header
    server:
      metrics_port: 9090
      health_check_enabled: true
    external_services:
      prometheus:
        url: http://prometheus:9090
      tracing:
        url: http://jaeger:16685
        use_grpc: true
```

### Bước 3: Jaeger (distributed tracing)

```bash
# Cài Jaeger
kubectl apply -f samples/addons/jaeger.yaml

# Truy cập
istioctl dashboard jaeger

# Config sampling
```
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio
  namespace: istio-system
data:
  mesh: |
    defaultConfig:
      tracing:
        sampling: 100   # 100% sampling trong dev
        # sampling: 1   # 1% trong production
        zipkin:
          address: zipkin.istio-system:9411
```

### Bước 4: Access logs

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
  - providers:
    - name: envoy
    defaultConfig:
      mode: "envoy"
      # Filter chỉ log error
      filter:
        expression: "response.code >= 400"
```

### Bước 5: Metrics customization

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: custom-metrics
spec:
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - tagOverrides:
        request_host:
          operation: UPSTREAM_REQUEST_HOST
      match:
        metric: REQUEST_COUNT
    - tagOverrides:
        response_code:
          operation: RESPONSE_CODE
        request_path:
          operation: REQUEST_PATH
      match:
        metric: REQUEST_DURATION
```

---

<a id="p5"></a>
## P5. Resilience

### Bước 1: Retries

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
  - api
  http:
  - route:
    - destination:
        host: api
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn:
      - "5xx"
      - "reset"
      - "connect-failure"
      - "envoy-failure"
      retryRemoteLocalities: true
```

### Bước 2: Timeouts

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
  - api
  http:
  - timeout: 10s         # Total timeout
    route:
    - destination:
        host: api

# Per-route timeout
  - match:
    - uri:
        prefix: /slow
    timeout: 60s
    route:
    - destination:
        host: slow-api
```

### Bước 3: Circuit Breaker

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api
spec:
  host: api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
        tcpKeepalive:
          time: 7200s
          interval: 75s
      http:
        h2UpgradePolicy: UPGRADE
        maxRequestsPerConnection: 10
        maxRetries: 3
        consecutive5xxErrors: 5
        interval: 30s
        baseEjectionTime: 30s
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
```

### Bước 4: Connection pooling

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: db-pool
spec:
  host: postgres
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30s
      http:
        maxRequestsPerConnection: 100
        maxPendingRequests: 200
        maxRequests: 1000
        maxRetries: 5
        consecutive5xxErrors: 10
        interval: 30s
        baseEjectionTime: 30s
```

### Bước 5: Rate limiting với EnvoyFilter

```yaml
apiVersion: networking.istio.io/v1alpha1
kind: EnvoyFilter
metadata:
  name: rate-limit
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
            subFilter:
              name: envoy.filters.http.router
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.local_ratelimit
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
          stat_prefix: http_local_rate_limiter
          token_bucket:
            max_tokens: 100
            tokens_per_fill: 100
            fill_interval: 60s
          filter_enabled:
            runtime_key: local_rate_limit_enabled
            default_value:
              numerator: 100
              denominator: HUNDRED
          filter_enforced:
            runtime_key: local_rate_limit_enforced
            default_value:
              numerator: 100
              denominator: HUNDRED
```

### Bước 6: Health checking

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-health
spec:
  host: api
  trafficPolicy:
    outlierDetection:
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      consecutive5xxErrors: 5
      consecutiveGatewayErrors: 5
      splitExternalLocalOriginErrors: true
```

---

<a id="p6"></a>
## P6. Multi-Cluster

### Bước 1: Primary-Remote

```bash
# Cluster 1: primary (control plane)
# Cluster 2: remote (chỉ gateway)

# Cài primary
istioctl install --set profile=remote -y \
  --set values.global.remotePilotAddress=<primary-pod-ip>

# Cài remote
istioctl install --set profile=remote -y
```

### Bước 2: Multi-Primary

```yaml
# Cài trên cả 2 clusters
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      multiCluster:
        enabled: true
      meshID: mesh1
      network:
        - network1
```

### BƯớc 3: Cross-cluster service discovery

```bash
# Install east-west gateway
samples/multicluster/gen-eastwest-gateway.sh \
  --mesh mesh1 --cluster-name cluster1 --network network1 | kubectl apply -f -

# Apply remote secret to cluster 1
istioctl x create-remote-secret \
  --name=cluster2 \
  --context=cluster2 | kubectl apply -f -
```

### Bước 4: ServiceEntry cho external services

```yaml
# External service accessible từ mesh
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
  - api.external.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

### BƯớc 5: Failover giữa clusters

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-failover
spec:
  host: api.production.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
```

---

<a id="p7"></a>
## P7. Production Best Practices

### Bước 1: Upgrade Istio

```bash
# Check version compatible với K8s
istioctl x precheck

# Upgrade canary (control plane only)
istioctl upgrade --revision=1-21-2

# Migrate data plane
# Rollout restart tất cả workloads để dùng revision mới
kubectl rollout restart deployment -n default

# Verify
istioctl proxy-status
```

### Bước 2: Performance tuning

```yaml
# Sidecar resource limits
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-sidecar-injector
  namespace: istio-system
data:
  values: |-
    global:
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 2000m
            memory: 1024Mi
```

### Bước 3: Ambient Mesh (sidecar-less)

```bash
# Cài ambient
istioctl install --set profile=ambient -y

# Migrate namespace
kubectl label namespace default istio.io/dataplane-mode=ambient

# Verify
istioctl experimental describe pod <pod-name>
```

### BƯớc 4: External traffic management

```yaml
# EnvoyFilter thêm header
apiVersion: networking.istio.io/v1alpha1
kind: EnvoyFilter
metadata:
  name: add-header
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.lua
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
          inlineCode: |
            function envoy_on_request(request_handle)
              request_handle:headers():add("x-custom-header", "value")
            end
```

### Bước 5: Troubleshooting

```bash
# Analyze cluster
istioctl analyze -A

# Proxy status
istioctl proxy-status

# Endpoint info
istioctl proxy-config endpoints <pod>.<ns>
istioctl proxy-config cluster <pod>.<ns>
istioctl proxy-config listener <pod>.<ns>
istioctl proxy-config route <pod>.<ns>

# Logs control plane
kubectl logs -n istio-system -l app=istiod --tail 100

# Logs sidecar
kubectl logs <pod-name> -c istio-proxy

# Debug envoy
kubectl exec <pod> -c istio-proxy -- pilot-agent request GET /debug/...

# Test connectivity
kubectl exec <pod> -c istio-proxy -- curl http://service.namespace:port
```

```bash
# Common issues

# 1. Sidecar không inject
kubectl get pod -n <ns> -o jsonpath='{.items[*].spec.containers[*].name}'
# Phải có 2 containers: app + istio-proxy

# 2. Service không reachable
istioctl proxy-config services <pod>
# Check service registry

# 3. 503 errors
# Check DestinationRule subsets labels match pod labels
kubectl get pod --show-labels

# 4. mTLS failure
# Check PeerAuthentication mode
kubectl get peerauthentication -A
```

### Bước 6: Production checklist

```yaml
# 1. mTLS STRICT
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT

# 2. AuthorizationPolicy (deny by default)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec: {}    # Empty spec = deny all

# 3. Resource limits cho sidecar
# 4. Sampling rate hợp lý (1-10% production)
# 5. Prometheus + Grafana dashboards
# 6. Alert khi sync status != Healthy
# 7. Backup Istio config (kubectl get -o yaml > backup.yaml)
# 8. Test upgrade trên staging trước
```

---

## 🎯 Bài tập P0-P7

1. Cài Istio lên K3s/Minikube, deploy Bookinfo
2. Configure VirtualService cho canary deployment 90/10
3. Setup mTLS STRICT cho namespace production
4. AuthorizationPolicy chỉ cho phép frontend gọi API
5. Configure retry + timeout cho slow API
6. Cài Kiali + Jaeger, xem service graph và traces
7. Deploy multi-cluster với 2 K3s clusters

---

> **💡 Tip cuối**: Istio rất mạnh nhưng phức tạp. Bắt đầu với VirtualService + DestinationRule + mTLS. Thêm AuthorizationPolicy khi đã thành thạo. Ambient mesh là tương lai, sidecar classic vẫn stable. Luôn dùng `istioctl analyze` trước khi apply.

---

*Tạo bởi tài liệu học Istio - Chúc bạn thành công! 🚀*
