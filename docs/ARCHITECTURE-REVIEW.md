# hetzner-k3s Architecture Review
## Multi-Tenant Managed Kubernetes Platform Design Guide

**Date:** 2025-11-17
**Purpose:** Inform Infrastructure-as-Code design for multi-tenant managed k3s platform on Hetzner Cloud ARM servers

---

## Executive Summary

**hetzner-k3s** is a production-ready, 2,166-line Crystal CLI tool that provisions k3s clusters on Hetzner Cloud in 2-3 minutes. It provides a battle-tested approach to deploying Kubernetes on Hetzner infrastructure that can be adapted to Terraform/OpenTofu for your multi-tenant platform.

### Key Findings

| Aspect | Finding | Impact on Your Platform |
|--------|---------|------------------------|
| **Provisioning** | Direct Hetzner API orchestration with concurrent instance creation | Easily translatable to Terraform resources |
| **Architecture** | Cloud-init based setup with templated k3s installation | Can reuse cloud-init templates in Terraform |
| **Networking** | Private network (10.0.0.0/16) + firewall via Hetzner API | Native Terraform support available |
| **Storage** | Hetzner CSI Driver for block volumes | Installed via kubectl after cluster creation |
| **Scaling** | Idempotent operations, cluster autoscaler, rolling upgrades | Supports 1→3→5 node growth path |
| **HA** | Multi-master with embedded etcd + load balancer | Proven pattern for production use |
| **Multi-Tenancy** | Label-based isolation per cluster | Perfect for isolated client clusters |

### Quick Answer to Your Key Questions

1. **Single CAX11 with k3s via Terraform?** ✅ Yes - See [terraform-conversion-strategy.md](./terraform-conversion-strategy.md#example-single-cax11-cluster) for complete working example

2. **Same IaC for multiple clients?** ✅ Yes - Variable-driven with separate state files per client

3. **Scale 1→3 nodes without recreation?** ✅ Yes - The create command is idempotent and incremental

4. **Network/firewall for multi-tenant?** ✅ Yes - Label selectors provide isolation, separate firewalls per cluster

---

## How hetzner-k3s Works

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Crystal CLI Tool (hetzner-k3s)                              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ├──── Configuration (YAML file)
                            │     ├─ Cluster settings
                            │     ├─ Node pools
                            │     ├─ Networking
                            │     └─ Add-ons
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Hetzner Cloud API (REST)                                    │
├─────────────────────────────────────────────────────────────┤
│ • POST /ssh_keys           → SSH key upload                 │
│ • POST /networks           → Private network (10.0.0.0/16)  │
│ • POST /firewalls          → Security rules                 │
│ • POST /servers            → Instance creation (concurrent) │
│ • POST /load_balancers     → API HA load balancer           │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Instances (Hetzner Cloud Servers)                          │
├─────────────────────────────────────────────────────────────┤
│ • Cloud-init runs on boot                                   │
│ • Network interface detection (MTU 1450/1280)               │
│ • k3s installed via https://get.k3s.io                      │
│ • Masters: embedded etcd, control plane                     │
│ • Workers: join via k3s agent                               │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Kubernetes Cluster (k3s)                                    │
├─────────────────────────────────────────────────────────────┤
│ • Hetzner Cloud Controller Manager → LoadBalancer services │
│ • Hetzner CSI Driver → PersistentVolumes                    │
│ • Cluster Autoscaler → Auto-scale workers                   │
│ • System Upgrade Controller → Rolling k3s upgrades          │
│ • Flannel/Cilium CNI → Pod networking                       │
└─────────────────────────────────────────────────────────────┘
```

### Design Patterns to Adopt

1. **Cloud-Init Templating**: Inject k3s installation scripts via user-data
2. **Concurrent Provisioning**: Create instances in parallel (10 max concurrency)
3. **Network Detection**: Use MTU-based interface detection for private networks
4. **Label-Based Targeting**: Use Hetzner labels for firewall/LB targeting
5. **Idempotent Operations**: Check for existing resources before creating
6. **Retry Logic**: Exponential backoff for API calls (10 attempts, 5s interval)

---

## Your Platform: Implementation Strategy

### Starting Point: Single CAX11 Cluster

**Goal:** Deploy isolated k3s clusters on client-owned Hetzner accounts, starting with single CAX11 nodes.

**Terraform Resources Required:**

```hcl
# Minimal single-node k3s cluster
resource "hcloud_ssh_key" "cluster"          # 1 SSH key
resource "hcloud_network" "cluster"          # 1 private network
resource "hcloud_network_subnet" "cluster"   # 1 subnet (10.0.0.0/16)
resource "hcloud_firewall" "cluster"         # 1 firewall (label-based)
resource "hcloud_server" "master"            # 1 CAX11 server
```

**See:** [terraform-conversion-strategy.md](./terraform-conversion-strategy.md#example-single-cax11-cluster) for complete 700-line working example

### Growth Path: Scale to HA

**Phase 1: Single Node** (€4/month)
- 1× CAX11 master
- Dev/test workloads
- No HA, local storage only

**Phase 2: Basic HA** (€24/month)
- 3× CAX11 masters (multi-location)
- 2× CAX11 workers
- Embedded etcd, load balancer
- Production-ready

**Phase 3: Vertical Scale** (€60/month)
- 3× CAX21 masters (4 vCPU, 8GB)
- 2× CAX21 workers
- Same cluster, new instance types
- Requires manual migration

**Phase 4: Auto-Scaling** (Variable cost)
- 3× CAX21 masters
- Worker pools with autoscaling
- Min 2, max 10 workers
- Cluster Autoscaler manages

**See:** [scaling-and-ha.md](./scaling-and-ha.md#practical-scaling-path) for detailed migration steps

### Multi-Tenant Pattern

**Directory Structure:**

```
terraform/
├── modules/
│   └── hetzner-k3s-cluster/       # Reusable module
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── templates/
│           └── cloud-init.yaml.tpl
├── environments/
│   ├── client-a-prod/
│   │   ├── main.tf               # Uses module
│   │   ├── terraform.tfvars      # Client-specific values
│   │   └── backend.tf            # Separate state
│   ├── client-b-prod/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── client-c-dev/
│       ├── main.tf
│       ├── terraform.tfvars
│       └── backend.tf
```

**Key Principles:**

1. **One module, many clusters**: Same Terraform code for all clients
2. **Variable-driven configuration**: tfvars file per client
3. **Isolated state**: Separate backend per client (S3/Terraform Cloud)
4. **Separate Hetzner tokens**: Each client provides their own API token
5. **Label-based isolation**: Firewall/LB use labels (`cluster=client-a-prod`)

**See:** [terraform-conversion-strategy.md](./terraform-conversion-strategy.md#multi-tenant-pattern) for complete examples

---

## Detailed Documentation

This review includes five detailed documents:

### 1. [Architecture Review](./ARCHITECTURE-REVIEW.md) (This Document)
- Executive summary
- High-level architecture
- Implementation strategy
- Quick reference

### 2. [Provisioning Workflow](./provisioning-workflow.md)
- Step-by-step cluster creation (12 phases)
- All Hetzner API calls with request/response formats
- Cloud-init generation and injection
- k3s installation process
- Timing and orchestration (2-3 minute timeline)

**Key Insights:**
- 10 concurrent instance creations via semaphore
- Network interface detection: 30 attempts × 10s (5 min max)
- SSH readiness: 20 attempts × 5s
- Exponential backoff for API retries

### 3. [Network Architecture](./network-architecture.md)
- Network topology diagrams (single-node, HA, multi-tenant)
- Complete firewall rule breakdown (15+ rules)
- Load balancer configuration (lb11, round-robin)
- Multi-tenant security patterns
- Private network design (10.0.0.0/16)

**Key Insights:**
- Firewall has 50-rule limit (Hetzner constraint)
- Label selectors enable per-cluster targeting
- Private network uses MTU 1450/1280 for detection
- Three isolation strategies for multi-tenancy

### 4. [Scaling and HA](./scaling-and-ha.md)
- Vertical scaling (instance type changes)
- Horizontal scaling (add/remove nodes)
- Cluster autoscaler configuration
- k3s upgrade process (System Upgrade Controller)
- Multi-master HA with embedded etcd
- Multi-location distribution

**Key Insights:**
- Create command is idempotent (no recreation needed)
- Autoscaler provisions nodes with cloud-init templates
- Upgrades use rolling strategy (masters: concurrency=1)
- Odd number of masters prevents split-brain (3, 5, 7)

### 5. [Terraform Conversion Strategy](./terraform-conversion-strategy.md)
- Required providers and versions
- Complete module structure
- Working single-CAX11 example (700 lines)
- Multi-tenant environment layout
- Resource mapping (hetzner-k3s → Terraform)
- CI/CD integration examples

**Key Insights:**
- Hetzner provider has native support for all features
- Cloud-init template can be embedded or external
- Remote state backend required for team collaboration
- Module outputs provide kubeconfig and cluster info

---

## Critical Technical Details

### 1. Cloud-Init Script Generation

**Process:**
1. Template rendering (Crinja - similar to Jinja2)
2. Files are gzip+base64 encoded
3. User-data passed to `POST /servers` API call
4. Cloud-init runs on first boot

**Key Template Variables:**
- `{{ k3s_version }}` - e.g., v1.30.2+k3s2
- `{{ k3s_token }}` - Shared secret for node joining
- `{{ private_network_enabled }}` - Boolean
- `{{ private_network_subnet }}` - e.g., 10.0.0.0/16
- `{{ flannel_backend }}` - CNI configuration
- `{{ labels_and_taints }}` - Node metadata

**See:** [provisioning-workflow.md#cloud-init-process](./provisioning-workflow.md#cloud-init-process)

### 2. Network Interface Detection

**Challenge:** Hetzner private network interfaces don't have predictable names.

**Solution:** MTU-based detection
```bash
NETWORK_INTERFACE=$(
  ip -o link show |
    awk -F': ' '/mtu (1450|1280)/ {print $2}' |
    grep -Ev 'cilium|br|flannel|docker|veth' |
    head -n1
)
```

**Why:** Hetzner private networks use MTU 1450 or 1280 (vs 1500 for public)

**Retry:** 30 attempts × 10 seconds = 5 minutes max wait

**Impact:** This is the longest single operation during provisioning.

**See:** [provisioning-workflow.md#network-detection-logic](./provisioning-workflow.md#network-detection-logic)

### 3. Firewall Rules

**Mode 1: Hetzner Cloud Firewall (Recommended)**

Managed via Hetzner API, applied via label selector:

```yaml
apply_to:
  - label_selector:
      selector: "cluster=my-cluster"
    type: label_selector
```

**Rules Created:**
- SSH (port 22, configurable allowed_networks)
- ICMP (ping)
- NodePort TCP/UDP (30000-32767)
- Kubernetes API (6443, private network only)
- Inter-node traffic (all ports on private subnet)
- etcd (2379, 2380, master IPs only)

**Mode 2: Local Kernel Firewall**

iptables + ipset, systemd services, auto-updates

**See:** [network-architecture.md#firewall-rules](./network-architecture.md#firewall-rules)

### 4. Load Balancer (HA Clusters)

**When Created:** `master_count > 1` AND `create_load_balancer_for_the_kubernetes_api: true`

**Configuration:**
- Type: lb11 (€6/month, 2 vCPU, 4GB RAM)
- Algorithm: round_robin
- Service: TCP 6443 → 6443
- Targets: Label selector `cluster=name,role=master`
- Health check: TCP connect to port 6443

**Kubeconfig:** Points to LB IP instead of single master

**See:** [network-architecture.md#load-balancer-configuration](./network-architecture.md#load-balancer-configuration)

### 5. k3s Installation

**Master:**
```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION="v1.30.2+k3s2" \
  K3S_TOKEN="<random-token>" \
  INSTALL_K3S_EXEC="server" \
  sh -s - \
    --cluster-cidr=10.244.0.0/16 \
    --service-cidr=10.43.0.0/16 \
    --advertise-address=<private-ip> \
    --node-external-ip=<public-ip> \
    --disable-cloud-controller \
    --disable traefik \
    --write-kubeconfig-mode=644 \
    --flannel-iface=<private-interface>
```

**Worker:**
```bash
curl -sfL https://get.k3s.io | \
  K3S_TOKEN="<token>" \
  K3S_URL="https://<master-ip>:6443" \
  INSTALL_K3S_EXEC="agent" \
  sh -s - \
    --node-label=<labels> \
    --node-taint=<taints>
```

**See:** [provisioning-workflow.md#k3s-installation-details](./provisioning-workflow.md#k3s-installation-details)

### 6. Storage Integration

**Hetzner CSI Driver:**
- Manifest URL: Configurable, default v2.17.0
- Installation: `kubectl apply -f <manifest-url>`
- Capabilities: Block volumes, snapshots, expansion
- Authentication: Kubernetes secret with Hetzner token

**Storage Classes:**
- `hcloud-volumes` (default): Hetzner Cloud Volumes (SSD)
- `local-path` (optional): Local node storage (hostPath)

**See:** [provisioning-workflow.md#addon-installation](./provisioning-workflow.md#addon-installation)

---

## Terraform Conversion: Key Decisions

### Decision 1: Cloud-Init Template Management

**Option A: Inline Template (Simpler)**
```hcl
resource "hcloud_server" "master" {
  user_data = templatefile("${path.module}/templates/cloud-init.yaml.tpl", {
    k3s_version = var.k3s_version
    # ... other variables
  })
}
```

**Option B: Template Module (More Flexible)**
```hcl
module "cloud_init" {
  source = "./modules/k3s-cloud-init"
  # ... variables
}

resource "hcloud_server" "master" {
  user_data = module.cloud_init.rendered
}
```

**Recommendation:** Option A for single-node, Option B for complex multi-pool setups

### Decision 2: Multi-Tenancy State Management

**Option A: Separate Workspaces (Not Recommended)**
```bash
terraform workspace new client-a
terraform workspace new client-b
```
❌ Shared state file, risk of cross-contamination

**Option B: Separate Backends (Recommended)**
```hcl
# client-a/backend.tf
terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "clients/client-a/prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```
✅ Complete isolation, separate state files

**Recommendation:** Option B with unique S3 keys per client

### Decision 3: Hetzner Token Management

**Option A: Environment Variables (Development)**
```bash
export HCLOUD_TOKEN="client-a-token"
terraform apply
```

**Option B: Terraform Variables (CI/CD)**
```hcl
variable "hcloud_token" {
  type      = string
  sensitive = true
}

provider "hcloud" {
  token = var.hcloud_token
}
```

**Option C: Secret Management (Production)**
- Terraform Cloud: Workspace variables (encrypted)
- Vault: Dynamic secrets with rotation
- GitHub Secrets: For GitHub Actions

**Recommendation:** Option C for production, Option A for local dev

---

## ARM Architecture (CAX Servers) Considerations

### ARM64 Compatibility

**Good News:** k3s has excellent ARM64 support out of the box.

**Verified Compatible:**
- k3s v1.30+ (tested by hetzner-k3s)
- Containerd runtime
- Flannel CNI
- Cilium CNI
- Hetzner Cloud Controller Manager
- Hetzner CSI Driver
- Cluster Autoscaler
- System Upgrade Controller

**Potential Issues:**

1. **Container Images:**
   - Most official images support `linux/arm64`
   - Third-party images may be `linux/amd64` only
   - Check with `docker manifest inspect <image>`

2. **Build Tools:**
   - Native ARM builds are faster
   - Cross-compilation requires QEMU (slower)

3. **Performance:**
   - CAX servers use Ampere Altra (ARM Neoverse N1)
   - Excellent single-threaded performance
   - Better price/performance than x86 for many workloads

**Recommendation:** Start with ARM (CAX), but be prepared to add x86 node pool if needed

**See:** Project has extensive testing on ARM64, no special configuration needed

---

## Implementation Roadmap

### Phase 1: Proof of Concept (Week 1-2)

**Goal:** Deploy single CAX11 cluster via Terraform

**Tasks:**
1. Set up Hetzner Cloud account and API token
2. Copy single-node example from [terraform-conversion-strategy.md](./terraform-conversion-strategy.md#example-single-cax11-cluster)
3. Customize variables (cluster name, SSH key, location)
4. Run `terraform init && terraform apply`
5. Validate k3s cluster access
6. Test workload deployment (nginx, curl)

**Success Criteria:**
- k3s cluster accessible via kubectl
- Pod deployed and reachable
- Provisioning time < 5 minutes

### Phase 2: Module Development (Week 3-4)

**Goal:** Create reusable Terraform module

**Tasks:**
1. Extract single-node code into module structure
2. Add variables for all configuration options
3. Support multi-master HA configuration
4. Add worker node pools with labels/taints
5. Implement cluster autoscaler configuration
6. Create module documentation
7. Add validation and error handling

**Success Criteria:**
- Module supports 1-7 master nodes
- Module supports 0-N worker pools
- Module works with private/public networks
- Module tested on 3 different cluster configs

### Phase 3: Multi-Tenant Setup (Week 5-6)

**Goal:** Deploy 3 isolated client clusters

**Tasks:**
1. Set up remote state backend (S3 or Terraform Cloud)
2. Create environment directory structure
3. Deploy Client A production cluster (3 masters, 2 workers)
4. Deploy Client B production cluster (different configuration)
5. Deploy Client C dev cluster (single node)
6. Validate complete isolation (network, firewall, state)
7. Document client onboarding process

**Success Criteria:**
- 3 clusters running independently
- No cross-cluster communication
- Each cluster has separate state
- Onboarding takes < 1 hour

### Phase 4: CI/CD Integration (Week 7-8)

**Goal:** Automate cluster provisioning

**Tasks:**
1. Set up CI/CD pipeline (GitHub Actions, GitLab CI, etc.)
2. Implement Terraform plan on pull requests
3. Implement Terraform apply on merge to main
4. Add approval gates for production changes
5. Implement drift detection (scheduled runs)
6. Add notification system (Slack, email)
7. Create runbooks for common operations

**Success Criteria:**
- New cluster provisioned via PR merge
- Configuration changes go through review
- Drift detected and alerted within 1 hour
- Zero manual terraform commands needed

### Phase 5: Production Hardening (Week 9-10)

**Goal:** Production-ready platform

**Tasks:**
1. Add monitoring (Prometheus, Grafana)
2. Implement backup strategy (etcd snapshots → S3)
3. Set up log aggregation (Loki, ELK)
4. Configure alerts (high CPU, disk full, node down)
5. Document disaster recovery procedures
6. Perform load testing
7. Create SLA definitions

**Success Criteria:**
- 99.9% uptime for HA clusters
- RTO < 1 hour, RPO < 15 minutes
- All alerts routed and actionable
- DR procedure tested and documented

---

## Cost Estimation

### Single Client Examples

**Dev/Test: Single CAX11**
- 1× CAX11: €3.85/month
- Network: €0
- Firewall: €0
- **Total: ~€4/month**

**Production: HA Cluster**
- 3× CAX21: 3 × €12.39 = €37.17/month
- 2× CAX21 workers: 2 × €12.39 = €24.78/month
- Load balancer (lb11): €5.95/month
- Network: €0
- **Total: ~€68/month**

**Production: With Autoscaling**
- 3× CAX21 masters: €37.17/month
- 2-10× CAX21 workers: €24.78 - €123.90/month (variable)
- Load balancer: €5.95/month
- Volumes (100GB each): 3 × €4.76 = €14.28/month
- **Total: €82 - €181/month**

### Multi-Tenant Platform (10 Clients)

**Scenario 1: All Dev Clusters**
- 10× single CAX11: €38.50/month
- Platform overhead: €50/month (monitoring, CI/CD)
- **Total: ~€90/month**

**Scenario 2: Mixed (5 Dev, 5 Prod)**
- 5× dev (CAX11): €19.25/month
- 5× prod (HA): 5 × €68 = €340/month
- Platform overhead: €100/month
- **Total: ~€460/month**

**Scenario 3: All Production HA**
- 10× prod (HA): 10 × €68 = €680/month
- Platform overhead: €150/month
- **Total: ~€830/month**

**Per-Client Pricing (Your Revenue):**
- Dev cluster: €10-20/month (2.5-5x markup)
- Prod HA: €100-150/month (1.5-2x markup)
- Margin: 50-75% on infrastructure costs

---

## Potential Pitfalls to Avoid

### 1. Network Interface Detection Timing

**Problem:** Cloud-init script may fail if private network interface isn't ready

**hetzner-k3s Solution:** 30 attempts × 10s delay (5 min timeout)

**Your Solution:** Implement same retry logic in cloud-init template

```bash
MAX_ATTEMPTS=30
for i in $(seq 1 $MAX_ATTEMPTS); do
  INTERFACE=$(ip -o link | awk '/mtu (1450|1280)/ {print $2}')
  [ -n "$INTERFACE" ] && break
  sleep 10
done
```

### 2. Firewall Rule Limit

**Problem:** Hetzner Cloud Firewalls have 50-rule hard limit

**hetzner-k3s Solution:** Validates rule count before creation

**Your Solution:**
- Use CIDR ranges instead of individual IPs
- Consolidate custom rules
- Add validation in Terraform

```hcl
locals {
  total_rules = (
    length(var.ssh_allowed_networks) +
    length(var.api_allowed_networks) +
    length(var.custom_firewall_rules) +
    10  # Base rules
  )
}

resource "null_resource" "validate_firewall_rules" {
  count = local.total_rules > 50 ? "ERROR: Firewall rules exceed 50 limit" : 0
}
```

### 3. Concurrent Instance Creation

**Problem:** Creating many instances simultaneously can hit rate limits

**hetzner-k3s Solution:** Semaphore limiting to 10 concurrent creates

**Your Solution:** Use Terraform's parallelism flag

```bash
terraform apply -parallelism=10
```

### 4. etcd Cluster Size

**Problem:** Even number of masters causes split-brain risk

**hetzner-k3s Solution:** Documentation recommends 3, 5, or 7 masters

**Your Solution:** Add validation

```hcl
variable "master_count" {
  type = number
  validation {
    condition     = var.master_count == 1 || var.master_count % 2 == 1
    error_message = "Master count must be 1 or odd number (3, 5, 7) for HA."
  }
}
```

### 5. Kubeconfig Retrieval

**Problem:** Kubeconfig not immediately available after instance creation

**hetzner-k3s Solution:** SSH polling with 20 retries × 5s

**Your Solution:** Use Terraform provisioner with retry

```hcl
resource "null_resource" "get_kubeconfig" {
  depends_on = [hcloud_server.master]

  provisioner "remote-exec" {
    inline = [
      "timeout 120 bash -c 'until [ -f /etc/rancher/k3s/k3s.yaml ]; do sleep 5; done'",
      "cat /etc/rancher/k3s/k3s.yaml"
    ]

    connection {
      type        = "ssh"
      host        = hcloud_server.master.ipv4_address
      user        = "root"
      private_key = file(var.ssh_private_key_path)
    }
  }
}
```

### 6. Load Balancer Readiness

**Problem:** LB created but public IP not assigned immediately

**hetzner-k3s Solution:** Poll until public_ip_address is not nil

**Your Solution:** Terraform handles this automatically, but add explicit depends_on

```hcl
resource "null_resource" "wait_for_lb" {
  depends_on = [hcloud_load_balancer.api]

  provisioner "local-exec" {
    command = "sleep 10"  # LB needs time to initialize
  }
}
```

### 7. State File Management

**Problem:** Accidentally using same state for multiple clients

**Your Solution:**
- Never use default local backend
- Unique S3 key per client: `clients/<client-id>/terraform.tfstate`
- Add safeguards in backend config

```hcl
terraform {
  backend "s3" {
    bucket = "your-terraform-state"
    key    = "clients/${var.client_id}/terraform.tfstate"  # Interpolation not allowed in backend!
    # Solution: Use separate backend.tf per environment
  }
}
```

### 8. Secrets in State

**Problem:** Hetzner tokens, k3s tokens stored in Terraform state

**Your Solution:**
- Encrypt state at rest (S3 encryption, Terraform Cloud)
- Restrict state file access (IAM policies, RBAC)
- Never commit state files to git
- Use `sensitive = true` on outputs

```hcl
output "k3s_token" {
  value     = random_password.k3s_token.result
  sensitive = true
}
```

---

## Quick Reference

### Terraform Resources Needed

```hcl
# Core resources for single-node cluster
resource "random_password" "k3s_token"       # k3s join token
resource "tls_private_key" "ssh"             # SSH keypair (or use existing)
resource "hcloud_ssh_key" "cluster"          # Upload to Hetzner
resource "hcloud_network" "cluster"          # Private network
resource "hcloud_network_subnet" "cluster"   # Subnet (10.0.0.0/16)
resource "hcloud_firewall" "cluster"         # Security rules
resource "hcloud_server" "master"            # CAX11 instance
resource "null_resource" "get_kubeconfig"    # Retrieve kubeconfig

# Additional for HA cluster
resource "hcloud_server" "master"            # 3+ instances
resource "hcloud_load_balancer" "api"        # API load balancer
resource "hcloud_load_balancer_network" "api"  # Attach to private network
resource "hcloud_load_balancer_service" "api"  # Port 6443 service
resource "hcloud_load_balancer_target" "masters"  # Target master nodes

# Additional for workers
resource "hcloud_server" "worker"            # Worker instances
```

### Key Configuration Values

```yaml
# Network CIDRs
private_network_subnet: 10.0.0.0/16
cluster_cidr: 10.244.0.0/16
service_cidr: 10.43.0.0/16
cluster_dns: 10.43.0.10

# Ports
ssh_port: 22
kubernetes_api: 6443
node_port_range: 30000-32767
etcd_client: 2379
etcd_peer: 2380

# Timeouts
ssh_timeout: 120s
cloud_init_timeout: 300s
instance_ready_timeout: 300s
```

### Essential kubectl Commands

```bash
# Get cluster info
kubectl cluster-info
kubectl get nodes -o wide

# Check component health
kubectl get componentstatuses
kubectl get pods -n kube-system

# View events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Test storage
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: hcloud-volumes
EOF

kubectl get pvc test-pvc
```

### hetzner-k3s CLI Commands (for reference)

```bash
# Create cluster
hetzner-k3s create --config cluster.yaml

# Upgrade k3s version
hetzner-k3s upgrade --config cluster.yaml --new-k3s-version v1.30.3+k3s1

# Delete cluster
hetzner-k3s delete --config cluster.yaml

# Run command on all nodes
hetzner-k3s run --config cluster.yaml --command "uptime"

# List available k3s versions
hetzner-k3s releases
```

---

## Additional Resources

### Official Documentation

- **hetzner-k3s GitHub:** https://github.com/vitobotta/hetzner-k3s
- **Hetzner Cloud API:** https://docs.hetzner.cloud/
- **Terraform Hetzner Provider:** https://registry.terraform.io/providers/hetznercloud/hcloud/latest/docs
- **k3s Documentation:** https://docs.k3s.io/
- **k3s on ARM64:** https://docs.k3s.io/advanced#additional-preparation-for-alpine-linux-setup

### Useful Examples

- **hetzner-k3s Examples:** See `e2e-tests/` directory in repo
- **Terraform Examples:** See [terraform-conversion-strategy.md](./terraform-conversion-strategy.md)
- **Cloud-init Templates:** See `templates/` directory in repo

### Community Resources

- **Hetzner Community:** https://community.hetzner.com/
- **k3s Slack:** https://slack.k3s.io/
- **r/kubernetes:** https://reddit.com/r/kubernetes

---

## Document Structure

This architecture review consists of five documents:

1. **ARCHITECTURE-REVIEW.md** (this document) - Executive summary and overview
2. **provisioning-workflow.md** - Detailed provisioning process
3. **network-architecture.md** - Network topology and security
4. **scaling-and-ha.md** - Scaling strategies and HA patterns
5. **terraform-conversion-strategy.md** - Terraform implementation guide

---

## Conclusion

**hetzner-k3s provides a production-proven blueprint** for deploying k3s clusters on Hetzner Cloud. The architecture is well-designed, performant (2-3 minute cluster creation), and battle-tested across hundreds of deployments.

**For your multi-tenant platform**, the key insights are:

1. ✅ **Single CAX11 deployment is straightforward** - 5 Terraform resources, cloud-init template, done
2. ✅ **Scaling path is clear** - Start small, grow incrementally without recreation
3. ✅ **Multi-tenancy is native** - Label-based isolation, separate state files
4. ✅ **ARM64 works great** - No special considerations needed
5. ✅ **Terraform conversion is feasible** - Direct mapping from CLI operations to Terraform resources

**Next Steps:**

1. Review [terraform-conversion-strategy.md](./terraform-conversion-strategy.md) for complete Terraform implementation
2. Deploy proof-of-concept using single-CAX11 example
3. Develop reusable module following recommended structure
4. Set up multi-tenant environment layout
5. Implement CI/CD pipeline for automation

**Questions or need clarification?** All implementation details are in the linked documents above.

---

**Document Version:** 1.0
**Last Updated:** 2025-11-17
**Maintainer:** Architecture Review Team
