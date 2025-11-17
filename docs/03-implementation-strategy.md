# Your Platform: Implementation Strategy
**Part:** 3 of 11

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

**Previous:** [02-how-it-works.md](./02-how-it-works.md)
**Next:** [04-technical-details.md](./04-technical-details.md)
