# 🏗️ Tài liệu Terraform: Từ Zero đến Chuyên gia

> **Mục tiêu**: Infrastructure as Code (IaC) chuyên nghiệp với Terraform trên AWS/GCP/Azure/K8s.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Chuẩn bị](#p0) | Cài Terraform, providers |
| [P1. HCL cơ bản](#p1) | Resource, variable, output |
| [P2. State](#p2) | Backend, lock, remote |
| [P3. Module](#p3) | Tái sử dụng, chuẩn |
| [P4. Nâng cao](#p4) | Loop, condition, function, lifecycle |
| [P5. Multi-Provider](#p5) | AWS, GCP, Azure, K8s |
| [P6. Workspace & Env](#p6) | Multi-env, workspaces |
| [P7. Terraform Cloud](#p7) | TFC, Sentinel, cost |
| [P8. Testing](#p8) | Validate, plan, tftest |
| [P9. Production](#p9) | CI/CD, security, drift |

---

<a id="p0"></a>
## P0. Chuẩn bị môi trường

### Bước 1: Cài đặt

```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Linux (Ubuntu/Debian)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Windows (scoop)
scoop install terraform

# Verify
terraform version
```

### Bước 2: Auto-completion & aliases

```bash
# Auto-completion
terraform -install-autocomplete

# Aliases
cat >> ~/.bashrc <<'EOF'
alias tf=terraform
alias tfa='terraform apply'
alias tfp='terraform plan'
alias tfi='terraform init'
alias tfd='terraform destroy'
alias tfo='terraform output'
alias tfs='terraform state'
complete -C $(which terraform) terraform
EOF

source ~/.bashrc
```

### Bước 3: Cấu hình chung

```bash
# Cấu hình CLI
cat > ~/.terraformrc <<EOF
provider_installation {
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
  network_mirror {
    url = "https://terraform-mirror.example.com/"
  }
}
EOF

# Hoặc dùng env var
export TF_VAR_region=us-east-1
export TF_INPUT=0           # Không hỏi input
export TF_LOG=DEBUG         # Logging
export TF_LOG_PATH=./tf.log
```

---

<a id="p1"></a>
## P1. HCL cơ bản

### Bước 1: Cấu trúc project

```bash
mkdir ~/terraform-lab && cd ~/terraform-lab
```
```bash
terraform-lab/
├── main.tf              # Resource chính
├── variables.tf         # Input variables
├── outputs.tf           # Output values
├── providers.tf         # Provider config
├── versions.tf          # Version constraints
├── terraform.tfstate    # State (KHÔNG commit)
├── .terraform/          # Cache (KHÔNG commit)
└── .terraform.lock.hcl  # Lock file (commit)
```
```bash
# .gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
!example.tfvars
crash.log
crash.*.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json
.terraformrc
terraform.rc
```

### Bước 2: Provider

```hcl
# providers.tf
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.27"
    }
  }
}

provider "aws" {
  region = "us-east-1"
  default_tags {
    tags = {
      Environment = terraform.workspace
      ManagedBy   = "Terraform"
      Project     = "MyApp"
    }
  }
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}
```

### Bước 3: Resource đầu tiên

```hcl
# main.tf
resource "aws_s3_bucket" "example" {
  bucket = "my-unique-bucket-name-12345"
}

resource "aws_s3_bucket_versioning" "example" {
  bucket = aws_s3_bucket.example.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "example" {
  bucket = aws_s3_bucket.example.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

```bash
terraform init
terraform plan
terraform apply
terraform show
terraform destroy
```

### Bước 4: Variables

```hcl
# variables.tf
variable "region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}

variable "environment" {
  type        = string
  description = "Environment name"
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "cidr_blocks" {
  type        = list(string)
  description = "Allowed CIDR blocks"
  default     = ["0.0.0.0/0"]
}

variable "tags" {
  type = map(string)
  default = {
    Project = "MyApp"
  }
}

variable "instance_config" {
  type = object({
    ami           = string
    instance_type = string
    disk_size     = number
  })
}

# Sensitive
variable "db_password" {
  type      = string
  sensitive = true
}
```

```bash
# Truyền biến
terraform apply -var="region=us-west-2"
terraform apply -var-file="prod.tfvars"

# Auto-load: terraform.tfvars hoặc *.auto.tfvars
cat > prod.tfvars <<EOF
region        = "us-east-1"
environment   = "production"
instance_type = "t3.large"
EOF

# Biến môi trường
export TF_VAR_db_password="secret"
```

### Bước 5: Output

```hcl
# outputs.tf
output "bucket_name" {
  value       = aws_s3_bucket.example.id
  description = "Name of the S3 bucket"
}

output "connection_info" {
  value = {
    host     = aws_db_instance.db.address
    port     = aws_db_instance.db.port
    username = aws_db_instance.db.username
    password = aws_db_instance.db.password
  }
  sensitive = true
}

output "all_instance_ids" {
  value = aws_instance.web[*].id
}
```

```bash
terraform output
terraform output bucket_name
terraform output -json
```

### Bước 6: Data sources

```hcl
# Lấy data có sẵn
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

# Dùng trong resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  availability_zone = data.aws_availability_zones.available.names[0]
}

# K8s
data "kubernetes_namespace" "example" {
  metadata {
    name = "production"
  }
}
```

### Bước 7: Locals

```hcl
# locals.tf
locals {
  common_tags = {
    Environment = var.environment
    Project     = "MyApp"
    ManagedBy   = "Terraform"
    Owner       = "Platform Team"
  }

  name_prefix = "${var.project}-${var.environment}"

  # Functions
  is_production = var.environment == "production"
}

# Dùng
resource "aws_instance" "web" {
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
  })
}
```

---

<a id="p2"></a>
## P2. State management

### Bước 1: Local state (mặc định)

```bash
# State lưu trong terraform.tfstate
terraform state list
terraform state show aws_instance.web
terraform state mv aws_instance.old aws_instance.new    # Rename
terraform state rm aws_instance.unused                   # Remove khỏi state
terraform state pull > state.json                        # Export
terraform state push state.json                          # Import (cẩn thận)
```

### Bước 2: Remote backend - S3

```hcl
# providers.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```
```bash
# Setup S3 + DynamoDB
aws s3 mb s3://mycompany-terraform-state --region us-east-1
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

terraform init
```

### Bước 3: Backend khác

```hcl
# Terraform Cloud
terraform {
  cloud {
    organization = "mycompany"
    workspaces {
      name = "my-app-prod"
    }
  }
}

# Azure Storage
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate"
    storage_account_name = "tfstatexxxx"
    container_name       = "tfstate"
    key                  = "prod.tfstate"
  }
}

# GCS
terraform {
  backend "gcs" {
    bucket  = "tfstate-bucket"
    prefix  = "terraform/state"
  }
}

# Consul
terraform {
  backend "consul" {
    address = "consul.example.com:8500"
    path    = "terraform/state"
  }
}

# HTTP (advanced)
terraform {
  backend "http" {
    address        = "https://tfstate.example.com/state"
    lock_address   = "https://tfstate.example.com/lock"
    unlock_address = "https://tfstate.example.com/unlock"
    username       = "terraform"
    password       = var.tf_password
  }
}
```

### Bước 4: Import existing infrastructure

```bash
# Terraform 1.5+: dùng import block
```
```hcl
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}

# Hoặc CLI
terraform import aws_instance.web i-1234567890abcdef0
```

```hcl
# Cho multiple resources
import {
  for_each = toset(["i-1234", "i-5678"])
  to       = aws_instance.web
  id       = each.value
}
```

### Bước 5: State locking

```bash
# S3 + DynamoDB: tự động lock
# Force unlock (cẩn thận)
terraform force-unlock <LOCK_ID>
```

---

<a id="p3"></a>
## P3. Modules

### Bước 1: Tạo module

```bash
mkdir -p modules/web-server
```
```
modules/web-server/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

```hcl
# modules/web-server/variables.tf
variable "name" {
  type        = string
  description = "Server name"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "ami_id" {
  type = string
}

variable "subnet_id" {
  type = string
}

variable "vpc_security_group_ids" {
  type = list(string)
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/web-server/main.tf
resource "aws_instance" "this" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.vpc_security_group_ids

  tags = merge(var.tags, {
    Name = var.name
  })
}
```

```hcl
# modules/web-server/outputs.tf
output "id" {
  value = aws_instance.this.id
}

output "public_ip" {
  value = aws_instance.this.public_ip
}

output "private_ip" {
  value = aws_instance.this.private_ip
}
```

### Bước 2: Dùng module

```hcl
# main.tf
module "web1" {
  source = "./modules/web-server"

  name                    = "web-1"
  ami_id                  = data.aws_ami.ubuntu.id
  subnet_id               = aws_subnet.public_a.id
  vpc_security_group_ids  = [aws_security_group.web.id]
  instance_type           = "t3.small"

  tags = {
    Environment = "production"
    Role        = "web"
  }
}

module "web2" {
  source = "./modules/web-server"

  name                    = "web-2"
  ami_id                  = data.aws_ami.ubuntu.id
  subnet_id               = aws_subnet.public_b.id
  vpc_security_group_ids  = [aws_security_group.web.id]
  instance_type           = "t3.small"
}

# Dùng for_each cho nhiều instances
module "web_servers" {
  source   = "./modules/web-server"
  for_each = toset(["web-1", "web-2", "web-3"])

  name                    = each.key
  ami_id                  = data.aws_ami.ubuntu.id
  subnet_id               = aws_subnet.public_a.id
  vpc_security_group_ids  = [aws_security_group.web.id]
}

output "web_server_ips" {
  value = { for k, m in module.web_servers : k => m.public_ip }
}
```

### Bước 3: Module từ Registry

```hcl
# VPC module từ Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = false

  tags = {
    Environment = "production"
  }
}

# EKS cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.8.0"

  name               = "my-cluster"
  kubernetes_version = "1.29"
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnets

  eks_managed_node_groups = {
    main = {
      instance_types = ["t3.medium"]
      min_size       = 2
      max_size       = 10
      desired_size   = 3
    }
  }
}
```

### Bước 4: Module từ Git

```hcl
module "vpc" {
  source = "git::https://github.com/terraform-aws-modules/terraform-aws-vpc.git?ref=v5.5.0"
  # Hoặc SSH
  # source = "git::ssh://[email protected]/org/repo.git?ref=v1.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

### Bước 5: Module versioning best practice

```hcl
# Luôn pin version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.5"   # Major.minor, allow patch

  # ...
}
```

---

<a id="p4"></a>
## P4. HCL nâng cao

### Bước 1: Loops

```hcl
# count
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index + 1}"
  }
}

# for_each với set
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.key
}

# for_each với map
resource "aws_route53_record" "dns" {
  for_each = {
    api    = "api.example.com"
    web    = "www.example.com"
    admin  = "admin.example.com"
  }

  zone_id = aws_route53_zone.main.zone_id
  name    = each.value
  type    = "A"
  ttl     = 300
  records = [aws_lb.main.dns_name]
}

# for_each với object phức tạp
resource "aws_instance" "web" {
  for_each = {
    web1 = { type = "t3.small", az = "a" }
    web2 = { type = "t3.medium", az = "b" }
    web3 = { type = "t3.large", az = "c" }
  }

  ami                    = data.aws_ami.ubuntu.id
  instance_type          = each.value.type
  availability_zone      = "us-east-1${each.value.az}"

  tags = {
    Name = each.key
  }
}
```

### Bước 2: Conditionals

```hcl
# ternary
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

  # Conditional attribute
  monitoring = var.environment == "production" ? true : false

  tags = {
    Name = "web-${var.environment}"
  }
}

# dynamic block (cho nested blocks)
resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }
}

variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = [
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP"
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTPS"
    }
  ]
}
```

### Bước 3: Functions

```hcl
# String
upper("hello")            # HELLO
lower("HELLO")            # hello
trim("  hello  ")         # hello
substr("hello", 0, 3)     # hel
replace("hello", "l", "L")  # heLLo
split(",", "a,b,c")       # ["a", "b", "c"]
join(",", ["a", "b"])     # a,b

# Number
max(1, 2, 3)              # 3
min(1, 2, 3)              # 1
ceil(1.4)                 # 2
floor(1.6)                # 1
abs(-5)                   # 5

# List
["a", "b", "c"][0]        # a
length(["a", "b"])        # 2
concat([1, 2], [3, 4])    # [1, 2, 3, 4]
distinct([1, 2, 2, 3])    # [1, 2, 3]
flatten([[1, 2], [3]])    # [1, 2, 3]

# Map
merge({a = 1}, {b = 2})   # {a = 1, b = 2}
keys({a = 1, b = 2})      # ["a", "b"]
values({a = 1, b = 2})    # [1, 2]

# Type conversion
tolist(["a", "b"])        # ["a", "b"]
tomap({a = 1})            # {a = 1}
toset(["a", "a", "b"])    # ["a", "b"]

# File
file("${path.module}/config.json")
fileexists("${path.module}/config.json")

# JSON/YAML
jsonencode({a = 1})
jsondecode("{\"a\": 1}")
yamlencode({a = 1})
yamldecode("a: 1")

# IP/CIDR
cidrsubnet("10.0.0.0/16", 8, 0)   # 10.0.0.0/24
cidrhost("10.0.0.0/24", 5)        # 10.0.0.5
cidrnetmask("10.0.0.0/24")        # 255.255.255.0

# Date
formatdate("YYYY-MM-DD", "2025-01-01T00:00:00Z")
timestamp()                         # RFC3339

# Crypto
md5("data")
sha256("data")
base64encode("data")
base64decode("ZGF0YQ==")

# Sensitive
nonsensitive(sensitive_value)
sensitive(plain_value)
```

### Bước 4: for expressions

```hcl
# Map comprehension
output "instance_summary" {
  value = {
    for k, v in aws_instance.web : k => {
      id         = v.id
      private_ip = v.private_ip
      type       = v.instance_type
    }
  }
}

# List comprehension
output "private_ips" {
  value = [for v in aws_instance.web : v.private_ip]
}

# Filter
output "large_instances" {
  value = {
    for k, v in aws_instance.web : k => v
    if v.instance_type == "t3.large"
  }
}

# Transform
locals {
  user_names = [for u in aws_iam_user.users : u.name]
}
```

### Bước 5: Lifecycle rules

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true   # Tạo mới trước khi xoá cũ
    prevent_destroy       = true   # Cấm xoá (production)
    ignore_changes        = [      # Bỏ qua drift
      ami,
      tags,
      user_data,
    ]
    replace_triggered_by  = [      # Trigger recreate
      aws_ami_update.app.id
    ]
  }
}
```

### Bước 6: depends_on (explicit dependency)

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id

  depends_on = [
    aws_iam_role_policy_attachment.s3_access
  ]
}
```

### Bước 7: Provisioners (cẩn thận)

```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt install -y nginx",
    ]
  }

  provisioner "local-exec" {
    command = "echo ${aws_instance.web.public_ip} > /tmp/instance_ip.txt"
  }

  provisioner "file" {
    source      = "config/app.conf"
    destination = "/etc/myapp/app.conf"
  }
}

# Best practice: dùng user_data hoặc Ansible thay vì provisioner
```

---

<a id="p5"></a>
## P5. Multi-Provider

### Bước 1: AWS

```hcl
provider "aws" {
  region = "us-east-1"
}

# Multiple regions
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Multiple accounts (assume role)
provider "aws" {
  alias = "prod"
  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/terraform"
  }
}

resource "aws_s3_bucket" "east" {
  bucket = "my-bucket-east"
}

resource "aws_s3_bucket" "west" {
  provider = aws.west
  bucket   = "my-bucket-west"
}
```

### Bước 2: K8s

```hcl
provider "kubernetes" {
  config_path = "~/.kube/config"
}

resource "kubernetes_deployment" "app" {
  metadata {
    name      = "my-app"
    namespace = "production"
    labels = {
      app = "my-app"
    }
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "my-app"
      }
    }

    template {
      metadata {
        labels = {
          app = "my-app"
        }
      }

      spec {
        container {
          image = "myapp:1.0"
          name  = "my-app"

          port {
            container_port = 8080
          }

          resources {
            limits = {
              cpu    = "500m"
              memory = "512Mi"
            }
            requests = {
              cpu    = "100m"
              memory = "128Mi"
            }
          }
        }
      }
    }
  }
}

resource "kubernetes_service" "app" {
  metadata {
    name      = "my-app"
    namespace = "production"
  }

  spec {
    selector = {
      app = "my-app"
    }

    port {
      port        = 80
      target_port = 8080
    }

    type = "ClusterIP"
  }
}
```

### Bước 3: Helm provider

```hcl
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

resource "helm_release" "nginx" {
  name       = "nginx-ingress"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"
  version    = "4.9.1"
  namespace  = "ingress-nginx"
  create_namespace = true

  values = [
    file("${path.module}/values/nginx-ingress.yaml")
  ]

  set {
    name  = "controller.replicaCount"
    value = "2"
  }

  set {
    name  = "controller.resources.limits.cpu"
    value = "500m"
  }
}
```

### Bước 4: GCP

```hcl
provider "google" {
  project = "my-project-id"
  region  = "us-central1"
  zone    = "us-central1-a"
}

provider "google-beta" {
  project = "my-project-id"
  region  = "us-central1"
}

resource "google_compute_instance" "web" {
  name         = "web-server"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    network = google_compute_network.main.self_link
    access_config {
      // Ephemeral public IP
    }
  }
}
```

### Bước 5: Multi-cloud

```hcl
# AWS S3 replica sang GCP
provider "aws" {
  region = "us-east-1"
}

provider "google" {
  project = "my-project"
  region  = "us-central1"
}

resource "aws_s3_bucket" "source" {
  bucket = "source-bucket"
}

resource "google_storage_bucket" "replica" {
  name     = "replica-bucket"
  location = "US"
}
```

---

<a id="p6"></a>
## P6. Workspace & Multi-Environment

### Bước 1: Workspaces

```bash
# List
terraform workspace list

# Tạo
terraform workspace new production
terraform workspace new staging
terraform workspace new dev

# Chuyển
terraform workspace select production

# Xoá
terraform workspace delete dev
```

```hcl
# Dùng workspace trong code
locals {
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"
  min_size      = terraform.workspace == "production" ? 3 : 1
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type

  tags = {
    Environment = terraform.workspace
  }
}
```

### Bước 2: Directory-based env (khuyến nghị)

```bash
terraform-multi-env/
├── modules/
│   └── web-app/
├── envs/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── production/
└── README.md
```
```hcl
# envs/production/main.tf
provider "aws" {
  region = "us-east-1"
}

module "web" {
  source = "../../modules/web-app"

  environment = "production"
  instance_type = "t3.large"
  replicas = 5
}
```
```hcl
# envs/production/backend.tf
terraform {
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "production/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### Bước 3: Terragrunt (DRY)

```bash
brew install terragrunt
```
```hcl
# terragrunt.hcl (cho mỗi env)
include "root" {
  path = "../../terragrunt.hcl"
}

include "env" {
  path = "./env.hcl"
}

inputs = {
  environment = include.env.locals.environment
  region      = include.env.locals.region
  instance_type = include.env.locals.instance_type
}
```
```hcl
# envs/production/env.hcl
locals {
  environment   = "production"
  region        = "us-east-1"
  instance_type = "t3.large"
}
```

```bash
cd envs/production
terragrunt init
terragrunt plan
terragrunt apply
terragrunt run-all plan   # Chạy cho tất cả modules
```

---

<a id="p7"></a>
## P7. Terraform Cloud / Enterprise

### Bước 1: Terraform Cloud setup

```hcl
# Trong providers.tf
terraform {
  cloud {
    organization = "mycompany"

    workspaces {
      name = "my-app-production"
    }
  }
}
```
```bash
terraform login
# Mở browser, tạo token

terraform init
```

### Bước 2: Remote operations

```
- VCS integration (GitHub/GitLab)
- Speculative plans cho PR
- Auto apply khi merge main
- State stored in TFC
- Variable management UI
- Sentinel policies
- Cost estimation
```

### Bước 3: Variables trong TFC

```bash
# Sensitive variables (TF_VAR_db_password)
# TFC sẽ encrypt và không hiển thị value
```

### Bước 4: Sentinel policies

```python
# policies/cost-limit.sentinel
import "tfplan/v2" as tfplan
import "strings"

# Phát hiện EC2 instance quá lớn
all_instances = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_instance"
}

violations = filter all_instances as _, rc {
    rc.change.after.instance_type is "t3.2xlarge" or
    rc.change.after.instance_type is "t3.4xlarge"
}

if length(violations) > 0 {
    print("Large instances not allowed:", violations)
    main = false
} else {
    main = true
}
```

### Bước 5: Atlantis (self-hosted GitOps)

```yaml
# atlantis.yaml
version: 3
autoplan:
  when_modified: ["*.tf", "*.tfvars"]
  enabled: true
projects:
  - name: production
    dir: envs/production
    workspace: production
    terraform_version: 1.6.0
  - name: staging
    dir: envs/staging
    workspace: staging
```

---

<a id="p8"></a>
## P8. Testing

### Bước 1: Validate

```bash
terraform validate
terraform fmt -recursive       # Format code
terraform fmt -check -diff     # Check format
```

### Bước 2: Plan workflow

```bash
# Plan chi tiết
terraform plan -out=tfplan
terraform apply tfplan

# Plan với target
terraform plan -target=aws_instance.web

# Plan chi tiết (dài)
terraform plan -detailed-exitcode
# 0 = success, no changes
# 1 = error
# 2 = success, changes present (dùng cho CI)
```

### Bước 3: tftest (Go)

```go
// main_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestTerraformWebApp(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../",
        Vars: map[string]interface{}{
            "environment": "test",
        },
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    bucketID := terraform.Output(t, terraformOptions, "bucket_name")
    assert.Contains(t, bucketID, "test-bucket")
}
```
```bash
cd test/
go test -v
```

### Bước 4: Terratest với AWS

```go
func TestEC2Instance(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../",
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    instanceID := terraform.Output(t, terraformOptions, "instance_id")

    // Verify với AWS API
    awsClient := aws.NewClient(t, "us-east-1")
    instance := awsClient.GetEc2Instance(t, instanceID)

    assert.Equal(t, "running", instance.State.Name)
    assert.True(t, instance.PublicIP != "")
}
```

### Bước 5: Checkov / tfsec (security scanning)

```bash
# Checkov
pip install checkov
checkov -d .

# tfsec
brew install tfsec
tfsec .

# Tích hợp vào CI
checkov -d . --output junit > checkov.xml
```

---

<a id="p9"></a>
## P9. Production Best Practices

### Bước 1: CI/CD cho Terraform

```yaml
# .github/workflows/terraform.yml
name: Terraform
on:
  pull_request:
    branches: [main]
    paths: ['**.tf', '**.tfvars']

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Terraform Init
        run: terraform init
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Format
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -no-color -out=tfplan
        env:
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}

      - name: Security Scan
        run: |
          pip install checkov
          checkov -d . --quiet --compact

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan

  apply:
    needs: terraform
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: tfplan
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
        env:
          AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE }}
```

### Bước 2: Security best practices

```hcl
# 1. Encryption mọi nơi
resource "aws_s3_bucket" "data" {
  bucket = "my-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.main.arn
    }
  }
}

# 2. Không hardcode secrets
variable "db_password" {
  type      = string
  sensitive = true
}

# 3. Tag mọi resource
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    CostCenter  = var.cost_center
  }
}

# 4. IAM least privilege
resource "aws_iam_role_policy" "app" {
  name = "app-policy"
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
        ]
        Resource = "${aws_s3_bucket.data.arn}/*"
      }
    ]
  })
}

# 5. Lifecycle cho cost
resource "aws_s3_bucket_lifecycle_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    id     = "transition-to-ia"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }
}
```

### Bước 3: Drift detection

```bash
# Detect drift
terraform plan -detailed-exitcode

# Schedule trong CI
0 8 * * * cd /terraform && terraform plan -out=plan.out
```

### Bước 4: Cost estimation

```bash
# Infracost
brew install infracost

# Setup API key
infracost register

# Generate cost estimate
infracost breakdown --path=. --format=json --out-file=infracost.json

# In PR comment
infracost comment github --path=infracost.json \
  --repo=myorg/myrepo --pull-request=123 \
  --github-token=$GITHUB_TOKEN

# Tích hợp vào GitHub Actions
- uses: infracost/actions/setup@v2
  with:
    api-key: ${{ secrets.INFRACOST_API_KEY }}

- run: infracost breakdown --path=. --format=json --out-file=/tmp/infracost.json
- run: infracost comment github --path=/tmp/infracost.json --repo=$GITHUB_REPOSITORY --pull-request=${{ github.event.number }} --github-token=${{ github.token }}
```

### Bước 5: Top 10 lỗi thường gặp

```
1. Không pin provider version -> update bất ngờ
2. Commit .tfstate -> lộ sensitive data
3. Hardcode secrets -> lộ password
4. Không dùng remote backend -> mất state
5. Quên lock file -> conflict khi nhiều người apply
6. Apply production trực tiếp từ local -> không audit
7. Không có state locking -> corruption
8. Quên run `terraform fmt` -> code khó đọc
9. Không validate sau khi sửa -> syntax error
10. Quên lifecycle cho resource không nên xoá
```

---

## 🎯 Bài tập P0-P9

1. Tạo VPC + 2 EC2 + Security Group + ALB bằng Terraform
2. Dùng module tái sử dụng cho 3 môi trường dev/staging/prod
3. Remote state với S3 + DynamoDB
4. Import existing AWS resource vào Terraform
5. Deploy EKS cluster bằng module
6. Tích hợp Checkov + tftest vào CI
7. Cost estimate cho 1 infrastructure

---

> **💡 Tip cuối**: Terraform giỏi cho static infrastructure, K8s/Helm giỏi cho application. Đừng cố Terraform hoá tất cả. Luôn dùng remote backend + state locking.

---

*Tạo bởi tài liệu học Terraform - Chúc bạn thành công! 🚀*
