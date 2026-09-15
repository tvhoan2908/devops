# ☁️ Tài liệu AWS: Từ Zero đến Chuyên gia

> **Mục tiêu**: Thành thạo AWS core services: EC2, S3, VPC, RDS, Lambda, IAM, và automation với Terraform/CDK.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Chuẩn bị](#p0) | Tài khoản, IAM, CLI |
| [P1. EC2 & Compute](#p1) | Instance, AMI, ASG |
| [P2. VPC & Network](#p2) | Subnet, Route, NAT, IGW |
| [P3. S3 & Storage](#p3) | Bucket, lifecycle, Glacier |
| [P4. Database](#p4) | RDS, DynamoDB, Aurora |
| [P5. IAM & Security](#p5) | User, Role, Policy, MFA |
| [P6. Lambda & Serverless](#p6) | Function, API Gateway, EventBridge |
| [P7. Container](#p7) | ECS, EKS, Fargate |
| [P8. Monitoring](#p8) | CloudWatch, X-Ray, CloudTrail |
| [P9. Cost & Best Practice](#p9) | Tối ưu chi phí, well-architected |

---

<a id="p0"></a>
## P0. Chuẩn bị

### Bước 1: Tạo tài khoản & MFA

```
1. Đăng ký: https://aws.amazon.com/
2. Root account (KHÔNG dùng để làm việc)
3. Enable MFA cho root
4. Tạo IAM admin user
5. Login với IAM user, enable MFA
```

### Bước 2: Cài AWS CLI v2

```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Windows
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# Verify
aws --version
```

### Bước 3: Cấu hình CLI

```bash
# Cấu hình nhanh
aws configure
# AWS Access Key ID: AKIA...
# AWS Secret Access Key: ...
# Default region: us-east-1
# Default output format: json

# File lưu tại ~/.aws/credentials và ~/.aws/config

# Multiple profiles
aws configure --profile dev
aws configure --profile prod

# Sử dụng
aws s3 ls --profile prod
AWS_PROFILE=prod aws s3 ls
```

```ini
# ~/.aws/config
[default]
region = us-east-1
output = json

[profile prod]
region = us-east-1
output = json
source_profile = default
role_arn = arn:aws:iam::123456789012:role/Admin
mfa_serial = arn:aws:iam::123456789012:mfa/user
```

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = ...

[prod]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
```

### Bước 4: SSO (khuyến nghị cho team)

```bash
# Configure SSO
aws configure sso
# SSO session name: my-sso
# SSO start URL: https://mycompany.awsapps.com/start
# SSO region: us-east-1
# Account ID: 123456789012
# Role name: AdminRole

# Login
aws sso login --profile my-sso

# Use
aws s3 ls --profile my-sso
```

### Bước 5: Helper tools

```bash
# aws-shell (interactive)
pip install aws-shell

# aws-vault (secure credentials)
brew install aws-vault
aws-vault add my-profile
aws-vault exec my-profile -- aws s3 ls

# cdk
npm install -g aws-cdk

# Terraform
brew install terraform

# eksctl
brew install eksctl
```

---

<a id="p1"></a>
## P1. EC2 & Compute

### Bước 1: Khởi tạo EC2

```bash
# Launch instance
aws ec2 run-instances \
  --image-id ami-0c7217cdde317cfec \
  --count 1 \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-xxx \
  --subnet-id subnet-xxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web1}]' \
  --user-data file://userdata.sh

# List
aws ec2 describe-instances
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' --output table

# Start/Stop/Reboot
aws ec2 start-instances --instance-ids i-xxx
aws ec2 stop-instances --instance-ids i-xxx
aws ec2 reboot-instances --instance-ids i-xxx

# Terminate
aws ec2 terminate-instances --instance-ids i-xxx
```

### Bước 2: User data

```bash
#!/bin/bash
# userdata.sh
apt-get update
apt-get install -y nginx
echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
systemctl enable nginx
systemctl start nginx
```

### Bước 3: Key pair

```bash
# Tạo key pair
aws ec2 create-key-pair --key-name my-key --query 'KeyMaterial' --output text > my-key.pem
chmod 400 my-key.pem

# SSH
ssh -i my-key.pem ec2-user@<public-ip>

# Import key
aws ec2 import-key-pair --key-name my-key --public-key-material fileb://~/.ssh/id_rsa.pub
```

### Bước 4: AMI

```bash
# Tạo AMI từ instance
aws ec2 create-image \
  --instance-id i-xxx \
  --name "myapp-$(date +%Y%m%d)" \
  --description "MyApp production AMI"

# List
aws ec2 describe-images --owners self

# Copy sang region khác
aws ec2 copy-image \
  --source-region us-east-1 \
  --source-image-id ami-xxx \
  --region us-west-2 \
  --name "myapp-west"

# Deregister
aws ec2 deregister-image --image-id ami-xxx
```

### Bước 5: Auto Scaling Group

```bash
# Tạo launch template
aws ec2 create-launch-template \
  --launch-template-name web-template \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId": "ami-xxx",
    "InstanceType": "t3.micro",
    "SecurityGroupIds": ["sg-xxx"],
    "UserData": "IyEvYmluL2Jhc2gKYXB0LWdldCB1cGRhdGUKYXB0LWdldCBpbnN0YWxsIC15IG5naW54Cg=="
  }'

# Tạo ASG
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --launch-template LaunchTemplateName=web-template,Version='$Latest' \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 3 \
  --vpc-zone-identifier "subnet-a,subnet-b,subnet-c" \
  --target-group-arns arn:aws:elasticloadbalancing:... \
  --health-check-type ELB \
  --health-check-grace-period 300

# Scaling policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name cpu-scale-up \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 70
  }'
```

### Bước 6: Elastic Load Balancer

```bash
# Application Load Balancer
aws elbv2 create-load-balancer \
  --name web-alb \
  --subnets subnet-a subnet-b subnet-c \
  --security-groups sg-alb \
  --scheme internet-facing \
  --type application

# Target group
aws elbv2 create-target-group \
  --name web-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-xxx \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --target-type instance

# Listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:...

# Register targets
aws elbv2 register-targets \
  --target-group-arn arn:... \
  --targets Id=i-xxx Id=i-yyy
```

---

<a id="p2"></a>
## P2. VPC & Network

### Bước 1: VPC

```bash
# Tạo VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-vpc}]'

# Lấy VPC ID
VPC_ID=vpc-xxx

# Enable DNS
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
```

### Bước 2: Subnets

```bash
# Public subnets
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1b
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone us-east-1c

# Private subnets
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.10.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 --availability-zone us-east-1b
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.12.0/24 --availability-zone us-east-1c
```

### Bước 3: Internet Gateway & Route Tables

```bash
# Tạo IGW
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# Public route table
RTB_PUBLIC=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB_PUBLIC --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# Associate public subnets
for SUBNET in subnet-public-a subnet-public-b subnet-public-c; do
    aws ec2 associate-route-table --route-table-id $RTB_PUBLIC --subnet-id $SUBNET
done

# Auto-assign public IP cho public subnets
for SUBNET in subnet-public-a subnet-public-b subnet-public-c; do
    aws ec2 modify-subnet-attribute --subnet-id $SUBNET --map-public-ip-on-launch
done
```

### Bước 4: NAT Gateway

```bash
# EIP cho NAT
EIP_ALLOCATION=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

# Tạo NAT Gateway (1 cho mỗi AZ - HA)
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-a \
  --allocation-id $EIP_ALLOCATION

# Private route table -> NAT
RTB_PRIVATE=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB_PRIVATE --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-xxx
```

### Bước 5: Security Groups

```bash
# Web SG
SG_WEB=$(aws ec2 create-security-group --group-name web-sg --description "Web servers" --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_WEB --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_WEB --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_WEB --protocol tcp --port 22 --cidr 10.0.0.0/16

# DB SG - chỉ từ web SG
SG_DB=$(aws ec2 create-security-group --group-name db-sg --description "Database" --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_DB --protocol tcp --port 3306 --source-group $SG_WEB
```

### Bước 6: VPC Endpoints (cost saving)

```bash
# S3 endpoint (gateway)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids $RTB_PRIVATE

# ECR endpoint (interface)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-private-a subnet-private-b
```

### Bước 7: VPC Peering & Transit Gateway

```bash
# Peering giữa 2 VPC
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-a \
  --peer-vpc-id vpc-b \
  --peer-region us-east-1

# Accept
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-xxx

# Transit Gateway (cho nhiều VPC)
aws ec2 create-transit-gateway --description "My TGW"
TGW_ID=tgw-xxx

# Attach VPCs
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id vpc-a \
  --subnet-ids subnet-a
```

---

<a id="p3"></a>
## P3. S3 & Storage

### Bước 1: Bucket

```bash
# Tạo bucket
aws s3 mb s3://my-unique-bucket-12345 --region us-east-1

# Hoặc với config chi tiết
aws s3api create-bucket \
  --bucket my-unique-bucket-12345 \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1

# List
aws s3 ls
aws s3 ls s3://my-bucket/

# Sync
aws s3 sync ./local-folder s3://my-bucket/path/

# Copy
aws s3 cp file.txt s3://my-bucket/
aws s3 cp s3://my-bucket/file.txt ./

# Delete
aws s3 rm s3://my-bucket/file.txt
aws s3 rm s3://my-bucket/ --recursive
```

### Bước 2: Versioning

```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# List versions
aws s3api list-object-versions --bucket my-bucket

# Restore old version
aws s3api copy-object \
  --bucket my-bucket \
  --key file.txt \
  --copy-source my-bucket/file.txt?versionId=xxx
```

### Bước 3: Lifecycle

```bash
cat > lifecycle.json <<EOF
{
  "Rules": [
    {
      "Id": "transition-to-ia",
      "Status": "Enabled",
      "Prefix": "logs/",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      }
    },
    {
      "Id": "cleanup-incomplete-uploads",
      "Status": "Enabled",
      "Prefix": "",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json
```

### Bước 4: Encryption

```bash
# Default encryption với SSE-S3
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# SSE-KMS
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/xxx"
      }
    }]
  }'
```

### Bước 5: Static website hosting

```bash
aws s3api put-bucket-website \
  --bucket my-bucket \
  --website-configuration '{
    "IndexDocument": {"Suffix": "index.html"},
    "ErrorDocument": {"Key": "error.html"}
  }'

# Policy cho public read
cat > policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicRead",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
EOF

aws s3api put-bucket-policy --bucket my-bucket --policy file://policy.json
```

### Bước 6: S3 + CloudFront

```bash
# Origin Access Identity
aws cloudfront create-cloud-front-origin-access-identity \
  --cloud-front-origin-access-identity-config \
  Comment="My OAI"

# Distribution
aws cloudfront create-distribution \
  --origin-domain-name my-bucket.s3.amazonaws.com \
  --default-root-object index.html
```

### Bước 7: S3 Glacier

```bash
# Upload to Glacier
aws s3 cp file.zip s3://my-bucket/archive/file.zip \
  --storage-class GLACIER \
  --recursive

# Initiate restore
aws s3api restore-object \
  --bucket my-bucket \
  --key archive/file.zip \
  --restore-request Days=7

# Tier options
# Expedited (1-5 min, expensive)
# Standard (3-5 hours)
# Bulk (5-12 hours, cheap)
```

### Bước 8: EBS Volumes

```bash
# Tạo volume
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 100 \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125

# Attach
aws ec2 attach-volume \
  --volume-id vol-xxx \
  --instance-id i-xxx \
  --device /dev/sdf

# Snapshot
aws ec2 create-snapshot --volume-id vol-xxx --description "backup"

# Copy snapshot sang region khác
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-xxx \
  --destination-region us-west-2 \
  --description "DR copy"
```

---

<a id="p4"></a>
## P4. Database

### Bước 1: RDS

```bash
# Tạo DB subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name myapp-db \
  --db-subnet-group-description "MyApp DB" \
  --subnet-ids subnet-private-a subnet-private-b

# Tạo RDS instance
aws rds create-db-instance \
  --db-instance-identifier myapp-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.4 \
  --master-username admin \
  --master-user-password "$DB_PASSWORD" \
  --allocated-storage 20 \
  --storage-type gp3 \
  --storage-encrypted \
  --vpc-security-group-ids $SG_DB \
  --db-subnet-group-name myapp-db \
  --backup-retention-period 7 \
  --multi-az \
  --no-publicly-accessible \
  --enable-cloudwatch-logs-exports postgresql \
  --deletion-protection

# Multi-AZ read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier myapp-db-replica \
  --source-db-instance-identifier myapp-db \
  --db-instance-class db.t3.small

# Snapshot
aws rds create-db-snapshot \
  --db-instance-identifier myapp-db \
  --db-snapshot-identifier myapp-db-snapshot

# Restore
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier myapp-db-restored \
  --db-snapshot-identifier myapp-db-snapshot
```

### Bước 2: Aurora

```bash
# Tạo Aurora cluster
aws rds create-db-cluster \
  --db-cluster-identifier myapp-aurora \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --master-username admin \
  --master-user-password "$DB_PASSWORD" \
  --vpc-security-group-ids $SG_DB \
  --db-subnet-group-name myapp-db \
  --storage-encrypted \
  --enable-cloudwatch-logs-exports postgresql

# Writer instance
aws rds create-db-instance \
  --db-instance-identifier myapp-aurora-writer \
  --db-cluster-identifier myapp-aurora \
  --db-instance-class db.r6g.large \
  --engine aurora-postgresql

# Reader instance
aws rds create-db-instance \
  --db-instance-identifier myapp-aurora-reader \
  --db-cluster-identifier myapp-aurora \
  --db-instance-class db.r6g.large \
  --engine aurora-postgresql
```

### Bước 3: DynamoDB

```bash
# Tạo table
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=userId,AttributeType=S \
  --key-schema AttributeName=userId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Với GSI
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
      AttributeName=orderId,AttributeType=S \
      AttributeName=customerId,AttributeType=S \
  --key-schema \
      AttributeName=orderId,KeyType=HASH \
  --global-secondary-indexes '[{
      "IndexName": "CustomerIndex",
      "KeySchema": [{"AttributeName": "customerId", "KeyType": "HASH"}],
      "Projection": {"ProjectionType": "ALL"}
  }]' \
  --billing-mode PAY_PER_REQUEST

# Item operations
aws dynamodb put-item \
  --table-name Users \
  --item '{"userId": {"S": "123"}, "name": {"S": "John"}}'

aws dynamodb get-item \
  --table-name Users \
  --key '{"userId": {"S": "123"}}'

aws dynamodb query \
  --table-name Users \
  --key-condition-expression "userId = :id" \
  --expression-attribute-values '{":id": {"S": "123"}}'

aws dynamodb scan --table-name Users --max-items 10

# Update
aws dynamodb update-item \
  --table-name Users \
  --key '{"userId": {"S": "123"}}' \
  --update-expression "SET #n = :name" \
  --expression-attribute-names '{"#n": "name"}' \
  --expression-attribute-values '{":name": {"S": "Jane"}}'

# Backup
aws dynamodb create-backup --table-name Users --backup-name UsersBackup

# Point-in-time recovery
aws dynamodb update-continuous-backups \
  --table-name Users \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
```

### Bước 4: ElastiCache (Redis)

```bash
# Subnet group
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name myapp-cache \
  --cache-subnet-group-description "Cache" \
  --subnet-ids subnet-private-a subnet-private-b

# Redis cluster
aws elasticache create-replication-group \
  --replication-group-id myapp-redis \
  --replication-group-description "Redis cache" \
  --engine redis \
  --cache-node-type cache.t3.micro \
  --num-cache-clusters 2 \
  --cache-subnet-group-name myapp-cache \
  --security-group-ids $SG_CACHE \
  --transit-encryption-enabled \
  --at-rest-encryption-enabled
```

### Bước 5: Backup & PITR

```bash
# RDS automated backup (mặc định)
# Backup retention: 0-35 days

# Manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier myapp-db \
  --db-snapshot-identifier manual-snap-$(date +%Y%m%d)

# Export sang S3
aws rds start-export-task \
  --export-task-identifier export-1 \
  --source-arn arn:aws:rds:us-east-1:123456789012:snapshot:myapp-db-snapshot \
  --s3-bucket-name my-exports \
  --iam-role-arn arn:aws:iam::123456789012:role/RDSExportRole

# Cross-region copy
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier myapp-db-snapshot \
  --target-db-snapshot-identifier myapp-db-snapshot-west \
  --source-region us-east-1 \
  --region us-west-2
```

---

<a id="p5"></a>
## P5. IAM & Security

### Bước 1: User & Group

```bash
# Group
aws iam create-group --group-name Developers

# User
aws iam create-user --user-name john

# Add to group
aws iam add-user-to-group --user-name john --group-name Developers

# Login profile (console access)
aws iam create-login-profile \
  --user-name john \
  --password "$PASSWORD" \
  --password-reset-required

# Access key (CLI access)
aws iam create-access-key --user-name john
# Lưu SecretAccessKey ngay - chỉ hiển thị 1 lần
```

### Bước 2: Policy

```json
// policy-developer.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:Region": "us-east-1"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::mycompany-dev-*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "rds:DeleteDBInstance",
        "s3:DeleteBucket"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
# Attach policy
aws iam put-user-policy \
  --user-name john \
  --policy-name DeveloperPolicy \
  --policy-document file://policy-developer.json
```

### Bước 3: Role

```bash
# Trust policy (cho EC2 assume)
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Tạo role
aws iam create-role \
  --role-name EC2S3AccessRole \
  --assume-role-policy-document file://trust-policy.json

# Attach policy cho role
aws iam attach-role-policy \
  --role-name EC2S3AccessRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Instance profile (cho EC2)
aws iam create-instance-profile --instance-profile-name EC2S3Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2S3Profile \
  --role-name EC2S3AccessRole

# Gán cho EC2
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxx \
  --iam-instance-profile Name=EC2S3Profile
```

### Bước 4: Cross-account role

```bash
# Account A: tạo role cho Account B assume
cat > trust-cross-account.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::ACCOUNT_B_ID:root"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "unique-external-id"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name CrossAccountReadOnly \
  --assume-role-policy-document file://trust-cross-account.json
```

### Bước 5: MFA

```bash
# Bắt buộc MFA cho sensitive operations
cat > policy-mfa-required.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAllIfMFA",
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    },
    {
      "Sid": "DenyAllIfNoMFA",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
EOF
```

### Bước 6: Secrets Manager & Parameter Store

```bash
# Parameter Store (đơn giản, free)
aws ssm put-parameter \
  --name "/myapp/db/password" \
  --value "secret123" \
  --type SecureString \
  --key-id alias/aws/ssm

aws ssm get-parameter \
  --name "/myapp/db/password" \
  --with-decryption

# Lấy nhiều
aws ssm get-parameters-by-path \
  --path "/myapp/" \
  --recursive \
  --with-decryption

# Secrets Manager (rotation tự động)
aws secretsmanager create-secret \
  --name myapp/db/password \
  --secret-string "secret123"

# Auto rotation (Lambda)
aws secretsmanager rotate-secret \
  --secret-id myapp/db/password \
  --rotation-lambda-arn arn:aws:lambda:... \
  --rotation-rules '{"AutomaticallyAfterDays": 30}'
```

### Bước 7: KMS Encryption

```bash
# Tạo KMS key
KMS_KEY=$(aws kms create-key \
  --description "MyApp encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --query 'KeyMetadata.KeyId' --output text)

# Tạo alias
aws kms create-alias \
  --alias-name alias/myapp-key \
  --target-key-id $KMS_KEY

# Encrypt
aws kms encrypt \
  --key-id alias/myapp-key \
  --plaintext fileb://secret.txt \
  --output text \
  --query CiphertextBlob > secret.encrypted

# Decrypt
aws kms decrypt \
  --ciphertext-blob fileb://secret.encrypted \
  --output text \
  --query Plaintext
```

---

<a id="p6"></a>
## P6. Lambda & Serverless

### Bước 1: Lambda function

```bash
# Tạo package
zip function.zip index.js

# Tạo function
aws lambda create-function \
  --function-name myapp-function \
  --runtime nodejs20.x \
  --role arn:aws:iam::123456789012:role/LambdaRole \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --memory-size 512 \
  --timeout 30 \
  --environment Variables={LOG_LEVEL=info,ENV=production}

# Update
aws lambda update-function-code \
  --function-name myapp-function \
  --zip-file fileb://function.zip

# Invoke
aws lambda invoke \
  --function-name myapp-function \
  --payload '{"key": "value"}' \
  --cli-binary-format raw-in-base64-out \
  output.json
```

### Bước 2: Node.js Lambda

```javascript
// index.js
const { DynamoDBClient } = require("@aws-sdk/client-dynamodb");
const { DynamoDBDocumentClient, GetCommand } = require("@aws-sdk/lib-dynamodb");

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
    console.log('Event:', JSON.stringify(event));

    try {
        // Path parameter từ API Gateway
        const userId = event.pathParameters?.userId;

        if (!userId) {
            return {
                statusCode: 400,
                body: JSON.stringify({ error: 'userId required' })
            };
        }

        // DynamoDB query
        const command = new GetCommand({
            TableName: process.env.TABLE_NAME,
            Key: { userId }
        });

        const result = await docClient.send(command);

        return {
            statusCode: 200,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            body: JSON.stringify(result.Item || {})
        };
    } catch (error) {
        console.error('Error:', error);
        return {
            statusCode: 500,
            body: JSON.stringify({ error: 'Internal error' })
        };
    }
};
```

### Bước 3: Python Lambda

```python
# lambda_function.py
import json
import os
import boto3
from botocore.exceptions import ClientError

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

def lambda_handler(event, context):
    print(f"Event: {event}")

    try:
        user_id = event.get('pathParameters', {}).get('userId')
        if not user_id:
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'userId required'})
            }

        response = table.get_item(Key={'userId': user_id})

        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps(response.get('Item', {}))
        }
    except ClientError as e:
        print(f"Error: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

### Bước 4: API Gateway

```bash
# HTTP API (đơn giản hơn REST API)
aws apigatewayv2 create-api \
  --name myapp-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:us-east-1:123456789012:function:myapp-function

# Integration
aws apigatewayv2 create-integration \
  --api-id api-xxx \
  --integration-type AWS_PROXY \
  --integration-uri arn:aws:lambda:us-east-1:123456789012:function:myapp-function \
  --payload-format-version 2.0

# Route
aws apigatewayv2 create-route \
  --api-id api-xxx \
  --route-key "GET /users/{userId}" \
  --target integrations/xxx

# Lambda permission
aws lambda add-permission \
  --function-name myapp-function \
  --statement-id apigateway \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:123456789012:api-xxx/*"

# Custom domain
aws apigatewayv2 create-domain-name \
  --domain-name api.example.com \
  --domain-name-configurations CertificateArn=arn:acm:...
```

### Bước 5: EventBridge

```bash
# Rule
aws events put-rule \
  --name myapp-schedule \
  --schedule-expression "cron(0 2 * * ? *)"

# Target
aws events put-targets \
  --rule myapp-schedule \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:myapp-function"

# Lambda permission
aws lambda add-permission \
  --function-name myapp-function \
  --statement-name eventbridge \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:us-east-1:123456789012:rule/myapp-schedule
```

### Bước 6: Step Functions

```json
// state-machine.json
{
  "Comment": "Order processing workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:validate",
      "Retry": [{
        "ErrorEquals": ["States.TaskFailed"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3
      }],
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:payment",
      "Catch": [{
        "ErrorEquals": ["PaymentError"],
        "ResultPath": "$.error",
        "Next": "RefundOrder"
      }],
      "Next": "ShipOrder"
    },
    "ShipOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:ship",
      "End": true
    },
    "RefundOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:refund",
      "End": true
    }
  }
}
```

```bash
aws stepfunctions create-state-machine \
  --name myapp-workflow \
  --definition file://state-machine.json \
  --role-arn arn:aws:iam::...:role/StepFunctionsRole
```

---

<a id="p7"></a>
## P7. Container Services

### Bước 1: ECR

```bash
# Tạo repository
aws ecr create-repository --repository-name myapp

# Login
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Tag & push
docker tag myapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
```

### Bước 2: ECS với Fargate

```bash
# Cluster
aws ecs create-cluster --cluster-name myapp-cluster

# Task definition
cat > task-def.json <<EOF
{
  "family": "myapp-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::...:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "myapp",
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest",
    "portMappings": [{
      "containerPort": 8080,
      "protocol": "tcp"
    }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/myapp",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]
}
EOF

aws ecs register-task-definition --cli-input-json file://task-def.json

# Service
aws ecs create-service \
  --cluster myapp-cluster \
  --service-name myapp-service \
  --task-definition myapp-task:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-a,subnet-b],securityGroups=[sg-app],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:...,containerName=myapp,containerPort=8080"
```

### BƯớc 3: EKS

```bash
# Tạo cluster với eksctl
eksctl create cluster \
  --name myapp-cluster \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 10 \
  --managed

# Hoặc Terraform
# Xem tài liệu Terraform
```

### Bước 4: Fargate profiles cho EKS

```bash
aws eks create-fargate-profile \
  --cluster-name myapp-cluster \
  --fargate-profile-name myapp-fargate \
  --namespace myapp \
  --selectors namespace=myapp \
  --subnets subnet-private-a subnet-private-b
```

---

<a id="p8"></a>
## P8. Monitoring

### Bước 1: CloudWatch Logs

```bash
# Log group
aws logs create-log-group --log-group-name /myapp/app

# Tạo metric filter
aws logs put-metric-filter \
  --log-group-name /myapp/app \
  --filter-name error-count \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=ErrorCount,metricNamespace=MyApp,metricValue=1

# Tail logs
aws logs tail /myapp/app --follow

# Query (CloudWatch Insights)
aws logs start-query \
  --log-group-name /myapp/app \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/'

aws logs get-query-results --query-id xxx
```

### Bước 2: CloudWatch Alarms

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu \
  --alarm-description "Alert when CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:alerts \
  --dimensions Name=InstanceId,Value=i-xxx
```

### Bước 3: SNS Topic

```bash
# Topic
aws sns create-topic --name alerts

# Subscribe email
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:alerts \
  --protocol email \
  --notification-endpoint [email protected]

# Subscribe Lambda
aws sns subscribe \
  --topic-arn arn:aws:sns:... \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:...:function:my-handler
```

### BƯớc 4: CloudTrail

```bash
# Trail
aws cloudtrail create-trail \
  --name my-trail \
  --s3-bucket-name my-cloudtrail-logs \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation

# Start logging
aws cloudtrail start-logging --name my-trail

# Lookup events
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=admin \
  --max-items 10
```

### Bước 5: X-Ray

```javascript
// Node.js với X-Ray
const AWSXRay = require('aws-xray-sdk');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));

// Trong Lambda
exports.handler = AWSXRay.captureAsyncFunc('myHandler', async (event) => {
    const segment = AWSXRay.getSegment();
    const subsegment = segment.addNewSubsegment('DynamoDB Call');

    try {
        const result = await dynamoCall();
        subsegment.close();
        return result;
    } catch (err) {
        subsegment.addError(err);
        subsegment.close();
        throw err;
    }
});
```

---

<a id="p9"></a>
## P9. Cost & Well-Architected

### Bước 1: Cost Explorer & Budgets

```bash
# Budget
aws budgets create-budget \
  --account-id 123456789012 \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```

```json
// budget.json
{
  "BudgetName": "monthly-budget",
  "BudgetLimit": {
    "Amount": "1000",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
```

### Bước 2: Reserved Instances & Savings Plans

```bash
# Mua EC2 RI
aws ec2 purchase-reserved-instances-offering \
  --reserved-instances-offering-id xxx \
  --instance-count 5

# Savings Plans
aws savingsplans create-savings-plan \
  --savings-plan-offering-id xxx \
  --commitment 100 \
  --upfront-payment-amount 0 \
  --purchase-time $(date -u +%Y-%m-%dT%H:%M:%SZ)
```

### Bước 3: Spot Instances

```bash
# Spot fleet request
aws ec2 request-spot-fleet \
  --spot-fleet-request-config file://spot-config.json
```

```json
// spot-config.json
{
  "AllocationStrategy": "lowestPrice",
  "TargetCapacity": 10,
  "IamFleetRole": "arn:aws:iam::...:role/aws-ec2-spot-fleet-tagging-role",
  "LaunchSpecifications": [{
    "ImageId": "ami-xxx",
    "InstanceType": "t3.medium",
    "SubnetId": "subnet-xxx"
  }]
}
```

### Bước 4: Cost optimization tips

```bash
# 1. Stop dev instances outside hours
aws ec2 stop-instances --instance-ids i-dev1 i-dev2

# 2. Delete unused EBS volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].VolumeId' --output text | \
  xargs -I {} aws ec2 delete-volume --volume-id {}

# 3. Release unused EIPs
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==`null`].AllocationId' \
  --output text | xargs -I {} aws ec2 release-address --allocation-id {}

# 4. S3 Intelligent-Tiering
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket my-bucket \
  --id my-config \
  --intelligent-tiering-configuration file://tiering.json

# 5. Use Compute Savings Plans
# 6. Right-size instances
# 7. Use Graviton (ARM) - 20% cheaper
```

### Bước 5: AWS Well-Architected Framework

```
5 pillars:
1. Operational Excellence
   - IaC (CloudFormation, CDK, Terraform)
   - Monitoring & logging
   - Runbooks

2. Security
   - IAM least privilege
   - Encryption everywhere
   - MFA, no long-lived keys
   - Audit với CloudTrail

3. Reliability
   - Multi-AZ
   - Auto Scaling
   - Backup (3-2-1 rule)
   - DR plan (RTO/RPO)

4. Performance Efficiency
   - Right-sized resources
   - Caching (CloudFront, ElastiCache)
   - Use managed services
   - Monitor & iterate

5. Cost Optimization
   - Pay only for what you use
   - Reserved capacity cho predictable workload
   - Cost allocation tags
   - Regular review
```

### Bước 6: Tagging strategy

```bash
# Standard tags
aws ec2 create-tags \
  --resources i-xxx vol-xxx \
  --tags \
    Key=Environment,Value=production \
    Key=Project,Value=myapp \
    Key=Team,Value=platform \
    Key=CostCenter,Value=engineering \
    Key=ManagedBy,Value=terraform \
    Key=Owner,Value=jane

# Cost allocation tag
aws ce create-cost-category-definition \
  --name Environment \
  --rules '[{
    "Value": "production",
    "Rule": {
      "Tags": {
        "Key": "Environment",
        "Values": ["production", "prod"]
      }
    }
  }]'
```

---

## 🎯 Bài tập P0-P9

1. Tạo IAM user với MFA, dùng SSO để login
2. Deploy 1 web app EC2 + ALB + ASG trong VPC 3-tier
3. RDS Multi-AZ + read replica + backup script
4. S3 với lifecycle, encryption, public static website
5. Lambda + API Gateway + DynamoDB (full serverless API)
6. ECS Fargate cluster với task definition + service
7. CloudWatch alarms + SNS + dashboards

---

> **💡 Tip cuối**: AWS có 200+ services, đừng học hết. Focus 20% services dùng 80%: EC2, S3, VPC, RDS, IAM, Lambda, CloudWatch. Dùng AWS Free Tier cho học. Tối ưu cost từ đầu với tagging + budgets.

---

*Tạo bởi tài liệu học AWS - Chúc bạn thành công! 🚀*
