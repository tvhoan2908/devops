# 🐳 Tài liệu Docker & Container nâng cao: Từ Zero đến Chuyên gia

> **Mục tiêu**: Thành thạo Docker, Dockerfile tối ưu, multi-stage build, security, registry và best practices production.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Cơ bản](#p0) | Cài Docker, lệnh cơ bản |
| [P1. Dockerfile](#p1) | FROM, RUN, COPY, layers |
| [P2. Multi-stage build](#p2) | Tối ưu image size |
| [P3. Networking](#p3) | Bridge, host, overlay |
| [P4. Volume & Storage](#p4) | Bind, named, tmpfs |
| [P5. Compose](#p5) | Multi-container app |
| [P6. Security](#p6) | User, scan, distroless, signing |
| [P7. Registry & Image](#p7) | Push, pull, Harbor, cache |
| [P8. Buildx & Multi-arch](#p8) | Build cho nhiều platform |
| [P9. Production](#p9) | Health check, log, monitoring |
| [P10. Troubleshooting](#p10) | Debug container, network |

---

<a id="p0"></a>
## P0. Cơ bản

### Bước 1: Cài Docker

```bash
# Linux (Ubuntu)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker

# macOS: tải Docker Desktop
# Windows: tải Docker Desktop (hoặc WSL2 + Docker)

# Test
docker run hello-world
docker --version
docker info
```

### Bước 2: Lệnh cơ bản

```bash
# === IMAGES ===
docker pull nginx:1.25
docker images
docker rmi nginx:1.25
docker image prune -a            # Xoá image không dùng
docker image ls --format "{{.Repository}}:{{.Tag}} {{.Size}}"

# === CONTAINERS ===
docker run -d --name web -p 8080:80 nginx:1.25    # Chạy nền
docker run -it --rm ubuntu:22.04 bash             # Interactive, xoá khi thoát
docker ps                                         # Đang chạy
docker ps -a                                      # Tất cả
docker stop web
docker start web
docker restart web
docker rm web
docker container prune                            # Xoá container stopped

# === LOGS & EXEC ===
docker logs web
docker logs -f --tail 100 web                     # Follow
docker logs web --since 10m                        # 10 phút gần
docker exec -it web bash
docker exec web ls /etc/nginx

# === INSPECT ===
docker inspect web
docker stats                                      # Live resource usage
docker top web                                     # Process trong container
docker port web                                    # Port mapping

# === SYSTEM ===
docker system df                                  # Disk usage
docker system prune -a                            # Cleanup tất cả
docker system info
```

### Bước 3: Container vs VM

```
┌─────────────────────────────────┐   ┌─────────────────────────────────┐
│          Virtual Machine        │   │           Container             │
├─────────────────────────────────┤   ├─────────────────────────────────┤
│  App A  │  App B  │  App C      │   │  App A  │  App B  │  App C      │
├─────────┴─────────┴─────────────┤   ├─────────┴─────────┴─────────────┤
│  Guest OS │ Guest OS │ Guest OS  │   │         Container Engine       │
│  (linux)  │ (linux)  │ (windows) │   │           (Docker)             │
├─────────────────────────────────┤   ├─────────────────────────────────┤
│         Hypervisor              │   │           Host OS               │
│      (VMware, KVM...)           │   │         (Linux kernel)          │
├─────────────────────────────────┤   ├─────────────────────────────────┤
│         Host OS                 │   │         Hardware                │
├─────────────────────────────────┤   │                                 │
│         Hardware                │   │                                 │
└─────────────────────────────────┘   └─────────────────────────────────┘
  Size: GB                          Size: MB
  Boot: minutes                     Boot: seconds
  Overhead: heavy                   Overhead: minimal
```

---

<a id="p1"></a>
## P1. Dockerfile

### Bước 1: Dockerfile đầu tiên

```dockerfile
# Dockerfile
FROM ubuntu:22.04

LABEL maintainer="[email protected]"
LABEL version="1.0"
LABEL description="My first app"

WORKDIR /app

COPY . .

RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

RUN pip3 install -r requirements.txt

EXPOSE 8080

ENV APP_ENV=production

CMD ["python3", "app.py"]
```

```bash
docker build -t myapp:1.0 .
docker run -d --name myapp -p 8080:8080 myapp:1.0
docker logs myapp
```

### Bước 2: Cú pháp Dockerfile

```dockerfile
# FROM - base image (bắt buộc đầu tiên)
FROM node:20-alpine

# LABEL - metadata
LABEL maintainer="[email protected]"
LABEL version="1.0"

# ARG - biến build-time (chỉ dùng khi build)
ARG VERSION=1.0
ARG NODE_ENV=production

# ENV - biến runtime (container dùng được)
ENV NODE_ENV=${NODE_ENV}
ENV PORT=8080
ENV DB_HOST=localhost

# WORKDIR - thư mục làm việc
WORKDIR /app

# COPY - copy từ local
COPY package*.json ./
COPY src/ ./src/
COPY --chown=node:node . .

# ADD - giống COPY nhưng hỗ trợ URL, tar auto-extract
ADD https://example.com/file.tar.gz /tmp/
ADD file.tar.gz /tmp/

# RUN - chạy command khi build
RUN apt-get update && apt-get install -y curl
RUN npm install
RUN echo "Built at $(date)" > /build-info.txt

# USER - chuyển user
USER node
USER 1000

# EXPOSE - document port (không tự động publish)
EXPOSE 8080

# VOLUME - khai báo mount point
VOLUME /data

# CMD - command mặc định (chỉ 1 CMD, lệnh cuối thắng)
CMD ["node", "server.js"]
CMD ["npm", "start"]

# ENTRYPOINT - command chính, không bị override
ENTRYPOINT ["node"]
CMD ["server.js"]
# Khi chạy: docker run image arg1 -> node arg1

# HEALTHCHECK
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# SHELL - shell mặc định cho RUN
SHELL ["/bin/bash", "-c"]

# STOPSIGNAL
STOPSIGNAL SIGTERM

# ONBUILD - trigger khi image này làm base cho image khác
ONBUILD RUN npm install
```

### Bước 3: Layer & caching

```dockerfile
# Mỗi instruction = 1 layer
# Layer được cache riêng
# Thứ tự: thay đổi ít -> nhiều

# TỐT: tách dependencies và code
FROM node:20-alpine
WORKDIR /app

# Copy package files trước (thay đổi ít)
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Copy code sau (thay đổi nhiều)
COPY . .

CMD ["node", "server.js"]
```

```bash
# Xem layer history
docker history myapp:1.0
docker history --no-trunc myapp:1.0

# Xem size mỗi layer
docker history myapp:1.0 --format "{{.Size}} {{.CreatedBy}}"
```

### Bước 4: .dockerignore

```
# .dockerignore
.git
.gitignore
node_modules
npm-debug.log
.env
.env.*
*.md
!README.md
Dockerfile
.dockerignore
docker-compose.yml
.vscode
.idea
*.log
coverage
.nyc_output
dist
build
test
tests
__tests__
*.test.js
*.spec.js
```

---

<a id="p2"></a>
## P2. Multi-stage build

### Bước 1: Pattern cơ bản

```dockerfile
# === STAGE 1: Build ===
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# === STAGE 2: Production ===
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY --from=builder /app/dist ./dist
USER node
EXPOSE 8080
CMD ["node", "dist/server.js"]
```

### Bước 2: Go multi-stage

```dockerfile
# Build stage
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o app .

# Final stage - distroless hoặc alpine
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/app /
USER nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

### Bước 3: Java Spring Boot

```dockerfile
# Build với Maven
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Runtime
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Bước 4: Python với venv

```dockerfile
# Build
FROM python:3.11 AS builder
WORKDIR /app
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

# Production
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /opt/venv /opt/venv
COPY --from=builder /app /app
ENV PATH="/opt/venv/bin:$PATH"
USER nobody
EXPOSE 8080
CMD ["python", "app.py"]
```

### Bước 5: Tối ưu size

```dockerfile
# Chọn base image nhỏ
FROM alpine:3.19                    # ~7MB
FROM node:20-alpine                 # ~50MB
FROM gcr.io/distroless/static       # ~2MB
FROM scratch                        # 0MB (cho binary tĩnh)

# Dùng slim
FROM python:3.11-slim               # ~45MB thay vì 350MB

# Cleanup sau apt
RUN apt-get update && apt-get install -y --no-install-recommends \
    pkg1 \
    pkg2 \
  && rm -rf /var/lib/apt/lists/*

# Combine RUN
RUN apt-get update \
  && apt-get install -y pkg1 pkg2 \
  && curl -L https://example.com/file.tar.gz | tar xz \
  && rm -rf /var/lib/apt/lists/* \
  && apt-get clean
```

```bash
# So sánh size
docker images | grep myapp
docker history myapp:1.0

# Dùng dive để xem layer chi tiết
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest myapp:1.0
```

---

<a id="p3"></a>
## P3. Networking

### Bước 1: Network drivers

```
┌─────────────────────────────────────────────────────────┐
│  Network Drivers                                        │
│                                                          │
│  bridge  - Default cho container trên 1 host            │
│  host    - Container dùng network của host               │
│  none    - Không có network                              │
│  overlay - Multi-host (Swarm, K8s)                      │
│  macvlan - Container có MAC riêng (như thiết bị thật)   │
│  ipvlan  - Tương tự macvlan nhưng chia sẻ MAC          │
└─────────────────────────────────────────────────────────┘
```

### Bước 2: Bridge network

```bash
# Default bridge (không khuyến nghị)
docker run -d --name web --network bridge nginx

# User-defined bridge (khuyến nghị)
docker network create --driver bridge mynet

# Container có thể resolve tên nhau
docker run -d --name db --network mynet postgres
docker run -d --name api --network mynet -e DB_HOST=db myapi

# Liệt kê network
docker network ls
docker network inspect mynet

# Xem IP container
docker inspect web | grep IPAddress
```

### Bước 3: Host network

```bash
# Container dùng trực tiếp network của host
docker run -d --name web --network host nginx
# Port 80 của container = port 80 của host
# Performance tốt nhất nhưng conflict port
```

### Bước 4: Port mapping

```bash
# -p host:container
docker run -d -p 8080:80 nginx            # All interfaces
docker run -d -p 127.0.0.1:8080:80 nginx  # Specific IP
docker run -d -p 8080:80 nginx            # TCP default
docker run -d -p 8080:80/udp nginx        # UDP

# Random port
docker run -d -P nginx
docker port web
```

### Bước 5: Multi-host overlay

```bash
# Khởi tạo Swarm
docker swarm init
docker swarm join --token xxx worker

# Tạo overlay network
docker network create --driver overlay --attachable mynet

# Service trên Swarm
docker service create --name web --network mynet -p 80:80 nginx
```

---

<a id="p4"></a>
## P4. Volume & Storage

### Bước 1: Volume types

```bash
# 1. Named volume (Docker quản lý)
docker volume create mydata
docker run -v mydata:/data alpine

# 2. Bind mount (mount từ host)
docker run -v /host/path:/container/path alpine
docker run -v $(pwd)/src:/app/src alpine

# 3. tmpfs (RAM, không persist)
docker run --tmpfs /tmp alpine
docker run --mount type=tmpfs,destination=/tmp,tmpfs-size=100m alpine
```

### Bước 2: Quản lý volume

```bash
docker volume ls
docker volume inspect mydata
docker volume prune
docker volume rm mydata

# Backup volume
docker run --rm -v mydata:/data -v $(pwd):/backup alpine \
  tar czf /backup/mydata.tar.gz /data

# Restore
docker run --rm -v mydata:/data -v $(pwd):/backup alpine \
  tar xzf /backup/mydata.tar.gz -C /
```

### Bước 3: Volume driver

```bash
# NFS
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.1,rw \
  --opt device=:/path/to/dir mynfs

# S3 (s3fs)
docker plugin install rexray/s3fs:latest \
  --alias s3fs \
  --grant-all-permissions
docker volume create --driver s3fs mys3

# SSHFS
docker plugin install --grant-all-permissions vieux/sshfs
docker volume create --driver vieux/sshfs \
  --opt sshcmd=user@host:/path \
  --opt password=xxx mysshfs
```

### Bước 4: Best practice

```yaml
# Dùng named volumes cho data
# Bind mount cho code dev
# tmpfs cho cache tạm
volumes:
  - db-data:/var/lib/postgresql/data  # Named
  - ./src:/app/src                    # Bind (dev)
  - /tmp                              # tmpfs
```

---

<a id="p5"></a>
## P5. Docker Compose

### Bước 1: Compose cơ bản

```yaml
# docker-compose.yml
version: "3.9"

services:
  web:
    image: nginx:1.25-alpine
    container_name: web
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - frontend
    depends_on:
      - api
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 3s
      retries: 3
    environment:
      - NGINX_HOST=example.com
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  api:
    build: ./api
    container_name: api
    environment:
      - DB_HOST=db
      - DB_USER=app
      - DB_PASSWORD_FILE=/run/secrets/db_pass
    secrets:
      - db_pass
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M

  db:
    image: postgres:16-alpine
    container_name: db
    environment:
      - POSTGRES_USER=app
      - POSTGRES_DB=appdb
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_pass
    secrets:
      - db_pass
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    networks:
      - backend
    command: redis-server --requirepass redispass

volumes:
  db-data:

networks:
  frontend:
  backend:

secrets:
  db_pass:
    file: ./secrets/db_pass.txt
```

### Bước 2: Lệnh Compose

```bash
# Khởi động
docker compose up -d
docker compose up -d --build
docker compose up -d --force-recreate

# Xem trạng thái
docker compose ps
docker compose top
docker compose logs -f
docker compose logs -f api

# Quản lý
docker compose stop
docker compose start
docker compose restart api
docker compose pause
docker compose unpause

# Xoá
docker compose down
docker compose down -v               # Xoá volumes
docker compose down --rmi all        # Xoá images

# Scale
docker compose up -d --scale api=3

# Validate
docker compose config

# Exec
docker compose exec api bash
docker compose exec db psql -U app
```

### Bước 3: Compose profiles

```yaml
services:
  web:
    image: nginx
    profiles: [frontend]

  api:
    build: ./api
    profiles: [backend]

  db:
    image: postgres
    profiles: [backend]

  monitoring:
    image: grafana/grafana
    profiles: [monitoring]
```
```bash
# Chỉ chạy các service trong profile
docker compose --profile backend up -d
docker compose --profile monitoring up -d
```

### Bước 4: Compose overrides

```yaml
# docker-compose.yml (base)
services:
  web:
    image: nginx
    ports: ["80:80"]

# docker-compose.override.yml (dev local, auto-load)
services:
  web:
    volumes:
      - ./html:/usr/share/nginx/html

# docker-compose.prod.yml (production)
services:
  web:
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
```
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

<a id="p6"></a>
## P6. Security

### Bước 1: Non-root user

```dockerfile
# Tạo user trước
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup package*.json ./
RUN npm ci --only=production
COPY --chown=appuser:appgroup . .

USER appuser
EXPOSE 8080
CMD ["node", "server.js"]
```

### Bước 2: Distroless & minimal images

```dockerfile
# Distroless - không có shell, package manager
FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app /app
USER nonroot
CMD ["server.js"]

# Scratch - chỉ cho binary tĩnh
FROM scratch
COPY --from=builder /app/app /
ENTRYPOINT ["/app"]
```

### Bước 3: Image scanning với Trivy

```bash
# Cài
brew install trivy
# hoặc
sudo apt-get install -y wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install trivy

# Scan
trivy image nginx:1.25
trivy image --severity HIGH,CRITICAL nginx:1.25
trivy image --format json --output result.json nginx:1.25
trivy image --exit-code 1 --severity CRITICAL myapp:1.0

# Scan filesystem
trivy fs .
trivy fs --security-checks vuln,secret,config .

# Scan K8s manifest
trivy config k8s.yaml
```

### Bước 4: Docker Scout / Snyk

```bash
# Docker Scout
docker scout cves myapp:1.0
docker scout recommendations myapp:1.0

# Snyk
npm install -g snyk
snyk auth
snyk container test myapp:1.0
snyk container monitor myapp:1.0
```

### Bước 5: Image signing (Cosign)

```bash
# Cài cosign
curl -O -L https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign

# Tạo key
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key myregistry/myapp:1.0

# Verify
cosign verify --key cosign.pub myregistry/myapp:1.0

# Keyless với GitHub Actions
COSIGN_EXPERIMENTAL=1 cosign sign myregistry/myapp:1.0
```

### Bước 6: Runtime security

```bash
# Không dùng privileged
docker run --rm alpine          # Mặc định không privileged
docker run --privileged alpine  # TRÁNH dùng

# Cap-drop
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Read-only filesystem
docker run --read-only --tmpfs /tmp nginx

# No-new-privileges
docker run --security-opt=no-new-privileges nginx

# Limit resources
docker run --memory=512m --cpus=1 nginx

# User namespace
docker run --userns=host nginx
```

---

<a id="p7"></a>
## P7. Registry

### Bước 1: Docker Hub

```bash
# Login
docker login -u username
docker login registry.example.com

# Tag
docker tag myapp:1.0 username/myapp:1.0
docker tag myapp:1.0 username/myapp:latest

# Push
docker push username/myapp:1.0
docker push username/myapp:latest

# Pull
docker pull username/myapp:1.0

# Search
docker search nginx
```

### Bước 2: Private registry

```bash
# Chạy registry local
docker run -d \
  -p 5000:5000 \
  --name registry \
  -v registry-data:/var/lib/registry \
  registry:2

# Self-signed cert
mkdir -p certs
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout certs/domain.key \
  -x509 -days 365 \
  -out certs/domain.crt \
  -subj "/CN=registry.local"

# Với TLS + auth
docker run -d \
  -p 5000:5000 \
  --name registry \
  --restart=always \
  -v $(pwd)/certs:/certs \
  -v $(pwd)/auth:/auth \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v registry-data:/var/lib/registry \
  registry:2

# Tạo user
docker run --rm \
  --entrypoint htpasswd \
  httpd:2 -Bbn admin password > auth/htpasswd
```

### Bước 3: Harbor (Enterprise registry)

```bash
# Tải và cài
wget https://github.com/goharbor/harbor/releases/latest/download/harbor-offline-installer.tgz
tar xzf harbor-offline-installer.tgz
cd harbor

cp harbor.yml.tmpl harbor.yml
# Sửa hostname, password

./install.sh

# Truy cập: https://hostname
```

### Bước 4: GitHub Container Registry / GitLab Registry

```bash
# GHCR
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
docker tag myapp:1.0 ghcr.io/username/myapp:1.0
docker push ghcr.io/username/myapp:1.0

# GitLab
docker login registry.gitlab.com -u username -p token
docker tag myapp:1.0 registry.gitlab.com/group/project/myapp:1.0
docker push registry.gitlab.com/group/project/myapp:1.0
```

### Bước 5: Image caching

```bash
# Pull-through cache
docker run -d \
  -p 5000:5000 \
  --name registry-cache \
  -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io \
  -v cache-data:/var/lib/registry \
  registry:2

# Configure daemon
# /etc/docker/daemon.json
{
  "registry-mirrors": ["https://mirror.gcr.io"],
  "insecure-registries": ["myregistry.local:5000"]
}
```

---

<a id="p8"></a>
## P8. Buildx & Multi-arch

### Bước 1: Buildx setup

```bash
# Tạo builder
docker buildx create --name multiarch --use --bootstrap
docker buildx ls

# Dùng mặc định
docker buildx use multiarch
```

### Bước 2: Multi-platform build

```bash
# Build cho nhiều platform
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  --tag myregistry/myapp:1.0 \
  --push \
  .

# Tag riêng
docker buildx build \
  --platform linux/amd64 -t myregistry/myapp:1.0-amd64 --push .
docker buildx build \
  --platform linux/arm64 -t myregistry/myapp:1.0-arm64 --push .
```

### Bước 3: Multi-platform trong Dockerfile

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.22 AS builder
ARG TARGETPLATFORM
ARG TARGETOS
ARG TARGETARCH

WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build -ldflags="-s -w" -o app .

FROM --platform=$TARGETPLATFORM gcr.io/distroless/static
COPY --from=builder /app/app /
ENTRYPOINT ["/app"]
```

### Bước 4: Cache nâng cao

```bash
# GHA cache
docker buildx build \
  --cache-from type=gha \
  --cache-to type=gha,mode=max \
  --tag myapp:1.0 \
  --push \
  .

# S3 cache (buildkit)
docker buildx create --name s3cache \
  --driver docker-container \
  --driver-opt "image=moby/buildkit:v0.13.0"

docker buildx build \
  --builder s3cache \
  --cache-from type=s3,region=us-east-1,bucket=docker-cache,name=myapp \
  --cache-to type=s3,region=us-east-1,bucket=docker-cache,name=myapp,mode=max \
  --tag myapp:1.0 \
  --push \
  .
```

---

<a id="p9"></a>
## P9. Production Best Practices

### Bước 1: HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

```bash
# Check status
docker inspect --format='{{.State.Health.Status}}' web

# Chỉ chạy container healthy
docker run -d --name web myapp
docker run -d --name api --link web myapi   # Cũ, không khuyến nghị
```

### Bước 2: Logging

```bash
# Xem logs
docker logs web
docker logs --since 10m web
docker logs --tail 100 -f web

# Driver config
# /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "production"
  }
}

# Hoặc ghi đè khi run
docker run --log-driver=fluentd \
  --log-opt fluentd-address=localhost:24224 \
  nginx
```

### Bước 3: Resource limits

```bash
# Memory
docker run -m 512m myapp
docker run --memory=512m --memory-swap=1g myapp

# CPU
docker run --cpus=1.5 myapp
docker run --cpuset-cpus=0,1 myapp

# I/O
docker run --device-read-bps=/dev/sda:1mb \
  --device-write-bps=/dev/sda:1mb myapp

# Pids
docker run --pids-limit=100 myapp
```

### Bước 4: Restart policies

```bash
docker run --restart=no myapp          # Không restart (default)
docker run --restart=on-failure:5 myapp # Tối đa 5 lần
docker run --restart=always myapp       # Luôn restart
docker run --restart=unless-stopped myapp # Trừ khi stop manually
```

### Bước 5: Labels & metadata

```bash
docker run \
  --label com.example.service=web \
  --label com.example.version=1.0 \
  --label com.example.environment=production \
  myapp

# Filter
docker ps --filter label=com.example.environment=production
```

---

<a id="p10"></a>
## P10. Troubleshooting

### Bước 1: Container không start

```bash
# Xem logs
docker logs <container>
docker logs --details <container>

# Xem events
docker events --filter container=<container>

# Inspect
docker inspect <container>
docker inspect <container> | jq '.[0].State'

# Vào container đã stop
docker commit <container> debug
docker run -it --entrypoint /bin/sh debug

# Override entrypoint
docker run -it --entrypoint /bin/sh myapp
```

### Bước 2: Network issues

```bash
# Test connection từ container
docker exec web ping db
docker exec web nslookup db
docker exec web curl http://api:8080

# Xem network
docker network inspect mynet

# Test port
docker run --rm --network mynet alpine nc -zv db 5432

# DNS debug
docker run --rm --network mynet alpine \
  nslookup db

# Capture network traffic
docker run --rm --net container:web --cap-add NET_ADMIN \
  alpine tcpdump -i any -w /tmp/cap.pcap
```

### Bước 3: Performance issues

```bash
# Resource usage
docker stats
docker stats --no-stream

# Process info
docker top web

# Disk I/O
docker run --rm \
  --pid container:web \
  alpine \
  sh -c 'for pid in $(ls /proc | grep -E "^[0-9]+$"); do \
    cat /proc/$pid/io 2>/dev/null; \
  done'
```

### Bước 4: Image size lớn

```bash
# Xem layer lớn
docker history myapp --format "{{.Size}} {{.CreatedBy}}" | sort -h

# Phân tích chi tiết
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest myapp

# Tối ưu:
# - Multi-stage build
# - Dùng alpine/slim
# - Cleanup apt cache
# - .dockerignore
# - Combine RUN
```

### Bước 5: Debug production-like

```bash
# Exec như user khác
docker exec -u root -it web bash

# Copy file ra/vào
docker cp web:/var/log/app.log ./app.log
docker cp ./config.json web:/app/config.json

# Vào network namespace
docker run --rm --net container:web --pid container:web --cap-add NET_ADMIN \
  alpine sh

# System tools
docker run --rm --pid container:web alpine \
  sh -c 'apk add strace && strace -p 1'
```

### 🎯 Bài tập P0-P10
1. Tạo Dockerfile cho 1 app (Python/Node/Go) - size < 200MB
2. Multi-stage build giảm 50% size
3. Compose stack: web + api + db + redis + nginx
4. Setup local registry, push/pull images
5. Trivy scan, fix hết CRITICAL
6. Multi-arch build cho amd64 + arm64
7. Cosign sign image, verify thành công

---

> **💡 Tip cuối**: Image nhỏ = nhanh, an toàn, ít vulnerability. Multi-stage build là chìa khoá. Luôn scan trước khi deploy.

---

*Tạo bởi tài liệu học Docker nâng cao - Chúc bạn thành công! 🚀*
