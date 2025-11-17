# Critical Technical Details
**Part:** 4 of 11

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

**Previous:** [03-implementation-strategy.md](./03-implementation-strategy.md)
**Next:** [05-terraform-decisions.md](./05-terraform-decisions.md)
