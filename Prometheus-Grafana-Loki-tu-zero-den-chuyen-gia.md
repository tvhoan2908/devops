# 📊 Tài liệu Prometheus + Grafana + Loki: Từ Zero đến Chuyên gia

> **Mục tiêu**: Xây dựng hệ thống observability hoàn chỉnh cho hạ tầng và ứng dụng.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Chuẩn bị](#p0) | Cài đặt stack |
| [P1. Prometheus cơ bản](#p1) | Metrics, PromQL, exporter |
| [P2. Prometheus nâng cao](#p2) | Recording rules, alert rules, federation |
| [P3. Grafana cơ bản](#p3) | Dashboard, panel, biến |
| [P4. Grafana nâng cao](#p4) | Alerting, datasource, provisioning |
| [P5. Loki & Promtail](#p5) | LogQL, labels, log pipeline |
| [P6. Tempo & Tracing](#p6) | Distributed tracing |
| [P7. K8s monitoring](#p7) | kube-prometheus-stack, servicemonitor |
| [P8. Production](#p8) | Scaling, HA, retention, cost |

---

<a id="p0"></a>
## P0. Chuẩn bị môi trường

### Bước 1: Docker Compose stack

```bash
mkdir ~/observability-lab && cd ~/observability-lab
```
```yaml
# docker-compose.yml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: prometheus
    volumes:
      - ./prometheus:/etc/prometheus
      - prom-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    volumes:
      - ./alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    ports:
      - "9093:9093"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:11.1.0
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    restart: unless-stopped

  loki:
    image: grafana/loki:3.1.0
    container_name: loki
    command: -config.file=/etc/loki/local-config.yaml
    ports:
      - "3100:3100"
    volumes:
      - ./loki:/etc/loki
    restart: unless-stopped

  promtail:
    image: grafana/promtail:3.1.0
    container_name: promtail
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail:/etc/promtail
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    command:
      - '--path.rootfs=/host'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /:/host:ro,rslave
    ports:
      - "9100:9100"
    restart: unless-stopped

volumes:
  prom-data:
  grafana-data:
```
```bash
mkdir -p prometheus alertmanager grafana/provisioning loki promtail
docker compose up -d
```

### Bước 2: Truy cập các UI

```
Prometheus:  http://localhost:9090
Alertmanager: http://localhost:9093
Grafana:     http://localhost:3000 (admin/admin)
Loki:        http://localhost:3100
Node Exp:    http://localhost:9100/metrics
```

---

<a id="p1"></a>
## P1. Prometheus cơ bản

### Bước 1: Cấu hình prometheus.yml

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: production
    region: us-east-1

scrape_configs:
  # Prometheus itself
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  # Node exporter
  - job_name: node
    static_configs:
      - targets: ['node-exporter:9100']

  # Application (custom metrics)
  - job_name: myapp
    metrics_path: /metrics
    scrape_interval: 10s
    static_configs:
      - targets: ['myapp:8080']
        labels:
          service: api
          env: production

  # Kubernetes service discovery
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
```

### Bước 2: Metric types

```yaml
# 4 loại metrics Prometheus thu thập:

# 1. Counter - chỉ tăng
# Ví dụ: số request, số lỗi
http_requests_total{service="api", status="200"} 1234

# 2. Gauge - tăng/giảm
# Ví dụ: CPU%, memory, queue size
node_cpu_percent 45.2

# 3. Histogram - phân phối giá trị trong bucket
# Ví dụ: request duration
http_request_duration_seconds_bucket{le="0.1"} 100
http_request_duration_seconds_bucket{le="0.5"} 250
http_request_duration_seconds_bucket{le="1.0"} 300
http_request_duration_seconds_sum 120.5
http_request_duration_seconds_count 350

# 4. Summary - tương tự histogram, có quantile
http_request_duration_seconds{quantile="0.5"} 0.234
http_request_duration_seconds{quantile="0.9"} 0.567
http_request_duration_seconds{quantile="0.99"} 1.234
http_request_duration_seconds_sum 120.5
http_request_duration_seconds_count 350
```

### Bước 3: PromQL - ngôn ngữ truy vấn

```promql
# === BASIC QUERIES ===
# Chọn metric
node_cpu_seconds_total

# Filter by label
node_cpu_seconds_total{mode="idle"}

# Multiple labels
node_cpu_seconds_total{mode="idle", cpu="0"}

# Regex
node_cpu_seconds_total{mode=~"idle|iowait"}

# Negation
node_cpu_seconds_total{mode!="idle"}

# === AGGREGATION ===
# Sum
sum(node_cpu_seconds_total)

# By labels
sum(node_cpu_seconds_total) by (instance)

# Count
count(up == 1)

# Average
avg(node_cpu_memory_MemFree_bytes)

# Min/Max
min(node_load1)
max(node_load1)

# === RATE & INCREASE ===
# Rate per second (cho counter)
rate(node_network_receive_bytes_total[5m])

# Increase (tổng tăng trong khoảng)
increase(http_requests_total[1h])

# Irate (tính trên 2 điểm gần nhất)
irate(node_cpu_seconds_total[1m])

# === COMMON QUERIES ===
# CPU usage %
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage %
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage %
100 - ((node_filesystem_avail_bytes{fstype!="tmpfs"} * 100) / node_filesystem_size_bytes)

# Request rate per second
rate(http_requests_total[5m])

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
  /
sum(rate(http_requests_total[5m])) by (service)

# P95 latency
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))

# P99 latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))

# Network bandwidth
rate(node_network_receive_bytes_total[5m])

# Top 10 CPU consumers
topk(10, sum(rate(container_cpu_usage_seconds_total[5m])) by (pod))

# Pods not ready
kube_pod_status_ready{condition="true"} == 0

# Container restarts
increase(kube_pod_container_status_restarts_total[1h]) > 0

# === FUNCTIONS ===
# Time offset
node_cpu_seconds_total offset 1h

# Math
node_memory_MemTotal_bytes / 1024 / 1024  # MB

# Abs
abs(node_load5_change[5m])

# Round
round(node_cpu_seconds_total)

# Sort
sort_desc(sum by(pod) (rate(container_cpu_usage_seconds_total[5m])))

# Standard deviation
stddev_over_time(node_load1[1h])
```

### Bước 4: Exporters phổ biến

```bash
# Node/system metrics
prom/node-exporter         # CPU, memory, disk, network
prom/process-exporter      # Per-process metrics
prom/redis-exporter        # Redis
prom/mysqld-exporter       # MySQL/MariaDB
prom/postgres-exporter     # PostgreSQL
prom/mongodb-exporter      # MongoDB
prom/nginx-exporter        # Nginx (nginx-prometheus-exporter)
prom/blackbox-exporter     # HTTP/TCP/ICMP probes
prom/rabbitmq-exporter     # RabbitMQ
prom/haproxy-exporter      # HAProxy
prom/kafka-exporter        # Kafka
```

```yaml
# Ví dụ: scrape MySQL
- job_name: mysql
  static_configs:
    - targets: ['mysql-exporter:9104']
  metric_relabel_configs:
    - source_labels: [__name__]
      regex: '(mysql_global_status|mysql_global_variables|mysql_info)'
      action: keep
```

### Bước 5: Application instrumentation

```python
# Python (Flask)
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from flask import Flask, Response
import time

app = Flask(__name__)

REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint']
)

@app.before_request
def start_timer():
    from flask import g
    g.start = time.time()

@app.after_request
def record_metrics(response):
    from flask import request, g
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.path,
        status=response.status_code
    ).inc()
    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=request.path
    ).observe(time.time() - g.start)
    return response

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

@app.route('/api/users')
def users():
    return {'users': []}
```

```javascript
// Node.js (Express)
const client = require('prom-client');

const register = new client.Registry();
client.collectDefaultMetrics({ register });

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.1, 0.5, 1, 2, 5]
});
register.registerMetric(httpRequestDuration);

app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    end({
      method: req.method,
      route: req.route?.path || req.path,
      status: res.statusCode
    });
  });
  next();
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

```go
// Go (chi router)
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
)

var (
    requests = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
    latency = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
)

func main() {
    prometheus.MustRegister(requests, latency)

    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

---

<a id="p2"></a>
## P2. Prometheus nâng cao

### Bước 1: Recording Rules (pre-computed queries)

```yaml
# prometheus/rules/recording.yml
groups:
  - name: api.recording
    interval: 30s
    rules:
      # CPU usage per pod
      - record: pod:cpu_usage:rate5m
        expr: |
          sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod, namespace)

      # Memory usage
      - record: pod:memory_usage:bytes
        expr: |
          sum(container_memory_working_set_bytes{namespace="production"}) by (pod, namespace)

      # Request rate per service
      - record: service:http_requests:rate5m
        expr: |
          sum(rate(http_requests_total[5m])) by (service, status)

      # Error ratio
      - record: service:http_error_ratio:rate5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            /
          sum(rate(http_requests_total[5m])) by (service)

      # P95 latency
      - record: service:http_latency:p95
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          )
```

### Bước 2: Alert Rules

```yaml
# prometheus/rules/alerts.yml
groups:
  - name: api.alerts
    rules:
      - alert: HighCPUUsage
        expr: |
          100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU > 80% for 5 minutes (current: {{ $value }}%)"
          runbook_url: "https://wiki.example.com/runbooks/high-cpu"

      - alert: HighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory on {{ $labels.instance }}"

      - alert: DiskWillFillIn4Hours
        expr: |
          predict_linear(node_filesystem_avail_bytes{fstype!="tmpfs"}[6h], 4*3600) < 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Disk will fill in 4h on {{ $labels.instance }}"

      - alert: PodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[10m]) > 0
          and
          kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"

      - alert: APILatencyHigh
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 latency > 1s for {{ $labels.service }}"

      - alert: ErrorRateHigh
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            /
          sum(rate(http_requests_total[5m])) by (service) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 5% for {{ $labels.service }}"

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} on {{ $labels.instance }} is down"
```

### Bước 3: Alertmanager

```yaml
# alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: '[email protected]'
  smtp_auth_username: '[email protected]'
  smtp_auth_password: 'app-password'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true
    - match:
        team: platform
      receiver: 'platform-team'

receivers:
  - name: 'default'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }} ({{ .Status }})'
        text: |
          {{ range .Alerts }}
          *{{ .Annotations.summary }}*
          {{ .Annotations.description }}
          {{ end }}

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'pagerduty-key'
        description: '{{ .GroupLabels.alertname }} - {{ .CommonAnnotations.summary }}'

  - name: 'platform-team'
    email_configs:
      - to: '[email protected]'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster']
```

### Bước 4: Federation & Remote Write

```yaml
# Global Prometheus -> Thanos/Cortex/Mimir
remote_write:
  - url: "http://cortex:9009/api/v1/push"
    write_relabel_configs:
      - source_labels: [__name__]
        regex: 'go_.*|process_.*|prometheus_.*'
        action: drop

# Federation giữa các Prometheus
- job_name: federate
  scrape_interval: 30s
  honor_labels: true
  metrics_path: /federate
  params:
    'match[]':
      - '{job="prometheus"}'
      - '{__name__=~"job:.*"}'
  static_configs:
    - targets: ['other-prometheus:9090']
```

### Bước 5: ServiceMonitor & PodMonitor (K8s)

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  labels:
    release: kube-prom
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
    scrapeTimeout: 10s
    relabelings:
      - sourceLabels: [__meta_kubernetes_pod_phase]
        regex: (Running|Succeeded)
        action: keep
```

```yaml
# podmonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp-pods
  labels:
    release: kube-prom
spec:
  selector:
    matchLabels:
      app: myapp
  podMetricsEndpoints:
  - port: metrics
    path: /metrics
```

---

<a id="p3"></a>
## P3. Grafana cơ bản

### Bước 1: Thêm Datasource

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: 15s
      queryTimeout: 60s
      httpMethod: POST

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
    jsonData:
      maxLines: 1000

  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
    editable: false
    jsonData:
      httpMethod: GET
      serviceMap:
        datasourceUid: prometheus
```

### Bước 2: Tạo Dashboard

```json
{
  "dashboard": {
    "title": "Application Overview",
    "uid": "app-overview",
    "tags": ["production", "app"],
    "timezone": "browser",
    "time": { "from": "now-6h", "to": "now" },
    "refresh": "30s",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps",
            "custom": {
              "drawStyle": "line",
              "lineWidth": 2,
              "fillOpacity": 10
            }
          }
        }
      },
      {
        "id": 2,
        "title": "P95 Latency",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
            "legendFormat": "p95 {{service}}",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": { "unit": "s" }
        }
      },
      {
        "id": 3,
        "title": "Error Rate",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 8 },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "color": "green", "value": null },
                { "color": "yellow", "value": 1 },
                { "color": "red", "value": 5 }
              ]
            }
          }
        }
      }
    ]
  }
}
```

### Bước 3: Dashboard Variables

```json
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_pod_info, namespace)",
        "refresh": 2,
        "multi": true,
        "includeAll": true
      },
      {
        "name": "pod",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_pod_info{namespace=\"$namespace\"}, pod)",
        "multi": true,
        "includeAll": true
      },
      {
        "name": "interval",
        "type": "interval",
        "auto": true,
        "auto_min": "30s",
        "auto_count": 30,
        "options": ["30s", "1m", "5m", "15m", "1h"]
      }
    ]
  }
}
```

### Bước 4: Provisioning Dashboard

```yaml
# grafana/provisioning/dashboards/provider.yml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: 'General'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 60
    allowUiUpdates: true
    options:
      path: /etc/grafana/dashboards
      foldersFromFilesStructure: true
```

```bash
# Cấu trúc
grafana/
├── provisioning/
│   ├── datasources/
│   │   └── prometheus.yml
│   └── dashboards/
│       └── provider.yml
└── dashboards/
    ├── api-overview.json
    ├── k8s-cluster.json
    └── node-metrics.json
```

### Bước 5: Panel types hay dùng

```
- timeseries       # Line chart
- barchart         # Bar chart
- stat             # Big number
- gauge            # Gauge
- table            # Table
- piechart         # Pie chart
- heatmap          # Heatmap
- bargauge         # Bar gauge
- logs             # Logs panel (Loki)
- traces           # Traces panel (Tempo)
- nodeGraph        # Service map
- geomap           # Map
- text             # Markdown/HTML
- row              # Section divider
- histogram        # Histogram
```

---

<a id="p4"></a>
## P4. Grafana nâng cao

### Bước 1: Alerting trong Grafana

```yaml
# grafana/provisioning/alerting/contact-points.yml
apiVersion: 1

contactPoints:
  - orgId: 1
    name: slack
    receivers:
      - uid: slack-1
        type: slack
        settings:
          url: https://hooks.slack.com/services/xxx
          channel: '#alerts'

  - orgId: 1
    name: pagerduty
    receivers:
      - uid: pd-1
        type: pagerduty
        settings:
          integrationKey: xxx
```

```yaml
# alert-rules.yml
apiVersion: 1

groups:
  - orgId: 1
    name: api-alerts
    folder: API
    interval: 1m
    rules:
      - uid: api-error-rate
        title: HighErrorRate
        condition: C
        data:
          - refId: A
            datasourceUid: prometheus
            relativeTimeRange:
              from: 300
              to: 0
            model:
              expr: sum(rate(http_requests_total{status=~"5.."}[5m]))
                    / sum(rate(http_requests_total[5m]))
          - refId: C
            datasourceUid: __expr__
            model:
              type: threshold
              expression: A
              conditions:
                - evaluator:
                    type: gt
                    params: [0.05]
        noDataState: NoData
        execErrState: Alerting
        for: 5m
        annotations:
          summary: "Error rate > 5%"
        labels:
          severity: critical
        isPaused: false
```

### Bước 2: Dashboard JSON model hoàn chỉnh

```bash
# Export dashboard từ UI: Share > Export > Save to file
# Hoặc dùng API:
curl -H "Authorization: Bearer $TOKEN" \
  http://grafana:3000/api/dashboards/uid/app-overview \
  > dashboard.json

# Import:
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://grafana:3000/api/dashboards/import \
  -d @dashboard.json
```

---

<a id="p5"></a>
## P5. Loki & Promtail

### Bước 1: Loki config

```yaml
# loki/loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /loki

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  tsdb_shipper:
    active_index_directory: /loki/tsdb-index
    cache_location: /loki/tsdb-cache
  filesystem:
    directory: /loki/chunks

limits_config:
  retention_period: 30d
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  max_query_parallelism: 32
```

### Bước 2: Promtail

```yaml
# promtail/config.yml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Docker logs
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'

  # System logs
  - job_name: system
    static_configs:
      - targets: [localhost]
        labels:
          job: syslog
          __path__: /var/log/*.log

  # K8s logs
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

### Bước 3: LogQL

```promql
# Basic: chọn stream
{app="myapp"}

# Filter content
{app="myapp"} |= "error"

# Không chứa
{app="myapp"} != "debug"

# Regex
{app="myapp"} |~ "timeout|connection refused"

# Parse JSON
{app="myapp"} | json | level="error"

# Extract field
{app="myapp"} | logfmt | status="500"

# Line filter với regex
{app="myapp"} |~ "user_id=(?P<user_id>\\w+)"

# Filter sau parse
{app="myapp"} | json | latency > 1

# Aggregations
count_over_time({app="myapp"}[5m])
sum by (level) (count_over_time({app="myapp"}[5m]))

# Rate
rate({app="myapp"}[5m])

# Top errors
topk(10, sum by (error) (rate({app="myapp"} |= "error" [5m])))

# Template variables
{app=~"$app", namespace=~"$namespace"}

# Multi-line
{app="myapp"} |= "Started"
{app="myapp"} |= "error"
```

### Bước 4: Structured logging tốt

```python
# Python - JSON logs
import json, logging, sys
from pythonjsonlogger import jsonlogger

logger = logging.getLogger()
handler = logging.StreamHandler(sys.stdout)
formatter = jsonlogger.JsonFormatter(
    '%(asctime)s %(levelname)s %(name)s %(message)s'
)
handler.setFormatter(formatter)
logger.addHandler(handler)

logger.info("User logged in", extra={
    "user_id": "123",
    "request_id": "abc-xyz",
    "duration_ms": 45
})
```

```javascript
// Node.js - Winston JSON
const winston = require('winston');
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [new winston.transports.Console()]
});
logger.info('User login', {
  user_id: '123',
  request_id: 'abc',
  duration_ms: 45
});
```

---

<a id="p6"></a>
## P6. Tempo & Distributed Tracing

### Bước 1: Tempo setup

```yaml
# tempo/tempo-config.yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

ingester:
  trace_idle_period: 10s
  max_block_duration: 5m

compactor:
  compaction:
    block_retention: 48h

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
    wal:
      path: /var/tempo/wal
```

### Bước 2: Application OpenTelemetry

```python
# Python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://tempo:4317"))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

tracer = trace.get_tracer(__name__)

@app.route("/api/users/<id>")
def get_user(id):
    with tracer.start_as_current_span("get_user") as span:
        span.set_attribute("user.id", id)
        user = db.query(id)
        span.set_attribute("user.found", user is not None)
        return user
```

```javascript
// Node.js
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { Resource } = require('@opentelemetry/resources');

const sdk = new NodeSDK({
  resource: new Resource({
    'service.name': 'myapp',
    'service.version': '1.0.0'
  }),
  traceExporter: new OTLPTraceExporter({
    url: 'http://tempo:4318/v1/traces'
  })
});
sdk.start();
```

### Bước 3: Service Map

```json
{
  "type": "nodeGraph",
  "title": "Service Map",
  "datasource": {
    "type": "prometheus",
    "uid": "prometheus"
  },
  "targets": [
    {
      "expr": "sum(rate(http_requests_total[5m])) by (source, destination)"
    }
  ]
}
```

### Bước 4: TraceQL

```promql
# Tìm traces có lỗi
{ status = error }

# Tìm traces chậm
{ duration > 5s }

# Tìm theo service
{ resource.service.name = "myapp" }

# Kết hợp
{ resource.service.name = "api" && status = error && span.http.status_code = 500 }

# Span có attribute
{ span.db.statement =~ "SELECT.*users.*" }
```

---

<a id="p7"></a>
## P7. K8s Monitoring với kube-prometheus-stack

### Bước 1: Cài đặt

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.ruleSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword='admin' \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=10Gi
```

### Bước 2: Truy cập

```bash
# Grafana
kubectl port-forward -n monitoring svc/kube-prom-grafana 3000:80

# Prometheus
kubectl port-forward -n monitoring svc/kube-prom-prometheus 9090:9090

# Alertmanager
kubectl port-forward -n monitoring svc/kube-prom-alertmanager 9093:9093

# Lấy Grafana password
kubectl get secret -n monitoring kube-prom-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

### Bước 3: ServiceMonitor cho app

```yaml
# app-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: monitoring
  labels:
    release: kube-prom
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
```

### Bước 4: PrometheusRule

```yaml
# prometheus-rule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
  labels:
    release: kube-prom
spec:
  groups:
  - name: app
    interval: 30s
    rules:
    - alert: PodOOMKilled
      expr: |
        kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
      for: 0m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} OOMKilled"
```

### Bước 5: Dashboard nâng cao

```bash
# Import dashboard JSON từ grafana.com/dashboards
# Ví dụ: dashboard ID 315 (Kubernetes cluster monitoring)

curl -X POST -H "Content-Type: application/json" \
  http://admin:[email protected]:3000/api/dashboards/import \
  -d '{
    "dashboard": <content>,
    "inputs": [{"name": "DS_PROMETHEUS", "type": "datasource", "pluginId": "prometheus", "value": "prometheus"}]
  }'
```

---

<a id="p8"></a>
## P8. Production Best Practices

### Bước 1: HA Prometheus

```yaml
# Dùng Thanos hoặc Cortex/Mimir
# Thanos đơn giản nhất cho bắt đầu

# prometheus.yml
global:
  external_labels:
    replica: $(POD_NAME)

# K8s StatefulSet với 2 replica
# Thêm sidecar Thanos
spec:
  containers:
    - name: prometheus
      ...
    - name: thanos-sidecar
      image: thanosio/thanos:v0.36.0
      args:
        - sidecar
        - --prometheus.url=http://localhost:9090
        - --tsdb.path=/prometheus
        - --gcs.bucket=thanos-bucket
        - --objstore.config-file=/etc/thanos/objectstore.yaml
```

### Bước 2: Retention & Storage

```bash
# Tính toán:
# 1 pod = ~1KB metric
# 1000 pods, 1 metric/pod/15s = 66667 samples/15s = 4.4M samples/min
# 30 ngày retention = ~190GB raw, ~50GB compressed
```

```yaml
# Storage class cho Prometheus (fast SSD)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: prometheus-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "5000"
  throughput: "250"
volumeBindingMode: WaitForFirstConsumer
```

### Bước 3: Cardinality management

```yaml
# Tránh label có cardinality cao (user_id, request_id)
# Chỉ dùng label: env, service, status, version, region

# Drop high-cardinality labels trong relabel_configs
metric_relabel_configs:
  - source_labels: [user_id]
    regex: '.*'
    action: labeldrop

# Drop unused metrics
  - source_labels: [__name__]
    regex: 'go_gc_.*|go_memstats_.*'
    action: drop

# Chỉ giữ metrics cần thiết
  - source_labels: [__name__]
    regex: 'http_requests_total|http_request_duration_seconds.*'
    action: keep
```

### Bước 4: Cost optimization

```yaml
# Dùng Mimir/Cortex cho long-term storage (cheaper)
# Downsampling ở recording rules
# Drop high-cardinality labels
# Giảm scrape_interval cho non-critical metrics
# Giảm retention ở local, dùng remote storage cho long-term
```

### Bước 5: Monitoring stack itself (meta-monitoring)

```yaml
# Dùng Prometheus + Grafana để monitor... chính nó
# Alert khi Prometheus không scrape được
- alert: PrometheusTargetMissing
  expr: up == 0
  for: 5m

- alert: PrometheusTSDBCorruption
  expr: prometheus_tsdb_compactions_failed_total > 0

- alert: PrometheusNotConnected
  expr: prometheus_notifications_alertmanagers_discovered < 1

- alert: HighIngestionRate
  expr: rate(prometheus_tsdb_head_series_created_total[5m]) > 100
```

### 🎯 Bài tập P0-P8
1. Cài stack Prometheus + Grafana + Loki bằng docker-compose
2. Instrument 1 app Python/Node expose metrics
3. Tạo dashboard với 5 panels: CPU, memory, req/s, error rate, latency
4. Cấu hình alert Slack khi service down
5. Dùng kube-prometheus-stack cho K8s cluster
6. Setup LogQL query cho errors
7. Tracing end-to-end với Tempo

---

> **💡 Tip cuối**: Observability không chỉ là metrics, logs và traces mà là khả năng trả lời câu hỏi "Tại sao?". Đầu tư vào label naming convention và dashboard từ đầu.

---

*Tạo bởi tài liệu học Observability - Chúc bạn thành công! 🚀*
