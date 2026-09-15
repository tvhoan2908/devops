# 🌐 Tài liệu GCP (Google Cloud): Từ Zero đến Chuyên gia

> **Mục tiêu**: Thành thạo Google Cloud Platform: Compute, Storage, K8s (GKE), Serverless, IAM, BigQuery.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Chuẩn bị](#p0) | gcloud CLI, project, auth |
| [P1. Compute](#p1) | GCE, MIG, templates |
| [P2. VPC & Network](#p2) | VPC, firewall, LB, Cloud CDN |
| [P3. Storage](#p3) | Cloud Storage, classes, lifecycle |
| [P4. Database](#p4) | Cloud SQL, Spanner, Firestore, Memorystore |
| [P5. GKE](#p5) | GKE Autopilot/Standard, workloads |
| [P6. IAM & Security](#p6) | IAM, Service Account, Secret Manager |
| [P7. Serverless](#p7) | Cloud Functions, Cloud Run, Workflows |
| [P8. BigQuery & Data](#p8) | BigQuery, Pub/Sub, Dataflow |
| [P9. Monitoring](#p9) | Cloud Monitoring, Logging, Trace |
| [P10. Best Practice](#p10) | Cost, security, well-architected |

---

<a id="p0"></a>
## P0. Chuẩn bị

### Bước 1: Tạo project & billing

```
1. https://console.cloud.google.com/
2. Tạo project mới (hoặc dùng có sẵn)
3. Link billing account (cần thẻ tín dụng)
4. Enable APIs cần thiết
```

```bash
# Enable APIs
gcloud services enable compute.googleapis.com \
  container.googleapis.com \
  sqladmin.googleapis.com \
  storage.googleapis.com \
  cloudfunctions.googleapis.com
```

### Bước 2: Cài gcloud CLI

```bash
# macOS
brew install --cask google-cloud-sdk

# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# Windows
# Tải installer từ https://cloud.google.com/sdk/docs/install

# Verify
gcloud version
```

### Bước 3: Khởi tạo

```bash
# Login
gcloud auth login

# Application credentials (cho tools)
gcloud auth application-default login

# Set project
gcloud config set project my-project-id

# Set region/zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# List configs
gcloud config list

# Multiple configs
gcloud config configurations create dev
gcloud config configurations activate dev
gcloud config set project my-dev-project
```

### Bước 4: Công cụ bổ sung

```bash
# gsutil (Storage CLI) - đã có
gsutil --version

# gke-gcloud-auth-plugin (cho kubectl)
gcloud components install gke-gcloud-auth-plugin

# Terraform
brew install terraform

# kubectl
gcloud components install kubectl

# Kubectx
brew install kubectx
```

---

<a id="p1"></a>
## P1. Compute Engine (GCE)

### Bước 1: Tạo VM

```bash
# VM đầu tiên
gcloud compute instances create web-1 \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-balanced \
  --tags=http-server,https-server \
  --network=default \
  --subnet=default \
  --metadata-from-file=startup-script=./startup.sh \
  --labels=env=dev,app=web \
  --scopes=cloud-platform

# List
gcloud compute instances list
gcloud compute instances describe web-1 --zone=us-central1-a

# SSH
gcloud compute ssh web-1 --zone=us-central1-a

# Stop/Start/Delete
gcloud compute instances stop web-1 --zone=us-central1-a
gcloud compute instances start web-1 --zone=us-central1-a
gcloud compute instances delete web-1 --zone=us-central1-a
```

### Bước 2: Startup script

```bash
#!/bin/bash
# startup.sh
apt-get update
apt-get install -y nginx
echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
systemctl enable nginx
```

### Bước 3: Instance templates & groups

```bash
# Template
gcloud compute instance-templates create web-template \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=http-server \
  --network=default \
  --metadata-from-file=startup-script=./startup.sh

# Managed Instance Group
gcloud compute instance-groups managed create web-mig \
  --base-instance-name=web \
  --template=web-template \
  --size=3 \
  --zone=us-central1-a

# Autoscaling
gcloud compute instance-groups managed set-autoscaling web-mig \
  --min-num-replicas=1 \
  --max-num-replicas=10 \
  --target-cpu-utilization=0.7 \
  --zone=us-central1-a

# Rolling update
gcloud compute instance-groups managed rolling-action start-update web-mig \
  --version=template=web-template-v2 \
  --max-surge=3 \
  --max-unavailable=0 \
  --zone=us-central1-a
```

### Bước 4: Preemptible/Spot VMs

```bash
# Spot VM (rẻ 60-90%)
gcloud compute instances create spot-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --preemptible \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

### Bước 5: GPU/TPU

```bash
# GPU
gcloud compute instances create gpu-vm \
  --zone=us-central1-a \
  --machine-type=n1-standard-8 \
  --accelerator=type=nvidia-tesla-t4,count=1 \
  --image-family=pytorch-latest-gpu \
  --image-project=deeplearning-platform-release \
  --maintenance-policy=TERMINATE

# TPU
gcloud compute tpus create my-tpu \
  --zone=us-central1-a \
  --accelerator-type=v3-8 \
  --version=2.13.0
```

---

<a id="p2"></a>
## P2. VPC & Network

### Bước 1: VPC

```bash
# Custom VPC
gcloud compute networks create my-vpc \
  --subnet-mode=custom

# Subnets (multi-region)
gcloud compute networks subnets create web-us-central1 \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24 \
  --enable-private-ip-google-access

gcloud compute networks subnets create db-us-central1 \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.10.0/24

# List
gcloud compute networks list
gcloud compute networks subnets list

# Shared VPC (multi-project)
gcloud compute shared-vpc host-project enable my-host-project
gcloud compute shared-vpc associated-projects add my-service-project \
  --host-project=my-host-project
```

### Bước 2: Firewall

```bash
# Allow HTTP
gcloud compute firewall-rules create allow-http \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server \
  --rules=tcp:80

# Allow HTTPS
gcloud compute firewall-rules create allow-https \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges=0.0.0.0/0 \
  --target-tags=https-server \
  --rules=tcp:443

# Allow internal
gcloud compute firewall-rules create allow-internal \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges=10.0.0.0/16 \
  --rules=tcp,udp,icmp

# Allow SSH from specific IP
gcloud compute firewall-rules create allow-ssh \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges=1.2.3.0/24 \
  --rules=tcp:22
```

### Bước 3: Cloud NAT

```bash
# Cho phép VM private ra internet
gcloud compute routers create my-router \
  --network=my-vpc \
  --region=us-central1

gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

### Bước 4: Load Balancer

```bash
# Health check
gcloud compute health-checks create http web-health-check \
  --port=80 \
  --request-path=/health \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3

# Backend service
gcloud compute backend-services create web-backend \
  --load-balancing-scheme=EXTERNAL \
  --protocol=HTTP \
  --health-checks=web-health-check \
  --global \
  --enable-cdn

# Add MIG
gcloud compute backend-services add-backend web-backend \
  --instance-group=web-mig \
  --instance-group-zone=us-central1-a \
  --global

# URL map
gcloud compute url-maps create web-map \
  --default-service=web-backend

# HTTP proxy
gcloud compute target-http-proxies create web-proxy \
  --url-map=web-map

# Forwarding rule (external IP)
gcloud compute forwarding-rules create web-rule \
  --load-balancing-scheme=EXTERNAL \
  --address=web-ip \
  --global \
  --target-http-proxy=web-proxy \
  --ports=80

# Reserve static IP
gcloud compute addresses create web-ip --global
```

### Bước 5: Cloud DNS

```bash
# Managed zone
gcloud dns managed-zones create my-zone \
  --dns-name=example.com. \
  --description="My domain"

# Record set
gcloud dns record-sets create app.example.com \
  --zone=my-zone \
  --type=A \
  --ttl=300 \
  --rrdatas=1.2.3.4

# CNAME
gcloud dns record-sets create www.example.com \
  --zone=my-zone \
  --type=CNAME \
  --ttl=300 \
  --rrdatas=app.example.com.

# List
gcloud dns record-sets list --zone=my-zone
```

### Bước 6: Cloud CDN

```bash
# Enable CDN cho backend
gcloud compute backend-services update web-backend \
  --enable-cdn \
  --cache-mode=CACHE_ALL_STATIC \
  --default-ttl=3600 \
  --max-ttl=86400 \
  --client-ttl=3600 \
  --global
```

---

<a id="p3"></a>
## P3. Cloud Storage

### Bước 1: Bucket

```bash
# Tạo bucket
gsutil mb -l us-central1 gs://my-unique-bucket-12345

# Class
gsutil mb -c STANDARD -l us-central1 gs://my-bucket
gsutil mb -c NEARLINE -l us-central1 gs://my-bucket
gsutil mb -c COLDLINE -l us-central1 gs://my-bucket
gsutil mb -c ARCHIVE -l us-central1 gs://my-bucket

# Versioning
gsutil versioning set on gs://my-bucket

# List
gsutil ls
gsutil ls -L gs://my-bucket
```

### Bước 2: Upload/Download

```bash
# Upload
gsutil cp file.txt gs://my-bucket/
gsutil cp -r directory/ gs://my-bucket/path/

# Sync
gsutil rsync -r ./local/ gs://my-bucket/

# Download
gsutil cp gs://my-bucket/file.txt ./
gsutil cp -r gs://my-bucket/path/ ./

# Parallel composite upload
gsutil -m cp -r bigdir/ gs://my-bucket/bigdir/
```

### Bước 3: Lifecycle

```bash
cat > lifecycle.json <<EOF
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30}
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
        "condition": {"age": 90}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 365}
      }
    ]
  }
}
EOF

gsutil lifecycle set lifecycle.json gs://my-bucket
```

### Bước 4: IAM cho bucket

```bash
# Public read
gsutil iam ch allUsers:objectViewer gs://my-bucket

# Specific user
gsutil iam ch user:[email protected]:objectViewer gs://my-bucket

# Service account
gsutil iam ch serviceAccount:[email protected]:objectAdmin gs://my-bucket

# Uniform bucket-level access (recommended)
gsutil uniformbucketlevelaccess set on gs://my-bucket

# Signed URL (temporary access)
gsutil signurl -d 1h key.json gs://my-bucket/private-file.txt
```

### Bước 5: Object versioning

```bash
gsutil versioning set on gs://my-bucket

# List versions
gsutil ls -a gs://my-bucket/file.txt

# Restore older version
gsutil cp gs://my-bucket/file.txt#1234567890 ./file.txt.old

# Delete version
gsutil rm gs://my-bucket/file.txt#1234567890
```

### Bước 6: Encryption

```bash
# CMEK (Customer-Managed Encryption Keys)
# 1. Tạo KMS key
gcloud kms keyrings create my-keyring --location=us-central1
gcloud kms keys create my-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --purpose=encryption

# 2. Apply cho bucket
gsutil kms encryption -k \
  projects/PROJECT/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key \
  gs://my-bucket
```

---

<a id="p4"></a>
## P4. Database

### Bước 1: Cloud SQL

```bash
# Tạo instance
gcloud sql instances create my-db \
  --database-version=POSTGRES_15 \
  --tier=db-g1-small \
  --region=us-central1 \
  --root-password="$DB_PASSWORD" \
  --network=projects/my-project/global/networks/my-vpc \
  --no-assign-ip \
  --backup-start-time=02:00 \
  --enable-point-in-time-recovery \
  --maintenance-window-day=SUN \
  --maintenance-window-hour=03 \
  --database-flags=log_min_duration_statement=1000

# Database
gcloud sql databases create myapp \
  --instance=my-db

# User
gcloud sql users create app \
  --instance=my-db \
  --password="$APP_PASSWORD"

# Connection
gcloud sql instances describe my-db --format="value(connectionName)"
# project:region:instance

# Connect from local
gcloud sql connect my-db --user=app

# High availability
gcloud sql instances patch my-db --availability-type=REGIONAL

# Read replica
gcloud sql instances create my-db-replica \
  --master-instance-name=my-db \
  --region=us-east1

# Backup
gcloud sql backups create --instance=my-db

# Restore
gcloud sql backups restore <backup-id> --restore-instance=my-db
```

### Bước 2: Cloud SQL Auth Proxy

```bash
# Tải
wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
chmod +x cloud_sql_proxy

# Chạy
./cloud_sql_proxy -instances=PROJECT:REGION:INSTANCE=tcp:5432

# K8s sidecar
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp
        env:
        - name: DB_HOST
          value: 127.0.0.1
        - name: DB_PORT
          value: "5432"
      - name: cloud-sql-proxy
        image: gcr.io/cloudsql-docker/gce-proxy:1.33.2
        command:
        - "/cloud_sql_proxy"
        - "-instances=PROJECT:REGION:INSTANCE=tcp:5432"
        securityContext:
          runAsNonRoot: true
```

### Bước 3: Spanner (distributed SQL)

```bash
# Tạo instance
gcloud spanner instances create my-instance \
  --config=regional-us-central1 \
  --description="My Spanner" \
  --nodes=3

# Database
gcloud spanner databases create mydb \
  --instance=my-instance

# DDL
gcloud spanner databases ddl update mydb \
  --instance=my-instance \
  --ddl="CREATE TABLE Users (UserId INT64 NOT NULL, Name STRING(100)) PRIMARY KEY (UserId)"
```

### BƯớc 4: Firestore (NoSQL)

```bash
# Enable
gcloud services enable firestore.googleapis.com
gcloud firestore databases create --location=us-central1

# Native mode vs Datastore mode

# Query
gcloud firestore databases describe

# Backup & restore
gcloud firestore export gs://my-bucket/firestore-backup
gcloud firestore import gs://my-bucket/firestore-backup
```

```javascript
// Node.js Firestore
const { Firestore } = require('@google-cloud/firestore');
const firestore = new Firestore();

async function getUser(userId) {
    const doc = await firestore.collection('users').doc(userId).get();
    return doc.exists ? doc.data() : null;
}

async function setUser(userId, data) {
    await firestore.collection('users').doc(userId).set(data);
}
```

### Bước 5: Memorystore (Redis)

```bash
# Tạo Redis
gcloud redis instances create my-redis \
  --size=1 \
  --region=us-central1 \
  --tier=basic \
  --network=projects/my-project/global/networks/my-vpc \
  --redis-version=redis_7_0 \
  --enable-auth

# Connect
gcloud redis instances describe my-redis --region=us-central1 \
  --format="value(host,port)"
```

### Bước 6: Bigtable (wide-column)

```bash
gcloud bigtable instances create my-instance \
  --cluster=my-cluster \
  --cluster-zone=us-central1-a \
  --cluster-num-nodes=3 \
  --display-name="My Bigtable"

gcloud bigtable clusters create my-cluster-2 \
  --instance=my-instance \
  --zone=us-east1-a \
  --num-nodes=3

# Tables (qua cbt tool)
cbt -instance my-instance -project my-project createtable mytable "families=cf1"
```

---

<a id="p5"></a>
## P5. GKE (Google Kubernetes Engine)

### Bước 1: Cluster

```bash
# Standard cluster
gcloud container clusters create my-cluster \
  --region=us-central1 \
  --num-nodes=3 \
  --machine-type=e2-standard-2 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --enable-autorepair \
  --enable-autoupgrade \
  --enable-ip-alias \
  --network=my-vpc \
  --subnetwork=us-central1 \
  --release-channel=regular \
  --enable-stackdriver-kubernetes

# Autopilot (fully managed)
gcloud container clusters create-auto my-cluster \
  --region=us-central1 \
  --release-channel=regular

# Get credentials
gcloud container clusters get-credentials my-cluster --region=us-central1

# List
gcloud container clusters list
gcloud container clusters describe my-cluster --region=us-central1
```

### Bước 2: Node pools

```bash
# Add node pool
gcloud container node-pools create high-memory \
  --cluster=my-cluster \
  --region=us-central1 \
  --machine-type=e2-highmem-4 \
  --num-nodes=1 \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=5

# Spot/Preemptible
gcloud container node-pools create spot-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --spot \
  --num-nodes=3

# GPU
gcloud container node-pools create gpu-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --accelerator=type=nvidia-tesla-t4,count=1 \
  --machine-type=n1-standard-4
```

### Bước 3: GKE workloads

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: gcr.io/my-project/myapp:v1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

### Bước 4: GKE Autopilot features

```yaml
# Autopilot tự động scale, manage nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: gcr.io/my-project/myapp:v1
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          # NO limits - Autopilot sẽ tự manage
```

### Bước 5: GKE Ingress & Cloud CDN

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    kubernetes.io/ingress.class: gce
    networking.gke.io/managed-certificates: myapp-cert
    cloud.google.com/neg: '{"ingress": true}'
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

```yaml
# Managed Certificate
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: myapp-cert
spec:
  domains:
  - app.example.com
```

### Bước 6: GKE Logging & Monitoring

```bash
# Enable
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --enable-stackdriver-kubernetes
```

---

<a id="p6"></a>
## P6. IAM & Security

### Bước 1: IAM

```bash
# Service account
gcloud iam service-accounts create myapp-sa \
  --display-name="MyApp Service Account"

# Grant role
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:[email protected]" \
  --role="roles/storage.objectViewer"

# Create key (cho app sử dụng)
gcloud iam service-accounts keys create key.json \
  --iam-account=[email protected]

# Use với app
export GOOGLE_APPLICATION_CREDENTIALS=key.json

# Short-lived token (recommended)
gcloud auth print-access-token --impersonate-service-account=[email protected]
```

### Bước 2: IAM conditions

```bash
# Conditional access - chỉ từ IP cụ thể
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:[email protected]" \
  --role="roles/storage.objectAdmin" \
  --condition="expression=request.time < timestamp('2025-12-31T00:00:00Z'),title=expiry,description=Token expires end of 2025"
```

### BƯớc 3: Workload Identity (GKE)

```bash
# Enable
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --workload-pool=my-project.svc.id.goog

# Bind K8s SA to GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  [email protected] \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-project.svc.id.goog[default/myapp-sa]"
```
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  annotations:
    iam.gke.io/gcp-service-account: [email protected]
```

### Bước 4: Secret Manager

```bash
# Tạo secret
echo -n "my-secret-value" | \
  gcloud secrets create db-password --data-file=-

# Version
gcloud secrets versions add db-password --data-file=-

# Access
gcloud secrets versions access latest --secret=db-password

# IAM
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:[email protected]" \
  --role="roles/secretmanager.secretAccessor"

# Mount trong VM
gcloud compute instances create web-1 \
  --metadata-from-file=db-password=./db-password.txt
```

### BƯớc 5: Cloud KMS

```bash
# Keyring + Key
gcloud kms keyrings create my-keyring --location=us-central1
gcloud kms keys create my-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --purpose=encryption \
  --rotation-period=90d

# Encrypt
echo -n "secret" | \
  gcloud kms encrypt \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=my-key \
  --plaintext-file=- \
  --ciphertext-file=encrypted.bin
```

---

<a id="p7"></a>
## P7. Serverless

### Bước 1: Cloud Functions

```javascript
// index.js
const functions = require('@google-cloud/functions-framework');

functions.http('hello', (req, res) => {
    const name = req.query.name || req.body.name || 'World';
    res.send(`Hello, ${name}!`);
});
```

```bash
# Deploy
gcloud functions deploy hello \
  --runtime=nodejs20 \
  --trigger-http \
  --region=us-central1 \
  --source=. \
  --entry-point=hello \
  --memory=256MB \
  --timeout=30s \
  --max-instances=100 \
  --allow-unauthenticated

# Call
URL=$(gcloud functions describe hello --region=us-central1 --format="value(serviceConfig.uri)")
curl $URL
```

```python
# main.py
import functions_framework
from flask import jsonify

@functions_framework.http
def hello(request):
    name = request.args.get('name', 'World')
    return jsonify({'message': f'Hello, {name}!'})
```

### Bước 2: Cloud Run

```bash
# Build & push image
gcloud builds submit --tag gcr.io/my-project/myapp

# Deploy
gcloud run deploy myapp \
  --image=gcr.io/my-project/myapp \
  --region=us-central1 \
  --platform=managed \
  --allow-unauthenticated \
  --memory=512Mi \
  --cpu=1 \
  --concurrency=80 \
  --max-instances=100 \
  --min-instances=1 \
  --port=8080 \
  --timeout=60 \
  --set-env-vars="ENV=production,DB_HOST=10.0.0.1" \
  --set-secrets="DB_PASSWORD=db-password:latest" \
  --vpc-connector=my-vpc-connector \
  --vpc-egress=private-ranges-only

# Custom domain
gcloud run domain-mappings create \
  --service=myapp \
  --domain=app.example.com \
  --region=us-central1
```

```yaml
# Knative-style service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - image: gcr.io/my-project/myapp
        resources:
          limits:
            memory: 512Mi
            cpu: 1
        env:
        - name: DB_HOST
          value: 10.0.0.1
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

### Bước 3: Event-driven (Eventarc)

```bash
# Storage trigger
gcloud eventarc triggers create storage-trigger \
  --destination-run-service=process-file \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-bucket" \
  --service-account=[email protected]

# Pub/Sub trigger
gcloud eventarc triggers create pubsub-trigger \
  --destination-run-service=process-message \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.pubsub.topic.v1.messagePublished" \
  --transport-topic=my-topic
```

### Bước 4: Cloud Workflows

```yaml
# workflow.yaml
main:
  params: [input]
  steps:
  - init:
      assign:
      - orderId: ${input.orderId}
  - validateOrder:
      call: http.post
      args:
        url: https://us-central1-my-project.cloudfunctions.net/validate
        body:
          orderId: ${orderId}
      result: validationResult
  - checkValidation:
      switch:
        - condition: ${validationResult.valid}
          next: processPayment
        - next: returnError
  - processPayment:
      call: http.post
      args:
        url: https://us-central1-my-project.cloudfunctions.net/payment
        body:
          orderId: ${orderId}
      result: paymentResult
      next: shipOrder
  - shipOrder:
      call: http.post
      args:
        url: https://us-central1-my-project.cloudfunctions.net/ship
        body:
          orderId: ${orderId}
      next: returnSuccess
  - returnSuccess:
      return: ${paymentResult}
  - returnError:
      raise:
        message: "Invalid order"
        code: 400
```

```bash
gcloud workflows deploy my-workflow --source=workflow.yaml
gcloud workflows execute my-workflow --data='{"orderId":"123"}'
```

---

<a id="p8"></a>
## P8. BigQuery & Data

### Bước 1: BigQuery cơ bản

```bash
# Dataset
bq mk my_dataset

# Table từ schema
bq mk -t my_dataset.users \
  userId:INTEGER,username:STRING,email:STRING,created_at:TIMESTAMP

# Query
bq query --use_legacy_sql=false \
  'SELECT COUNT(*) FROM `my-project.my_dataset.users`'

# Load CSV
bq load --source_format=CSV \
  my_dataset.users \
  gs://my-bucket/data.csv \
  userId:INTEGER,username:STRING,email:STRING

# Export
bq extract my_dataset.users gs://my-bucket/export/users-*.csv
```

### Bước 2: Pub/Sub

```bash
# Topic
gcloud pubsub topics create my-topic

# Subscription
gcloud pubsub subscriptions create my-sub \
  --topic=my-topic \
  --ack-deadline=60 \
  --message-retention-duration=7d \
  --expiration-period=never

# Publish
gcloud pubsub topics publish my-topic --message="Hello"

# Pull
gcloud pubsub subscriptions pull my-sub --limit=10

# Push subscription
gcloud pubsub subscriptions create my-sub-push \
  --topic=my-topic \
  --push-endpoint=https://my-app.com/webhook
```

```python
# Publisher
from google.cloud import pubsub_v1
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path('my-project', 'my-topic')
future = publisher.publish(topic_path, b'Message data', attr='value')
print(future.result())

# Subscriber
from google.cloud import pubsub_v1
subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path('my-project', 'my-sub')

def callback(message):
    print(f"Received: {message.data}")
    message.ack()

subscriber.subscribe(subscription_path, callback=callback)
```

### Bước 3: Dataflow (Apache Beam)

```python
# pipeline.py
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

options = PipelineOptions(
    project='my-project',
    runner='DataflowRunner',
    region='us-central1',
    job_name='my-pipeline',
    temp_location='gs://my-bucket/temp',
    staging_location='gs://my-bucket/staging',
)

with beam.Pipeline(options=options) as pipeline:
    (pipeline
     | 'Read from Pub/Sub' >> beam.io.ReadFromPubSub(topic='projects/my-project/topics/my-topic')
     | 'Parse JSON' >> beam.Map(lambda x: x.decode('utf-8'))
     | 'Transform' >> beam.Map(lambda x: {'id': x.split(',')[0], 'value': x.split(',')[1]})
     | 'Write to BigQuery' >> beam.io.WriteToBigQuery(
         'my-project:my_dataset.events',
         schema='id:STRING,value:STRING',
         write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND
     ))
```

```bash
python pipeline.py
```

---

<a id="p9"></a>
## P9. Monitoring & Logging

### BƯớc 1: Cloud Logging

```bash
# View logs
gcloud logging read "resource.type=k8s_container AND resource.labels.namespace_name=default" \
  --limit=50 \
  --format=json \
  --project=my-project

# Tail logs
gcloud logging tail "resource.type=global" --project=my-project

# Logs từ specific service
gcloud logging read "resource.type=k8s_container AND labels.k8s-pod/app=myapp" \
  --limit=20

# Export sang BigQuery
gcloud logging sinks create my-sink \
  bigquery.googleapis.com/projects/my-project/datasets/my_logs \
  --log-filter='resource.type="k8s_container" AND severity>=ERROR'

# Export sang GCS
gcloud logging sinks create my-gcs-sink \
  storage.googleapis.com/my-logs-bucket \
  --log-filter='resource.type="k8s_container"'
```

### Bước 2: Cloud Monitoring

```bash
# Uptime check
gcloud monitoring uptime create my-app-check \
  --resource-type=uptime-url \
  --host=app.example.com \
  --path=/health \
  --check-interval=60s

# Alert policy
cat > alert.json <<EOF
{
  "displayName": "High CPU",
  "combiner": "OR",
  "conditions": [{
    "displayName": "CPU > 80%",
    "conditionThreshold": {
      "filter": "metric.type=\"compute.googleapis.com/instance/cpu/utilization\"",
      "comparison": "COMPARISON_GT",
      "thresholdValue": 0.8,
      "duration": "300s"
    }
  }],
  "alertStrategy": {
    "autoClose": "1800s"
  }
}
EOF

gcloud alpha monitoring policies create --policy-from-file=alert.json
```

### Bước 3: Cloud Trace

```javascript
// Node.js auto-instrumentation
require('@google-cloud/trace-agent').start({
    projectId: 'my-project',
    samplingRate: 10
});

// Custom spans
const tracer = require('@google-cloud/trace-agent').get();
tracer.runInRootSpan({name: 'processOrder'}, (rootSpan) => {
    tracer.runInSpanWithParent('validateOrder', rootSpan, () => {
        // ...
    });
});
```

### Bước 4: Error Reporting

```javascript
// Stackdriver Error Reporting tự động
require('@google-cloud/error-reporting').start({
    projectId: 'my-project',
    reportMode: 'always',
    serviceContext: {
        service: 'myapp',
        version: '1.0.0'
    }
});
```

---

<a id="p10"></a>
## P10. Best Practices

### Bước 1: Cost optimization

```bash
# Committed Use Discounts (1-3 năm)
gcloud compute commitments create my-commitment \
  --region=us-central1 \
  --resources=vcpu=100,memory=400GB \
  --plan=12-month

# Spot VMs
gcloud compute instances create spot-vm --preemptible

# Right-sizing recommendation
gcloud recommender recommendations list \
  --project=my-project \
  --location=us-central1-a \
  --recommender=google.compute.instance.MachineTypeRecommender

# Budget alerts
gcloud billing budgets create \
  --billing-account=BILLING_ACCOUNT_ID \
  --display-name="Monthly Budget" \
  --budget-amount=1000USD \
  --threshold-rule=percent=50 \
  --threshold-rule=percent=90
```

### BƯớc 2: Labels & Organization

```bash
# Labels
gcloud compute instances create web-1 \
  --labels=env=production,team=platform,project=myapp

# Organization policies
gcloud org-policies set-policy /tmp/policy.yaml \
  --project=my-project

# Folder structure
gcloud resource-manager folders create my-folder \
  --display-name="Production" \
  --organization=ORG_ID
```

### Bước 3: Security checklist

```
1. Service account ít quyền nhất có thể
2. Workload Identity thay vì key files
3. VPC Service Controls cho sensitive data
4. IAM Conditions với expiry
5. Audit logs enabled
6. Encryption với CMEK cho data nhạy cảm
7. Binary Authorization cho container
8. VPC firewall rules chặt
```

### Bước 4: VPC Service Controls

```bash
# Tạo service perimeter
gcloud access-context-manager perimeters create my-perimeter \
  --title="My Perimeter" \
  --resources=projects/PROJECT_NUMBER \
  --restricted-services=storage.googleapis.com,bigquery.googleapis.com \
  --policy=POLICY_ID
```

### BƯớc 5: Well-Architected Framework

```
5 pillars:
1. Operational Excellence
   - IaC (Terraform, Deployment Manager)
   - Cloud Monitoring + Logging
   - Runbooks

2. Security
   - BeyondCorp (zero-trust)
   - IAM least privilege
   - VPC SC, CMEK
   - Binary Authorization

3. Reliability
   - Multi-zone (GKE regional cluster)
   - Backup (Cloud SQL, GCS versioning)
   - DR plan

4. Performance
   - Premium network tier
   - Cloud CDN
   - Memorystore
   - Right-sized machines

5. Cost Optimization
   - Committed Use Discounts
   - Spot VMs
   - Sustained Use Discounts (auto)
   - Resource labeling
```

---

## 🎯 Bài tập P0-P10

1. Tạo project, cài gcloud, setup multi-region
2. Deploy web app trên GCE + Load Balancer + CDN
3. Setup VPC + Cloud SQL private + Memorystore
4. GKE cluster với Autopilot, deploy app qua Ingress + Cloud CDN
5. Cloud Function trigger từ Pub/Sub, ghi vào BigQuery
6. Workload Identity cho GKE app access Secret Manager
7. IAM với conditions, audit logs review

---

> **💡 Tip cuối**: GCP focus vào data, ML, K8s. GKE Autopilot giảm ops overhead rất nhiều. BigQuery miễn phí 10GB, dùng để log analysis. Workload Identity + Service Account thay vì key files. Always Free Tier có thể dùng để học.

---

*Tạo bởi tài liệu học GCP - Chúc bạn thành công! 🚀*
