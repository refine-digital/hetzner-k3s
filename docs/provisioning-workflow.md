# Provisioning Workflow

This document provides a detailed technical breakdown of how hetzner-k3s provisions a k3s cluster from zero to a running state, including all Hetzner API calls, cloud-init processes, and orchestration logic.

## Table of Contents

- [Overview](#overview)
- [Step-by-Step Provisioning Flow](#step-by-step-provisioning-flow)
- [Hetzner API Operations](#hetzner-api-operations)
- [Cloud-Init Process](#cloud-init-process)
- [k3s Installation Details](#k3s-installation-details)
- [Timing and Orchestration](#timing-and-orchestration)
- [Network Detection Logic](#network-detection-logic)
- [Error Handling and Retries](#error-handling-and-retries)

## Overview

The provisioning process is orchestrated by the `Cluster::Create` class (`src/cluster/create.cr`) which coordinates infrastructure creation, cloud-init injection, and k3s installation across master and worker nodes concurrently.

**Typical Timeline**: 2-3 minutes for an HA cluster (3 masters + workers)

## Step-by-Step Provisioning Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     INITIALIZATION PHASE                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Configuration Loading                                         │
│    - Load cluster configuration from YAML                        │
│    - Initialize Hetzner API client                              │
│    - Validate configuration (locations, instance types, etc.)   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Infrastructure Setup (Pre-Instance)                          │
│    a. Network Manager: Create/find private network (if enabled) │
│       - POST /networks (if new)                                 │
│       - GET /networks (if existing)                             │
│    b. SSH Key: Create/find SSH key                             │
│       - POST /ssh_keys (if new)                                 │
│       - GET /ssh_keys (if existing)                             │
│    c. Initialize instance builders for masters and workers      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MASTER CREATION PHASE                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Create Master Instances (CONCURRENT - max 10 at a time)     │
│    For each master:                                             │
│    a. Generate cloud-init configuration                         │
│       - Render cloud_init.yaml template                         │
│       - Include SSH configuration                               │
│       - Include firewall rules (if public network)              │
│       - Include package list (fail2ban, wireguard)              │
│       - Add hostname setup commands                             │
│    b. POST /servers with:                                       │
│       - Instance name, location, type, image                    │
│       - SSH key ID                                              │
│       - Network ID (if private network enabled)                 │
│       - Cloud-init user_data                                    │
│       - Labels: cluster=<name>, role=master                     │
│       - Public IPv4/IPv6 settings                               │
│    c. Wait for instance creation (with retry)                   │
│    d. Ensure instance is ready:                                 │
│       - Poll instance status until "running"                    │
│       - If private network: POST /servers/{id}/actions/poweron  │
│       - If private network: POST /servers/{id}/actions/attach_to_network │
│       - Wait for SSH to respond (echo ready)                    │
│    e. Send instance to kubernetes_masters_installation_queue    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Load Balancer Creation (if HA cluster)                      │
│    - Only if master_count > 1 and create_load_balancer enabled │
│    - POST /load_balancers with:                                 │
│      - Type: lb11                                               │
│      - Location: first master location                          │
│      - Network ID (if private network)                          │
│      - Service: TCP 6443 → 6443                                │
│      - Target: label_selector (cluster=<name>,role=master)      │
│      - use_private_ip: based on network mode                    │
│    - Wait for public IP assignment                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Firewall Manager (if enabled)                               │
│    - Configure firewall rules for master instances             │
│    - Attach firewall resources                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    K3S INSTALLATION PHASE                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. Initiate k3s Setup (SPAWNED - runs in background)           │
│    Kubernetes::Installer spawned as fiber                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. Install First Master                                         │
│    a. Receive first master from queue                           │
│    b. Wait for cloud-init completion:                           │
│       - SSH: wait for /var/lib/cloud/instance/boot-finished     │
│       - Timeout: 5 minutes                                      │
│    c. Generate and deploy master install script:                │
│       - Render master_install_script.sh template                │
│       - Include network configuration                           │
│       - Include k3s installation via get.k3s.io                 │
│    d. Execute script via SSH                                    │
│    e. Wait 10 seconds for control plane startup                 │
│    f. Save kubeconfig from master                               │
│    g. Validate control plane:                                   │
│       - Run: kubectl cluster-info                               │
│       - Retry: 3 attempts with 30s timeout each                 │
│       - Must see "running" in output                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. Install Additional Masters (CONCURRENT)                     │
│    For each additional master (if HA):                          │
│    a. Receive master from queue                                 │
│    b. Wait for cloud-init completion                            │
│    c. Generate and deploy master install script                 │
│       - Includes --server flag pointing to first master/LB      │
│    d. Execute script via SSH                                    │
│    All masters installed in parallel                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    WORKER CREATION PHASE                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 9. Create Worker Instances (CONCURRENT - max 10 at a time)     │
│    Started immediately after k3s setup is initiated             │
│    For each worker:                                             │
│    a. Generate cloud-init (similar to masters)                  │
│    b. POST /servers with worker configuration                   │
│    c. Wait for instance ready (same as masters)                 │
│    d. Send to kubernetes_workers_installation_queue             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 10. Install Workers (CONCURRENT - max 10 at a time)            │
│     For each worker:                                            │
│     a. Receive worker from queue                                │
│     b. Wait for cloud-init completion                           │
│     c. Generate and deploy worker install script:               │
│        - Render worker_install_script.sh template               │
│        - Include K3S_URL pointing to master/LB                  │
│        - Include k3s agent installation                         │
│     d. Execute script via SSH                                   │
│     All workers installed in parallel                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 11. Post-Installation Software                                  │
│     a. Install Hetzner Cloud Controller Manager                 │
│     b. Install Hetzner CSI Driver                               │
│     c. Install Cilium CNI (if configured)                       │
│     d. Install System Upgrade Controller (if configured)        │
│     e. Install Cluster Autoscaler (if configured)               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 12. Finalization                                                │
│     a. Wait for at least one worker to be Ready                 │
│        - kubectl get nodes | grep worker | grep Ready           │
│        - Timeout: 5 minutes                                     │
│     b. Switch kubectl context to cluster                        │
│     c. Signal completion via completed_channel                  │
│     d. Display protection warning if not enabled                │
└─────────────────────────────────────────────────────────────────┘
```

## Hetzner API Operations

All API operations use the base URL: `https://api.hetzner.cloud/v1`

### Authentication

Every request includes:
```
Authorization: Bearer <hetzner_token>
```

### API Endpoints Hit During Provisioning (in order)

#### 1. Preflight Checks

**GET /locations**
- Purpose: Fetch available data center locations
- Used for: Validating location configuration
- Pagination: Single page
- Response: List of location objects with names, cities, countries

**GET /server_types**
- Purpose: Fetch available server types
- Used for: Validating instance type configuration
- Pagination: Yes (loops through all pages)
- Request params: `?page=<number>`
- Response: List of server type objects with names, cores, memory, disk

#### 2. Network Setup (if private network enabled)

**POST /networks**
- Purpose: Create private network
- Request body:
  ```json
  {
    "name": "<cluster_name>",
    "ip_range": "10.0.0.0/16",
    "subnets": [
      {
        "ip_range": "10.0.0.0/16",
        "network_zone": "eu-central",
        "type": "cloud"
      }
    ]
  }
  ```
- Retry: 10 attempts, 5s interval
- Response: Network object with ID

**GET /networks**
- Purpose: Find existing network by name
- Used when: Network already exists or after creation

#### 3. SSH Key Setup

**POST /ssh_keys**
- Purpose: Upload SSH public key
- Request body:
  ```json
  {
    "name": "<cluster_name>",
    "public_key": "<ssh_public_key_content>"
  }
  ```
- Retry: 10 attempts, 5s interval
- Response: SSH key object with ID

**GET /ssh_keys**
- Purpose: Find existing SSH key
- Used when: Key already exists

#### 4. Instance Creation (per server)

**POST /servers**
- Purpose: Create compute instance
- Request body (with private network):
  ```json
  {
    "name": "<cluster_name>-master-<location>-<n>",
    "location": "fsn1",
    "image": "ubuntu-22.04",
    "server_type": "cx11",
    "ssh_keys": [<ssh_key_id>],
    "user_data": "<cloud_init_yaml>",
    "labels": {
      "cluster": "<cluster_name>",
      "role": "master"
    },
    "start_after_create": true,
    "public_net": {
      "enable_ipv4": true,
      "enable_ipv6": true
    },
    "networks": [<network_id>]
  }
  ```
- Retry: Infinite with exponential backoff (1s → 2s → 4s → ... → max 60s)
- Random initial delay: 0-3 seconds (to prevent thundering herd)
- Response: Server object with ID and status

**GET /servers/{id}** (polling)
- Purpose: Check instance status
- Called repeatedly every 10s until status = "running"
- Response: Server object with current status

#### 5. Instance Lifecycle Operations (if private network)

**POST /servers/{id}/actions/poweron**
- Purpose: Power on instance that was created in stopped state
- Called when: Instance status != "running" and private network enabled
- Response: Action object

**POST /servers/{id}/actions/attach_to_network**
- Purpose: Attach instance to private network
- Request body:
  ```json
  {
    "network": <network_id>
  }
  ```
- Called when: Private network enabled and instance not yet attached
- Mutex-protected: Only one attachment at a time across all instances
- Response: Action object

#### 6. Load Balancer Creation (if HA cluster)

**POST /load_balancers**
- Purpose: Create load balancer for API server
- Conditions: master_count > 1 AND create_load_balancer enabled
- Request body (with private network):
  ```json
  {
    "algorithm": {
      "type": "round_robin"
    },
    "load_balancer_type": "lb11",
    "location": "<first_master_location>",
    "name": "<cluster_name>-api",
    "network": <network_id>,
    "public_interface": true,
    "services": [
      {
        "destination_port": 6443,
        "listen_port": 6443,
        "protocol": "tcp",
        "proxyprotocol": false
      }
    ],
    "targets": [
      {
        "label_selector": {
          "selector": "cluster=<cluster_name>,role=master"
        },
        "type": "label_selector",
        "use_private_ip": true
      }
    ]
  }
  ```
- Retry: 10 attempts, 5s interval
- Response: Load balancer object with ID

**GET /load_balancers** (polling)
- Purpose: Wait for load balancer public IP assignment
- Called repeatedly every 1s until public_ip_address is set
- Response: Load balancer object with public IP

### Rate Limiting

The API client handles rate limiting (HTTP 429) automatically:
- Detects 429 status code
- Mutex-protected wait (prevents multiple goroutines from waiting separately)
- Waits 1 hour from rate limit time
- Progress updates every 5 seconds
- Retries request after wait completes

### Request/Response Format

All requests:
- Content-Type: `application/json`
- Response format: JSON
- Error handling: Check success field, log response body on failure

## Cloud-Init Process

Cloud-init is generated by `Hetzner::Instance::CloudInitGenerator` and injected via the `user_data` field in the server creation request.

### Template Structure

Base template: `templates/cloud_init.yaml`

```yaml
#cloud-config
preserve_hostname: true

{{ growpart_str }}

write_files:
{{ growroot_disabled_file }}
{{ eth1_str }}
{{ firewall_files }}
{{ ssh_files }}
{{ init_files }}

packages: [{{ packages_str }}]

runcmd:
{{ post_create_commands_str }}
```

### Packages Installed

**Always installed:**
- `fail2ban` - Intrusion prevention
- `wireguard` (or `wireguard-tools` on MicroOS) - VPN support

**Additional packages:**
- Can be specified via `additional_packages` configuration

### Files Written

#### SSH Configuration
Location: `/etc/systemd/system/ssh.socket.d/listen.conf`
- Content: SSH port configuration
- Encoding: gzip+base64

Location: `/etc/configure_ssh.sh`
- Content: SSH setup script
- Permissions: 0755
- Encoding: gzip+base64

#### Firewall Configuration (if public network without private network)
- `/etc/allowed-networks-kubernetes-api.conf` - Allowed API networks
- `/etc/allowed-networks-ssh.conf` - Allowed SSH networks
- `/usr/local/lib/firewall/configure_firewall.sh` - Firewall config script (0755)
- `/usr/local/lib/firewall/firewall_setup.sh` - Setup script (0755)
- `/usr/local/lib/firewall/firewall_status.sh` - Status script (0755)
- `/usr/local/lib/firewall/firewall_updater.sh` - Updater script (0755)
- `/etc/systemd/system/firewall_updater.service` - Systemd service
- `/etc/systemd/system/iptables_restore.service` - Systemd service
- `/etc/systemd/system/ipset_restore.service` - Systemd service
- `/usr/local/lib/firewall/setup_service.sh` - Service setup (0755)

All firewall files are gzip+base64 encoded.

#### OS-Specific Files

**MicroOS only:**
Location: `/etc/sysconfig/network/ifcfg-eth1`
```
BOOTPROTO='dhcp'
STARTMODE='auto'
```

### runcmd Commands (executed in order)

#### 1. Hostname Setup
```bash
hostnamectl set-hostname $(curl http://169.254.169.254/hetzner/v1/metadata/hostname)
```
- Fetches hostname from Hetzner metadata service
- Sets system hostname

#### 2. Crypto Policies
```bash
update-crypto-policies --set DEFAULT:SHA1 || true
```
- Updates system crypto policies
- Allows SHA1 for compatibility
- Non-fatal (|| true)

#### 3. SSH Configuration
```bash
/etc/configure_ssh.sh
```
- Executes SSH setup script written via write_files

#### 4. DNS Configuration
```bash
echo "nameserver 8.8.8.8" > /etc/k8s-resolv.conf
```
- Creates k8s-specific resolv.conf with Google DNS
- Used by k3s for DNS resolution

#### 5. Firewall Setup (if public network)
```bash
/usr/local/lib/firewall/firewall_setup.sh
```
- Only runs if: private_network disabled AND use_local_firewall enabled
- Configures iptables/ipset rules
- Sets up firewall updater service

#### 6. MicroOS-Specific Commands (if snapshot_os = "microos")
```bash
btrfs filesystem resize max /var
sed -i 's/NETCONFIG_DNS_STATIC_SERVERS=\"\"/NETCONFIG_DNS_STATIC_SERVERS=\"1.1.1.1 1.0.0.1\"/g' /etc/sysconfig/network/config
sed -i 's/#SystemMaxUse=/SystemMaxUse=3G/g' /etc/systemd/journald.conf
sed -i 's/#MaxRetentionSec=/MaxRetentionSec=1week/g' /etc/systemd/journald.conf
sed -i 's/NUMBER_LIMIT=\"2-10\"/NUMBER_LIMIT=\"4\"/g' /etc/snapper/configs/root
sed -i 's/NUMBER_LIMIT_IMPORTANT=\"4-10\"/NUMBER_LIMIT_IMPORTANT=\"3\"/g' /etc/snapper/configs/root
sed -i 's/NETCONFIG_NIS_SETDOMAINNAME=\"yes\"/NETCONFIG_NIS_SETDOMAINNAME=\"no\"/g' /etc/sysconfig/network/config
sed -i 's/DHCLIENT_SET_HOSTNAME=\"yes\"/DHCLIENT_SET_HOSTNAME=\"no\"/g' /etc/sysconfig/network/dhcp
```
- Resize /var partition
- Configure DNS
- Limit journald size (3GB, 1 week retention)
- Configure snapper snapshots (4 regular, 3 important)
- Disable NIS domain name and DHCP hostname setting

#### 7. Additional Pre-k3s Commands
- Custom commands from configuration: `additional_pre_k3s_commands`

#### 8. Custom Init Scripts
- Commands from `init_commands` configuration
- Written as `/etc/init-<n>.sh` scripts
- Executed in order

#### 9. Additional Post-k3s Commands
- Custom commands from configuration: `additional_post_k3s_commands`

### Cloud-Init Completion Signal

Location: `/var/lib/cloud/instance/boot-finished`
- Created by cloud-init when all tasks complete
- Contains timestamp
- Polled by hetzner-k3s via `cloud_init_wait_script.sh`

## k3s Installation Details

k3s installation is handled by rendered shell scripts executed via SSH after cloud-init completes.

### Master Node Installation

Script: `templates/master_install_script.sh`

#### 1. Initialization Marker
```bash
touch /etc/initialized
```
- Creates file to track installation status
- Can contain "true" on success

#### 2. Network Detection

**Get Public IP:**
```bash
HOSTNAME=$(hostname -f)
PUBLIC_IP=$(hostname -I | awk '{print $1}')
```

**Private Network Detection (if enabled):**
```bash
if [ "{{ private_network_enabled }}" = "true" ]; then
  SUBNET="{{ private_network_subnet }}"
  MAX_ATTEMPTS=30
  DELAY=10

  for i in $(seq 1 $MAX_ATTEMPTS); do
    NETWORK_INTERFACE=$(
      ip -o link show |
        awk -F': ' '/mtu (1450|1280)/ {print $2}' |
        grep -Ev 'cilium|br|flannel|docker|veth' |
        head -n1
    )

    if [ -n "$NETWORK_INTERFACE" ]; then
      break
    fi

    echo "Waiting for private network interface... (Attempt $i/$MAX_ATTEMPTS)"
    sleep $DELAY
  done
fi
```

**Key Points:**
- **30 attempts** maximum
- **10 second delay** between attempts
- **Total timeout**: 5 minutes (30 × 10s)
- **Interface detection**: Looks for MTU 1450 or 1280 (Hetzner private network signature)
- **Exclusions**: Filters out cilium, bridge, flannel, docker, veth interfaces
- **Logging**: All to `/var/log/hetzner-k3s.log`

**Get Private IP:**
```bash
PRIVATE_IP=$(
  ip -4 -o addr show dev "$NETWORK_INTERFACE" |
    awk '{print $4}' |
    cut -d'/' -f1 |
    head -n1
)
```

**Fallback (public network):**
```bash
else
  PRIVATE_IP="${PUBLIC_IP}"
  NETWORK_INTERFACE=""
fi
```

#### 3. CNI Configuration

**Flannel (if CNI is flannel and private network):**
```bash
if [ "{{ cni }}" = "true" ] && [ "{{ cni_mode }}" = "flannel" ] && [ "{{ private_network_enabled }}" = "true" ] && [ -n "$NETWORK_INTERFACE" ]; then
  FLANNEL_SETTINGS="{{ flannel_backend }} --flannel-iface=$NETWORK_INTERFACE"
else
  FLANNEL_SETTINGS="{{ flannel_backend }}"
fi
```

#### 4. Component Configuration

**Embedded Registry Mirror:**
```bash
if [ "{{ embedded_registry_mirror_enabled }}" = "true" ]; then
  EMBEDDED_REGISTRY_MIRROR="--embedded-registry"
else
  EMBEDDED_REGISTRY_MIRROR=""
fi
```

**Local Path Storage:**
```bash
if [ "{{ local_path_storage_class_enabled }}" = "true" ]; then
  LOCAL_PATH_STORAGE_CLASS=""
else
  LOCAL_PATH_STORAGE_CLASS="--disable local-storage"
fi
```

**Traefik:**
```bash
if [ "{{ traefik_enabled }}" = "true" ]; then
  TRAEFIK_PLUGIN=""
else
  TRAEFIK_PLUGIN="--disable traefik"
fi
```

**ServiceLB:**
```bash
if [ "{{ servicelb_enabled }}" = "true" ]; then
  SERVICELB_PLUGIN=""
else
  SERVICELB_PLUGIN="--disable servicelb"
fi
```

**Metrics Server:**
```bash
if [ "{{ metrics_server_enabled }}" = "true" ]; then
  METRICS_SERVER_PLUGIN=""
else
  METRICS_SERVER_PLUGIN="--disable metrics-server"
fi
```

#### 5. k3s Directory Setup

```bash
mkdir -p /etc/rancher/k3s

cat >/etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "*":
EOF
```

#### 6. Instance ID (for public network)

```bash
KUBELET_INSTANCE_ID=""
if [ "{{ private_network_enabled }}" = "false" ]; then
  INSTANCE_ID=$(curl -s http://169.254.169.254/hetzner/v1/metadata/instance-id)
  if [ -n "$INSTANCE_ID" ]; then
    KUBELET_INSTANCE_ID="--kubelet-arg=provider-id=hcloud://$INSTANCE_ID"
  fi
fi
```
- Fetches instance ID from metadata service
- Used for Hetzner Cloud Controller Manager integration
- Only for public network mode

#### 7. k3s Installation Command

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION="{{ k3s_version }}" \
  K3S_TOKEN="{{ k3s_token }}" \
  {{ datastore_endpoint }} \
  INSTALL_K3S_SKIP_START=false \
  INSTALL_K3S_EXEC="server" \
  sh -s - \
    --disable-cloud-controller \
    $TRAEFIK_PLUGIN \
    $SERVICELB_PLUGIN \
    $METRICS_SERVER_PLUGIN \
    --write-kubeconfig-mode=644 \
    --node-name=$HOSTNAME \
    --cluster-cidr={{ cluster_cidr }} \
    --service-cidr={{ service_cidr }} \
    --cluster-dns={{ cluster_dns }} \
    --kube-controller-manager-arg="bind-address=0.0.0.0" \
    --kube-proxy-arg="metrics-bind-address=0.0.0.0" \
    --kube-scheduler-arg="bind-address=0.0.0.0" \
    {{ master_taint }} {{ labels_and_taints }} {{ extra_args }} {{ etcd_arguments }} \
    $KUBELET_INSTANCE_ID \
    $FLANNEL_SETTINGS \
    $EMBEDDED_REGISTRY_MIRROR \
    $LOCAL_PATH_STORAGE_CLASS \
    --advertise-address=$PRIVATE_IP \
    --node-ip=$PRIVATE_IP \
    --node-external-ip=$PUBLIC_IP \
    {{ server }} {{ tls_sans }} 2>&1 | tee -a /var/log/hetzner-k3s.log
```

**Key Flags:**
- `--disable-cloud-controller`: Disabled for Hetzner CCM
- `--write-kubeconfig-mode=644`: Make kubeconfig readable
- `--node-name`: FQDN hostname
- `--cluster-cidr`: Pod CIDR (default: 10.244.0.0/16)
- `--service-cidr`: Service CIDR (default: 10.43.0.0/16)
- `--cluster-dns`: DNS server IP (default: 10.43.0.10)
- Component bind addresses: 0.0.0.0 for metrics exposure
- `--advertise-address`: Private IP (for etcd/API)
- `--node-ip`: Private IP (for kubelet)
- `--node-external-ip`: Public IP (for external access)
- `{{ server }}`: Join URL for additional masters (first master omits this)
- `{{ tls_sans }}`: Additional SANs for API cert (includes LB IP)

**Installation Source:**
- `https://get.k3s.io` - Official k3s installation script
- Downloads k3s binary
- Installs systemd service
- Starts k3s-server immediately

#### 8. Success Verification

```bash
if [ ${PIPESTATUS[0]} -ne 0 ]; then
  echo "ERROR: k3s installation failed"
  exit 1
fi

echo "k3s installation completed successfully"
echo true >/etc/initialized
```

### Worker Node Installation

Script: `templates/worker_install_script.sh`

Very similar to master, with these key differences:

#### Network Detection
Same 30 attempts × 10s delay logic as master

#### k3s Installation Command

```bash
curl -sfL https://get.k3s.io | \
  K3S_TOKEN="{{ k3s_token }}" \
  INSTALL_K3S_VERSION="{{ k3s_version }}" \
  K3S_URL=https://{{ api_server_ip_address }}:6443 \
  INSTALL_K3S_EXEC="agent" \
  sh -s - \
    --node-name=$HOSTNAME \
    {{ extra_args }} {{ labels_and_taints }} \
    --node-ip=$PRIVATE_IP \
    --node-external-ip=$PUBLIC_IP \
    $KUBELET_INSTANCE_ID \
    $FLANNEL_SETTINGS 2>&1 | tee -a /var/log/hetzner-k3s.log
```

**Key Differences:**
- `K3S_URL`: Points to first master or load balancer
- `INSTALL_K3S_EXEC="agent"`: Worker mode instead of server
- No cluster/service CIDR, DNS, or component settings
- Simpler configuration focused on joining cluster

## Timing and Orchestration

### Concurrency Model

The provisioning process uses Crystal fibers (green threads) and channels for concurrency:

```
Channel: masters_installation_queue (capacity: 5)
Channel: workers_installation_queue (capacity: 10)
Channel: completed_channel (capacity: 1)
Semaphore: Instance creation (capacity: 10)
Semaphore: Worker installation (capacity: 10)
Mutex: Network attachment (prevents concurrent attachments)
```

### Execution Timeline

```
Time    Masters                         Workers                         Control Plane
────────────────────────────────────────────────────────────────────────────────────
T+0s    Create 3 masters (concurrent)   -                               -
        ├─ master-1 POST /servers
        ├─ master-2 POST /servers
        └─ master-3 POST /servers

T+15s   Wait for instances ready        -                               -
        ├─ Poll status
        ├─ Attach to network
        └─ Wait for SSH

T+30s   All masters created             -                               -
        Send to queue →

T+31s   -                               Create workers (concurrent)     Install first master
                                        ├─ worker-1 POST /servers       ├─ Wait cloud-init (5min)
                                        ├─ worker-2 POST /servers       ├─ Run install script
                                        └─ worker-3 POST /servers       ├─ Wait 10s
                                                                        └─ Save kubeconfig

T+45s   Install additional masters      Wait for instances ready        Validate control plane
        ├─ master-2 (concurrent)        ├─ Poll status                  ├─ kubectl cluster-info
        └─ master-3 (concurrent)        └─ Wait for SSH                 └─ Retry 3×30s

T+60s   -                               All workers created             Additional masters joining
                                        Send to queue →                 ├─ master-2 script
                                                                        └─ master-3 script

T+75s   -                               Install workers (concurrent)    Masters fully joined
                                        ├─ worker-1 script
                                        ├─ worker-2 script
                                        └─ worker-3 script

T+90s   -                               Workers joining                 Install Hetzner CCM
                                                                        Install Hetzner CSI

T+120s  -                               Wait for 1 worker Ready         Install Cilium (optional)
                                        kubectl get nodes                Install autoscaler

T+150s  -                               -                               Completed!
                                                                        Signal done
```

### Wait Mechanisms

#### SSH Readiness Wait
Location: `Util::SSH.wait_for_instance`
- **Default attempts**: 20
- **Retry delay**: 5 seconds
- **Command timeout**: 30 seconds
- **Total timeout**: ~100 seconds (20 × 5s)
- **Test command**: `echo ready`
- **Expected result**: `ready`

Retry logic:
```crystal
Retriable.retry(
  max_attempts: 20,
  on: [Tasker::Timeout, IO::Error],
  backoff: false,
  sleep_timer: 5.seconds
) do
  Tasker.timeout(30.seconds) do
    result = run(instance, port, test_command, use_ssh_agent, false)
    # Must match expected_result exactly
  end
end
```

#### Cloud-Init Wait
Script: `templates/cloud_init_wait_script.sh`
- **Max wait**: 300 seconds (5 minutes)
- **Check interval**: 1 second
- **Progress dots**: Every 10 seconds
- **File checked**: `/var/lib/cloud/instance/boot-finished`

#### Control Plane Ready Wait
Location: `Kubernetes::ControlPlane::Setup.wait_for_control_plane`
- **Retries**: 3 attempts
- **Timeout per attempt**: 30 seconds
- **Check interval**: 1 second
- **Command**: `kubectl cluster-info`
- **Expected**: Output contains "running"

#### Worker Ready Wait
Location: `Kubernetes::Worker::Setup.wait_for_one_worker_to_be_ready`
- **Timeout**: 5 minutes
- **Check interval**: 5 seconds
- **Command**: `kubectl get nodes`
- **Expected**: At least one line with "worker" AND "Ready"

### Parallel vs Sequential Operations

**Parallel (concurrent):**
- Instance creation (10 concurrent max via semaphore)
- Master k3s installation (after first master, all others concurrent)
- Worker instance creation (10 concurrent max)
- Worker k3s installation (10 concurrent max)

**Sequential:**
- Network creation → SSH key → Instance creation
- First master install → Additional masters install
- Control plane validation → Software installation
- Must complete within same operation (e.g., cloud-init wait)

**Mixed:**
- Instance creation starts parallel while k3s setup is spawned
- Workers created immediately after k3s setup spawned (don't wait for masters to finish installing k3s)

## Network Detection Logic

The network interface detection is critical for private network operation.

### Detection Algorithm

```bash
NETWORK_INTERFACE=$(
  ip -o link show |
    awk -F': ' '/mtu (1450|1280)/ {print $2}' |
    grep -Ev 'cilium|br|flannel|docker|veth' |
    head -n1
)
```

**How it works:**
1. `ip -o link show` - List all network interfaces (one per line)
2. `awk -F': ' '/mtu (1450|1280)/ {print $2}'` - Extract interface name where MTU is 1450 or 1280
   - Hetzner private networks use MTU 1450 or 1280 (vs 1500 for public)
3. `grep -Ev 'cilium|br|flannel|docker|veth'` - Exclude virtual interfaces
   - Prevents selecting CNI interfaces
   - Prevents selecting Docker bridges
   - Prevents selecting veth pairs
4. `head -n1` - Take first match

### Why MTU 1450/1280?

Hetzner Cloud private networks use:
- **MTU 1450**: Standard private network MTU (accounts for VPN/tunnel overhead)
- **MTU 1280**: IPv6 minimum MTU, used in some configurations

Public interfaces use MTU 1500.

### Retry Loop

```bash
MAX_ATTEMPTS=30
DELAY=10

for i in $(seq 1 $MAX_ATTEMPTS); do
  # ... detection logic ...

  if [ -n "$NETWORK_INTERFACE" ]; then
    echo "Private network interface $NETWORK_INTERFACE found"
    break
  fi

  echo "Waiting for private network interface... (Attempt $i/$MAX_ATTEMPTS)"
  sleep $DELAY
done

if [ -z "$NETWORK_INTERFACE" ]; then
  echo "ERROR: Timeout waiting for private network interface"
  exit 1
fi
```

**Timeline:**
- Attempt 1: Immediate
- Attempt 2: 10s
- Attempt 3: 20s
- ...
- Attempt 30: 290s
- **Total possible wait**: 290 seconds (~5 minutes)

**Why needed:**
- Network attachment via Hetzner API is async
- Interface may not appear immediately after `attach_to_network` returns
- Cloud-init network configuration may still be running
- DHCP IP assignment takes time

### IP Address Extraction

Once interface is found:

```bash
PRIVATE_IP=$(
  ip -4 -o addr show dev "$NETWORK_INTERFACE" |
    awk '{print $4}' |
    cut -d'/' -f1 |
    head -n1
)
```

1. `ip -4 -o addr show dev "$NETWORK_INTERFACE"` - Show IPv4 addresses for interface
2. `awk '{print $4}'` - Extract CIDR notation (e.g., 10.0.0.2/16)
3. `cut -d'/' -f1` - Remove netmask, keep IP only
4. `head -n1` - Take first address

**Validation:**
```bash
if [ -z "$PRIVATE_IP" ]; then
  echo "ERROR: Could not determine private IP address"
  exit 1
fi
```

## Error Handling and Retries

### Instance Creation Retry

```crystal
loop do
  attempts += 1
  log_line "Creating instance #{instance_name} (attempt #{attempts})..."

  sleep rand(3001).milliseconds  # Random jitter: 0-3s

  success, response = hetzner_client.post("/servers", instance_config)

  if success
    break
  else
    log_line "Creating instance failed: #{response}"
    delay = [INITIAL_DELAY * (2 ** (attempts - 1)), MAX_DELAY].min
    log_line "Waiting #{delay} seconds before retry..."
    sleep delay.seconds
  end
end
```

**Backoff:**
- Attempt 1: 1s
- Attempt 2: 2s
- Attempt 3: 4s
- Attempt 4: 8s
- Attempt 5: 16s
- Attempt 6: 32s
- Attempt 7+: 60s (capped)

**Random jitter**: 0-3 seconds before each attempt prevents thundering herd

### Network/SSH Key Creation Retry

```crystal
Retriable.retry(max_attempts: 10, backoff: false, base_interval: 5.seconds) do
  success, response = hetzner_client.post("/networks", config)

  unless success
    STDERR.puts "Failed to create resource: #{response}"
    STDERR.puts "Retrying in 5 seconds..."
    raise "Failed to create resource"
  end
end
```

**Fixed interval**: 5 seconds between attempts
**Max attempts**: 10
**Total timeout**: 45 seconds (9 × 5s delays)

### Load Balancer Creation Retry

Same as network/SSH key: 10 attempts, 5s interval

### SSH Command Retry

```crystal
Retriable.retry(
  max_attempts: 20,
  on: [Tasker::Timeout, IO::Error],
  backoff: false,
  sleep_timer: 5.seconds
) do
  Tasker.timeout(30.seconds) do
    result = run(instance, port, command, use_agent, false)
    unless result == expected
      raise IO::Error.new("Result mismatch")
    end
  end
end
```

**Max attempts**: 20
**Delay**: 5 seconds
**Command timeout**: 30 seconds per attempt
**Exceptions caught**: Tasker::Timeout, IO::Error

### Control Plane Validation Retry

```crystal
Retriable.retry(max_attempts: 3, on: Tasker::Timeout, backoff: false) do
  Tasker.timeout(30.seconds) do
    loop do
      result = run_shell_command("kubectl cluster-info", ...)
      break if result.output.includes?("running")
      sleep 1.seconds
    end
  end
end
```

**Outer retry**: 3 attempts
**Timeout**: 30 seconds per attempt
**Check interval**: 1 second
**Total possible wait**: 90 seconds (3 × 30s)

### Rate Limit Handling

```crystal
while true
  response = yield

  if response.status_code == 429
    mutex.synchronize do
      handle_rate_limit(response)  # Wait 1 hour
    end
  else
    return response
  end
end
```

**Wait time**: 3600 seconds (1 hour)
**Progress updates**: Every 5 seconds
**Mutex protected**: Only one fiber waits, others block on mutex

## Summary

The provisioning workflow orchestrates:
- **10+ Hetzner API endpoints** for infrastructure creation
- **Cloud-init injection** with gzip+base64 encoded files and commands
- **Network detection** with 30 attempts × 10s delay (5 min timeout)
- **k3s installation** via get.k3s.io with extensive configuration
- **Concurrent execution** of up to 10 instances and 10 worker installations
- **Comprehensive retry logic** with exponential backoff and fixed delays
- **Multiple wait mechanisms** for SSH, cloud-init, control plane, and workers

**Result**: A production-ready k3s cluster typically provisioned in 2-3 minutes.
