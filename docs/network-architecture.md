# Network and Security Architecture

This document provides a comprehensive overview of the network and security architecture for hetzner-k3s clusters.

## Table of Contents

1. [Network Topology](#network-topology)
2. [Firewall Rules](#firewall-rules)
3. [Load Balancer Configuration](#load-balancer-configuration)
4. [Security Patterns](#security-patterns)
5. [Multi-Tenant Security](#multi-tenant-security)

## Network Topology

### Overview

hetzner-k3s supports two distinct networking modes:
- **Private Network Mode** (default): Nodes communicate via a dedicated Hetzner private network
- **Public Network Mode**: Nodes communicate over public IPs with encrypted tunnels (WireGuard)

### Single-Node Cluster

```
┌─────────────────────────────────────────┐
│  Hetzner Cloud                          │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  Master Node                      │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │ Public IP: x.x.x.x          │  │  │
│  │  │ Role: master                │  │  │
│  │  │ Cluster Label: cluster-name │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
│                                         │
│  Internet → SSH (port 22/custom)        │
│  Internet → K8s API (port 6443)         │
│  Internet → NodePorts (30000-32767)     │
└─────────────────────────────────────────┘
```

### Multi-Node Cluster with Private Network

```
┌──────────────────────────────────────────────────────────────────────┐
│  Hetzner Cloud                                                       │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Load Balancer (lb11)                                          │ │
│  │  Public IP: lb.x.x.x.x                                         │ │
│  │  Algorithm: Round Robin                                        │ │
│  │  Port: 6443 → Masters:6443 (private IPs)                       │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                 ↓                                    │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Private Network: 10.0.0.0/16 (default)                        │ │
│  │  Network Zone: eu-central / us-east / us-west / ap-southeast   │ │
│  │                                                                 │ │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐       │ │
│  │  │ Master-1     │   │ Master-2     │   │ Master-3     │       │ │
│  │  │ Public: x.1  │   │ Public: x.2  │   │ Public: x.3  │       │ │
│  │  │ Private:10.0 │   │ Private:10.0 │   │ Private:10.0 │       │ │
│  │  │              │←─→│              │←─→│              │       │ │
│  │  │ etcd:2379/80 │   │ etcd:2379/80 │   │ etcd:2379/80 │       │ │
│  │  └──────────────┘   └──────────────┘   └──────────────┘       │ │
│  │         ↓                   ↓                   ↓              │ │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐       │ │
│  │  │ Worker-1     │   │ Worker-2     │   │ Worker-N     │       │ │
│  │  │ Public: y.1  │   │ Public: y.2  │   │ Public: y.N  │       │ │
│  │  │ Private:10.0 │   │ Private:10.0 │   │ Private:10.0 │       │ │
│  │  └──────────────┘   └──────────────┘   └──────────────┘       │ │
│  │                                                                 │ │
│  │  All nodes communicate via private IPs (10.0.0.0/16)           │ │
│  │  All TCP/UDP traffic allowed within private network            │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  Internet → SSH via public IPs                                      │
│  Internet → K8s API via Load Balancer                               │
│  Internet → NodePorts (30000-32767) via node public IPs             │
└──────────────────────────────────────────────────────────────────────┘
```

### Multi-Node Cluster without Private Network

```
┌──────────────────────────────────────────────────────────────────────┐
│  Hetzner Cloud                                                       │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Load Balancer (lb11)                                          │ │
│  │  Public IP: lb.x.x.x.x                                         │ │
│  │  Algorithm: Round Robin                                        │ │
│  │  Port: 6443 → Masters:6443 (public IPs)                        │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                 ↓                                    │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐            │
│  │ Master-1     │   │ Master-2     │   │ Master-3     │            │
│  │ Public: x.1  │   │ Public: x.2  │   │ Public: x.3  │            │
│  │              │←─→│              │←─→│              │            │
│  │ WireGuard    │   │ WireGuard    │   │ WireGuard    │            │
│  │ (51820/51871)│   │ (51820/51871)│   │ (51820/51871)│            │
│  │ etcd:2379/80 │   │ etcd:2379/80 │   │ etcd:2379/80 │            │
│  └──────────────┘   └──────────────┘   └──────────────┘            │
│         ↓                   ↓                   ↓                   │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐            │
│  │ Worker-1     │   │ Worker-2     │   │ Worker-N     │            │
│  │ Public: y.1  │   │ Public: y.2  │   │ Public: y.N  │            │
│  │ WireGuard    │   │ WireGuard    │   │ WireGuard    │            │
│  └──────────────┘   └──────────────┘   └──────────────┘            │
│                                                                      │
│  Node-to-node encrypted via WireGuard tunnels over public Internet  │
│  etcd traffic restricted to master public IPs only                  │
└──────────────────────────────────────────────────────────────────────┘
```

### Public IP Addressing

All nodes receive public IPv4 and IPv6 addresses by default:
- **IPv4**: Enabled by default (`networking.public_network.ipv4: true`)
- **IPv6**: Enabled by default (`networking.public_network.ipv6: true`)

Public IPs are used for:
- SSH access from allowed networks
- NodePort service exposure (30000-32767)
- Load balancer targets (if private network is disabled)
- External cluster API access

### Private Network Configuration

**Default Subnet**: `10.0.0.0/16`

The private network is:
- **Enabled by default** (`networking.private_network.enabled: true`)
- Created automatically per cluster (named after cluster_name)
- Used for all node-to-node communication when enabled
- Supports custom subnet configuration
- Can use existing Hetzner network (`networking.private_network.existing_network_name`)

**Private Network Benefits**:
- Lower latency between nodes
- No bandwidth costs for internal traffic
- Enhanced security (traffic doesn't traverse public internet)
- Simplified firewall rules (allow all traffic within subnet)

### Network Zones and Location Mapping

Hetzner Cloud requires all nodes in a private network to be in the same network zone:

| Location Code | Location Name       | Network Zone    |
|---------------|---------------------|-----------------|
| ash           | Ashburn, VA         | us-east         |
| hil           | Hillsboro, OR       | us-west         |
| sin           | Singapore           | ap-southeast    |
| fsn1          | Falkenstein         | eu-central      |
| nbg1          | Nuremberg           | eu-central      |
| hel1          | Helsinki            | eu-central      |
| Other EU      | Various EU          | eu-central      |

**Important Constraints**:
- All master nodes must be in the same network zone
- All worker nodes must be in the same network zone as masters
- This is enforced when `private_network.enabled: true` or `datastore.mode: etcd`

### Cluster CIDR and Service CIDR

**Pod Network (Cluster CIDR)**: `10.244.0.0/16` (default)
- Used for pod-to-pod communication
- Managed by CNI (Flannel or Cilium)

**Service Network (Service CIDR)**: `10.43.0.0/16` (default)
- Used for Kubernetes service IPs
- Cluster DNS: `10.43.0.10`

**Network Separation**:
```
┌─────────────────────────────────────────────┐
│ Network Layers                              │
├─────────────────────────────────────────────┤
│ Private Network:     10.0.0.0/16            │
│   └─ Node-to-node infrastructure traffic    │
│                                             │
│ Pod Network:         10.244.0.0/16          │
│   └─ Container-to-container communication   │
│                                             │
│ Service Network:     10.43.0.0/16           │
│   └─ Service discovery and load balancing   │
└─────────────────────────────────────────────┘
```

## Firewall Rules

hetzner-k3s creates Hetzner Cloud Firewalls that are applied to all cluster nodes via label selectors (`cluster=<cluster-name>`).

### Firewall Application

Firewalls are applied using **label selectors**:
```
apply_to:
  - label_selector: "cluster=<cluster-name>"
    type: "label_selector"
```

All nodes in the cluster automatically inherit these firewall rules based on their cluster label.

### Rule Limit

**Hetzner Cloud imposes a hard limit of 50 rules per firewall**. The system validates this and will fail fast if the total rules (default + custom) exceed 50.

### Common Rules (All Configurations)

These rules are created regardless of private network settings:

#### 1. SSH Access
```yaml
Description: Allow SSH port
Direction: in
Protocol: tcp
Port: 22 (or custom via networking.ssh.port)
Source IPs: networking.allowed_networks.ssh (default: ["0.0.0.0/0"])
```
Controls SSH access to all nodes. Restrict via `allowed_networks.ssh` for enhanced security.

#### 2. ICMP (Ping)
```yaml
Description: Allow ICMP (ping)
Direction: in
Protocol: icmp
Source IPs: ["0.0.0.0/0", "::/0"]
```
Allows ping from anywhere for network diagnostics.

#### 3. NodePort Range TCP
```yaml
Description: Node port range TCP
Direction: in
Protocol: tcp
Port: 30000-32767
Source IPs: ["0.0.0.0/0", "::/0"]
```
Exposes Kubernetes NodePort services to the internet.

#### 4. NodePort Range UDP
```yaml
Description: Node port range UDP
Direction: in
Protocol: udp
Port: 30000-32767
Source IPs: ["0.0.0.0/0", "::/0"]
```
Exposes UDP NodePort services to the internet.

### Private Network Mode Rules

When `networking.private_network.enabled: true`, these additional rules are created:

#### 5. Kubernetes API Access
```yaml
Description: Allow Kubernetes API access to allowed networks
Direction: in
Protocol: tcp
Port: 6443
Source IPs: networking.allowed_networks.api (default: ["0.0.0.0/0"])
```
Controls access to the Kubernetes API server. Restrict via `allowed_networks.api` for production.

#### 6. Private Network TCP Traffic
```yaml
Description: Allow all TCP traffic between nodes on the private network
Direction: in
Protocol: tcp
Port: any
Source IPs: [<private_network.subnet>]  # e.g., ["10.0.0.0/16"]
```
Permits unrestricted TCP communication between cluster nodes on private network.

#### 7. Private Network UDP Traffic
```yaml
Description: Allow all UDP traffic between nodes on the private network
Direction: in
Protocol: udp
Port: any
Source IPs: [<private_network.subnet>]  # e.g., ["10.0.0.0/16"]
```
Permits unrestricted UDP communication between cluster nodes on private network.

### Public Network Mode Rules

When `networking.private_network.enabled: false`, these rules are created instead:

#### 5. Kubernetes API Access (Public)
```yaml
Description: Allow port 6443 (Kubernetes API server) between masters
Direction: in
Protocol: tcp
Port: 6443
Source IPs: ["0.0.0.0/0", "::/0"]
```
Allows public access to Kubernetes API on all nodes.

#### 6. WireGuard Traffic
```yaml
Description: Allow wireguard traffic (Cilium)
Direction: in
Protocol: tcp
Port: 51820 (Flannel) or 51871 (Cilium)
Source IPs: ["0.0.0.0/0", "::/0"]
```
Enables encrypted node-to-node communication via WireGuard tunnels. Port depends on CNI selection:
- Flannel CNI: Port 51820
- Cilium CNI: Port 51871

#### 7. etcd Traffic (Master-to-Master)
```yaml
Description: Allow etcd traffic between masters
Direction: in
Protocol: tcp
Port: 2379
Source IPs: [<master_public_ips>/32]  # e.g., ["1.2.3.4/32", "5.6.7.8/32"]
```
Allows etcd client communication between master nodes. Only created when:
- `datastore.mode: etcd`
- Multiple masters exist

#### 8. etcd Peer Traffic (Master-to-Master)
```yaml
Description: Allow etcd traffic between masters
Direction: in
Protocol: tcp
Port: 2380
Source IPs: [<master_public_ips>/32]
```
Allows etcd peer communication between master nodes. Only created when:
- `datastore.mode: etcd`
- Multiple masters exist

### Embedded Registry Mirror Rule

When embedded registry mirror is enabled without private network:

#### 9. Peer-to-Peer Image Distribution
```yaml
Description: Allow traffic between nodes for peer-to-peer image distribution
Direction: in
Protocol: tcp
Port: 5001
Source IPs: ["0.0.0.0/0", "::/0"]
```
Only created when:
- `embedded_registry_mirror.enabled: true`
- `networking.private_network.enabled: false`

### Custom Firewall Rules

Users can define custom rules via `networking.allowed_networks.custom_firewall_rules`:

```yaml
networking:
  allowed_networks:
    custom_firewall_rules:
      - description: "Allow custom service"
        direction: "in"
        protocol: "tcp"
        port: "8080"
        source_ips:
          - "203.0.113.0/24"
      - description: "Block outbound to specific network"
        direction: "out"
        protocol: "tcp"
        port: "443"
        destination_ips:
          - "198.51.100.0/24"
```

**Custom Rule Fields**:
- `description` (optional): Human-readable description
- `protocol`: tcp, udp, icmp, esp, gre (default: tcp)
- `direction`: in or out (default: in)
- `port`: Single port ("80"), range ("30000-32767"), or "any" (default: any)
- `source_ips`: CIDR ranges for incoming traffic (when direction=in)
- `destination_ips`: CIDR ranges for outgoing traffic (when direction=out)

**Important**: Total rules (default + custom) cannot exceed **50 rules**.

### Firewall Update Behavior

- Firewalls are created during cluster creation
- Existing firewalls are **updated** (not recreated) when rules change
- Updates use the `set_rules` action to replace all rules atomically
- Retries up to 10 times with 5-second intervals on failure

### Local Firewall Bypass

Set `networking.public_network.use_local_firewall: true` to skip Hetzner Cloud Firewall creation entirely and manage firewalls locally on each node (e.g., via iptables, ufw).

## Load Balancer Configuration

### When Load Balancer is Created

A Hetzner Cloud Load Balancer is created **only** when:
1. `create_load_balancer_for_the_kubernetes_api: true` (setting exists)
2. **AND** number of master nodes > 1

For single-master clusters, the master's public IP is used directly for API access.

### Load Balancer Specifications

**Type**: `lb11` (Hetzner's smallest load balancer)
- Up to 20,000 concurrent connections
- Up to 20,000 requests per second
- Included traffic: 20 TB

**Naming**: `<cluster-name>-api`

**Location**: Placed in the same location as the first master node

**Algorithm**: Round Robin
```yaml
algorithm:
  type: "round_robin"
```
Distributes traffic evenly across all healthy master nodes.

### Load Balancer Configuration

#### With Private Network
```yaml
load_balancer:
  type: "lb11"
  location: <first_master_location>
  network: <network_id>
  public_interface: true

  services:
    - destination_port: 6443
      listen_port: 6443
      protocol: "tcp"
      proxyprotocol: false

  targets:
    - label_selector: "cluster=<cluster-name>,role=master"
      type: "label_selector"
      use_private_ip: true
```

#### Without Private Network
```yaml
load_balancer:
  type: "lb11"
  location: <first_master_location>
  public_interface: true

  services:
    - destination_port: 6443
      listen_port: 6443
      protocol: "tcp"
      proxyprotocol: false

  targets:
    - label_selector: "cluster=<cluster-name>,role=master"
      type: "label_selector"
      use_private_ip: false
```

### Target Selection

**Label Selector**: `cluster=<cluster-name>,role=master`

The load balancer automatically discovers and targets all master nodes based on their labels:
- All nodes with `cluster=<cluster-name>` AND `role=master` are added as targets
- Dynamic: New masters are automatically added when scaled up
- Health checking: Unhealthy masters are automatically removed from rotation

### Public vs Private IP Usage

**Private Network Enabled**:
- `use_private_ip: true`
- Load balancer targets masters via their private network IPs (e.g., 10.0.0.x)
- More secure, no public API exposure from individual masters
- Lower latency

**Private Network Disabled**:
- `use_private_ip: false`
- Load balancer targets masters via their public IPs
- Masters' API servers must be accessible on public IPs
- Traffic traverses public internet

### Load Balancer Traffic Flow

```
External Client (kubectl, CI/CD)
         ↓
Load Balancer Public IP (lb.x.x.x.x:6443)
         ↓
Round Robin Algorithm
         ↓
    ┌────┴────┬────────┬────────┐
    ↓         ↓        ↓        ↓
Master-1  Master-2  Master-3  Master-N
(Private or Public IP:6443)
```

### Load Balancer Lifecycle

**Creation**: During cluster creation when conditions met
**Update**: Automatically when masters are added/removed (via label selector)
**Deletion**:
- Automatically when scaling down to 1 master
- During cluster destruction

## Security Patterns

### Allowed Networks Configuration

Control access to critical services via `networking.allowed_networks`:

```yaml
networking:
  allowed_networks:
    ssh:
      - "0.0.0.0/0"  # Default: Allow SSH from anywhere
    api:
      - "0.0.0.0/0"  # Default: Allow API access from anywhere
```

**Production Recommendations**:
```yaml
networking:
  allowed_networks:
    ssh:
      - "203.0.113.0/24"      # Office network
      - "198.51.100.50/32"    # VPN gateway
    api:
      - "203.0.113.0/24"      # Office network
      - "198.51.100.0/24"     # CI/CD network
      - "192.0.2.100/32"      # Bastion host
```

### Node-to-Node Communication

#### Private Network Mode (Recommended)
- **Security**: All node-to-node traffic on isolated private network (10.0.0.0/16)
- **Encryption**: Not required (private network is isolated at Hetzner infrastructure level)
- **Firewall**: Simple rules - allow all TCP/UDP from private subnet
- **Performance**: High performance, low latency, no bandwidth costs

#### Public Network Mode
- **Security**: All traffic encrypted via WireGuard tunnels
- **Encryption**: Mandatory CNI encryption (Flannel WireGuard or Cilium WireGuard)
- **Firewall**: More complex rules - WireGuard, etcd restricted to specific IPs
- **Performance**: Slightly higher latency due to encryption overhead

### NodePort Security

**Range**: 30000-32767 (Kubernetes standard)

**Exposure**: Fully open to internet by default
```yaml
Port Range: 30000-32767
Protocol: TCP and UDP
Source: 0.0.0.0/0, ::/0
```

**Security Recommendations**:

1. **Use LoadBalancer Services**: Instead of NodePorts, use LoadBalancer services with Hetzner Cloud Controller Manager
2. **Ingress Controllers**: Deploy NGINX/Traefik ingress with single LoadBalancer
3. **Network Policies**: Implement Kubernetes Network Policies to restrict pod-level access
4. **Custom Firewall Rules**: Add custom rules to restrict NodePort access if needed:
   ```yaml
   custom_firewall_rules:
     - description: "Restrict NodePort to specific clients"
       direction: "in"
       protocol: "tcp"
       port: "30000-32767"
       source_ips:
         - "203.0.113.0/24"
   ```
   **Warning**: This would override the default open rule

### etcd Port Security

#### Private Network Mode
- **Ports**: 2379 (client), 2380 (peer)
- **Access**: Allowed from all IPs in private network
- **Security**: Private network isolation provides security
- **No specific firewall rules**: Covered by "allow all from private subnet"

#### Public Network Mode (High Security)
- **Port 2379**: Restricted to master node public IPs only
  ```yaml
  Source IPs: ["1.2.3.4/32", "5.6.7.8/32", "9.10.11.12/32"]
  ```
- **Port 2380**: Restricted to master node public IPs only
  ```yaml
  Source IPs: ["1.2.3.4/32", "5.6.7.8/32", "9.10.11.12/32"]
  ```
- **Dynamic Updates**: Firewall rules update when masters are added/removed
- **Recommendation**: Always use private network for etcd in production

### SSH Port Security

**Default Port**: 22 (customizable via `networking.ssh.port`)

**Default Access**: Open to internet (0.0.0.0/0)

**Production Security**:
```yaml
networking:
  ssh:
    port: 2222  # Non-standard port
  allowed_networks:
    ssh:
      - "203.0.113.0/24"  # Corporate network
      - "198.51.100.1/32" # Jump host
```

### API Server Security

**Port**: 6443

**Access Pattern**:
- **With Load Balancer**: Access via load balancer IP only
- **Without Load Balancer**: Direct access to master node IP

**Security Layers**:
1. **Network Level**: Firewall rules via `allowed_networks.api`
2. **Authentication**: Kubernetes RBAC and authentication
3. **Authorization**: Kubernetes RBAC policies
4. **Audit**: Kubernetes audit logging

**Production Recommendation**:
```yaml
networking:
  allowed_networks:
    api:
      - "203.0.113.0/24"      # Office network
      - "198.51.100.0/24"     # CI/CD runners
      - "<lb_public_ip>/32"   # Load balancer (if external access needed)
```

### CNI Encryption

**Flannel**:
- WireGuard backend (when encryption enabled)
- Port: 51820
- Encryption: Automatic when `networking.cni.encryption: true`

**Cilium**:
- WireGuard mode
- Port: 51871
- Encryption: Automatic when `networking.cni.encryption: true`
- Enhanced features: Network policies, observability, egress gateway

**Default**: Encryption enabled (`networking.cni.encryption: true`)

## Multi-Tenant Security

### Isolation Strategies

#### 1. Separate Clusters (Recommended)
**Approach**: Deploy one cluster per tenant

**Network Isolation**:
- Each cluster has its own private network (10.0.0.0/16)
- Separate Hetzner Cloud Firewalls per cluster
- No shared infrastructure between tenants

**Configuration**:
```yaml
# Tenant A cluster
cluster_name: tenant-a-prod
networking:
  private_network:
    subnet: "10.0.0.0/16"

# Tenant B cluster
cluster_name: tenant-b-prod
networking:
  private_network:
    subnet: "10.0.0.0/16"  # Same subnet, different network
```

**Benefits**:
- Complete network isolation
- Independent scaling
- Blast radius containment
- Simple security model

**Drawbacks**:
- Higher costs (minimum 3 nodes per HA cluster)
- More management overhead

#### 2. Shared Cluster with Network Policies
**Approach**: Single cluster with namespace isolation

**Network Isolation**:
- Kubernetes Network Policies per namespace
- Calico/Cilium for policy enforcement
- Shared private network, logical segmentation

**Not Handled by hetzner-k3s**:
- Network Policies must be deployed manually
- Requires CNI with policy support (Cilium recommended)

**Example Network Policy**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-tenant
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          tenant: tenant-a
  egress:
  - to:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          tenant: tenant-a
```

### Firewall Separation

#### Per-Cluster Firewalls (Automatic)
Each cluster gets its own Hetzner Cloud Firewall:
- Firewall name: `<cluster-name>`
- Applied via label: `cluster=<cluster-name>`
- Complete isolation at infrastructure level

#### Per-Tenant Custom Rules
Use custom firewall rules for tenant-specific restrictions:

```yaml
# Tenant A cluster
networking:
  allowed_networks:
    ssh:
      - "203.0.113.0/24"  # Tenant A office
    api:
      - "203.0.113.0/24"
    custom_firewall_rules:
      - description: "Allow tenant A application port"
        port: "8443"
        protocol: "tcp"
        source_ips:
          - "203.0.113.0/24"

# Tenant B cluster
networking:
  allowed_networks:
    ssh:
      - "198.51.100.0/24"  # Tenant B office
    api:
      - "198.51.100.0/24"
    custom_firewall_rules:
      - description: "Allow tenant B application port"
        port: "9443"
        protocol: "tcp"
        source_ips:
          - "198.51.100.0/24"
```

### Recommendations for Isolated Clients

#### High Security Requirements

**Strategy**: Separate Cluster per Tenant

**Configuration**:
```yaml
cluster_name: client-<tenant-id>
networking:
  private_network:
    enabled: true
    subnet: "10.0.0.0/16"
  allowed_networks:
    ssh:
      - "<tenant-vpn-ip>/32"
    api:
      - "<tenant-vpn-ip>/32"
      - "<tenant-office-cidr>"
  cni:
    encryption: true
```

**Additional Measures**:
1. Separate Hetzner Cloud projects per tenant
2. Different API tokens per project
3. No shared credentials or SSH keys
4. Separate monitoring and logging stacks

#### Medium Security Requirements

**Strategy**: Shared Cluster with Strong Isolation

**Requirements**:
1. Use Cilium CNI with network policies enabled:
   ```yaml
   networking:
     cni:
       mode: cilium
       encryption: true
   ```

2. Deploy per-tenant namespaces with network policies
3. Use Kubernetes RBAC for access control
4. Implement resource quotas per namespace
5. Use pod security policies/standards

**Network Policy Template**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: tenant-namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: tenant-namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
  egress:
  - to:
    - podSelector: {}
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```

#### Cost-Optimized Multi-Tenancy

**Strategy**: Shared Infrastructure with Namespace Isolation

**Suitable For**:
- Development environments
- Non-production workloads
- Trusted tenants
- Cost-sensitive deployments

**Configuration**:
```yaml
cluster_name: shared-dev
networking:
  private_network:
    enabled: true
  allowed_networks:
    ssh:
      - "<admin-network>/24"
    api:
      - "0.0.0.0/0"  # Rely on Kubernetes auth
```

**Additional Controls**:
- Namespace-level resource quotas
- Pod security standards enforcement
- Network policies for tenant isolation
- Separate service accounts per tenant
- Audit logging enabled

### Multi-Tenant Network Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Approach 1: Separate Clusters (Highest Isolation)             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────┐  ┌──────────────────────────┐    │
│  │ Tenant A Cluster         │  │ Tenant B Cluster         │    │
│  │ Private Net: 10.0.0.0/16 │  │ Private Net: 10.0.0.0/16 │    │
│  │ Firewall: tenant-a       │  │ Firewall: tenant-b       │    │
│  │ SSH: 203.0.113.0/24      │  │ SSH: 198.51.100.0/24     │    │
│  │ API: 203.0.113.0/24      │  │ API: 198.51.100.0/24     │    │
│  └──────────────────────────┘  └──────────────────────────┘    │
│                                                                 │
│  Infrastructure-level isolation, separate networks              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Approach 2: Shared Cluster with Network Policies              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Shared Cluster: multi-tenant                            │   │
│  │ Private Network: 10.0.0.0/16                            │   │
│  │ Firewall: multi-tenant                                  │   │
│  │                                                         │   │
│  │ ┌────────────────────┐  ┌────────────────────┐         │   │
│  │ │ Namespace: tenant-a│  │ Namespace: tenant-b│         │   │
│  │ │ Network Policy: ✓  │  │ Network Policy: ✓  │         │   │
│  │ │ Resource Quota: ✓  │  │ Resource Quota: ✓  │         │   │
│  │ └────────────────────┘  └────────────────────┘         │   │
│  │                                                         │   │
│  │ Logical isolation via Kubernetes                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Security Checklist for Multi-Tenant Deployments

- [ ] Use separate clusters for high-security tenants
- [ ] Restrict SSH access to specific IP ranges per tenant
- [ ] Restrict API access to tenant-specific networks
- [ ] Enable private networking for all production clusters
- [ ] Enable CNI encryption
- [ ] Deploy Network Policies in shared clusters
- [ ] Implement resource quotas per tenant namespace
- [ ] Use separate Hetzner projects for critical tenants
- [ ] Rotate and isolate SSH keys per cluster
- [ ] Enable Kubernetes audit logging
- [ ] Implement pod security standards
- [ ] Monitor network traffic between namespaces
- [ ] Regular security reviews of firewall rules
- [ ] Document tenant-specific firewall requirements
- [ ] Test isolation via penetration testing

---

## Additional Resources

- [Hetzner Cloud Firewall Documentation](https://docs.hetzner.com/cloud/firewalls/overview/)
- [Hetzner Cloud Networks Documentation](https://docs.hetzner.com/cloud/networks/overview/)
- [Hetzner Cloud Load Balancers Documentation](https://docs.hetzner.com/cloud/load-balancers/overview/)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Cilium Network Policies](https://docs.cilium.io/en/stable/security/policy/)
