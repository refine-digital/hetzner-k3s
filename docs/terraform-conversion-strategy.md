# Terraform/OpenTofu Conversion Strategy for hetzner-k3s

This document provides a comprehensive strategy for converting the hetzner-k3s CLI tool approach into a pure Terraform/OpenTofu infrastructure-as-code solution for deploying k3s clusters on Hetzner Cloud in a multi-tenant managed Kubernetes platform.

## Table of Contents

1. [Required Terraform Providers](#1-required-terraform-providers)
2. [Recommended Module Structure](#2-recommended-module-structure)
3. [Core Module: hetzner-k3s-cluster](#3-core-module-hetzner-k3s-cluster)
4. [Example: Single CAX11 Cluster](#4-example-single-cax11-cluster)
5. [Multi-Tenant Pattern](#5-multi-tenant-pattern)
6. [Resource Mapping](#6-resource-mapping)

---

## 1. Required Terraform Providers

### Provider Configuration

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    hcloud = {
      source  = "hetznercloud/hcloud"
      version = "~> 1.45.0"
    }

    random = {
      source  = "hashicorp/random"
      version = "~> 3.6.0"
    }

    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0.0"
    }

    null = {
      source  = "hashicorp/null"
      version = "~> 3.2.0"
    }
  }
}

provider "hcloud" {
  token = var.hcloud_token
}
```

### Authentication Setup

The Hetzner Cloud provider requires an API token. This can be provided in multiple ways:

**Option 1: Environment Variable (Recommended for CI/CD)**
```bash
export HCLOUD_TOKEN="your-api-token-here"
terraform apply
```

**Option 2: Variable File (Recommended for Multi-Tenant)**
```hcl
# terraform.tfvars (DO NOT commit to git)
hcloud_token = "your-api-token-here"
```

**Option 3: Terraform Cloud/Enterprise Workspace Variables**
- Set `HCLOUD_TOKEN` as a sensitive environment variable in your workspace

---

## 2. Recommended Module Structure

### Directory Layout

```
terraform-hetzner-k3s/
├── modules/
│   ├── hetzner-k3s-cluster/          # Main cluster module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   ├── network.tf                # Private network resources
│   │   ├── firewall.tf               # Firewall rules
│   │   ├── ssh.tf                    # SSH key management
│   │   ├── masters.tf                # Master node resources
│   │   ├── workers.tf                # Worker node resources
│   │   ├── loadbalancer.tf           # API load balancer
│   │   ├── cloud-init/               # Cloud-init templates
│   │   │   ├── master.tftpl
│   │   │   ├── worker.tftpl
│   │   │   └── common.tftpl
│   │   └── README.md
│   │
│   ├── k3s-nodepool/                 # Reusable node pool module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   │
│   └── hetzner-network/              # Network module (optional)
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── environments/                      # Multi-tenant environments
│   ├── client-a/
│   │   ├── prod/
│   │   │   ├── main.tf
│   │   │   ├── terraform.tfvars
│   │   │   └── backend.tf
│   │   └── staging/
│   │       ├── main.tf
│   │       ├── terraform.tfvars
│   │       └── backend.tf
│   │
│   ├── client-b/
│   │   └── prod/
│   │       ├── main.tf
│   │       ├── terraform.tfvars
│   │       └── backend.tf
│   │
│   └── shared/                       # Shared infrastructure
│       └── networking/
│           ├── main.tf
│           └── outputs.tf
│
├── examples/                         # Example configurations
│   ├── single-node/
│   ├── ha-cluster/
│   └── with-autoscaling/
│
└── README.md
```

### Module Separation Strategy

**Primary Module: `hetzner-k3s-cluster`**
- Self-contained, creates a complete k3s cluster
- Handles networking, compute, firewall, and k3s installation
- Suitable for most use cases

**Sub-modules:**
- `k3s-nodepool`: For dynamic worker pool management
- `hetzner-network`: For shared network scenarios across multiple clusters

---

## 3. Core Module: hetzner-k3s-cluster

### Input Variables

```hcl
# modules/hetzner-k3s-cluster/variables.tf

variable "hcloud_token" {
  description = "Hetzner Cloud API token"
  type        = string
  sensitive   = true
}

variable "cluster_name" {
  description = "Name of the k3s cluster (used for resource naming and labels)"
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.cluster_name))
    error_message = "Cluster name must contain only lowercase letters, numbers, and hyphens."
  }
}

variable "k3s_version" {
  description = "k3s version to install (e.g., v1.30.2+k3s2)"
  type        = string
  default     = "v1.30.2+k3s2"
}

# Networking Configuration
variable "networking" {
  description = "Networking configuration for the cluster"
  type = object({
    ssh = object({
      port              = number
      public_key_path   = string
      private_key_path  = string
    })

    allowed_networks = object({
      ssh = list(string)
      api = list(string)
    })

    public_network = object({
      ipv4 = bool
      ipv6 = bool
    })

    private_network = object({
      enabled              = bool
      subnet               = string
      existing_network_id  = optional(string)
    })

    cni = object({
      enabled    = bool
      mode       = string  # "flannel" or "cilium"
      encryption = bool
    })

    cluster_cidr = optional(string, "10.42.0.0/16")
    service_cidr = optional(string, "10.43.0.0/16")
  })

  default = {
    ssh = {
      port             = 22
      public_key_path  = "~/.ssh/id_rsa.pub"
      private_key_path = "~/.ssh/id_rsa"
    }
    allowed_networks = {
      ssh = ["0.0.0.0/0"]
      api = ["0.0.0.0/0"]
    }
    public_network = {
      ipv4 = true
      ipv6 = true
    }
    private_network = {
      enabled             = true
      subnet              = "10.0.0.0/16"
      existing_network_id = null
    }
    cni = {
      enabled    = true
      mode       = "flannel"
      encryption = false
    }
    cluster_cidr = "10.42.0.0/16"
    service_cidr = "10.43.0.0/16"
  }
}

# Master Pool Configuration
variable "masters_pool" {
  description = "Configuration for master nodes"
  type = object({
    instance_type  = string
    instance_count = number
    location       = string
    locations      = optional(list(string))
  })

  default = {
    instance_type  = "cpx11"
    instance_count = 3
    location       = "fsn1"
    locations      = ["fsn1", "nbg1", "hel1"]
  }
}

# Worker Pools Configuration
variable "worker_node_pools" {
  description = "List of worker node pool configurations"
  type = list(object({
    name           = string
    instance_type  = string
    instance_count = number
    location       = string

    labels = optional(map(string), {})
    taints = optional(list(object({
      key    = string
      value  = string
      effect = string
    })), [])

    autoscaling = optional(object({
      enabled       = bool
      min_instances = number
      max_instances = number
    }))
  }))

  default = [
    {
      name           = "pool1"
      instance_type  = "cpx11"
      instance_count = 1
      location       = "fsn1"
      labels         = {}
      taints         = []
    }
  ]
}

# K3s Configuration
variable "schedule_workloads_on_masters" {
  description = "Allow workloads to be scheduled on master nodes"
  type        = bool
  default     = false
}

variable "datastore" {
  description = "Datastore configuration (etcd or external)"
  type = object({
    mode = string  # "etcd" or "external"
    external_datastore_endpoint = optional(string)
  })

  default = {
    mode = "etcd"
  }
}

variable "image" {
  description = "OS image to use for all nodes"
  type        = string
  default     = "ubuntu-24.04"
}

variable "additional_packages" {
  description = "Additional packages to install on all nodes"
  type        = list(string)
  default     = []
}

variable "additional_post_k3s_commands" {
  description = "Additional commands to run after k3s installation"
  type        = list(string)
  default     = []
}

variable "create_kubeconfig" {
  description = "Whether to create and output kubeconfig"
  type        = bool
  default     = true
}

variable "create_load_balancer" {
  description = "Create a load balancer for HA API access"
  type        = bool
  default     = true
}

variable "firewall_use_current_ip" {
  description = "Automatically add current public IP to SSH allowed networks"
  type        = bool
  default     = false
}

variable "labels" {
  description = "Additional labels to add to all resources"
  type        = map(string)
  default     = {}
}
```

### Resource Definitions

```hcl
# modules/hetzner-k3s-cluster/main.tf

locals {
  # Generate k3s token for cluster authentication
  k3s_token = random_password.k3s_token.result

  # Determine network zone from location
  network_zone = lookup({
    "fsn1" = "eu-central",
    "nbg1" = "eu-central",
    "hel1" = "eu-central",
    "ash"  = "us-east"
  }, var.masters_pool.location, "eu-central")

  # Master node locations (spread across availability zones if specified)
  master_locations = var.masters_pool.locations != null ? var.masters_pool.locations : [
    var.masters_pool.location
  ]

  # Common labels for all resources
  common_labels = merge(
    {
      cluster   = var.cluster_name
      terraform = "true"
      managed-by = "terraform-hetzner-k3s"
    },
    var.labels
  )
}

# Generate random k3s token
resource "random_password" "k3s_token" {
  length  = 64
  special = false
}

# Data source for current public IP (if needed)
data "http" "current_ip" {
  count = var.firewall_use_current_ip ? 1 : 0
  url   = "https://ipv4.icanhazip.com"
}
```

### Output Values

```hcl
# modules/hetzner-k3s-cluster/outputs.tf

output "cluster_name" {
  description = "Name of the created cluster"
  value       = var.cluster_name
}

output "master_ips" {
  description = "Public IP addresses of master nodes"
  value = {
    public  = [for s in hcloud_server.masters : s.ipv4_address]
    private = var.networking.private_network.enabled ? [for s in hcloud_server.masters : s.network[0].ip] : []
  }
}

output "worker_ips" {
  description = "Public IP addresses of worker nodes by pool"
  value = {
    for pool in var.worker_node_pools : pool.name => {
      public  = [for s in hcloud_server.workers[pool.name] : s.ipv4_address]
      private = var.networking.private_network.enabled ? [for s in hcloud_server.workers[pool.name] : s.network[0].ip] : []
    }
  }
}

output "api_endpoint" {
  description = "Kubernetes API endpoint"
  value       = var.create_load_balancer ? "https://${hcloud_load_balancer.k3s_api[0].ipv4}:6443" : "https://${hcloud_server.masters[0].ipv4_address}:6443"
}

output "load_balancer_ip" {
  description = "Load balancer public IP (if created)"
  value       = var.create_load_balancer ? hcloud_load_balancer.k3s_api[0].ipv4 : null
}

output "network_id" {
  description = "ID of the private network (if created)"
  value       = var.networking.private_network.enabled ? (var.networking.private_network.existing_network_id != null ? var.networking.private_network.existing_network_id : hcloud_network.k3s[0].id) : null
}

output "firewall_id" {
  description = "ID of the cluster firewall"
  value       = hcloud_firewall.k3s.id
}

output "ssh_key_id" {
  description = "ID of the SSH key used for nodes"
  value       = hcloud_ssh_key.k3s.id
}

output "k3s_token" {
  description = "k3s cluster token"
  value       = random_password.k3s_token.result
  sensitive   = true
}

output "kubeconfig" {
  description = "Kubeconfig for accessing the cluster (base64 encoded)"
  value       = var.create_kubeconfig ? base64encode(local.kubeconfig_content) : null
  sensitive   = true
}
```

---

## 4. Example: Single CAX11 Cluster

This example demonstrates a complete, minimal k3s cluster on a single CAX11 (ARM64) instance.

### Complete Terraform Configuration

```hcl
# examples/single-node/main.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    hcloud = {
      source  = "hetznercloud/hcloud"
      version = "~> 1.45.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6.0"
    }
  }
}

provider "hcloud" {
  token = var.hcloud_token
}

# Generate k3s token
resource "random_password" "k3s_token" {
  length  = 64
  special = false
}

# SSH Key
resource "hcloud_ssh_key" "k3s" {
  name       = "${var.cluster_name}-ssh-key"
  public_key = file(var.ssh_public_key_path)

  labels = {
    cluster = var.cluster_name
  }
}

# Network
resource "hcloud_network" "k3s" {
  name     = var.cluster_name
  ip_range = "10.0.0.0/16"

  labels = {
    cluster = var.cluster_name
  }
}

resource "hcloud_network_subnet" "k3s" {
  network_id   = hcloud_network.k3s.id
  type         = "cloud"
  network_zone = "eu-central"
  ip_range     = "10.0.0.0/16"
}

# Firewall
resource "hcloud_firewall" "k3s" {
  name = "${var.cluster_name}-firewall"

  labels = {
    cluster = var.cluster_name
  }

  # SSH access
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "22"
    source_ips = var.allowed_ssh_cidrs
  }

  # Kubernetes API
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "6443"
    source_ips = var.allowed_api_cidrs
  }

  # ICMP (ping)
  rule {
    direction = "in"
    protocol  = "icmp"
    source_ips = [
      "0.0.0.0/0",
      "::/0"
    ]
  }

  # NodePort range
  rule {
    direction = "in"
    protocol  = "tcp"
    port      = "30000-32767"
    source_ips = [
      "0.0.0.0/0",
      "::/0"
    ]
  }

  rule {
    direction = "in"
    protocol  = "udp"
    port      = "30000-32767"
    source_ips = [
      "0.0.0.0/0",
      "::/0"
    ]
  }

  # Private network - all traffic
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "any"
    source_ips = [hcloud_network.k3s.ip_range]
  }

  rule {
    direction  = "in"
    protocol   = "udp"
    port       = "any"
    source_ips = [hcloud_network.k3s.ip_range]
  }
}

# Cloud-init for k3s installation
locals {
  cloud_init = templatefile("${path.module}/cloud-init.tftpl", {
    k3s_version     = var.k3s_version
    k3s_token       = random_password.k3s_token.result
    cluster_name    = var.cluster_name
    private_subnet  = hcloud_network.k3s.ip_range
    cluster_cidr    = "10.42.0.0/16"
    service_cidr    = "10.43.0.0/16"
  })
}

# Single CAX11 server (ARM64)
resource "hcloud_server" "k3s" {
  name        = "${var.cluster_name}-node"
  image       = "ubuntu-24.04"
  server_type = "cax11"
  location    = var.location

  ssh_keys = [hcloud_ssh_key.k3s.id]

  public_net {
    ipv4_enabled = true
    ipv6_enabled = true
  }

  network {
    network_id = hcloud_network.k3s.id
  }

  firewall_ids = [hcloud_firewall.k3s.id]

  user_data = local.cloud_init

  labels = {
    cluster = var.cluster_name
    role    = "master"
  }

  lifecycle {
    ignore_changes = [
      user_data,  # Prevent recreation on cloud-init changes
      ssh_keys    # SSH keys are immutable after creation
    ]
  }

  depends_on = [
    hcloud_network_subnet.k3s
  ]
}

# Variables
variable "hcloud_token" {
  description = "Hetzner Cloud API token"
  type        = string
  sensitive   = true
}

variable "cluster_name" {
  description = "Name of the k3s cluster"
  type        = string
  default     = "k3s-single"
}

variable "location" {
  description = "Hetzner datacenter location"
  type        = string
  default     = "fsn1"
}

variable "k3s_version" {
  description = "k3s version to install"
  type        = string
  default     = "v1.30.2+k3s2"
}

variable "ssh_public_key_path" {
  description = "Path to SSH public key"
  type        = string
  default     = "~/.ssh/id_rsa.pub"
}

variable "allowed_ssh_cidrs" {
  description = "CIDRs allowed to SSH to nodes"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}

variable "allowed_api_cidrs" {
  description = "CIDRs allowed to access Kubernetes API"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}

# Outputs
output "server_ip" {
  description = "Public IP of the k3s server"
  value       = hcloud_server.k3s.ipv4_address
}

output "api_endpoint" {
  description = "Kubernetes API endpoint"
  value       = "https://${hcloud_server.k3s.ipv4_address}:6443"
}

output "k3s_token" {
  description = "k3s cluster token"
  value       = random_password.k3s_token.result
  sensitive   = true
}

output "ssh_command" {
  description = "SSH command to connect to the server"
  value       = "ssh root@${hcloud_server.k3s.ipv4_address}"
}
```

### Cloud-Init Template

```yaml
# examples/single-node/cloud-init.tftpl
#cloud-config
preserve_hostname: true

packages:
  - fail2ban
  - wireguard

write_files:
  - path: /etc/systemd/system/ssh.socket.d/listen.conf
    content: |
      [Socket]
      ListenStream=
      ListenStream=22

runcmd:
  # Set hostname
  - hostnamectl set-hostname $(curl -s http://169.254.169.254/hetzner/v1/metadata/hostname)

  # Configure DNS
  - echo "nameserver 8.8.8.8" > /etc/k8s-resolv.conf

  # Wait for private network interface
  - |
    for i in {1..30}; do
      PRIVATE_IP=$(ip -4 addr show | grep -oP '(?<=inet\s)10\.0\.\d+\.\d+(?=/)')
      if [ -n "$PRIVATE_IP" ]; then
        echo "Private IP found: $PRIVATE_IP"
        break
      fi
      echo "Waiting for private network... ($i/30)"
      sleep 10
    done

  # Get public IP
  - PUBLIC_IP=$(hostname -I | awk '{print $1}')
  - HOSTNAME=$(hostname -f)

  # Install k3s
  - |
    curl -sfL https://get.k3s.io | \
      INSTALL_K3S_VERSION="${k3s_version}" \
      K3S_TOKEN="${k3s_token}" \
      INSTALL_K3S_EXEC="server" \
      sh -s - \
        --disable-cloud-controller \
        --disable traefik \
        --disable servicelb \
        --disable metrics-server \
        --write-kubeconfig-mode=644 \
        --node-name=$HOSTNAME \
        --cluster-cidr=${cluster_cidr} \
        --service-cidr=${service_cidr} \
        --kubelet-arg="cloud-provider=external" \
        --kubelet-arg="resolv-conf=/etc/k8s-resolv.conf" \
        --advertise-address=$PRIVATE_IP \
        --node-ip=$PRIVATE_IP \
        --node-external-ip=$PUBLIC_IP \
        --flannel-iface=ens10 \
        --cluster-init

  # Wait for k3s to be ready
  - until kubectl get nodes; do sleep 5; done

  # Install Hetzner Cloud Controller Manager
  - kubectl -n kube-system create secret generic hcloud --from-literal=token=${k3s_token} || true
```

### Usage

```bash
# Initialize
terraform init

# Plan
terraform plan -var="hcloud_token=$HCLOUD_TOKEN"

# Apply
terraform apply -var="hcloud_token=$HCLOUD_TOKEN"

# Get kubeconfig
ssh root@$(terraform output -raw server_ip) cat /etc/rancher/k3s/k3s.yaml > kubeconfig
sed -i "s/127.0.0.1/$(terraform output -raw server_ip)/g" kubeconfig
export KUBECONFIG=$(pwd)/kubeconfig

# Verify
kubectl get nodes
```

---

## 5. Multi-Tenant Pattern

### Directory Structure for Multi-Tenancy

```
environments/
├── client-a/
│   ├── prod/
│   │   ├── main.tf
│   │   ├── backend.tf
│   │   ├── terraform.tfvars
│   │   └── .terraform.lock.hcl
│   └── staging/
│       ├── main.tf
│       ├── backend.tf
│       └── terraform.tfvars
│
└── client-b/
    └── prod/
        ├── main.tf
        ├── backend.tf
        └── terraform.tfvars
```

### Example: Client-A Production

```hcl
# environments/client-a/prod/main.tf

terraform {
  required_version = ">= 1.5.0"
}

module "k3s_cluster" {
  source = "../../../modules/hetzner-k3s-cluster"

  hcloud_token = var.hcloud_token
  cluster_name = "client-a-prod"
  k3s_version  = "v1.30.2+k3s2"

  networking = {
    ssh = {
      port             = 22
      public_key_path  = var.ssh_public_key_path
      private_key_path = var.ssh_private_key_path
    }

    allowed_networks = {
      ssh = var.allowed_ssh_cidrs
      api = var.allowed_api_cidrs
    }

    public_network = {
      ipv4 = true
      ipv6 = true
    }

    private_network = {
      enabled             = true
      subnet              = "10.10.0.0/16"
      existing_network_id = null
    }

    cni = {
      enabled    = true
      mode       = "flannel"
      encryption = true
    }

    cluster_cidr = "10.42.0.0/16"
    service_cidr = "10.43.0.0/16"
  }

  masters_pool = {
    instance_type  = "cpx21"
    instance_count = 3
    location       = "fsn1"
    locations      = ["fsn1", "nbg1", "hel1"]
  }

  worker_node_pools = [
    {
      name           = "general"
      instance_type  = "cpx31"
      instance_count = 3
      location       = "fsn1"
      labels         = {
        pool = "general"
      }
      taints = []
    },
    {
      name           = "compute"
      instance_type  = "ccx33"
      instance_count = 2
      location       = "nbg1"
      labels         = {
        pool     = "compute"
        workload = "cpu-intensive"
      }
      taints = [
        {
          key    = "workload"
          value  = "compute"
          effect = "NoSchedule"
        }
      ]
    }
  ]

  create_load_balancer = true

  labels = {
    client      = "client-a"
    environment = "production"
    managed-by  = "platform-team"
  }
}

# Store kubeconfig in Terraform state
resource "local_sensitive_file" "kubeconfig" {
  content  = base64decode(module.k3s_cluster.kubeconfig)
  filename = "${path.module}/kubeconfig"
}

output "api_endpoint" {
  value = module.k3s_cluster.api_endpoint
}

output "load_balancer_ip" {
  value = module.k3s_cluster.load_balancer_ip
}
```

```hcl
# environments/client-a/prod/backend.tf

terraform {
  backend "s3" {
    bucket         = "terraform-state-client-a"
    key            = "k3s/prod/terraform.tfstate"
    region         = "eu-central-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

```hcl
# environments/client-a/prod/terraform.tfvars

hcloud_token = "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"  # Use env var or secret manager

ssh_public_key_path  = "~/.ssh/client-a-prod.pub"
ssh_private_key_path = "~/.ssh/client-a-prod"

allowed_ssh_cidrs = [
  "203.0.113.0/24",  # Office IP
  "198.51.100.0/24"  # VPN IP
]

allowed_api_cidrs = [
  "203.0.113.0/24",  # Office IP
  "198.51.100.0/24"  # VPN IP
]
```

### State Management Recommendations

1. **Use Remote State Backend**
   - AWS S3 + DynamoDB for state locking
   - Terraform Cloud/Enterprise
   - Azure Blob Storage
   - Google Cloud Storage

2. **Separate State Files per Environment**
   - One state file per cluster
   - Prevents blast radius of changes
   - Enables independent deployment cycles

3. **State Locking**
   - Always enable state locking
   - Prevents concurrent modifications
   - Critical for team environments

4. **Workspace Strategy**
   - Use separate directories (recommended) over Terraform workspaces
   - Clearer separation
   - Easier to manage in CI/CD

### CI/CD Integration Example

```yaml
# .github/workflows/terraform-apply.yml
name: Terraform Apply

on:
  push:
    branches:
      - main
    paths:
      - 'environments/client-a/prod/**'

env:
  TF_VERSION: 1.5.0

jobs:
  terraform:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        working-directory: environments/client-a/prod
        env:
          HCLOUD_TOKEN: ${{ secrets.CLIENT_A_HCLOUD_TOKEN }}
        run: terraform init

      - name: Terraform Plan
        working-directory: environments/client-a/prod
        env:
          HCLOUD_TOKEN: ${{ secrets.CLIENT_A_HCLOUD_TOKEN }}
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        working-directory: environments/client-a/prod
        env:
          HCLOUD_TOKEN: ${{ secrets.CLIENT_A_HCLOUD_TOKEN }}
        run: terraform apply tfplan
```

---

## 6. Resource Mapping

### hetzner-k3s → Terraform Resource Equivalents

| hetzner-k3s Component | Hetzner API Call | Terraform Resource | Notes |
|-----------------------|------------------|-------------------|-------|
| **SSH Key Creation** | `POST /ssh_keys` | `hcloud_ssh_key` | Created once per cluster |
| **Private Network** | `POST /networks` | `hcloud_network` + `hcloud_network_subnet` | Optional, for private networking |
| **Network Attachment** | `POST /servers/{id}/actions/attach_to_network` | Embedded in `hcloud_server.network` block | Automatic when network ID specified |
| **Firewall Creation** | `POST /firewalls` | `hcloud_firewall` | Rules applied via label selector |
| **Firewall Rules Update** | `POST /firewalls/{id}/actions/set_rules` | `hcloud_firewall.rule` blocks | Declarative in Terraform |
| **Master Instance** | `POST /servers` | `hcloud_server` | With `user_data` for cloud-init |
| **Worker Instance** | `POST /servers` | `hcloud_server` | With `user_data` for cloud-init |
| **Load Balancer** | `POST /load_balancers` | `hcloud_load_balancer` + `hcloud_load_balancer_target` | For HA API access |
| **Cloud-init Script** | User data in server creation | Templatefile or heredoc in `user_data` | Can use `templatefile()` function |
| **k3s Installation** | SSH execution of install script | Cloud-init `runcmd` | Runs on first boot |
| **Kubeconfig Retrieval** | SSH to master, cat `/etc/rancher/k3s/k3s.yaml` | `ssh` provisioner or `null_resource` with `local-exec` | Can be automated post-creation |

### Detailed Resource Mapping

#### SSH Key

```hcl
# hetzner-k3s: Hetzner::SSHKey::Create
# Terraform equivalent:
resource "hcloud_ssh_key" "k3s" {
  name       = "${var.cluster_name}-key"
  public_key = file(var.networking.ssh.public_key_path)

  labels = local.common_labels
}
```

#### Private Network

```hcl
# hetzner-k3s: Hetzner::Network::Create
# Terraform equivalent:
resource "hcloud_network" "k3s" {
  count = var.networking.private_network.enabled ? 1 : 0

  name     = var.cluster_name
  ip_range = var.networking.private_network.subnet

  labels = local.common_labels
}

resource "hcloud_network_subnet" "k3s" {
  count = var.networking.private_network.enabled ? 1 : 0

  network_id   = hcloud_network.k3s[0].id
  type         = "cloud"
  network_zone = local.network_zone
  ip_range     = var.networking.private_network.subnet
}
```

#### Firewall

```hcl
# hetzner-k3s: Hetzner::Firewall::Create
# Terraform equivalent:
resource "hcloud_firewall" "k3s" {
  name = "${var.cluster_name}-firewall"

  labels = local.common_labels

  # SSH access
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = tostring(var.networking.ssh.port)
    source_ips = var.networking.allowed_networks.ssh
  }

  # Kubernetes API
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "6443"
    source_ips = var.networking.allowed_networks.api
  }

  # ICMP (ping)
  rule {
    direction = "in"
    protocol  = "icmp"
    source_ips = [
      "0.0.0.0/0",
      "::/0"
    ]
  }

  # NodePort range TCP
  rule {
    direction = "in"
    protocol  = "tcp"
    port      = "30000-32767"
    source_ips = [
      "0.0.0.0/0",
      "::/0"
    ]
  }

  # NodePort range UDP
  rule {
    direction = "in"
    protocol  = "udp"
    port      = "30000-32767"
    source_ips = [
      "0.0.0.0/0",
      "::/0"
    ]
  }

  # Private network - all TCP
  dynamic "rule" {
    for_each = var.networking.private_network.enabled ? [1] : []
    content {
      direction  = "in"
      protocol   = "tcp"
      port       = "any"
      source_ips = [var.networking.private_network.subnet]
    }
  }

  # Private network - all UDP
  dynamic "rule" {
    for_each = var.networking.private_network.enabled ? [1] : []
    content {
      direction  = "in"
      protocol   = "udp"
      port       = "any"
      source_ips = [var.networking.private_network.subnet]
    }
  }
}
```

#### Master Server

```hcl
# hetzner-k3s: Hetzner::Instance::Create (master)
# Terraform equivalent:
resource "hcloud_server" "masters" {
  count = var.masters_pool.instance_count

  name        = "${var.cluster_name}-master-${count.index + 1}"
  image       = var.image
  server_type = var.masters_pool.instance_type
  location    = local.master_locations[count.index % length(local.master_locations)]

  ssh_keys = [hcloud_ssh_key.k3s.id]

  public_net {
    ipv4_enabled = var.networking.public_network.ipv4
    ipv6_enabled = var.networking.public_network.ipv6
  }

  dynamic "network" {
    for_each = var.networking.private_network.enabled ? [1] : []
    content {
      network_id = local.network_id
    }
  }

  firewall_ids = [hcloud_firewall.k3s.id]

  user_data = templatefile("${path.module}/cloud-init/master.tftpl", {
    # Template variables
    k3s_version              = var.k3s_version
    k3s_token                = local.k3s_token
    cluster_name             = var.cluster_name
    private_network_enabled  = var.networking.private_network.enabled
    private_network_subnet   = var.networking.private_network.subnet
    cluster_cidr             = var.networking.cluster_cidr
    service_cidr             = var.networking.service_cidr
    is_first_master          = count.index == 0
    first_master_ip          = count.index == 0 ? "" : hcloud_server.masters[0].network[0].ip
    # ... more variables
  })

  labels = merge(
    local.common_labels,
    {
      role = "master"
    }
  )

  lifecycle {
    ignore_changes = [
      user_data,
      ssh_keys
    ]
  }

  depends_on = [
    hcloud_network_subnet.k3s
  ]
}
```

#### Load Balancer

```hcl
# hetzner-k3s: Hetzner::LoadBalancer::Create
# Terraform equivalent:
resource "hcloud_load_balancer" "k3s_api" {
  count = var.create_load_balancer ? 1 : 0

  name               = "${var.cluster_name}-api"
  load_balancer_type = "lb11"
  location           = var.masters_pool.location

  labels = merge(
    local.common_labels,
    {
      purpose = "api-server"
    }
  )
}

resource "hcloud_load_balancer_network" "k3s_api" {
  count = var.create_load_balancer && var.networking.private_network.enabled ? 1 : 0

  load_balancer_id = hcloud_load_balancer.k3s_api[0].id
  network_id       = local.network_id
}

resource "hcloud_load_balancer_service" "k3s_api" {
  count = var.create_load_balancer ? 1 : 0

  load_balancer_id = hcloud_load_balancer.k3s_api[0].id
  protocol         = "tcp"
  listen_port      = 6443
  destination_port = 6443
}

resource "hcloud_load_balancer_target" "k3s_api" {
  count = var.create_load_balancer ? 1 : 0

  type             = "label_selector"
  load_balancer_id = hcloud_load_balancer.k3s_api[0].id
  label_selector   = "cluster=${var.cluster_name},role=master"
  use_private_ip   = var.networking.private_network.enabled

  depends_on = [
    hcloud_load_balancer_network.k3s_api
  ]
}
```

### Key Differences: CLI vs Terraform

| Aspect | hetzner-k3s CLI | Terraform/OpenTofu |
|--------|-----------------|-------------------|
| **State Management** | None (recreates if exists) | Full state tracking, drift detection |
| **Concurrency** | Manual with Crystal fibers | Dependency graph, automatic parallelization |
| **Idempotency** | Manual checks in code | Built-in via state comparison |
| **Updates** | Limited (upgrade command) | Full lifecycle management |
| **Drift Detection** | None | `terraform plan` shows drift |
| **Resource Dependencies** | Manual orchestration | Automatic via `depends_on` and references |
| **Rollback** | Manual | State-based rollback possible |
| **Secrets Management** | Environment variables | Integration with Vault, AWS Secrets Manager, etc. |
| **Multi-Cloud** | Hetzner only | Extensible to multiple providers |
| **GitOps Ready** | Requires wrapper | Native (Atlantis, Terraform Cloud) |

---

## Next Steps

1. **Start with Single-Node Example**: Test the basic pattern with one CAX11 node
2. **Build Core Module**: Develop the full `hetzner-k3s-cluster` module with all features
3. **Create Sub-Modules**: Extract reusable components (networking, node pools)
4. **Test Multi-Tenant**: Set up 2-3 test clients with separate state files
5. **Implement CI/CD**: Automate apply with GitHub Actions or GitLab CI
6. **Add Monitoring**: Integrate with Prometheus/Grafana for cluster health
7. **Document Runbooks**: Create operational procedures for common tasks

## Additional Resources

- [Hetzner Cloud Terraform Provider Docs](https://registry.terraform.io/providers/hetznercloud/hcloud/latest/docs)
- [k3s Installation Documentation](https://docs.k3s.io/installation)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [hetzner-k3s Source Code](https://github.com/vitobotta/hetzner-k3s) (for reference)
