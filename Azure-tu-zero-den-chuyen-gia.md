# ☁️ Tài liệu Azure: Từ Zero đến Chuyên gia

> **Mục tiêu**: Thành thạo Microsoft Azure: Compute, Networking, Storage, AKS, Serverless, DevOps.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Chuẩn bị](#p0) | Azure CLI, subscription, auth |
| [P1. Compute](#p1) | VM, VMSS, Availability Zones |
| [P2. Network](#p2) | VNet, NSG, Load Balancer, App Gateway |
| [P3. Storage](#p3) | Blob, Files, lifecycle |
| [P4. Database](#p4) | SQL DB, Cosmos DB, PostgreSQL |
| [P5. AKS](#p5) | Azure Kubernetes Service |
| [P6. IAM & Security](#p6) | Azure AD, RBAC, Key Vault |
| [P7. Serverless](#p7) | Functions, Container Apps, Logic Apps |
| [P8. DevOps](#p8) | Azure DevOps, GitHub integration |
| [P9. Monitoring](#p9) | Monitor, Log Analytics, App Insights |
| [P10. Best Practice](#p10) | Cost, security, well-architected |

---

<a id="p0"></a>
## P0. Chuẩn bị

### Bước 1: Tài khoản & Subscription

```
1. https://portal.azure.com/
2. Tạo tài khoản Microsoft
3. Tạo Subscription (có thể dùng Free Trial $200/30 ngày)
4. Tạo Resource Group
```

### BƯớc 2: Cài Azure CLI

```bash
# macOS
brew install azure-cli

# Linux
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Windows
winget install Microsoft.AzureCLI

# Verify
az --version

# Login
az login

# List subscriptions
az account list --output table
az account set --subscription "My Subscription"
az account show
```

### Bước 3: Cấu hình

```bash
# Default location
az configure --defaults location=eastus

# Default resource group
az configure --defaults group=my-rg

# Format output
az configure --defaults output=json

# Multiple clouds
az cloud list --output table
az cloud set --name AzureUSGovernment
```

### Bước 4: Resource Group

```bash
# Tạo
az group create --name my-rg --location eastus

# List
az group list --output table

# Xoá (cẩn thận)
az group delete --name my-rg --yes --no-wait

# Tags
az group update --name my-rg --set tags.Environment=dev Team=platform
```

### Bước 5: ARM Template / Bicep

```bash
# Bicep (DSL cho ARM)
az bicep install
az bicep build --file main.bicep

# Deploy
az deployment group create \
  --resource-group my-rg \
  --template-file main.bicep \
  --parameters parameters.json

# Validate
az deployment group validate \
  --resource-group my-rg \
  --template-file main.bicep \
  --parameters parameters.json
```

---

<a id="p1"></a>
## P1. Azure Compute

### Bước 1: VM

```bash
# Tạo VM
az vm create \
  --resource-group my-rg \
  --name web-vm \
  --image UbuntuLTS \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name my-vnet \
  --subnet web-subnet \
  --public-ip-address web-ip \
  --nsg web-nsg \
  --custom-data ./cloud-init.txt

# List
az vm list --output table

# Show details
az vm show --resource-group my-rg --name web-vm --show-details

# SSH
az vm ssh --resource-group my-rg --name web-vm

# Stop/Start
az vm stop --resource-group my-rg --name web-vm
az vm start --resource-group my-rg --name web-vm
az vm deallocate --resource-group my-rg --name web-vm  # Free compute charges

# Delete
az vm delete --resource-group my-rg --name web-vm --yes
```

### Bước 2: VMSS (Virtual Machine Scale Set)

```bash
# Tạo VMSS
az vmss create \
  --resource-group my-rg \
  --name web-vmss \
  --image UbuntuLTS \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --instance-count 3 \
  --vnet-name my-vnet \
  --subnet web-subnet \
  --load-balancer web-lb \
  --backend-pool-name web-backend-pool \
  --upgrade-policy-mode automatic \
  --health-probe-path /health

# Autoscaling
az monitor autoscale create \
  --resource-group my-rg \
  --resource web-vmss \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-web \
  --min-count 1 \
  --max-count 10 \
  --count 3

az monitor autoscale rule create \
  --resource-group my-rg \
  --autoscale-name autoscale-web \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 1

az monitor autoscale rule create \
  --resource-group my-rg \
  --autoscale-name autoscale-web \
  --condition "CpuPercentage < 30 avg 5m" \
  --scale in 1
```

### Bước 3: Managed Disks

```bash
# Tạo disk
az disk create \
  --resource-group my-rg \
  --name data-disk \
  --size-gb 100 \
  --sku Premium_LRS \
  --zone 1

# Attach
az vm disk attach \
  --resource-group my-rg \
  --vm-name web-vm \
  --name data-disk \
  --lun 0

# Snapshot
az snapshot create \
  --resource-group my-rg \
  --name web-vm-snapshot \
  --source web-vm-osdisk

# Image
az image create \
  --resource-group my-rg \
  --name web-image \
  --source web-vm
```

### Bước 4: Availability Zones

```bash
# VM in specific zone
az vm create \
  --resource-group my-rg \
  --name web-vm-z1 \
  --zone 1 \
  --image UbuntuLTS \
  --size Standard_B2s

# VMSS across zones
az vmss create \
  --resource-group my-rg \
  --name web-vmss \
  --zones 1 2 3 \
  --instance-count 3
```

---

<a id="p2"></a>
## P2. Network

### Bước 1: VNet

```bash
# Tạo VNet
az network vnet create \
  --resource-group my-rg \
  --name my-vnet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name web-subnet \
  --subnet-prefix 10.0.1.0/24

# Subnets
az network vnet subnet create \
  --resource-group my-rg \
  --vnet-name my-vnet \
  --name db-subnet \
  --address-prefix 10.0.10.0/24 \
  --private-endpoint-network-policies Disabled

# List
az network vnet list --output table
az network vnet subnet list --vnet-name my-vnet --resource-group my-rg --output table
```

### Bước 2: Network Security Group

```bash
# Tạo NSG
az network nsg create \
  --resource-group my-rg \
  --name web-nsg

# Rules
az network nsg rule create \
  --resource-group my-rg \
  --nsg-name web-nsg \
  --name allow-http \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 80

az network nsg rule create \
  --resource-group my-rg \
  --nsg-name web-nsg \
  --name allow-https \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443

# Allow SSH từ IP
az network nsg rule create \
  --resource-group my-rg \
  --nsg-name web-nsg \
  --name allow-ssh \
  --priority 120 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 1.2.3.0/24 \
  --destination-port-ranges 22

# Bind vào subnet
az network vnet subnet update \
  --resource-group my-rg \
  --vnet-name my-vnet \
  --name web-subnet \
  --network-security-group web-nsg
```

### Bước 3: Public IP & Load Balancer

```bash
# Static public IP
az network public-ip create \
  --resource-group my-rg \
  --name web-ip \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

# Load Balancer (Standard)
az network lb create \
  --resource-group my-rg \
  --name web-lb \
  --sku Standard \
  --public-ip-address web-ip \
  --frontend-ip-name web-frontend \
  --backend-pool-name web-backend

# Health probe
az network lb probe create \
  --resource-group my-rg \
  --lb-name web-lb \
  --name web-probe \
  --protocol Http \
  --port 80 \
  --path /health \
  --interval 15 \
  --threshold 4

# Rule
az network lb rule create \
  --resource-group my-rg \
  --lb-name web-lb \
  --name web-rule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name web-frontend \
  --backend-pool-name web-backend \
  --probe-name web-probe
```

### Bước 4: Application Gateway

```bash
# App Gateway (L7)
az network application-gateway create \
  --resource-group my-rg \
  --name web-appgw \
  --location eastus \
  --sku Standard_v2 \
  --capacity 2 \
  --vnet-name my-vnet \
  --subnet appgw-subnet \
  --public-ip-address appgw-ip \
  --frontend-port 80 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --servers web-vm-1 web-vm-2

# HTTPS listener
az network application-gateway ssl-cert create \
  --resource-group my-rg \
  --gateway-name web-appgw \
  --name appgw-cert \
  --cert-file ./cert.pfx \
  --cert-password "$CERT_PASSWORD"

az network application-gateway http-listener create \
  --resource-group my-rg \
  --gateway-name web-appgw \
  --name https-listener \
  --frontend-port appgw-https-port \
  --ssl-cert appgw-cert

# WAF (Web Application Firewall)
az network application-gateway waf-config set \
  --resource-group my-rg \
  --gateway-name web-appgw \
  --enabled true \
  --mode Prevention \
  --rule-set-version 3.2
```

### BƯớc 5: Private Endpoint

```bash
# Cho SQL DB
az network private-endpoint create \
  --resource-group my-rg \
  --name sql-pe \
  --vnet-name my-vnet \
  --subnet db-subnet \
  --private-connection-resource-id $(az sql db show --name mydb --server myserver --resource-group my-rg --query id -o tsv) \
  --group-ids sqlServer \
  --connection-name sql-connection
```

### Bước 6: Front Door (CDN)

```bash
az afd profile create \
  --resource-group my-rg \
  --profile-name web-afd \
  --sku Standard_AzureFrontDoor

az afd endpoint create \
  --resource-group my-rg \
  --profile-name web-afd \
  --endpoint-name web-endpoint \
  --origin-response-timeout-seconds 60

az afd origin create \
  --resource-group my-rg \
  --profile-name web-afd \
  --origin-group web-og \
  --origin-name web-origin \
  --host-name app.example.com \
  --http-port 80 \
  --https-port 443 \
  --origin-host-header app.example.com

az afd route create \
  --resource-group my-rg \
  --profile-name web-afd \
  --endpoint-name web-endpoint \
  --route-name web-route \
  --origin-group web-og \
  --https-redirect Enabled
```

---

<a id="p3"></a>
## P3. Storage

### Bước 1: Storage Account

```bash
# Tạo
az storage account create \
  --resource-group my-rg \
  --name mystorageaccount \
  --sku Standard_LRS \
  --location eastus \
  --kind StorageV2 \
  --access-tier Hot \
  --https-only true \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

# Get connection string
az storage account show-connection-string \
  --name mystorageaccount \
  --resource-group my-rg \
  --output tsv

# List keys
az storage account keys list \
  --account-name mystorageaccount \
  --resource-group my-rg
```

### BƯớc 2: Blob Container

```bash
# Container
az storage container create \
  --name mycontainer \
  --account-name mystorageaccount \
  --public-access off

# Upload
az storage blob upload \
  --account-name mystorageaccount \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./local-file.txt \
  --overwrite

# Download
az storage blob download \
  --account-name mystorageaccount \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./downloaded.txt

# List
az storage blob list \
  --account-name mystorageaccount \
  --container-name mycontainer \
  --output table
```

### Bước 3: Lifecycle Management

```json
// lifecycle.json
{
  "rules": [
    {
      "enabled": true,
      "name": "moveToCool",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        }
      }
    },
    {
      "enabled": true,
      "name": "deleteOld",
      "definition": {
        "actions": {
          "baseBlob": {
            "delete": {
              "daysAfterModificationGreaterThan": 365
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"]
        }
      }
    }
  ]
}
```

```bash
az storage account management-policy create \
  --account-name mystorageaccount \
  --resource-group my-rg \
  --policy @lifecycle.json
```

### Bước 4: Static Website

```bash
az storage blob service-properties update \
  --account-name mystorageaccount \
  --static-website \
  --index-document index.html \
  --404-document 404.html

# Get URL
az storage account show \
  --name mystorageaccount \
  --resource-group my-rg \
  --query "primaryEndpoints.web" \
  --output tsv
```

### Bước 5: Azure Files (SMB/NFS)

```bash
# File share
az storage share create \
  --name myshare \
  --account-name mystorageaccount \
  --quota 100

# Mount trên VM
# Linux
sudo mount -t cifs //mystorageaccount.file.core.windows.net/myshare /mnt/share \
  -o vers=3.0,username=mystorageaccount,password=$STORAGE_KEY,dir_mode=0777,file_mode=0777

# Hoặc thêm vào /etc/fstab
```

---

<a id="p4"></a>
## P4. Database

### Bước 1: Azure SQL

```bash
# Server
az sql server create \
  --resource-group my-rg \
  --name mysqlserver \
  --location eastus \
  --admin-user sqladmin \
  --admin-password "$SQL_PASSWORD"

# Database
az sql db create \
  --resource-group my-rg \
  --server mysqlserver \
  --name mydb \
  --service-objective S0 \
  --backup-storage-redundancy Local

# Firewall rule (allow Azure services)
az sql server firewall-rule create \
  --resource-group my-rg \
  --server mysqlserver \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Allow specific IP
az sql server firewall-rule create \
  --resource-group my-rg \
  --server mysqlserver \
  --name my-ip \
  --start-ip-address 1.2.3.4 \
  --end-ip-address 1.2.3.4

# Elastic pool
az sql elastic-pool create \
  --resource-group my-rg \
  --server mysqlserver \
  --name mypool \
  --edition Standard \
  --dtu 50 \
  --db-dtu-min 0 \
  --db-dtu-max 50

# Backup
az sql db export \
  --resource-group my-rg \
  --server mysqlserver \
  --name mydb \
  --admin-user sqladmin \
  --admin-password "$SQL_PASSWORD" \
  --storage-key-type SharedAccessSignature \
  --storage-key "$SAS_TOKEN" \
  --storage-uri "https://mystorageaccount.blob.core.windows.net/backups/mydb.bacpac"

# Auto-failover group
az sql failover-group create \
  --resource-group my-rg \
  --server mysqlserver \
  --partner-server mysqlserver-west \
  --name my-failover-group \
  --partner-resource-group my-rg-west
```

### Bước 2: PostgreSQL Flexible Server

```bash
# Server
az postgres flexible-server create \
  --resource-group my-rg \
  --name mypgserver \
  --location eastus \
  --admin-user pgadmin \
  --admin-password "$PG_PASSWORD" \
  --sku-name Standard_B1ms \
  --tier Burstable \
  --storage-size 32 \
  --version 15 \
  --high-availability Disabled

# Database
az postgres flexible-server db create \
  --resource-group my-rg \
  --server-name mypgserver \
  --database-name mydb

# Firewall
az postgres flexible-server firewall-rule create \
  --resource-group my-rg \
  --name mypgserver \
  --rule-name my-ip \
  --start-ip-address 1.2.3.4 \
  --end-ip-address 1.2.3.4

# Read replica
az postgres flexible-server replica create \
  --resource-group my-rg \
  --replica-name mypgserver-replica \
  --source-server mypgserver
```

### Bước 3: Cosmos DB

```bash
# Account (SQL API)
az cosmosdb create \
  --resource-group my-rg \
  --name mycosmos \
  --kind GlobalDocumentDB \
  --default-consistency-level Session \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=false \
  --locations regionName=westus failoverPriority=1 isZoneRedundant=false \
  --enable-multiple-write-locations false

# Database
az cosmosdb sql database create \
  --resource-group my-rg \
  --account-name mycosmos \
  --name mydb

# Container
az cosmosdb sql container create \
  --resource-group my-rg \
  --account-name mycosmos \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/userId" \
  --throughput 1000 \
  --idx @index.json
```

```json
// index.json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {"path": "/*"}
  ],
  "excludedPaths": [
    {"path": "/\"_etag\"/?"}
  ]
}
```

### Bước 4: Redis Cache

```bash
az redis create \
  --resource-group my-rg \
  --name myredis \
  --location eastus \
  --sku Premium \
  --vm-size P1 \
  --redis-version 6

# Get key
az redis list-keys --resource-group my-rg --name myredis
```

---

<a id="p5"></a>
## P5. AKS (Azure Kubernetes Service)

### Bước 1: Cluster

```bash
# Tạo AKS
az aks create \
  --resource-group my-rg \
  --name my-aks \
  --node-count 3 \
  --node-vm-size Standard_DS2_v2 \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --kubernetes-version 1.29 \
  --network-plugin azure \
  --network-policy calico \
  --service-cidr 172.16.0.0/16 \
  --dns-service-ip 172.16.0.10 \
  --docker-bridge-address 172.17.0.1/16 \
  --vnet-subnet-id $(az network vnet subnet show --resource-group my-rg --vnet-name my-vnet --name aks-subnet --query id -o tsv) \
  --load-balancer-sku standard \
  --api-server-vnet-integration \
  --enable-managed-identity \
  --enable-aad \
  --aad-admin-group-objects-ids <group-id> \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10

# Get credentials
az aks get-credentials --resource-group my-rg --name my-aks

# List
az aks list --output table
```

### Bước 2: Node pools

```bash
# Add user node pool
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-aks \
  --name userpool \
  --node-count 3 \
  --node-vm-size Standard_DS3_v2 \
  --mode User \
  --labels workload=production \
  --tags env=prod

# Spot node pool
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-aks \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --node-count 3 \
  --node-vm-size Standard_DS2_v2

# GPU node pool
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-aks \
  --name gpupool \
  --node-vm-size Standard_NC6s_v3 \
  --node-count 1 \
  --aks-custom-headers UseGPUDedicatedVHD=true
```

### Bước 3: ACR (Container Registry) integration

```bash
# Tạo ACR
az acr create \
  --resource-group my-rg \
  --name myregistry \
  --sku Standard \
  --admin-enabled true

# Attach to AKS
az aks update \
  --resource-group my-rg \
  --name my-aks \
  --attach-acr myregistry

# Build image
az acr build \
  --registry myregistry \
  --image myapp:v1 \
  --file Dockerfile .

# List images
az acr repository list --name myregistry --output table
```

### Bước 4: Application Routing (Nginx)

```bash
# Enable
az aks enable-addons \
  --resource-group my-rg \
  --name my-aks \
  --addons ingress_application_routing

# Get public IP
kubectl get svc -n app-routing-system nginx
```

### BƯớc 5: Azure AD integration với AKS

```bash
# Enable OIDC
az aks update \
  --resource-group my-rg \
  --name my-aks \
  --enable-oidc-issuer

# Workload Identity
az aks update \
  --resource-group my-rg \
  --name my-aks \
  --enable-workload-identity

# Get OIDC issuer
AKS_OIDC_ISSUER=$(az aks show --resource-group my-aks-rg --name my-aks --query "oidcIssuerProfile.issuerUrl" -o tsv)
```

### Bước 6: AKS Backup

```bash
# Enable backup
az dataprotection backup-instance create \
  --resource-group my-rg \
  --vault-name my-backup-vault \
  --backup-instance my-aks-backup \
  --policy-name my-aks-policy

# Trigger backup
az dataprotection backup-instance trigger \
  --resource-group my-rg \
  --vault-name my-backup-vault \
  --backup-instance-name my-aks-backup
```

---

<a id="p6"></a>
## P6. IAM & Security

### Bước 1: Azure AD / Entra ID

```bash
# User
az ad user create \
  --display-name "John Doe" \
  --user-principal-name [email protected] \
  --password "$PASSWORD"

# Group
az ad group create \
  --display-name "Developers" \
  --mail-nickname "devs"

# Add user to group
az ad group member add \
  --group "Developers" \
  --member-id $(az ad user show --id [email protected] --query objectId -o tsv)

# Service Principal (cho app)
az ad sp create --name myapp-sp
# Lưu lại: appId, password, tenant

# Managed Identity (cho AKS/VM)
az identity create \
  --resource-group my-rg \
  --name my-managed-identity
```

### Bước 2: RBAC

```bash
# Built-in roles
# Owner, Contributor, Reader
# Storage Blob Data Contributor
# Virtual Machine Contributor
# AKS Cluster Admin
# AcrPull, AcrPush

# Custom role
cat > custom-role.json <<EOF
{
  "Name": "Custom VM Operator",
  "Description": "Can start/stop VMs",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/read"
  ],
  "AssignableScopes": ["/subscriptions/SUB_ID"]
}
EOF

az role definition create --role-definition custom-role.json

# Assign
az role assignment create \
  --assignee [email protected] \
  --role "Custom VM Operator" \
  --resource-group my-rg
```

### Bước 3: Key Vault

```bash
# Tạo
az keyvault create \
  --resource-group my-rg \
  --name mykeyvault \
  --sku Standard \
  --enable-rbac-authorization

# Secret
az keyvault secret set \
  --vault-name mykeyvault \
  --name db-password \
  --value "$PASSWORD"

# Certificate
az keyvault certificate import \
  --vault-name mykeyvault \
  --name app-cert \
  --file ./cert.pfx \
  --password "$CERT_PASSWORD"

# Key
az keyvault key set \
  --vault-name mykeyvault \
  --name mykey \
  --protection software

# Access policy
az keyvault set-policy \
  --name mykeyvault \
  --object-id $(az ad sp show --id <app-id> --query objectId -o tsv) \
  --secret-permissions get list

# RBAC role
az role assignment create \
  --assignee <object-id> \
  --role "Key Vault Secrets User" \
  --scope $(az keyvault show --name mykeyvault --query id -o tsv)
```

### BƯớc 4: Managed Identity access

```bash
# Gán cho VM
az vm identity assign \
  --resource-group my-rg \
  --name web-vm \
  --identities my-managed-identity

# Cho AKS
az aks update \
  --resource-group my-rg \
  --name my-aks \
  --enable-managed-identity \
  --assign-identity <identity-resource-id>
```

### Bước 5: Azure Policy

```bash
# List policies
az policy definition list --query "[?policyType=='BuiltIn']" --output table

# Assign policy
az policy assignment create \
  --name allowed-locations \
  --policy "/subscriptions/SUB_ID/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c" \
  --params '{"listOfAllowedLocations":{"value":["eastus","westus"]}}' \
  --scope /subscriptions/SUB_ID

# Initiative (multiple policies)
az policy set-definition create \
  --name my-initiative \
  --definitions @policies.json

az policy assignment create \
  --name my-initiative-assignment \
  --policy-set-definition my-initiative \
  --scope /subscriptions/SUB_ID
```

### Bước 6: Defender for Cloud

```bash
# Enable
az security pricing create \
  --name VirtualMachines \
  --tier Standard

az security pricing create \
  --name SqlServers \
  --tier Standard

az security pricing create \
  --name StorageAccounts \
  --tier Standard

# Enable auto-provisioning
az security auto-provisioning-setting update \
  --name default \
  --auto-provision true
```

---

<a id="p7"></a>
## P7. Serverless

### Bước 1: Azure Functions

```javascript
// index.js
const { app } = require('@azure/functions');

app.http('hello', {
    methods: ['GET', 'POST'],
    authLevel: 'anonymous',
    handler: async (request, context) => {
        const name = request.query.get('name') || 'World';
        return { body: `Hello, ${name}!` };
    },
});
```

```bash
# Tạo Function App
az functionapp create \
  --resource-group my-rg \
  --consumption-plan-location eastus \
  --runtime node \
  --runtime-version 20 \
  --functions-version 4 \
  --name my-func-app \
  --storage-account mystorageaccount

# Deploy
func azure functionapp publish my-func-app

# Hoặc qua zip
az functionapp deployment source config-zip \
  --resource-group my-rg \
  --name my-func-app \
  --src ./function.zip
```

### Bước 2: Azure Functions - Python

```python
# function_app.py
import azure.functions as func
import json

app = func.FunctionApp()

@app.route(route="hello", auth_level=func.AuthLevel.ANONYMOUS)
def hello(req: func.HttpRequest) -> func.HttpResponse:
    name = req.params.get('name', 'World')
    return func.HttpResponse(
        json.dumps({"message": f"Hello, {name}!"}),
        mimetype="application/json"
    )

@app.queue_trigger(arg_name="msg", queue_name="myqueue",
                    connection="AzureWebJobsStorage")
def process_queue(msg: func.QueueMessage) -> None:
    logging.info(f"Processing: {msg.get_body().decode()}")
```

### Bước 3: Container Apps

```bash
# Tạo environment
az containerapp env create \
  --resource-group my-rg \
  --name my-env \
  --location eastus

# Deploy
az containerapp create \
  --resource-group my-rg \
  --name myapp \
  --environment my-env \
  --image myregistry.azurecr.io/myapp:v1 \
  --registry-server myregistry.azurecr.io \
  --registry-username myregistry \
  --registry-password "$ACR_PASSWORD" \
  --target-port 8080 \
  --ingress external \
  --cpu 0.5 \
  --memory 1Gi \
  --min-replicas 1 \
  --max-replicas 10 \
  --env-vars "DB_HOST=10.0.0.1" \
  --secrets "db-password=secretvalue" \
  --scale-rule-name http-rule \
  --scale-rule-type http \
  --scale-rule-http-concurrency 100

# Custom domain
az containerapp hostname add \
  --resource-group my-rg \
  --name myapp \
  --hostname app.example.com
```

```yaml
# Bicep cho Container App
resources:
  containerApp:
    type: Microsoft.App/containerApps@2023-05-01
    properties:
      managedEnvironmentId: containerEnv.id
      configuration:
        ingress:
          external: true
          targetPort: 8080
        registries:
        - server: acr.properties.loginServer
          identity: systemAssigned
      template:
        containers:
        - image: myapp:latest
          name: myapp
          resources:
            cpu: 0.5
            memory: 1Gi
        scale:
          minReplicas: 1
          maxReplicas: 10
```

### Bước 4: Logic Apps

```bash
# Consumption plan
az logic workflow create \
  --resource-group my-rg \
  --name my-workflow \
  --definition @workflow.json
```

```json
// workflow.json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "actions": {
      "HTTP_GET": {
        "type": "Http",
        "inputs": {
          "method": "GET",
          "uri": "https://api.example.com/data"
        },
        "runAfter": {}
      },
      "Parse_JSON": {
        "type": "ParseJson",
        "inputs": {
          "content": "@body('HTTP_GET')",
          "schema": {}
        },
        "runAfter": {"HTTP_GET": ["Succeeded"]}
      }
    },
    "triggers": {
      "Recurrence": {
        "type": "Recurrence",
        "recurrence": {
          "frequency": "Hour",
          "interval": 1
        }
      }
    }
  }
}
```

---

<a id="p8"></a>
## P8. Azure DevOps

### Bước 1: Service Connection & Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
    - main
  paths:
    exclude:
    - docs/*

pool:
  vmImage: ubuntu-latest

variables:
  - group: my-variables
  - name: IMAGE_NAME
    value: myapp
  - name: CONTAINER_REGISTRY
    value: myregistry.azurecr.io

stages:
- stage: Build
  displayName: 'Build & Test'
  jobs:
  - job: Build
    steps:
    - task: Docker@2
      displayName: 'Build & Push image'
      inputs:
        containerRegistry: 'acr-service-connection'
        repository: $(IMAGE_NAME)
        command: buildAndPush
        tags: |
          $(Build.BuildId)
          latest

    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: $(Build.ArtifactStagingDirectory)
        artifact: manifest

- stage: Deploy
  displayName: 'Deploy to AKS'
  dependsOn: Build
  jobs:
  - deployment: Deploy
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            inputs:
              action: deploy
              kubernetesServiceConnection: 'aks-service'
              manifests: |
                $(Pipeline.Workspace)/manifest/deployment.yaml
```

### Bước 2: Azure DevOps CLI

```bash
# Install extension
az extension add --name azure-devops

# Configure
az devops configure --defaults organization=myorg project=myproject

# Pipelines
az pipelines list
az pipelines run --name my-pipeline

# Repos
az repos list

# Artifacts
az artifacts universal publish \
  --organization https://dev.azure.com/myorg \
  --feed myfeed \
  --name mypackage \
  --version 1.0.0 \
  --path ./dist

# Service connections
az devops service-endpoint list
```

### Bước 3: GitHub Actions cho Azure

```yaml
# .github/workflows/azure.yml
name: Deploy to Azure
on:
  push:
    branches: [main]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Build & Push to ACR
      run: |
        az acr build \
          --registry myregistry \
          --image myapp:${{ github.sha }} \
          --file Dockerfile .

    - name: Get AKS credentials
      uses: azure/aks-set-context@v3
      with:
        resource-group: my-rg
        cluster-name: my-aks

    - name: Deploy to AKS
      run: |
        kubectl set image deployment/myapp \
          myapp=myregistry.azurecr.io/myapp:${{ github.sha }}
        kubectl rollout status deployment/myapp
```

---

<a id="p9"></a>
## P9. Monitoring

### Bước 1: Log Analytics Workspace

```bash
# Workspace
az monitor log-analytics workspace create \
  --resource-group my-rg \
  --workspace-name my-workspace

# Diagnostic settings cho VM
az monitor diagnostic-settings create \
  --resource /subscriptions/SUB_ID/resourceGroups/my-rg/providers/Microsoft.Compute/virtualMachines/web-vm \
  --name my-diagnostic \
  --workspace my-workspace \
  --logs '[{"category": "AllLogs", "enabled": true}]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'
```

### Bước 2: Application Insights

```bash
# Tạo
az monitor app-insights component create \
  --resource-group my-rg \
  --app my-appinsights \
  --location eastus \
  --application-type web \
  --workspace my-workspace

# Connection string
az monitor app-insights component show \
  --resource-group my-rg \
  --app my-appinsights \
  --query connectionString

# Cấu hình app
```

```python
# Python
from opencensus.ext.azure.log_exporter import AzureLogHandler
from applicationinsights import TelemetryClient

import logging
logger = logging.getLogger(__name__)
logger.addHandler(AzureLogHandler(connection_string='InstrumentationKey=...'))

tc = TelemetryClient('instrumentation-key')

@tc.timer("ProcessOrder")
def process_order(order):
    tc.track_event('Order processed')
    tc.track_metric('Order count', 1)
    return {'status': 'ok'}
```

### Bước 3: Alerts

```bash
# Metric alert
az monitor metrics alert create \
  --name high-cpu \
  --resource-group my-rg \
  --scopes /subscriptions/SUB_ID/resourceGroups/my-rg/providers/Microsoft.Compute/virtualMachines/web-vm \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action-group my-action-group

# Action group
az monitor action-group create \
  --resource-group my-rg \
  --name my-action-group \
  --action email admin email [email protected] \
  --action webhook slack https://hooks.slack.com/xxx

# Log query alert
az monitor scheduled-query create \
  --resource-group my-rg \
  --name error-spike \
  --scopes /subscriptions/SUB_ID/resourceGroups/my-rg/providers/Microsoft.OperationalInsights/workspaces/my-workspace \
  --condition-result "count" \
  --condition "Heartbeat | where TimeGenerated > ago(5m) | count" \
  --condition-threshold 0 \
  --evaluation-frequency 5m \
  --window-size 10m
```

### Bước 4: Dashboards

```bash
# Tạo dashboard
az portal dashboard create \
  --resource-group my-rg \
  --name my-dashboard \
  --input-path ./dashboard.json

# Hoặc qua Portal UI
# https://portal.azure.com/#blade/Microsoft_Azure_Portal/DashboardBlade
```

```json
// dashboard.json
{
  "properties": {
    "lenses": [
      {
        "order": 0,
        "parts": [
          {
            "position": {"x": 0, "y": 0, "rowSpan": 4, "colSpan": 6},
            "metadata": {
              "type": "Extension/Microsoft_OperationsManagementSuite_Workspace/PartType/LogsDashboardPart",
              "inputs": [
                {
                  "name": "resourceId",
                  "value": "/subscriptions/SUB/resourceGroups/RG/providers/Microsoft.OperationalInsights/workspaces/WORKSPACE"
                }
              ]
            }
          }
        ]
      }
    ]
  }
}
```

### Bước 5: KQL queries

```kusto
// Performance counters
Perf
| where TimeGenerated > ago(1h)
| where CounterName == "% Processor Time"
| summarize avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart

// Errors
AppExceptions
| where TimeGenerated > ago(24h)
| summarize count() by ProblemId
| order by count_ desc

// Heartbeat (uptime)
Heartbeat
| where TimeGenerated > ago(1h)
| summarize Last = max(TimeGenerated) by Computer
| extend UpTime = datetime_diff('minute', now(), Last)
```

---

<a id="p10"></a>
## P10. Best Practices

### Bước 1: Cost optimization

```bash
# Reserved Instances (1-3 năm)
az reservations reservation-order create \
  --reserved-resource-type VirtualMachines \
  --sku Standard_B2s \
  --term P1Y \
  --quantity 10 \
  --location eastus

# Spot VMs
az vmss create \
  --resource-group my-rg \
  --name spot-vmss \
  --priority Spot \
  --eviction-policy Deallocate \
  --max-price -1

# Auto-shutdown (cho dev)
az vm auto-shutdown \
  --resource-group my-rg \
  --name dev-vm \
  --time 1900

# Advisor recommendations
az advisor recommendation list
```

### BƯớc 2: Naming convention

```
Pattern: {resource-type}-{workload}-{environment}-{region}-{instance}

Examples:
- rg-myapp-prod-eastus-001      (Resource Group)
- vm-web-prod-eastus-001        (VM)
- sql-db-prod-eastus-001        (SQL DB)
- st-myappprod001               (Storage)
- kv-myapp-prod-001             (Key Vault)
```

### Bước 3: Tags strategy

```bash
# Apply tags
az resource update \
  --resource-group my-rg \
  --name web-vm \
  --resource-type Microsoft.Compute/virtualMachines \
  --set tags.Environment=production Team=platform CostCenter=engineering Project=myapp ManagedBy=terraform

# Tag policy
az policy assignment create \
  --name require-tags \
  --policy "/subscriptions/SUB/providers/Microsoft.Authorization/policyDefinitions/96670d81-39a4-40ec-991f-8a6d77098b23" \
  --params '{"tagName":{"value":"Environment"}}' \
  --scope /subscriptions/SUB
```

### Bước 4: Backup strategy

```bash
# Recovery Services Vault
az backup vault create \
  --resource-group my-rg \
  --name my-vault \
  --location eastus

# Enable backup cho VM
az backup protection enable-for-vm \
  --resource-group my-rg \
  --vault-name my-vault \
  --vm web-vm \
  --policy-name DefaultPolicy

# Backup SQL
az backup protection enable-for-azurefileshare \
  --resource-group my-rg \
  --vault-name my-vault \
  --policy-name DefaultPolicy \
  --storage-account mystorageaccount \
  --fileshare myshare
```

### Bước 5: Landing Zone (Cloud Adoption Framework)

```
Subscriptions:
- Identity subscription (Azure AD)
- Management subscription (Log Analytics, Backup)
- Connectivity subscription (Hub VNet, VPN)
- Production subscription
- Non-production subscription

Hub-spoke topology:
- Hub VNet (firewall, VPN, Bastion)
- Spoke VNets per workload
- Peering
```

### Bước 6: Well-Architected Framework

```
5 pillars:
1. Cost Optimization
   - Reserved Instances
   - Spot VMs
   - Auto-shutdown
   - Right-sizing

2. Operational Excellence
   - IaC (Bicep/Terraform)
   - Azure Monitor
   - Runbooks

3. Performance Efficiency
   - Premium SSD
   - Azure CDN
   - Azure Cache for Redis

4. Reliability
   - Availability Zones
   - Auto-scaling
   - Backup
   - DR (paired regions)

5. Security
   - Azure AD + MFA
   - RBAC least privilege
   - Key Vault
   - Defender for Cloud
   - Network segmentation (NSG, Private Endpoints)
```

---

## 🎯 Bài tập P0-P10

1. Tạo resource group + VM với VNet + NSG
2. Setup Storage Account với lifecycle + static website
3. Azure SQL với failover group, restore backup
4. AKS cluster + ACR, deploy app với Azure DevOps
5. Azure Functions trigger từ Storage Queue, ghi Cosmos DB
6. Container Apps với auto-scaling
7. Monitor + Application Insights + alerts

---

> **💡 Tip cuối**: Azure rất enterprise-friendly, tích hợp tốt với Microsoft stack. Dùng Bicep cho IaC (DSL tốt hơn ARM JSON). Azure AD/Entra ID là central identity. Hybrid cloud với Arc. Free Tier có nhiều resource để học.

---

*Tạo bởi tài liệu học Azure - Chúc bạn thành công! 🚀*
