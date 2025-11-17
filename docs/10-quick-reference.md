# Quick Reference
**Part:** 10 of 11

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

## Navigation

**Previous:** [09-pitfalls-and-solutions.md](./09-pitfalls-and-solutions.md)
**Next:** [11-resources.md](./11-resources.md)
**Back to:** [ARCHITECTURE-REVIEW.md](./ARCHITECTURE-REVIEW.md)
