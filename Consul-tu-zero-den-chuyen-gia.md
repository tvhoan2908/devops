# 🏛️ Tài liệu Consul: Từ Zero đến Chuyên gia

> **Mục tiêu**: Thành thạo HashiCorp Consul: service discovery, service mesh, KV store, ACL, production HA.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Chuẩn bị](#p0) | Cài Consul, kiến trúc |
| [P1. Service Discovery](#p1) | Agent, registration, health check |
| [P2. DNS & HTTP API](#p2) | Query, watch, tag |
| [P3. Service Mesh](#p3) | Connect, intentions, mTLS |
| [P4. Security](#p4) | ACL, gossip encryption, TLS |
| [P5. KV Store](#p5) | Read, write, watch, lock |
| [P6. Consul Template](#p6) | Auto-render config |
| [P7. HA & Multi-DC](#p7) | Cluster, federation, WAN |
| [P8. Production](#p8) | Backup, monitoring, troubleshooting |

---

<a id="p0"></a>
## P0. Chuẩn bị

### Bước 1: Cài Consul

```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/consul

# Linux (Debian/Ubuntu)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install consul

# Windows (scoop)
scoop install consul

# Verify
consul version
```

### Bước 2: Single node dev mode

```bash
# Dev mode (mọi thứ ở localhost, không persist)
consul agent -dev

# Trong terminal khác - test
consul members
dig @127.0.0.1 -p 8600 consul.service.consul
curl http://localhost:8500/v1/catalog/services
```

### Bước 3: Kiến trúc Consul

```
┌────────────────────────────────────────────────────────────┐
│                  Consul Cluster                             │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              Server nodes (3 or 5 recommended)       │ │
│  │  - Raft consensus                                     │ │
│  │  - Store catalog & KV                                │ │
│  │  - Anti-entropy                                       │ │
│  │  - Gossip protocol (LAN/WAN)                         │ │
│  │                                                       │ │
│  │   ┌──────┐    ┌──────┐    ┌──────┐                  │ │
│  │   │ S1   │◄──►│ S2   │◄──►│ S3   │                  │ │
│  │   │Leader│    │Follow│    │Follow│                  │ │
│  │   └──┬───┘    └──┬───┘    └──┬───┘                  │ │
│  └──────┼──────────┼──────────┼─────────────────────────┘ │
│         │          │          │                            │
│   Gossip│        Gossip│    Gossip│ (LAN gossip)            │
│         ▼          ▼          ▼                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              Client agents                            │ │
│  │  - Service registration                               │ │
│  │  - Health checks                                      │ │
│  │  - Local cache                                        │ │
│  │  - Connect sidecar proxy                              │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  WAN Federation:                                            │
│  DC1 ──────── WAN gossip ──────── DC2                     │
│  (Primary)                       (Secondary)               │
└────────────────────────────────────────────────────────────┘
```

### Bước 4: Ports

```
Server nodes:
- 8300  TCP - Server RPC
- 8301  TCP/UDP - Serf LAN gossip
- 8302  TCP - Serf WAN gossip (cho federation)

Client agents:
- 8500  HTTP - HTTP API & UI
- 8501  gRPC - gRPC API
- 8502  HTTP - Envoy xDS
- 8503  HTTP - Connect ca TLS
- 8600  DNS - DNS interface (UDP/TCP)

All nodes:
- 8301/udp - Gossip
```

### Bước 5: Cấu hình file

```hcl
# consul.hcl
datacenter = "dc1"
data_dir   = "/opt/consul/data"
log_level  = "INFO"

server           = true
bootstrap_expect = 3

# Network
client_addr = "0.0.0.0"
bind_addr   = "10.0.1.10"
advertise_addr = "10.0.1.10"

# UI
ui_config {
  enabled = true
}

# Connect (Service Mesh)
connect {
  enabled = true
}

# ACL
acl {
  enabled        = true
  default_policy = "deny"
  enable_token_persistence = true
}

# TLS
tls {
  defaults {
    ca_file   = "/etc/consul/ca.pem"
    cert_file = "/etc/consul/server.pem"
    key_file  = "/etc/consul/server-key.pem"
    verify_incoming = true
    verify_outgoing = true
  }

  internal_rpc {
    verify_server_hostname = true
  }
}

# Auto reload (reload config on file change)
auto_reload_config = true

# Performance
performance {
  raft_multiplier = 1
}
```

```bash
# Start
consul agent -config-file=./consul.hcl

# Or with HCL directory
consul agent -config-dir=/etc/consul.d
```

---

<a id="p1"></a>
## P1. Service Discovery

### Bước 1: Service registration - Service definition file

```json
// /etc/consul.d/web.json
{
  "service": {
    "name": "web",
    "tags": ["v1", "production"],
    "port": 80,
    "address": "10.0.1.10",
    "meta": {
      "version": "1.2.3",
      "environment": "production"
    },
    "check": {
      "id": "web-http",
      "name": "HTTP check on port 80",
      "http": "http://localhost:80/health",
      "method": "GET",
      "interval": "10s",
      "timeout": "2s",
      "tls_skip_verify": false
    },
    "checks": [
      {
        "id": "web-tcp",
        "name": "TCP check on port 80",
        "tcp": "localhost:80",
        "interval": "10s",
        "timeout": "2s"
      }
    ]
  }
}
```

```bash
# Reload config
consul reload

# Verify
consul catalog services
consul catalog nodes -service=web
consul health service web
```

### BƯớc 2: Service registration - HTTP API

```bash
# Register
curl -X PUT http://localhost:8500/v1/agent/service/register \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "api",
    "Tags": ["v2"],
    "Port": 8080,
    "Address": "10.0.1.20",
    "Check": {
      "HTTP": "http://10.0.1.20:8080/health",
      "Interval": "10s"
    }
  }'

# Deregister
curl -X PUT http://localhost:8500/v1/agent/service/deregister \
  -d '{"Name": "api"}'
```

### Bước 3: Health checks

```json
// TTL check (app-driven)
{
  "service": {
    "name": "worker",
    "check": {
      "id": "worker-ttl",
      "name": "Worker alive",
      "ttl": "30s",
      "notes": "App must heartbeat every 30s"
    }
  }
}
```

```bash
# Update TTL status
curl -X PUT http://localhost:8500/v1/agent/check/pass/check-id:worker-ttl
curl -X PUT http://localhost:8500/v1/agent/check/warn/check-id:worker-ttl
curl -X PUT http://localhost:8500/v1/agent/check/fail/check-id:worker-ttl
```

```json
// Script check
{
  "check": {
    "id": "disk-usage",
    "name": "Disk usage",
    "args": ["/bin/sh", "-c", "df -h | awk '$5 > 80 {exit 1} {exit 0}'"],
    "interval": "30s"
  }
}
```

```json
// Docker check
{
  "check": {
    "id": "docker-alive",
    "name": "Docker alive",
    "dockercontainerid": "f7c0e40a5b87",
    "shell": "/bin/sh -c 'ps aux | grep myapp'",
    "interval": "30s"
  }
}
```

### Bước 4: Health check states

```
States:
- passing   - Healthy
- warning   - Soft failure (TTL, low disk...)
- critical  - Hard failure

# Script check exit codes
- 0 = passing
- 1 = warning
- 2+ = critical
```

### Bước 5: Service discovery qua DNS

```bash
# Lookup service
dig @127.0.0.1 -p 8600 web.service.consul

# Lookup với tag
dig @127.0.0.1 -p 8600 web.v1.service.consul

# SRV record (port)
dig @127.0.0.1 -p 8600 web.service.consul SRV

# All nodes in datacenter
dig @127.0.0.1 -p 8600 *.node.consul

# Specific node
dig @127.0.0.1 -p 8600 node-name.node.dc1.consul

# Lookup trong node (system DNS)
# Update /etc/resolv.conf
nameserver 127.0.0.1
search service.consul
options rotate timeout:1 attempts:2

# Sau đó:
nslookup web
nslookup api.v2
```

### Bước 6: CLI commands

```bash
# List services
consul catalog services -tags
consul catalog services -detail

# Service instances
consul catalog nodes -service=web
consul catalog nodes -service=web -tag=v1

# Health
consul health service web
consul health service web -passing
consul health state any    # Any failing checks

# Members
consul members
consul members -status=alive
consul members -role=server

# Force leave (khi node bị lỗi)
consul force-leave <node-name>
```

---

<a id="p2"></a>
## P2. DNS & HTTP API

### Bước 1: DNS prepared queries

```bash
# Tạo prepared query
curl -X POST http://localhost:8500/v1/query \
  -d '{
    "Name": "web",
    "Service": "web",
    "Tags": ["v1"],
    "OnlyPassing": true,
    "NearestN": 3,
    "Failover": {
      "Datacenters": ["dc2"]
    },
    "DNS": {
      "TTL": "30s"
    }
  }'
```

```bash
# DNS lookup (returns prepared query ID)
dig @127.0.0.1 -p 8600 web.query.consul
```

### Bước 2: HTTP API - Service discovery

```bash
# Get all services
curl http://localhost:8500/v1/catalog/services

# Get service info
curl http://localhost:8500/v1/catalog/service/web

# Get healthy instances
curl 'http://localhost:8500/v1/health/service/web?passing=true'

# Get with filter (1.8+)
curl 'http://localhost:8500/v1/health/service/web?filter=Meta.environment==production'
curl 'http://localhost:8500/v1/catalog/service/web?filter=Tags contains v1'

# Specific service
curl 'http://localhost:8500/v1/catalog/service/web?dc=dc1&tag=v1&near=agent'

# Connect-enabled service
curl http://localhost:8500/v1/health/connect/web
```

### Bước 3: Filter expressions

```
Supported operators:
==  Equal
!=  Not equal
<, >, <=, >=
contains
matches   (regex)
in
not in
is empty
is not empty

Examples:
filter=Status==passing
filter=Meta.version==1.2.3
filter=Tags contains "v1" and Meta.env=="prod"
filter=Node.Datacenter=="dc1" and Checks.Status=="passing"
filter=Service.Tags in ["v1","v2"]
filter=Service.Meta.region matches "us-.*"
```

### Bước 4: Watches

```hcl
// consul.hcl
watches = [
  {
    type = "service"
    service = "web"
    passingonly = true
    handler = "/usr/local/bin/reload-nginx.sh"
  },
  {
    type = "key"
    key = "config/nginx"
    handler = "/usr/local/bin/reload-nginx.sh"
  },
  {
    type = "checks"
    state = "critical"
    handler = "/usr/local/bin/alert.sh"
  }
]
```

```bash
# reload-nginx.sh
#!/bin/bash
SERVICES=$(curl -s "http://localhost:8500/v1/health/service/web?passing=true")
echo "$SERVICES" | jq -r '.[].Service.Address + ":" + (.[].Service.Port | tostring)' > /etc/nginx/conf.d/upstream.conf
nginx -s reload
```

### Bước 5: Token-based API

```bash
# Get token
TOKEN=$(curl -s http://localhost:8500/v1/acl/token/self | jq -r '.[] | select(.SecretID) | .SecretID')

# Use token
curl -H "X-Consul-Token: $TOKEN" http://localhost:8500/v1/catalog/services
```

---

<a id="p3"></a>
## P3. Service Mesh (Connect)

### Bước 1: Enable Connect

```hcl
// consul.hcl (server config)
connect {
  enabled = true
}
```

```bash
# Restart server
consul reload
```

### Bước 2: Service với Connect

```json
// /etc/consul.d/api.json
{
  "service": {
    "name": "api",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "port": 21000,
        "checks": [
          {
            "name": "Sidecar",
            "tcp": "localhost:21000",
            "interval": "10s"
          }
        ]
      }
    }
  }
}
```

```bash
# Register sidecar proxy
consul connect sidecar proxy -sidecar-for api

# Hoặc qua API
curl -X PUT http://localhost:8500/v1/agent/service/register \
  -d @sidecar.json

# sidecar.json
{
  "name": "api-sidecar-proxy",
  "port": 21000,
  "connect": {
    "sidecar_service": {
      "proxy": {
        "local_service_port": 8080
      }
    }
  }
}
```

### BƯớc 3: Service intentions (allow/deny)

```bash
# Allow api -> db
consul config write -kind service-intentions -name db - <<EOF
{
  "Sources": [
    {
      "Name": "api",
      "Action": "allow"
    }
  ]
}
EOF

# Allow cụ thể method
consul config write -kind service-intentions -name db - <<EOF
{
  "Sources": [
    {
      "Name": "api",
      "Action": "allow",
      "Permissions": [
        {
          "Action": "allow",
          "HTTP": {
            "PathExact": "/query"
          }
        }
      ]
    }
  ]
}
EOF

# Deny all others (default)

# Deny specific
consul config write -kind service-intentions -name db - <<EOF
{
  "Sources": [
    {
      "Name": "rogue-service",
      "Action": "deny"
    }
  ]
}
EOF

# List
consul config list -kind service-intentions
consul config read -kind service-intentions -name db
```

### Bước 4: Upstreams

```json
// api sidecar with upstreams
{
  "service": {
    "name": "api",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "proxy": {
          "local_service_port": 8080,
          "upstreams": [
            {
              "destination_name": "db",
              "local_bind_port": 21001
            },
            {
              "destination_name": "cache",
              "local_bind_port": 21002,
              "config": {
                "connect_timeout_ms": 1000
              }
            }
          ]
        }
      }
    }
  }
}
```

### Bước 5: Transparent proxy mode

```hcl
// In agent config
connect {
  enabled = true
}

ports {
  grpc = 8502
}

# Transparent proxy (route mọi outbound qua sidecar)
```

```json
// service definition
{
  "service": {
    "name": "api",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "proxy": {
          "mode": "transparent",
          "config": {
            "transparent_proxy": {
              "outbound_listener_port": 15001
            }
          }
        }
      }
    }
  }
}
```

### Bước 6: Service mesh routing (L7)

```bash
# Service splitter (canary)
consul config write -kind service-splitter -name api - <<EOF
{
  "Splits": [
    {
      "ServiceSubset": "v1",
      "Weight": 90
    },
    {
      "ServiceSubset": "v2",
      "Weight": 10
    }
  ]
}
EOF

# Service router (header-based)
consul config write -kind service-router -name api - <<EOF
{
  "Routes": [
    {
      "Match": {
        "HTTP": {
          "Header": {
            "x-canary": [{"Exact": "true"}]
          }
        }
      },
      "Destination": {
        "Service": "api",
        "ServiceSubset": "v2"
      }
    }
  ]
}
EOF

# Service resolver (subsets)
consul config write -kind service-resolver -name api - <<EOF
{
  "DefaultSubset": "v1",
  "Subsets": {
    "v1": {
      "Filter": "Tags.v1"
    },
    "v2": {
      "Filter": "Tags.v2"
    }
  }
}
EOF

# Service defaults
consul config write -kind service-defaults -name api - <<EOF
{
  "Protocol": "http",
  "MeshGateway": {
    "Mode": "local"
  },
  "ConnectTimeoutSec": 5
}
EOF
```

### Bước 7: Consul Gateway

```bash
# Mesh Gateway (for multi-DC)
consul config write -kind mesh -name mesh - <<EOF
{
  "Peers": [
    {
      "Name": "cluster-2"
    }
  ]
}
EOF

# Terminating Gateway (cho external services)
consul config write -kind terminating-gateway -name external-gateway - <<EOF
{
  "Services": [
    {
      "Name": "external-api"
    }
  ]
}
EOF
```

---

<a id="p4"></a>
## P4. Security

### BƯớc 1: ACL system

```hcl
// consul.hcl
acl {
  enabled        = true
  default_policy = "deny"
  enable_token_persistence = true

  # Tokens
  tokens {
    initial_management = "root-token-uuid"
    agent              = "agent-token-uuid"
    agent_recovery     = "recovery-token-uuid"
  }
}
```

```bash
# Bootstrap ACL
consul acl bootstrap

# List tokens
consul acl token list

# Create token
consul acl token create \
  -description "Web service token" \
  -policy-name "web-policy"

# Anonymous token (default)
consul acl token create -anonymous

# Delete
consul acl token delete <token-id>
```

### Bước 2: ACL Policies

```hcl
// web-policy.hcl
node_prefix "" {
  policy = "read"
}

service "web" {
  policy = "write"
}

service "db" {
  policy = "read"
}

key_prefix "config/web/" {
  policy = "read"
}
```

```bash
# Apply policy
consul acl policy create \
  -name "web-policy" \
  -description "Web service policy" \
  -rules @web-policy.hcl

# Update
consul acl policy update -name "web-policy" -rules @web-policy.hcl

# List
consul acl policy list

# Read
consul acl policy read -name "web-policy"
```

### Bước 3: Token roles & binding rules

```bash
# Create role
consul acl role create \
  -name "web-role" \
  -description "Web service role" \
  -policy-name "web-policy"

# Bind to services (auto-create token for service)
consul acl binding-rule create \
  -method "service" \
  -bind-name "web" \
  -bind-type "service" \
  -role-name "web-role"
```

### Bước 4: TLS encryption

```bash
# Generate CA
consul tls ca create

# Generate cert cho server
consul tls cert create -server -dc dc1 -days 1825
# Outputs: dc1-server-consul.pem, dc1-server-consul-key.pem

# Cert cho client
consul tls cert create -client -dc dc1 -days 1825
# Outputs: dc1-client-consul.pem, dc1-client-consul-key.pem

# Cert cho gossip
consul tls cert create -gossip -dc dc1 -days 1825
```

```hcl
// consul.hcl (TLS)
tls {
  defaults {
    ca_file   = "/etc/consul/dc1-ca.pem"
    cert_file = "/etc/consul/server.pem"
    key_file  = "/etc/consul/server-key.pem"
  }

  internal_rpc {
    verify_server_hostname = true
  }
}

# Gossip encryption (key file)
encrypt = "T0pZSGet0ver=="
# Hoặc dùng key file:
# encrypt_key_file = "/etc/consul/gossip.key"
```

```bash
# Generate gossip key
consul keygen
# Output: Base64 key
```

### Bước 5: Intentions as ACL

```bash
# Deny by default
consul config write -kind service-intentions -name '*' - <<EOF
{
  "Sources": []
}
EOF

# Allow specific
consul config write -kind service-intentions -name web - <<EOF
{
  "Sources": [
    {"Name": "lb", "Action": "allow"},
    {"Name": "monitoring", "Action": "allow"}
  ]
}
EOF
```

### Bước 6: Audit logging

```hcl
// consul.hcl
audit {
  enabled = true
  sink {
    type = "file"
    format = "json"
    path = "/var/log/consul/audit.log"
    delivery_guarantee = "best-effort"
  }
}
```

---

<a id="p5"></a>
## P5. KV Store

### Bước 1: Read & Write

```bash
# Put
consul kv put config/db/host "db.example.com"
consul kv put config/db/port 5432
consul kv put config/db/password "secret" -flags=1

# Get
consul kv get config/db/host
consul kv get -detailed config/db/host

# Get recursive
consul kv get -recurse config/
consul kv get -recurse config/db

# Delete
consul kv delete config/db/password
consul kv delete -recurse config/db
```

### BƯớc 2: KV qua API

```bash
# Put
curl -X PUT http://localhost:8500/v1/kv/config/db/host \
  -d 'db.example.com'

# Put với flags
curl -X PUT http://localhost:8500/v1/kv/config/db/host?flags=42 \
  -d 'db.example.com'

# Get
curl http://localhost:8500/v1/kv/config/db/host?raw
curl http://localhost:8500/v1/kv/config/?recurse

# CAS (Compare-And-Swap) - tránh race condition
# Get current value (note ModifyIndex)
curl -s http://localhost:8500/v1/kv/config/db/host | jq '.[0].ModifyIndex'

# Update only if not changed
curl -X PUT 'http://localhost:8500/v1/kv/config/db/host?cas=123' \
  -d 'new-db.example.com'
# Returns true if success, false if conflict
```

### Bước 3: Sessions & locks

```bash
# Create session
SESSION=$(consul session create -ttl 30s)

# Acquire lock
LOCK=$(consul kv put -acquire -session=$SESSION service/web/lock "" | jq -r .)
if [ "$LOCK" = "true" ]; then
    echo "Got lock"
fi

# Renew lock
consul session renew -id=$SESSION

# Release lock
consul kv release -session=$SESSION service/web/lock
consul session destroy -id=$SESSION
```

### Bước 4: KV watches

```hcl
// consul.hcl
watches {
  {
    type = "key"
    key = "config/database/main"
    handler = "/opt/consul/reload-app.sh"
  }

  {
    type = "keyprefix"
    prefix = "config/services/"
    handler = "/opt/consul/regenerate-config.sh"
  }
}
```

### BƯớc 5: Use cases cho KV

```bash
# Feature flags
consul kv put features/new-login true
consul kv put features/dark-mode false

# Service config
consul kv put services/api/timeout 30
consul kv put services/api/rate-limit 1000

# Dynamic credentials
consul kv put secrets/db/password "rotated-secret"

# Leader election (kết hợp sessions)
```

---

<a id="p6"></a>
## P6. Consul Template

### BƯc 1: Cài

```bash
# Tải
wget https://releases.hashicorp.com/consul-template/0.31.0/consul-template_0.31.0_linux_amd64.zip
unzip consul-template_0.31.0_linux_amd64.zip
sudo mv consul-template /usr/local/bin/

# Verify
consul-template -version
```

### Bước 2: Template cơ bản

```hcl
// /etc/consul-template/config.hcl
consul {
  address = "127.0.0.1:8500"
  retry {
    enabled     = true
    attempts    = 5
    backoff     = "250ms"
    max_backoff = "1m"
  }
}

template {
  source      = "/etc/consul-template/templates/nginx.ctmpl"
  destination = "/etc/nginx/conf.d/upstream.conf"
  perms       = 0644
  command     = "systemctl reload nginx"

  # Sử dụng wait
  wait {
    min = "2s"
    max = "10s"
  }
}

# Multiple templates
template {
  source      = "/etc/consul-template/templates/haproxy.ctmpl"
  destination = "/etc/haproxy/haproxy.cfg"
  command     = "systemctl reload haproxy"
}
```

### BƯớc 3: Template ngôn ngữ

```jinja2
{# /etc/consul-template/templates/nginx.ctmpl #}
upstream web_backend {
    {{- range service "web" "production" }}
    server {{ .Address }}:{{ .Port }};
    {{- end }}
}

server {
    listen 80;

    location / {
        proxy_pass http://web_backend;
    }
}

# Config từ KV
upstream db {
    {{- $host := key "config/db/host" }}
    {{- $port := key "config/db/port" }}
    server {{ $host }}:{{ $port }};
}
```

```bash
# Run
consul-template -config=/etc/consul-template/config.hcl

# Test
echo 'config/db/host: db.example.com' | consul kv import -
```

### Bước 4: Functions & helpers

```jinja2
{{ range service "web" }}
  # {{ .ID }} - {{ .Address }}:{{ .Port }}
  {{- if .Tags.Contains "v2" }}
    # This is v2
  {{- end }}
{{ end }}

# Filter by tag
{{ range service "web" "production" }}
{{ end }}

# By health
{{ range service "web" | byTag "production" }}
{{ end }}

# KV access
{{ key "config/database/host" }}
{{ keyOrDefault "config/database/host" "localhost" }}
{{ key "config/database/password" | protect }}

# Loop KV
{{ range $key, $value := keys "config/" }}
  {{ $key }} = {{ $value }}
{{ end }}

# File
{{ file "/path/to/file" }}

# By functions
{{ .Tags | contains "v1" }}
{{ env "CONSUL_HTTP_TOKEN" }}
```

### Bước 5: Envconsul

```bash
# Set env var from KV
envconsul -prefix=myapp/ sh -c 'echo $DB_HOST; myapp'
```

---

<a id="p7"></a>
## P7. HA & Multi-DC

### Bước 1: Cluster HA

```bash
# 3 server nodes
# Server 1
consul agent -server -bootstrap-expect=3 \
  -bind=10.0.1.10 \
  -data-dir=/opt/consul \
  -node=consul-1 \
  -datacenter=dc1 \
  -ui

# Server 2
consul agent -server -bootstrap-expect=3 \
  -bind=10.0.1.11 \
  -data-dir=/opt/consul \
  -node=consul-2 \
  -datacenter=dc1 \
  -retry-join=10.0.1.10

# Server 3
consul agent -server -bootstrap-expect=3 \
  -bind=10.0.1.12 \
  -data-dir=/opt/consul \
  -node=consul-3 \
  -datacenter=dc1 \
  -retry-join=10.0.1.10
```

### Bước 2: Auto-join

```hcl
// consul.hcl
retry_join = [
  "provider=aws tag_key=Name tag_value=consul-server-1",
  "provider=aws tag_key=Name tag_value=consul-server-2",
  "provider=aws tag_key=Name tag_value=consul-server-3"
]

# Hoặc với IP cứng
retry_join = ["10.0.1.10", "10.0.1.11", "10.0.1.12"]
```

### Bước 3: Performance tuning

```hcl
// consul.hcl (server)
performance {
  raft_multiplier        = 1
  serf_wan_timeout       = "60s"
}

server_maintenance_mode = false

# Snapshot
snapshot {
  interval = "1h"
  retention = "24h"
}
```

### Bước 4: Multi-DC federation

```bash
# DC1 - Primary
consul agent -server -bootstrap-expect=3 \
  -datacenter=dc1 \
  -bind=10.0.1.10

# DC2 - Secondary
consul agent -server -bootstrap-expect=3 \
  -datacenter=dc2 \
  -bind=10.0.2.10 \
  -retry-join-wan=<dc1-server-ip>
```

```bash
# Verify federation
consul members -wan
consul operator raft list-peers
```

### Bước 5: Mesh Gateway (multi-DC)

```hcl
// consul.hcl
connect {
  enabled = true
}

# Enable mesh gateway
mesh_gateways {
  mode = "local"
}
```

```bash
# Deploy gateway
curl -X PUT http://localhost:8500/v1/agent/service/register \
  -d '{
    "Name": "mesh-gateway",
    "Port": 8443,
    "Connect": {
      "SidecarService": {}
    },
    "Proxy": {
      "Config": {
        "envoy_mesh_gateway_port": 8443
      }
    }
  }'
```

---

<a id="p8"></a>
## P8. Production Best Practices

### BƯớc 1: Backup & restore

```bash
# Snapshot server state
consul snapshot save /backup/consul-$(date +%Y%m%d).snap

# Snapshot agent state (agent-only data, KV không có ở client)
consul snapshot save -stale /backup/agent-$(date +%Y%m%d).snap

# Automated với cron
0 */6 * * * consul snapshot save /backup/consul-$(date +\%Y\%m\%d-\%H\%M).snap

# Restore
consul snapshot restore /backup/consul-20250915.snap

# Verify
consul catalog services | wc -l
```

```bash
# Inspect snapshot
consul snapshot inspect /backup/consul-20250915.snap
```

### Bước 2: Metrics

```hcl
// consul.hcl
telemetry {
  prometheus_retention_time = "60s"
  disable_hostname = true
}
```

```bash
# Scrape endpoint
curl http://localhost:8500/v1/agent/metrics?format=prometheus
```

```yaml
# Prometheus scrape config
- job_name: 'consul-server'
  static_configs:
    - targets: ['consul-1:8500', 'consul-2:8500', 'consul-3:8500']
  metrics_path: /v1/agent/metrics
  params:
    format: [prometheus]
```

```promql
# Important metrics
# Raft
consul_raft_leader
consul_raft_peers

# Health
sum(consul_catalog_service) by (service)

# Memory
consul_runtime_allocated_bytes

# Goroutines
consul_runtime_num_goroutines

# Server check failed
consul_health_service_status{status="critical"}
```

### Bước 3: Health check operations

```bash
# Check cluster
consul operator raft list-peers

# Force leader step down (để leader khác được bầu)
consul operator raft step-down

# Force leave node (khi bị lỗi)
consul force-leave <node-name>

# Check gossip
consul members
consul debug -duration 30s   # Generate debug bundle
```

### Bước 4: Resource limits

```hcl
// consul.hcl
limits {
  http_max_conns_per_client = 200
  https_handshake_timeout   = "5s"
  rpc_handshake_timeout     = "5s"
  rpc_rate                 = 500
  rpc_max_burst            = 1000
  kv_max_value_size        = 524288  # 512KB
}
```

### Bước 5: Production checklist

```yaml
# 1. HA cluster (3 or 5 servers, not 2 or 4)
# 2. Gossip encryption enabled
# 3. TLS for RPC + HTTP
# 4. ACL enabled with deny default
# 5. Backups automated
# 6. Monitoring with Prometheus
# 7. Service mesh with intentions (deny by default)
# 8. UI access restricted (network policy)
# 9. Audit logging enabled
# 10. Resource limits configured
# 11. Multi-DC if needed
# 12. Tested upgrade path
```

### Bước 6: Upgrade

```bash
# 1. Backup
consul snapshot save backup-pre-upgrade.snap

# 2. Upgrade 1 server first
# Replace binary, restart
sudo systemctl restart consul
# Verify cluster still healthy
consul operator raft list-peers

# 3. Upgrade other servers
# Roll one at a time, wait for healthy

# 4. Upgrade clients

# 5. Verify
consul members
consul catalog services | wc -l
```

### Bước 7: Troubleshooting

```bash
# Common issues

# 1. "No leader"
# 3+ servers có vấn đề
consul operator raft list-peers
# Check logs
journalctl -u consul -f
# Verify network connectivity giữa servers

# 2. "Connection refused" từ agent
# Check gossip key match
consul members
# Check token
consul info

# 3. Service registered nhưng failing
consul health service <name>
consul catalog nodes -service=<name>

# 4. Snapshot fails
# Check disk space
df -h
# Check data dir permissions
ls -la /opt/consul

# 5. ACL denied
# Check default policy
consul acl token list
# Test với management token

# 6. Slow queries
# Enable trace
curl 'http://localhost:8500/v1/catalog/services?dc=dc1&token=<management>'

# Debug bundle
consul debug -duration 30s -http-addr=http://localhost:8500
```

---

## 🎯 Bài tập P0-P8

1. Cài Consul dev mode, explore UI
2. Register 3 services qua JSON config + health check
3. Test service discovery qua DNS và HTTP API
4. Setup Connect với sidecar proxy
5. Create intentions, test allow/deny
6. Setup ACL bootstrap, tạo policy cho 2 services
7. Setup Consul Template, render Nginx config tự động
8. Setup 3-server HA cluster, test failover
9. Setup backup script + restore test
10. Setup Prometheus scrape, tạo dashboard

---

> **💡 Tip cuối**: Consul là Swiss army knife cho service mesh. Bắt đầu với service discovery + KV. Service mesh (Connect) thêm sau khi đã thành thạo. Dùng 3 hoặc 5 server nodes (KHÔNG dùng 1, 2, 4). Gossip encryption + TLS là bắt buộc cho production.

---

*Tạo bởi tài liệu học Consul - Chúc bạn thành công! 🚀*
