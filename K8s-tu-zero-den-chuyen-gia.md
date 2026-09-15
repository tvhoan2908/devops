# 🚢 Tài liệu Kubernetes: Từ Zero đến Chuyên gia

> **Mục tiêu**: Sau khi hoàn thành tài liệu này, bạn có thể tự tin thiết kế, triển khai, vận hành và debug các hệ thống Kubernetes ở quy mô production.
>
> **Cách học**: Mỗi bước đều có lệnh thực thi và ví dụ thực tế. Hãy chạy từng lệnh trên cluster của bạn.

---

## 📑 Mục lục

| Phần | Nội dung | Thời gian |
|------|----------|-----------|
| [P0. Chuẩn bị môi trường](#p0) | Cài K8s local | 30 phút |
| [P1. Nền tảng](#p1) | Pod, Deployment, Service | 2 giờ |
| [P2. Networking](#p2) | Service types, Ingress, DNS | 3 giờ |
| [P3. Storage](#p3) | PV, PVC, StorageClass | 2 giờ |
| [P4. Config & Secrets](#p4) | ConfigMap, Secret, RBAC | 2 giờ |
| [P5. Scaling & Scheduling](#p5) | HPA, VPA, Affinity, Taints | 3 giờ |
| [P6. Security](#p6) | RBAC, NetworkPolicy, PSP/PSA | 3 giờ |
| [P7. Observability](#p7) | Metrics, Logs, Tracing | 3 giờ |
| [P8. Production](#p8) | HA, Backup, Upgrade, DR | 4 giờ |
| [P9. Advanced](#p9) | Operator, CRD, GitOps | 4 giờ |
| [P10. Troubleshooting](#p10) | Debug thực chiến | 4 giờ |

---

<a id="p0"></a>
## P0. Chuẩn bị môi trường

### Bước 1: Cài đặt K8s cluster local

Chọn **1 trong 3** tuỳ nhu cầu:

#### 🅰️ Dùng K3s (khuyến nghị - nhẹ, nhanh)
```bash
# Linux/macOS/WSL2
curl -sfL https://get.k3s.io | sh -

# Kiểm tra
sudo k3s kubectl get nodes
sudo k3s kubectl get pods -A
```

#### 🅱️ Dùng Minikube
```bash
minikube start --driver=docker --cpus=4 --memory=4g
minikube addons enable ingress
minikube dashboard
```

#### 🅲️ Dùng Kind (multi-node trong Docker)
```yaml
# kind-multi-node.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```
```bash
kind create cluster --config kind-multi-node.yaml --name k8s-lab
```

### Bước 2: Cài các công cụ cần thiết
```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# kubectl autocomplete & alias
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# K9s - Terminal UI cho K8s
curl -sS https://webi.sh/k9s | sh

# Stern - Multi-pod log tailer
curl -LO https://github.com/stern/stern/releases/download/v1.30.0/stern_1.30.0_linux_amd64.tar.gz
tar xf stern_*.tar.gz && sudo mv stern /usr/local/bin/

# Các công cụ debug
brew install jq yq kubectx  # macOS
# hoặc
sudo apt install -y jq && sudo curl -Lo /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
```

### Bước 3: Khám phá cluster
```bash
# Xem thông tin cluster
kubectl cluster-info
kubectl version
kubectl get nodes -o wide
kubectl get pods -A

# Xem cấu hình kubectl
kubectl config view
kubectl config get-contexts
kubectl config use-context <context-name>

# Xem tài nguyên của node đầu tiên
kubectl describe node $(kubectl get nodes -o name | head -1)
```

**📚 Kiến thức nền tảng cần nắm:**
- **Control Plane**: API Server, etcd, Scheduler, Controller Manager
- **Worker Node**: kubelet, kube-proxy, container runtime
- **Pod**: đơn vị nhỏ nhất, chứa 1+ container
- **Node**: máy vật lý/ảo chạy workload

---

<a id="p1"></a>
## P1. Nền tảng: Pod, Deployment, Service

### Bước 1: Tạo Pod đầu tiên

```bash
# Cách 1: imperative
kubectl run nginx --image=nginx:1.25 --port=80

# Kiểm tra
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx

# Xem log
kubectl logs nginx

# Exec vào container
kubectl exec -it nginx -- bash

# Xóa
kubectl delete pod nginx
```

```bash
# Cách 2: declarative (khuyến nghị)
mkdir -p ~/k8s-lab/p1 && cd ~/k8s-lab/p1
```
```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi
    env:
    - name: GREETING
      value: "Hello from Pod"
```
```bash
kubectl apply -f pod.yaml
kubectl get pod nginx-pod -o yaml | less
```

### Bước 2: Multi-container Pod (Sidecar pattern)

```yaml
# pod-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logger
spec:
  containers:
  - name: web
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx

  - name: log-shipper        # sidecar
    image: busybox
    command: ["sh", "-c", "tail -f /var/log/nginx/access.log"]
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx

  volumes:
  - name: logs
    emptyDir: {}
```
```bash
kubectl apply -f pod-sidecar.yaml
kubectl logs web-with-logger -c log-shipper  # log của container cụ thể
kubectl logs web-with-logger --all-containers=true # cả 2 container
```

### Bước 3: Deployment - quản lý Pod vĩnh viễn

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Tăng tối đa 1 pod khi update
      maxUnavailable: 0   # Không được downtime
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 3
        startupProbe:
          httpGet:
            path: /
            port: 80
          failureThreshold: 30
          periodSeconds: 5
```
```bash
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pod -l app=nginx  # Xem Deployment, ReplicaSet, Pod

# Scaling
kubectl scale deployment/nginx-deploy --replicas=5

# Rolling update image
kubectl set image deployment/nginx-deploy nginx=nginx:1.26
# Hoặc edit
kubectl edit deployment nginx-deploy
# Hoặc patch
kubectl patch deployment nginx-deploy -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.27"}]}}}}'

# Xem lịch sử rollout
kubectl rollout history deployment/nginx-deploy

# Rollback
kubectl rollout undo deployment/nginx-deploy
kubectl rollout undo deployment/nginx-deploy --to-revision=1

# Pause/resume (cho blue-green, canary)
kubectl rollout pause deployment/nginx-deploy
kubectl rollout resume deployment/nginx-deploy
```

### Bước 4: Service - expose Pod ra ngoài

```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP            # Default - chỉ trong cluster
  selector:
    app: nginx                # Match với pod có label này
  ports:
  - protocol: TCP
    port: 80                  # Port của Service
    targetPort: 80            # Port của Pod
```
```bash
kubectl apply -f service-clusterip.yaml
kubectl get svc nginx-svc
kubectl get endpoints nginx-svc   # IP các Pod backend

# Test từ trong cluster
kubectl run debug --rm -it --image=busybox --restart=Never -- sh
# Trong pod debug:
wget -qO- http://nginx-svc
```
```yaml
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080          # Range: 30000-32767
```
```yaml
# service-lb.yaml (cloud)
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### Bước 5: Labels & Selectors - kỹ năng phải thành thạo

```bash
# Xem label
kubectl get pods --show-labels

# Lọc theo label
kubectl get pods -l app=nginx
kubectl get pods -l 'app in (nginx, redis)'
kubectl get pods -l app=nginx,tier=frontend

# Thêm/sửa label
kubectl label pod nginx-pod version=v1
kubectl label pod nginx-pod version=v2 --overwrite

# Annotation (metadata cho tool, không selector được)
kubectl annotate pod nginx-pod description="My first pod"
```

### Bước 6: Job & CronJob

```yaml
# job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calc
spec:
  completions: 5              # Số pod phải chạy thành công
  parallelism: 2              # Số pod chạy đồng thời
  backoffLimit: 3             # Số lần retry khi fail
  activeDeadlineSeconds: 100  # Timeout toàn job
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
```
```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-db
spec:
  schedule: "0 2 * * *"         # 2h sáng mỗi ngày
  timeZone: "Asia/Ho_Chi_Minh"  # K8s 1.27+
  concurrencyPolicy: Forbid     # Không chạy 2 job cùng lúc
  startingDeadlineSeconds: 60
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:16
            command: ["/bin/sh", "-c", "pg_dump -h db -U $USER > /backup/db.sql"]
```

### Bước 7: StatefulSet - workload có trạng thái

```yaml
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis-headless   # Bắt buộc
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```
```yaml
# Headless service cho StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

### Bước 8: DaemonSet - chạy 1 Pod/node

```yaml
# daemonset-fluentd.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1.16
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      tolerations:
      - operator: Exists  # Chạy trên cả node có taint
```

### 🎯 Bài tập P1
1. Deploy nginx với 3 replicas, rolling update không downtime
2. Tạo Service NodePort, truy cập từ browser
3. Update lên version mới, rollback nếu lỗi
4. Tạo CronJob backup thư mục `/data` mỗi 5 phút

---

<a id="p2"></a>
## P2. Networking

### Bước 1: Hiểu mô hình network K8s

```
┌─────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                      │
│                                                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐            │
│  │ Node 1   │   │ Node 2   │   │ Node 3   │            │
│  │ ┌──────┐ │   │ ┌──────┐ │   │ ┌──────┐ │            │
│  │ │ Pod A│ │   │ │ Pod C│ │   │ │ Pod E│ │            │
│  │ └──┬───┘ │   │ └──┬───┘ │   │ └──┬───┘ │            │
│  │    │     │   │    │     │   │    │     │            │
│  │ ┌──▼──┐  │   │ ┌──▼──┐  │   │ ┌──▼──┐  │            │
│  │ │Pod B│  │   │ │Pod D│  │   │ │Pod F│  │            │
│  │ └─────┘  │   │ └─────┘  │   │ └─────┘  │            │
│  │          │   │          │   │          │            │
│  │ eth0 ───┼───┼──────┼───┼───┼──────┐  │            │
│  │ 10.244.0 │   │10.244.1 │   │10.244.2 │  │            │
│  └──────────┘   └──────────┘   └──────────┘            │
│                                                          │
│  Service Network: 10.96.0.0/12 (ClusterIP)              │
└─────────────────────────────────────────────────────────┘
```

**4 điều kiện K8s yêu cầu:**
1. Mỗi Pod có IP riêng
2. Pod-Pod cùng node giao tiếp được (không cần NAT)
3. Pod-Pod khác node giao tiếp được
4. Pod-Service giao tiếp được

### Bước 2: DNS trong K8s

```bash
# Service DNS
<service-name>.<namespace>.svc.cluster.local

# Ví dụ: từ pod trong namespace "production"
# Truy cập service "api" trong namespace "backend":
curl http://api.backend.svc.cluster.local

# Pod DNS (nếu có pod hostname)
<pod-ip>.<namespace>.pod.cluster.local
```
```yaml
# pod-dns.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-test
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
  dnsPolicy: ClusterFirst       # Default
  # dnsConfig:                   # Custom DNS
  #   nameservers:
  #     - 8.8.8.8
  #   searches:
  #     - ns1.svc.cluster.local
  #   options:
  #     - name: ndots
  #       value: "2"
```
```bash
kubectl exec -it dns-test -- nslookup kubernetes.default
kubectl exec -it dns-test -- nslookup nginx-svc
```

### Bước 3: Ingress - HTTP/HTTPS routing

```bash
# Cài NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller/v1.9.4/deploy/static/provider/cloud/deploy.yaml

# Minikube
minikube addons enable ingress
```
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```
```bash
# Test không có DNS, dùng --resolve
curl --resolve app.example.com:443:<INGRESS_IP> https://app.example.com

# Lấy IP ingress
kubectl get ingress
```

### Bước 4: Gateway API (thế hệ mới)

```yaml
# gateway.yaml (cần cài Gateway API CRDs)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    hostname: app.example.com
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames: ["app.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-svc
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-svc
      port: 80
```

### Bước 5: NetworkPolicy - micro-segmentation

```yaml
# networkpolicy-default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}           # Áp dụng cho tất cả pod
  policyTypes:
  - Ingress
  - Egress
```
```yaml
# networkpolicy-allow.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:                          # DNS
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```
```bash
# Test network policy
kubectl run nettest --rm -it --image=nicolaka/netshoot --restart=Never -- bash
# Trong pod:
nc -zv api-svc 8080       # Test vào
nc -zv postgres 5432      # Test ra
```

### 🎯 Bài tập P2
1. Deploy 2 service (frontend, backend), frontend gọi backend
2. Tạo Ingress route `/api` → backend, `/` → frontend
3. Thêm TLS với cert-manager
4. NetworkPolicy chỉ cho phép frontend → backend

---

<a id="p3"></a>
## P3. Storage

### Bước 1: Volume cơ bản

```yaml
# pod-with-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir: {}                  # Tạm thời, mất khi pod xoá
    # emptyDir.medium: Memory    # RAM-backed
```

### Bước 2: PersistentVolume & PersistentVolumeClaim

```yaml
# pv.yaml (static provisioning)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
```
```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage
```
```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pvc
spec:
  containers:
  - name: app
    image: postgres:16
    env:
    - name: POSTGRES_PASSWORD
      value: secret
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

### Bước 3: StorageClass (dynamic provisioning)

```bash
# Xem storage class có sẵn
kubectl get sc
```
```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs   # AWS EBS
# provisioner: kubernetes.io/gce-pd  # GCP
# provisioner: disk.csi.azure.com    # Azure
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer  # Quan trọng cho multi-zone
allowVolumeExpansion: true
reclaimPolicy: Delete
```
```yaml
# pvc-dynamic.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd
```
```bash
kubectl apply -f storageclass.yaml
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc dynamic-pvc        # STATUS = Bound
kubectl get pv                     # PV được tạo tự động
```

### Bước 4: CSI Driver nâng cao

```bash
# Cài NFS CSI driver
helm repo add csi-driver-smb https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts
helm install csi-driver-smb csi-driver-smb/csi-driver-smb --namespace kube-system --set vmss=true

# Hoặc Longhorn (distributed storage)
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.5.3/deploy/longhorn.yaml
```

### Bước 5: Snapshot & Restore

```bash
# Enable snapshot CRDs
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.2.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.2.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.2.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
```
```yaml
# snapshotclass.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  tagSpecification_1: "tag=backup"
```
```yaml
# snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: dynamic-pvc
```
```yaml
# restore.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 20Gi
```

### 🎯 Bài tập P3
1. Deploy Postgres với PVC 5Gi
2. Tạo StorageClass dynamic, test với NFS CSI
3. Snapshot/restore dữ liệu

---

<a id="p4"></a>
## P4. Config & Secrets

### Bước 1: ConfigMap

```bash
# Tạo từ literal
kubectl create configmap app-config \
  --from-literal=DB_HOST=postgres \
  --from-literal=DB_PORT=5432 \
  --from-literal=LOG_LEVEL=info

# Tạo từ file
kubectl create configmap app-config --from-file=app.properties

# Tạo từ folder
kubectl create configmap app-config --from-file=config/
```
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: postgres
  DB_PORT: "5432"
  app.properties: |
    server.port=8080
    server.tomcat.threads.max=200
  nginx.conf: |
    worker_processes auto;
    events { worker_connections 1024; }
```
```yaml
# pod-with-config.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
    envFrom:
    - configMapRef:
        name: app-config       # Tất cả key trở thành env
    volumeMounts:
    - name: config
      mountPath: /etc/app
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
```

### Bước 2: Secret

```bash
# Tạo secret
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password='S3cr3t!'

# Từ file
kubectl create secret generic tls-cert --from-file=tls.crt --from-file=tls.key

# Docker registry
kubectl create secret docker-registry regcred \
  --docker-server=ghcr.io \
  --docker-username=user \
  --docker-password=token

# SSH
kubectl create secret generic ssh-key --from-file=id_rsa=~/.ssh/id_rsa
```
```yaml
# pod-with-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
      defaultMode: 0400
  imagePullSecrets:
  - name: regcred
```

### Bước 3: Mã hoá Secret tại rest (EncryptionConfiguration)

```bash
# Generate key
head -c 32 /dev/urandom | base64
# Output: dGhpcy1pcy1hLXNhbXBsZS1rZXktMTIzNDU2Nzg5MA==
```
```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: dGhpcy1pcy1hLXNhbXBsZS1rZXktMTIzNDU2Nzg5MA==
      - identity: {}
```
```bash
# Apply vào K8s (cách tùy platform - ví dụ k3s)
sudo mkdir -p /etc/rancher/k3s
sudo cp encryption-config.yaml /etc/rancher/k3s/
sudo systemctl restart k3s

# Test
kubectl create secret generic test --from-literal=key=mysecret
kubectl get secret test -o yaml | grep -A2 encryptedData
```

### Bước 4: External Secrets Operator (sync từ Vault/AWS SSM)

```bash
# Cài ESO
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```
```yaml
# secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "k8s-role"
          serviceAccountRef:
            name: default
```
```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-credentials
  data:
  - secretKey: username
    remoteRef:
      key: prod/db
      property: user
  - secretKey: password
    remoteRef:
      key: prod/db
      property: pass
```

### Bước 5: RBAC cơ bản

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
```
```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update", "patch"]
```
```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rb
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io
```
```yaml
# pod-with-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: myapp:1.0
```

```bash
# Kiểm tra quyền
kubectl auth can-i list pods --as=system:serviceaccount:production:app-sa -n production
kubectl auth can-i create deployments --as=system:serviceaccount:production:app-sa -n production

# Xem role của SA
kubectl auth can-i --list --as=system:serviceaccount:production:app-sa -n production
```

### 🎯 Bài tập P4
1. Tạo ConfigMap, mount vào nginx làm config
2. Tạo Secret DB, inject vào Postgres
3. Tạo ServiceAccount riêng cho app, giới hạn quyền
4. Sync secret từ Vault bằng External Secrets

---

<a id="p5"></a>
## P5. Scaling & Scheduling

### Bước 1: HorizontalPodAutoscaler (HPA)

```bash
# Cài metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 30
      selectPolicy: Max
```
```bash
# Test HPA bằng cách tạo load
kubectl run -it --rm load-gen --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://web-svc; done"

# Xem HPA
kubectl get hpa
kubectl describe hpa app-hpa
```

### Bước 2: VerticalPodAutoscaler (VPA)

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```
```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Auto"      # Auto, Initial, Off
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
```

### Bước 3: KEDA (Event-driven autoscaling)

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
```
```yaml
# keda-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaler
spec:
  scaleTargetRef:
    name: worker
  pollingInterval: 10
  cooldownPeriod: 60
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
  - type: rabbitmq
    metadata:
      protocol: amqp
      queueName: tasks
      mode: QueueLength
      value: "30"
      host: rabbitmq.default.svc.cluster.local
```

### Bước 4: Node affinity / Pod affinity

```yaml
# pod-affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      affinity:
        # Cứng - phải match
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values: ["high-memory"]
              - key: topology.kubernetes.io/zone
                operator: NotIn
                values: ["us-east-1a"]
          # Mềm - ưu tiên
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: instance-type
                operator: In
                values: ["r6i.2xlarge"]
        # Pod cùng node
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["api"]
              topologyKey: kubernetes.io/hostname
        # Pod khác node
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["cache"]
              topologyKey: kubernetes.io/hostname
      containers:
      - name: cache
        image: redis:7
```

### Bước 5: Taints & Tolerations

```bash
# Taint node
kubectl taint nodes node1 dedicated=prod:NoSchedule
kubectl taint nodes node1 dedicated=prod:NoExecute

# Bỏ taint
kubectl taint nodes node1 dedicated-  # Note dấu -

# Xem taint
kubectl describe node node1 | grep Taints
```
```yaml
# pod-with-tolerations.yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-app
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "prod"
    effect: "NoSchedule"
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 30
  containers:
  - name: app
    image: myapp:1.0
```

### Bước 6: PodDisruptionBudget (PDB)

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2           # Hoặc
  # maxUnavailable: 1
  selector:
    matchLabels:
      app: api
```

### Bước 7: Pod Priority & Preemption

```yaml
# priorityclass.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Critical workloads"
```
```yaml
# pod-with-priority.yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: critical-app:1.0
```

### 🎯 Bài tập P5
1. Deploy app + HPA, tạo load xem scale
2. Taint node, deploy app có toleration
3. Deploy cache với pod anti-affinity
4. Tạo PDB cho app quan trọng

---

<a id="p6"></a>
## P6. Security

### Bước 1: Pod Security Standards (thay PSP)

```yaml
# Namespace labels - enforce restricted
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Các profile:**

| Profile | Mô tả |
|---------|--------|
| **privileged** | Không restrict |
| **baseline** | Ngăn chặn privilege escalation |
| **restricted** | Best practices (no root, no privileged...) |

### Bước 2: SecurityContext chi tiết

```yaml
# secure-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

### Bước 3: RBAC nâng cao

```yaml
# clusterrole-readonly.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-readonly
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/healthz", "/readyz", "/metrics"]
  verbs: ["get"]
```
```yaml
# clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dev-readonly
subjects:
- kind: Group
  name: developers         # OIDC group
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: [email protected]
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-readonly
  apiGroup: rbac.authorization.k8s.io
```

### Bước 4: Service Account token hardening

```yaml
# sa-with-token.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
automountServiceAccountToken: false   # Tắt auto-mount
```

### Bước 5: Image security - ImagePolicyWebhook

```yaml
# admission-control.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionReview
```

### Bước 6: OPA / Kyverno Policy

```yaml
# kyverno-disallow-privileged.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: deny-privileged
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Privileged containers are not allowed."
      pattern:
        spec:
          containers:
          - securityContext:
              privileged: false
```
```yaml
# kyverno-require-labels.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Audit
  rules:
  - name: check-team
    match:
      any:
      - resources:
          kinds:
          - Deployment
    validate:
      message: "Label 'team' is required."
      pattern:
        metadata:
          labels:
            team: "?*"
```

### Bước 7: Falco - runtime security

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco --namespace falco --create-namespace
```

### 🎯 Bài tập P6
1. Áp dụng pod-security=restricted cho namespace
2. Tạo RBAC cho team dev chỉ đọc được namespace của họ
3. Cài Kyverno, chặn pod có privileged
4. Cài Falco, xem alert khi shell vào container

---

<a id="p7"></a>
## P7. Observability

### Bước 1: Metrics - Prometheus + Grafana

```bash
# Cài kube-prometheus-stack (bao gồm Prometheus, Grafana, Alertmanager)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```
```bash
# Truy cập Grafana
kubectl port-forward -n monitoring svc/kube-prom-grafana 3000:80
# User: admin / Password: prom-operator
# Lấy password:
kubectl get secret -n monitoring kube-prom-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

```bash
# Cú pháp PromQL cơ bản
# CPU usage của pod
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod)

# Memory
sum(container_memory_working_set_bytes{namespace="production"}) by (pod)

# Request per second
sum(rate(http_requests_total[1m])) by (service)
```

### Bước 2: Alerting

```yaml
# prometheusrule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  labels:
    release: kube-prom
spec:
  groups:
  - name: app
    interval: 30s
    rules:
    - alert: HighCPUUsage
      expr: |
        sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU on {{ $labels.pod }}"
        description: "CPU > 80% for 5 minutes"

    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total[10m]) > 0
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
```
```yaml
# alertmanager-slack.yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: slack-config
spec:
  route:
    receiver: 'slack'
    groupBy: ['alertname', 'namespace']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
  receivers:
  - name: 'slack'
    slackConfigs:
    - apiURL:
        name: slack-webhook
        key: url
      channel: '#alerts'
      title: '{{ .CommonAnnotations.summary }}'
      text: '{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}'
```

### Bước 3: Logging với Loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace logging --create-namespace \
  --set promtail.enabled=true
```
```bash
# Query log với LogQL
{namespace="production"} |= "error" | json | level="error"

# Tail log real-time
stern -n production -l app=api
```

### Bước 4: Distributed Tracing với Tempo + OpenTelemetry

```yaml
# otel-collector.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    exporters:
      otlp:
        endpoint: tempo-distributor.tracing:4317
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [otlp]
```

### Bước 5: Profiling với Pyroscope / Parca

```bash
helm install pyroscope grafana/pyroscope --namespace profiling --create-namespace
```

### 🎯 Bài tập P7
1. Cài Prometheus stack, truy cập Grafana
2. Tạo dashboard cho application của bạn
3. Cấu hình alert Slack khi pod crash
4. Cài Loki, query log theo app

---

<a id="p8"></a>
## P8. Production Operations

### Bước 1: High Availability Cluster

**Cấu trúc HA K8s:**
```
       ┌─────────────────────────┐
       │      Load Balancer      │
       │   (HAProxy/NLB/ELB)     │
       └───────────┬─────────────┘
                   │
       ┌───────────┼───────────┐
       │           │           │
   ┌───▼───┐   ┌───▼───┐   ┌───▼───┐
   │ CP 1  │   │ CP 2  │   │ CP 3  │
   │API+ETCD│◄──┤API+ETCD│◄──┤API+ETCD│
   └───┬───┘   └───┬───┘   └───┬───┘
       └───────────┼───────────┘
                   │
   ┌───────────┬───┴───────┬───────────┐
   │           │           │           │
┌──▼──┐    ┌──▼──┐    ┌──▼──┐    ┌──▼──┐
│ W1  │    │ W2  │    │ W3  │    │ W4  │
└─────┘    └─────┘    └─────┘    └─────┘
```

**Dùng Kube-VIP cho HA control plane:**
```bash
# Cài Kube-VIP
kubectl apply -f https://raw.githubusercontent.com/kube-vip/kube-vip/v0.6.4/manifests/kube-vip-rbac.yaml
```

### Bước 2: etcd Backup & Restore

```bash
# Backup etcd (K3s/MicroK8s dùng SQLite, K8s kubeadm dùng etcd)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Crontab
0 2 * * * /usr/local/bin/etcd-backup.sh

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-20250915.db \
  --data-dir /var/lib/etcd-restore
```

**Velero - backup toàn bộ resources:**
```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero --namespace velero --create-namespace \
  --set configuration.backupStorageLocation[0].name=default \
  --set configuration.backupStorageLocation[0].bucket=my-bucket \
  --set configuration.backupStorageLocation[0].config.region=us-east-1 \
  --set configuration.provider=aws \
  --set credentials.existingSecret=velero-credentials
```
```bash
# Backup toàn cluster
velero backup create full-backup-$(date +%Y%m%d)

# Backup theo namespace
velero backup create ns-backup --include-namespaces production

# Schedule
velero schedule create daily-backup --schedule="0 2 * * *"

# Restore
velero restore create --from-backup full-backup-20250915
```

### Bước 3: Upgrade Cluster

```bash
# Kiểm tra version có sẵn
kubeadm upgrade plan

# Upgrade control plane
sudo kubeadm upgrade apply v1.30.0
sudo apt-get install -y kubelet=1.30.0-00 kubectl=1.30.0-00
sudo systemctl restart kubelet

# Upgrade worker từng node
kubectl cordon node-2
kubectl drain node-2 --ignore-daemonsets --delete-emptydir-data
# SSH vào node-2:
sudo kubeadm upgrade node
sudo apt-get install -y kubelet=1.30.0-00
sudo systemctl restart kubelet
# Quay lại control plane:
kubectl uncordon node-2

# K3s upgrade
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.30.0+k3s1 sh -
```

### Bước 4: Quota & LimitRange

```yaml
# resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    pods: "100"
    persistentvolumeclaims: "20"
    services: "50"
    secrets: "100"
    configmaps: "100"
    requests.storage: 1Ti
---
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: 50m
      memory: 64Mi
  - type: PersistentVolumeClaim
    max:
      storage: 100Gi
    min:
      storage: 1Gi
```

### Bước 5: Node maintenance

```bash
# Cordon (không schedule pod mới)
kubectl cordon node-1

# Drain (đẩy pod đi)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data --force

# Sau khi bảo trì
kubectl uncordon node-1

# Xem trạng thái
kubectl get nodes
```

### Bước 6: Certificates Management

```bash
# Cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```
```yaml
# cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: [email protected]
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

### 🎯 Bài tập P8
1. Setup Velero, backup toàn cluster lên S3
2. Test restore trên cluster mới
3. Cài cert-manager, cấp TLS cho app
4. Apply ResourceQuota cho namespace

---

<a id="p9"></a>
## P9. Advanced

### Bước 1: CustomResourceDefinition (CRD)

```yaml
# crd-application.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.example.com
spec:
  group: example.com
  scope: Namespaced
  names:
    plural: applications
    singular: application
    kind: Application
    shortNames: [app]
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              image:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 100
              env:
                type: array
                items:
                  type: object
                  properties:
                    name: {type: string}
                    value: {type: string}
```
```bash
kubectl apply -f crd-application.yaml
kubectl get crd
```
```yaml
# application.yaml
apiVersion: example.com/v1
kind: Application
metadata:
  name: my-app
spec:
  image: nginx:1.25
  replicas: 3
  env:
  - name: ENV
    value: production
```

### Bước 2: Operator pattern (Go)

```go
// controllers/application_controller.go
package controllers

import (
    context"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
)

type ApplicationReconciler struct {
    Client client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
}

func (r *ApplicationReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    app := &examplev1.Application{}
    if err := r.Get(ctx, req.NamespacedName, app); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Tìm deployment hiện tại
    deploy := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{Name: app.Name, Namespace: app.Namespace}, deploy)
    if err != nil && errors.IsNotFound(err) {
        // Tạo mới
        newDeploy := r.deploymentForApp(app)
        return ctrl.Result{}, r.Create(ctx, newDeploy)
    } else if err != nil {
        return ctrl.Result{}, err
    }

    // Update replicas
    desired := *app.Spec.Replicas
    if *deploy.Spec.Replicas != desired {
        deploy.Spec.Replicas = &desired
        return ctrl.Result{}, r.Update(ctx, deploy)
    }
    return ctrl.Result{}, nil
}
```

### Bước 3: Helm Chart - đóng gói ứng dụng

```bash
# Tạo chart
helm create mychart
```
```yaml
# mychart/Chart.yaml
apiVersion: v2
name: mychart
description: My application
version: 0.1.0
appVersion: "1.0"
type: application
```
```yaml
# mychart/values.yaml
replicaCount: 3

image:
  repository: nginx
  tag: 1.25
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: app.example.com

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```
```yaml
# mychart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 80
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```
```bash
# Test template
helm template myapp ./mychart --set replicaCount=5

# Install
helm install myapp ./mychart --namespace production --create-namespace

# Upgrade
helm upgrade myapp ./mychart -f custom-values.yaml

# Rollback
helm rollback myapp 1
```

### Bước 4: Kustomize

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
commonLabels:
  app: myapp
resources:
- deployment.yaml
- service.yaml
- ingress.yaml
configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=info
secretGenerator:
- name: db-secret
  literals:
  - username=admin
  - password=secret
patches:
- target:
    kind: Deployment
    name: myapp
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/resources
      value:
        limits:
          cpu: 1
          memory: 1Gi
```
```bash
# Build
kubectl kustomize ./overlay/production > manifest.yaml

# Apply
kubectl apply -k ./overlay/production
```

### Bước 5: GitOps với ArgoCD

```bash
# Cài ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-config
    targetRevision: HEAD
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
```
```bash
# Truy cập UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# User: admin
argocd admin initial-password -n argocd
```

### Bước 6: Service Mesh (Istio)

```bash
# Cài Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.21.*
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
```
```bash
# Inject sidecar vào namespace
kubectl label namespace production istio-injection=enabled
```
```yaml
# virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-vs
spec:
  hosts:
  - api
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: api
        subset: v2
  - route:
    - destination:
        host: api
        subset: v1
      weight: 90
    - destination:
        host: api
        subset: v2
      weight: 10
```
```yaml
# destinationrule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-dr
spec:
  host: api
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### 🎯 Bài tập P9
1. Tạo Helm chart cho app, cài qua helm install
2. Tạo Kustomize overlay cho 2 môi trường dev/prod
3. Cài ArgoCD, deploy app từ Git repo
4. Thử canary deployment với Istio

---

<a id="p10"></a>
## P10. Troubleshooting - Sửa lỗi thực chiến

### Bước 1: Lệnh debug không thể thiếu

```bash
# Mô tả tổng quan
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl describe svc <svc-name>
kubectl describe pvc <pvc-name>

# Logs
kubectl logs <pod>                           # Logs hiện tại
kubectl logs <pod> --previous                # Logs container trước (khi restart)
kubectl logs <pod> -c <container>            # Container cụ thể
kubectl logs <pod> --since=10m               # 10 phút gần nhất
kubectl logs -l app=api --tail=100           # Theo label
stern -n prod -l app=api                     # Multi-pod

# Exec
kubectl exec -it <pod> -- bash
kubectl exec -it <pod> -- sh
kubectl exec <pod> -- ps aux                 # Process bên trong
kubectl exec <pod> -- ls /var/log            # Xem file

# Port-forward
kubectl port-forward <pod> 8080:80
kubectl port-forward svc/<svc> 8080:80

# Copy file
kubectl cp <pod>:/var/log/app.log ./app.log
kubectl cp ./config.json <pod>:/app/config.json

# Debug ephemeral container (K8s 1.23+)
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>

# Debug node
kubectl debug node/<node> -it --image=ubuntu
```

### Bước 2: Top 10 lỗi thường gặp & cách sửa

#### ❌ Lỗi 1: `ImagePullBackOff` / `ErrImagePull`
```bash
kubectl describe pod <pod> | grep -A5 Events
# Nguyên nhân:
# - Sai tên image
# - Private registry chưa có imagePullSecrets
# - Network không ra ngoài
kubectl get events --field-selector involvedObject.name=<pod>
```
```bash
# Fix
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass>
```
```yaml
# Thêm vào pod/deployment
spec:
  imagePullSecrets:
  - name: regcred
```

#### ❌ Lỗi 2: `CrashLoopBackOff`
```bash
kubectl logs <pod> --previous
kubectl describe pod <pod>
```
```bash
# Nguyên nhân thường gặp:
# - Lỗi app (config sai, env thiếu)
# - Health check fail
# - Resource limit quá thấp (OOMKilled)
# - Permission denied
```
```yaml
# Fix - thêm probes tốt
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
```

#### ❌ Lỗi 3: `Pending` - Pod không được schedule
```bash
kubectl describe pod <pod> | grep Events -A10
```
```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  ...   default-scheduler  0/3 nodes are available: 
    3 Insufficient cpu, 2 Insufficient memory, 1 node(s) had taint...
```
```bash
# Fix:
# 1. Giảm resource requests
# 2. Thêm node mới
# 3. Thêm tolerations
# 4. Check affinity
kubectl describe nodes | grep -A5 "Allocated resources"
```

#### ❌ Lỗi 4: Service không truy cập được
```bash
# Checklist:
kubectl get svc <svc>                      # ClusterIP có?
kubectl get endpoints <svc>               # Endpoint có IP của pod?
kubectl get pods -l <selector của svc>     # Pod có match label?
kubectl get pods -o wide                   # IP pod đúng?

# Test trong cluster
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
nslookup <svc>
curl <svc>:<port>
# Nếu lỗi DNS:
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

#### ❌ Lỗi 5: PVC `Pending`
```bash
kubectl describe pvc <pvc>
```
```
Events:
  Warning  ProvisioningFailed  ...  failed to provision volume with StorageClass "fast": 
    error generating accessibility requirements: no topology key found on CSI node
```
```yaml
# Fix: thường do topology, đổi binding mode
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

#### ❌ Lỗi 6: Node `NotReady`
```bash
kubectl describe node <node>
# Check:
systemctl status kubelet
journalctl -u kubelet -f
systemctl status containerd
```
```bash
# Common fix:
sudo systemctl restart containerd
sudo systemctl restart kubelet
# Nếu network plugin lỗi:
kubectl -n kube-system logs <cni-pod>
```

#### ❌ Lỗi 7: OOMKilled
```bash
kubectl describe pod <pod> | grep -i oom
# Last State: Terminated, Reason: OOMKilled
```
```yaml
# Fix:
resources:
  requests:
    memory: 256Mi    # Đảm bảo scheduler biết
  limits:
    memory: 1Gi      # Tránh OOMKilled nhưng đủ lớn
```

#### ❌ Lỗi 8: Hết IP trong cluster
```
Events:
  Warning  FailedCreatePodSandBox  ...  network: failed to allocate for range 
  "10.244.0.0/16": no IP addresses available
```
```bash
# Fix: mở rộng CIDR hoặc dùng IPv6
# Cấu hình kubelet --max-pods, --pod-cidr hoặc dùng Cilium với larger pool
```

#### ❌ Lỗi 9: ConfigMap/Secret không update vào Pod
```bash
# Nguyên nhân: env từ ConfigMap chỉ load khi pod start
# Fix:
kubectl rollout restart deployment <name>

# Volume mount thì tự động update (sau 30-60s)
```

#### ❌ Lỗi 10: Pod không giao tiếp được sau khi áp NetworkPolicy
```bash
# Default-deny không có allow rule
# Fix: thêm rule allow cho namespace kube-system (DNS)
```
```yaml
# DNS rule
- to:
  - namespaceSelector: {}
  ports:
  - protocol: UDP
    port: 53
```

### Bước 3: Debug workflow chuẩn

```bash
#!/bin/bash
# debug-pod.sh - Script debug tự động
POD=$1
NS=${2:-default}

echo "=== 1. Describe Pod ==="
kubectl describe pod $POD -n $NS | tail -30

echo "=== 2. Recent Events ==="
kubectl get events -n $NS --field-selector involvedObject.name=$POD --sort-by='.lastTimestamp' | tail -10

echo "=== 3. Logs ==="
kubectl logs $POD -n $NS --tail=50 --timestamps

echo "=== 4. Previous Logs (if restarted) ==="
kubectl logs $POD -n $NS --previous --tail=50 2>/dev/null

echo "=== 5. Resource usage ==="
kubectl top pod $POD -n $NS 2>/dev/null || echo "metrics-server not installed"

echo "=== 6. Service endpoints ==="
kubectl get endpoints -n $NS

echo "=== 7. Node resources ==="
kubectl describe node $(kubectl get pod $POD -n $NS -o jsonpath='{.spec.nodeName}') | grep -A 10 "Allocated resources"

echo "=== 8. Network test ==="
kubectl run debug --rm -it --image=nicolaka/netshoot -n $NS --restart=Never -- bash -c "
  echo 'DNS test:'; nslookup kubernetes.default
  echo 'Ping pod IP:'; ping -c 2 \$(kubectl get pod $POD -n $NS -o jsonpath='{.status.podIP}')
"
```

### Bước 4: Debug nâng cao

```bash
# Cài nicolaka/netshoot - container có đủ tool debug
kubectl run netshoot --rm -it --image=nicolaka/netshoot --restart=Never -- bash
# Trong container:
nmap api-svc           # Port scan
tcpdump -i eth0        # Bắt gói tin
curl -v http://api     # Verbose HTTP
dig api-svc            # DNS lookup
```

```bash
# CPU/Memory profiling
kubectl exec <pod> -- top
kubectl exec <pod> -- cat /proc/<pid>/status
kubectl exec <pod> -- cat /sys/fs/cgroup/memory.current
```

```bash
# Trace system call
kubectl debug <pod> -it --image=ubuntu --target=<container>
apt install strace
strace -p <pid>
```

### Bước 5: Các công cụ debug hữu ích

```bash
# kubectl-debug plugin
kubectl krew install debug

# K9s - terminal UI
k9s
# Phím tắt:
# :pods, :svc, :deploy
# / để filter
# d để describe
# l để logs
# s để shell

# Kube-ops-view - cluster visualizer
kubectl apply -f https://raw.githubusercontent.com/hjacobs/kube-ops-view/master/kubernetes-kube-ops-view.yaml
```

### 🎯 Bài tập P10
1. Cố tình đặt image sai, fix ImagePullBackOff
2. Tạo resource limit quá thấp → OOMKilled → fix
3. Áp NetworkPolicy sai → debug connectivity
4. Dùng K9s thay thế hoàn toàn kubectl

---

## 🎓 Lộ trình trở thành Chuyên gia

### 🗓️ 3 tháng đầu - Nền tảng
- Hoàn thành P1-P3
- Thi **CKA** (Certified Kubernetes Administrator)
- Thực hành mỗi ngày trên K3s/Minikube

### 🗓️ 3-6 tháng - Trung cấp
- Hoàn thành P4-P7
- Thi **CKAD** (Certified Kubernetes Application Developer)
- Đóng góp vào project open-source K8s

### 🗓️ 6-12 tháng - Nâng cao
- Hoàn thành P8-P10
- Thi **CKS** (Certified Kubernetes Security Specialist)
- Xây dựng multi-cluster, multi-region
- Tham gia cộng đồng K8s Vietnam, KubeCon

### 📚 Tài liệu tham khảo
- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [KillerCoda](https://killercoda.com/kubernetes) - Lab miễn phí
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [Awesome Kubernetes](https://github.com/ramitsurana/awesome-kubernetes)

### 🛠️ Project cá nhân nên làm
1. **Home Lab**: Cluster K3s 3-node trên Raspberry Pi
2. **GitOps**: Deploy full-stack app (frontend + API + DB) qua ArgoCD
3. **Multi-cluster**: Dùng KubeFed hoặc Rancher
4. **E-commerce demo**: Microservices với Istio, monitoring với Prometheus

---

> **💡 Mẹo cuối cùng**: K8s là công nghệ rộng. Hãy chọn **1 vertical** sâu:
> - Platform Engineer: Cluster admin, networking, storage
> - Application Developer: Helm, GitOps, service mesh
> - Security: RBAC, Policy, runtime security
> - SRE: Observability, autoscaling, chaos engineering
>
> Và **luôn thực hành trên cluster thật**!

---

*Tạo bởi tài liệu học Kubernetes - Chúc bạn thành công! 🚀*
