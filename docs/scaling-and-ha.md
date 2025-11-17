# Scaling and High Availability

This guide covers scaling strategies, high availability configurations, and the upgrade process for hetzner-k3s clusters.

## Table of Contents

- [Scaling Approaches](#scaling-approaches)
- [High Availability Setup](#high-availability-setup)
- [Cluster Autoscaler](#cluster-autoscaler)
- [Upgrade Process](#upgrade-process)
- [Practical Scaling Path](#practical-scaling-path)

---

## Scaling Approaches

hetzner-k3s supports both vertical and horizontal scaling strategies, allowing you to grow your cluster as your needs evolve.

### Vertical Scaling

Vertical scaling involves upgrading to larger instance types with more resources (CPU, RAM, storage). Hetzner offers various instance families:

**Shared vCPU (CPX series - Cost-Effective)**
- `cpx11`: 2 vCPU, 2 GB RAM
- `cpx21`: 3 vCPU, 4 GB RAM
- `cpx31`: 4 vCPU, 8 GB RAM
- `cpx41`: 8 vCPU, 16 GB RAM
- `cpx51`: 16 vCPU, 32 GB RAM

**Dedicated vCPU (CCX series - Performance)**
- `ccx13`: 2 vCPU, 8 GB RAM
- `ccx23`: 4 vCPU, 16 GB RAM
- `ccx33`: 8 vCPU, 32 GB RAM
- `ccx43`: 16 vCPU, 64 GB RAM
- `ccx53`: 32 vCPU, 128 GB RAM

**ARM-based (CAX series - Efficiency)**
- `cax11`: 2 vCPU, 4 GB RAM
- `cax21`: 4 vCPU, 8 GB RAM
- `cax31`: 8 vCPU, 16 GB RAM
- `cax41`: 16 vCPU, 32 GB RAM

#### How to Vertically Scale

**For Master Nodes:**

1. Update your configuration file:
```yaml
masters_pool:
  instance_type: cpx31  # Changed from cpx21
  instance_count: 3
  locations:
    - fsn1
    - hel1
    - nbg1
```

2. Follow these steps for each master:
   - Drain the master: `kubectl drain <master-name> --ignore-daemonsets`
   - Delete the node: `kubectl delete node <master-name>`
   - Remove the instance from Hetzner Console or using `hcloud` CLI
   - Run `hetzner-k3s create --config cluster_config.yaml`
   - Verify the new master joins: `kubectl get nodes`
   - Repeat for the next master

**For Worker Nodes:**

1. Update the worker pool configuration:
```yaml
worker_node_pools:
- name: workers
  instance_type: cpx31  # Changed from cpx21
  instance_count: 3
  location: hel1
```

2. For each worker:
   - Drain the worker: `kubectl drain <worker-name> --ignore-daemonsets --delete-emptydir-data`
   - Delete the node: `kubectl delete node <worker-name>`
   - Remove the instance from Hetzner Console
   - Run `hetzner-k3s create --config cluster_config.yaml`
   - Verify the new worker joins and workloads are rescheduled

### Horizontal Scaling

Horizontal scaling involves adding more nodes to distribute workload across the cluster.

#### Scaling Masters (1 → 3 → 5)

**Converting Single Master to HA (3 masters):**

```yaml
# Before: Single master (non-HA)
masters_pool:
  instance_type: cpx21
  instance_count: 1
  locations:
    - fsn1

# After: High Availability (3 masters)
masters_pool:
  instance_type: cpx21
  instance_count: 3
  locations:
    - fsn1
    - hel1
    - nbg1
```

Run the create command:
```bash
hetzner-k3s create --config cluster_config.yaml
```

This will:
- Create `master2` in Helsinki
- Create `master3` in Nuremberg
- Set up etcd clustering across all three masters
- Update kubeconfig to use the API load balancer (if enabled)

**Best Practices for Master Scaling:**
- Always use an odd number of masters (1, 3, 5) to prevent split-brain scenarios
- For production: minimum 3 masters
- Place masters in different locations for regional redundancy
- Consider enabling `create_load_balancer_for_the_kubernetes_api: true` for HA clusters

#### Scaling Workers

Adding workers is straightforward and non-disruptive:

```yaml
worker_node_pools:
- name: general-purpose
  instance_type: cpx31
  instance_count: 5  # Increased from 3
  location: hel1
```

```bash
hetzner-k3s create --config cluster_config.yaml
```

The tool is idempotent and will only create the missing nodes.

**Scaling Down Workers:**

1. Update the instance count in your configuration file
2. Drain the nodes to be removed:
   ```bash
   kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
   ```
3. Delete the nodes from Kubernetes:
   ```bash
   kubectl delete node <node-name>
   ```
4. Remove the instances from Hetzner Console

### Can You Scale Without Cluster Recreation?

**Yes!** The `create` command is idempotent and designed for incremental changes:

- **Adding nodes**: Update instance count and run `create` - only new nodes are provisioned
- **Removing nodes**: Manually drain/delete nodes, then remove instances
- **Changing instance types**: Requires node replacement (drain, delete, recreate)
- **Adding node pools**: Simply add to configuration and run `create`

You do NOT need to destroy and recreate the entire cluster for these operations.

---

## High Availability Setup

High availability ensures your cluster continues operating even when individual components fail.

### Multi-Master Configuration

A highly available control plane requires **3 or more master nodes** (always an odd number).

```yaml
masters_pool:
  instance_type: cpx21
  instance_count: 3  # HA configuration
  locations:
    - fsn1   # Falkenstein, Germany
    - hel1   # Helsinki, Finland
    - nbg1   # Nuremberg, Germany
```

**Why 3 Masters?**
- etcd requires a quorum (majority) for consensus
- 3 masters can tolerate 1 failure: (3-1)/2 = 1
- 5 masters can tolerate 2 failures: (5-1)/2 = 2
- Split-brain prevention requires an odd number

**Architecture Diagram:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer (Optional)                  │
│              Distributes API requests to masters             │
└──────────────┬──────────────┬──────────────┬─────────────────┘
               │              │              │
       ┌───────▼──────┐ ┌────▼──────┐ ┌─────▼──────┐
       │  Master 1    │ │ Master 2  │ │  Master 3  │
       │  (fsn1)      │ │  (hel1)   │ │   (nbg1)   │
       │              │ │           │ │            │
       │ - API Server │ │- API Srv  │ │- API Srv   │
       │ - etcd       │ │- etcd     │ │- etcd      │
       │ - Scheduler  │ │- Scheduler│ │- Scheduler │
       │ - Controller │ │- Ctrl Mgr │ │- Ctrl Mgr  │
       └──────────────┘ └───────────┘ └────────────┘
                 ▲           ▲           ▲
                 │   etcd    │   etcd    │
                 └───────────┴───────────┘
                   Raft Consensus
```

### Embedded etcd Setup

hetzner-k3s uses **embedded etcd** as the default datastore mode, which provides:

- **High availability**: Built-in replication across masters
- **Automatic backups**: Configurable snapshot schedule
- **S3 integration**: Optional offsite backup storage
- **No external dependencies**: etcd runs within k3s

#### Basic etcd Configuration

```yaml
datastore:
  mode: etcd  # Default mode
```

#### Advanced etcd Configuration

```yaml
datastore:
  mode: etcd
  etcd:
    # Snapshot configuration
    snapshot_retention: 24      # Keep 24 snapshots
    snapshot_schedule_cron: "0 * * * *"  # Hourly snapshots

    # S3 backup configuration (optional)
    s3_enabled: true
    s3_endpoint: "s3.amazonaws.com"
    s3_region: "us-east-1"
    s3_bucket: "my-k3s-backups"
    s3_access_key: "AKIAIOSFODNN7EXAMPLE"
    s3_secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
    s3_folder: "cluster-backups/"
```

**etcd Performance Considerations:**

For regional clusters (masters in different locations):
- Default heartbeat interval: 100ms
- Latency between EU locations: 25-27ms
- Round-trip time (RTT): ~60-70ms
- **Result**: Well within etcd's acceptable limits

No etcd configuration changes are needed for regional clusters.

#### External Datastore Option

For very large clusters (>200 nodes), consider using PostgreSQL or MySQL:

```yaml
datastore:
  mode: external
  external_datastore_endpoint: "postgres://user:pass@host:5432/k3s"
```

Benefits:
- Better scalability for large clusters
- Managed database services available
- Separate backup/recovery strategies

### Load Balancer for API High Availability

When running multiple masters, enable an API load balancer for seamless failover:

```yaml
create_load_balancer_for_the_kubernetes_api: true
```

**How it works:**

1. A Hetzner Load Balancer is created automatically
2. All masters are registered as targets (port 6443)
3. Health checks ensure only healthy masters receive traffic
4. Kubeconfig is updated to point to the load balancer
5. API requests are distributed across available masters

**Without Load Balancer:**
- Kubeconfig includes contexts for each master
- Manual switching required if one master fails
- Less seamless failover experience

**With Load Balancer:**
- Single API endpoint for all requests
- Automatic failover if a master becomes unavailable
- Better for production environments

### Multi-Location Master Distribution

Distributing masters across multiple locations provides geographical redundancy.

#### Regional Cluster (EU Network Zone)

```yaml
masters_pool:
  instance_type: cpx21
  instance_count: 3
  locations:
    - fsn1  # Falkenstein, Germany
    - hel1  # Helsinki, Finland
    - nbg1  # Nuremberg, Germany
```

**Benefits:**
- Survives datacenter outage
- Lower latency for geographically distributed teams
- Complies with data residency requirements

**Limitations:**
- Only available in `eu-central` network zone
- Limited to 3 masters (only 3 locations available)
- Other regions (US, Singapore) support zonal clusters only

#### Zonal Cluster (Single Location)

For other regions or simplified setup:

```yaml
masters_pool:
  instance_type: cpx21
  instance_count: 3
  locations:
    - nbg1  # All masters in same location
```

**Benefits:**
- Lower inter-master latency
- Simpler network configuration
- Available in all regions

**Trade-offs:**
- Single point of failure at datacenter level
- Not protected against regional outages

#### Converting Zonal to Regional Cluster

See the detailed guide in [Masters in Different Locations](Masters_in_different_locations.md) for step-by-step instructions on migrating an existing cluster to a regional configuration.

---

## Cluster Autoscaler

The Cluster Autoscaler automatically adjusts the number of worker nodes based on resource demands.

### How It Works with Hetzner Provider

```
┌─────────────────────────────────────────────────────────────┐
│                   Kubernetes Scheduler                       │
│  Detects: Pending pods that cannot be scheduled             │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Cluster Autoscaler (Pod)                        │
│  - Monitors pending pods and node utilization                │
│  - Calculates required capacity                              │
│  - Calls Hetzner Cloud API                                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Hetzner Cloud API                           │
│  - Creates new server instances                              │
│  - Applies cloud-init configuration                          │
│  - Configures networking and firewall                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              New Worker Node                                 │
│  - Installs k3s automatically                                │
│  - Joins cluster with correct labels/taints                  │
│  - Becomes ready to accept workloads                         │
└─────────────────────────────────────────────────────────────┘
```

**Key Features:**
- Automatically provisions nodes when pods are pending
- Scales down underutilized nodes to save costs
- Respects node pool min/max constraints
- Preserves node labels and taints
- Uses cloud-init for zero-touch provisioning

### Node Pool Configuration

Define autoscaling pools in your configuration:

```yaml
worker_node_pools:
# Static pool (no autoscaling)
- name: critical-workloads
  instance_type: cpx31
  instance_count: 3
  location: hel1
  labels:
    - key: workload-type
      value: critical

# Autoscaled pool
- name: batch-processing
  instance_type: cpx41
  location: fsn1
  autoscaling:
    enabled: true
    min_instances: 1      # Always keep at least 1 node
    max_instances: 10     # Never exceed 10 nodes
  labels:
    - key: workload-type
      value: batch
  taints:
    - key: batch-only
      value: "true:NoSchedule"

# GPU autoscaled pool
- name: ml-workloads
  instance_type: ccx33
  location: nbg1
  autoscaling:
    enabled: true
    min_instances: 0      # Can scale to zero
    max_instances: 5
  labels:
    - key: workload-type
      value: machine-learning
```

### Min/Max Instances

The autoscaler respects the boundaries you define:

**min_instances:**
- Minimum number of nodes to maintain
- Set to `0` to allow complete scale-down (cost savings)
- Set to `1+` to ensure baseline capacity

**max_instances:**
- Maximum number of nodes to provision
- Prevents runaway scaling and unexpected costs
- Consider your budget and workload requirements

**Example Scenarios:**

```yaml
# Development: Cost optimization, can scale to zero
autoscaling:
  enabled: true
  min_instances: 0
  max_instances: 3

# Production: Always-on baseline with burst capacity
autoscaling:
  enabled: true
  min_instances: 3
  max_instances: 20

# Batch Processing: Large burst capacity
autoscaling:
  enabled: true
  min_instances: 1
  max_instances: 50
```

### Autoscaler Timing Configuration

Fine-tune the autoscaler's behavior with these parameters:

```yaml
cluster_autoscaler:
  scan_interval: "10s"                        # How often to check for scaling needs
  scale_down_delay_after_add: "10m"           # Wait after scaling up before scaling down
  scale_down_delay_after_delete: "10s"        # Wait after node deletion
  scale_down_delay_after_failure: "3m"        # Wait after a failed scale-down
  max_node_provision_time: "15m"              # Timeout for node provisioning
```

**Parameter Descriptions:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `scan_interval` | `10s` | Frequency of scaling evaluations. Shorter = faster response, more API calls. |
| `scale_down_delay_after_add` | `10m` | Prevents immediate scale-down after adding nodes (workloads may still be starting). |
| `scale_down_delay_after_delete` | `10s` | Stabilization period after removing a node. |
| `scale_down_delay_after_failure` | `3m` | Backoff period after a failed scale-down attempt. |
| `max_node_provision_time` | `15m` | Maximum time to wait for a new node to become ready. |

**Tuning Recommendations:**

- **Aggressive scaling** (development): Shorter intervals, shorter delays
- **Conservative scaling** (production): Longer delays to prevent thrashing
- **Cost optimization**: Longer `scale_down_delay_after_add` to avoid unnecessary churn
- **Large clusters**: Increase `max_node_provision_time` due to network setup overhead

### Cloud-Init Templates for Auto-Provisioned Nodes

The autoscaler automatically generates cloud-init configurations for new nodes, ensuring they:

1. **Match cluster configuration**: Same k3s version, packages, and settings
2. **Apply labels and taints**: From node pool definition
3. **Join the cluster**: Automatically connect to masters
4. **Configure networking**: Private network, firewall rules, etc.

**Cloud-init generation** (`src/kubernetes/software/cluster_autoscaler.cr`):

```crystal
# Generates worker install script
worker_install_script = ::Kubernetes::Script::WorkerGenerator.new(
  configuration,
  settings
).generate_script(masters, first_master, pool)

# Creates cloud-init with:
# - SSH configuration
# - Additional packages
# - Pre-k3s commands
# - k3s installation script
# - Post-k3s commands
cloud_init = ::Hetzner::Instance::Create.cloud_init(
  settings,
  ssh_port,
  snapshot_os,
  additional_packages,
  additional_pre_k3s_commands,
  additional_post_k3s_commands,
  [worker_install_script]
)
```

**Node configuration passed to autoscaler:**

```yaml
# Encoded in HCLOUD_CLUSTER_CONFIG environment variable
{
  "imagesForArch": {
    "arm64": "ubuntu-24.04",
    "amd64": "ubuntu-24.04"
  },
  "nodeConfigs": {
    "cluster-name-batch-processing": {
      "cloudInit": "<base64-encoded-cloud-init>",
      "labels": {
        "workload-type": "batch"
      },
      "taints": [
        {
          "key": "batch-only",
          "value": "true",
          "effect": "NoSchedule"
        }
      ]
    }
  }
}
```

**Important Notes:**

- Autoscaling only works with default images (Ubuntu, Debian, etc.)
- Snapshots are not currently supported for autoscaled nodes
- Different autoscaling pools can use different images via `autoscaling_image` setting
- Cloud-init ensures consistent node configuration across manual and auto-provisioned nodes

---

## Upgrade Process

hetzner-k3s uses the Rancher System Upgrade Controller for automated, rolling upgrades.

### k3s Version Upgrades via System Upgrade Controller

The System Upgrade Controller (SUC) manages k3s version upgrades using Custom Resource Definitions (CRDs).

**Upgrade Architecture:**

```
┌──────────────────────────────────────────────────────────────┐
│  hetzner-k3s upgrade --config config.yaml --new-k3s-version  │
│                    v1.30.3+k3s1                               │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│         Creates Upgrade Plans in system-upgrade ns           │
│  - Plan: k3s-server (for masters)                            │
│  - Plan: k3s-agent (for workers)                             │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│       System Upgrade Controller (watches Plans)              │
└────────────────────────┬─────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         ▼                               ▼
┌─────────────────────┐         ┌──────────────────┐
│  Upgrade Masters    │         │  Upgrade Workers │
│  (concurrency: 1)   │         │  (concurrency: N)│
│                     │         │                  │
│  1. Cordon node     │         │  1. Cordon node  │
│  2. Run upgrade job │         │  2. Drain node   │
│  3. Uncordon node   │         │  3. Run upgrade  │
│  4. Next master     │         │  4. Uncordon     │
└─────────────────────┘         └──────────────────┘
```

### Rolling Upgrade Strategy

Upgrades are performed in phases to maintain cluster availability:

**Phase 1: Master Upgrades (Serial)**
```
Master 1: Upgrade → Verify → Complete
           ↓
Master 2: Upgrade → Verify → Complete
           ↓
Master 3: Upgrade → Verify → Complete
```

**Phase 2: Worker Upgrades (Parallel with Concurrency Control)**
```
Worker 1: Upgrade → Complete ─┐
Worker 2: Upgrade → Complete  ├─ Concurrency: N
Worker 3: Upgrade → Complete ─┘
           ↓
Worker 4: Upgrade → Complete
  ... (and so on)
```

**Safety Mechanisms:**

1. **Cordon**: Prevents new pods from scheduling on upgrading nodes
2. **Drain** (workers only): Gracefully evicts pods before upgrade
3. **Health checks**: Verifies node readiness before proceeding
4. **Rollback capability**: Manual rollback possible if issues occur

### Concurrency Control

Control how many nodes upgrade simultaneously:

#### Masters: Always 1 (Fixed)

```yaml
# templates/upgrade_plan_for_masters.yaml
spec:
  concurrency: 1  # Fixed: one master at a time
  version: v1.30.3+k3s1
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/master, operator: In, values: ["true"]}
```

**Why concurrency=1 for masters?**
- Maintains quorum during upgrades
- Prevents API server disruption
- Ensures etcd cluster stability

#### Workers: Configurable

```yaml
# In your configuration file
k3s_upgrade_concurrency: 2  # Upgrade 2 workers simultaneously

# templates/upgrade_plan_for_workers.yaml
spec:
  concurrency: {{ worker_upgrade_concurrency }}
  version: v1.30.3+k3s1
```

**Choosing Worker Concurrency:**

| Cluster Size | Recommended Concurrency | Rationale |
|--------------|-------------------------|-----------|
| 1-5 workers  | 1 | Minimize disruption |
| 5-20 workers | 2-3 | Balance speed and stability |
| 20-50 workers | 3-5 | Faster upgrades with safety buffer |
| 50+ workers | 5-10 | Significant speedup, ensure adequate capacity |

**Factors to Consider:**
- Available cluster capacity (can it handle drained pods?)
- Workload disruption tolerance (stateful vs stateless)
- Upgrade urgency (critical security patch vs routine update)

### Upgrade Plans (Rancher System Upgrade Controller)

Upgrade Plans are Custom Resources that define how upgrades are executed.

#### Master Upgrade Plan

```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: k3s-server
  namespace: system-upgrade
  labels:
    k3s-upgrade: server
spec:
  concurrency: 1
  version: v1.30.3+k3s1
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/master, operator: In, values: ["true"]}
  serviceAccountName: system-upgrade
  tolerations:
  - key: "CriticalAddonsOnly"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
  cordon: true
  upgrade:
    image: rancher/k3s-upgrade
```

**Key Elements:**
- `concurrency: 1`: One master at a time
- `nodeSelector`: Targets only master nodes
- `tolerations`: Allows upgrades on tainted masters
- `cordon: true`: Prevents pod scheduling during upgrade
- `upgrade.image`: Container that performs the upgrade

#### Worker Upgrade Plan

```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: k3s-agent
  namespace: system-upgrade
  labels:
    k3s-upgrade: agent
spec:
  concurrency: {{ worker_upgrade_concurrency }}
  version: v1.30.3+k3s1
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/master, operator: NotIn, values: ["true"]}
  serviceAccountName: system-upgrade
  tolerations:
  - {key: '', effect: NoSchedule, operator: Exists, value: ''}
  prepare:
    image: rancher/k3s-upgrade
    args: ["prepare", "k3s-server"]  # Wait for masters to complete
  cordon: true
  upgrade:
    image: rancher/k3s-upgrade
```

**Key Differences from Master Plan:**
- Configurable concurrency
- Targets non-master nodes
- `prepare` step: Waits for master upgrades to complete
- More permissive tolerations for diverse workloads

### Performing an Upgrade

**Step 1: Check Available Releases**

```bash
hetzner-k3s releases
```

**Step 2: Initiate the Upgrade**

```bash
hetzner-k3s upgrade --config cluster_config.yaml --new-k3s-version v1.30.3+k3s1
```

This command:
- Creates upgrade Plans in `system-upgrade` namespace
- Updates `k3s_version` in your configuration file
- Displays monitoring instructions

**Step 3: Monitor the Upgrade**

```bash
# Watch node versions
watch kubectl get nodes -owide

# Monitor upgrade jobs
watch kubectl get jobs,pods -n system-upgrade

# Check Plan status
kubectl get plan -n system-upgrade -o wide

# View upgrade logs
kubectl logs -n system-upgrade -f job/<upgrade-job-name>
```

**Step 4: Verify Completion**

```bash
# All nodes should show new version
kubectl get nodes -o wide

# All upgrade jobs completed
kubectl get jobs -n system-upgrade

# No failed jobs
kubectl get jobs -n system-upgrade --field-selector status.failed=1
```

**Step 5: Update Cluster State (CRITICAL)**

```bash
hetzner-k3s create --config cluster_config.yaml
```

**Why this step is essential:**
- Updates master node configurations
- Ensures new nodes provision with correct k3s version
- Synchronizes cluster state with configuration
- Prevents version mismatches for future nodes

### Troubleshooting Upgrades

**Stalled Upgrades:**

```bash
# Clean up upgrade resources
kubectl -n system-upgrade delete job --all
kubectl -n system-upgrade delete plan --all

# Remove upgrade labels
kubectl label node --all plan.upgrade.cattle.io/k3s-server- plan.upgrade.cattle.io/k3s-agent-

# Restart upgrade controller
kubectl -n system-upgrade rollout restart deployment system-upgrade-controller
kubectl -n system-upgrade rollout status deployment system-upgrade-controller
```

**Masters Upgraded, Workers Stuck:**

```bash
# Mark masters as upgraded
kubectl label node <master1> <master2> <master3> plan.upgrade.cattle.io/k3s-server=upgraded
```

**Check Controller Logs:**

```bash
kubectl -n system-upgrade logs -f \
  $(kubectl -n system-upgrade get pod -l pod-template-hash -o jsonpath="{.items[0].metadata.name}")
```

---

## Practical Scaling Path

This section provides a step-by-step guide for scaling from a minimal cluster to a production-ready infrastructure.

### Starting Point: Single Node (Development)

**Use Case:** Development, testing, learning Kubernetes

```yaml
hetzner_token: <your token>
cluster_name: dev-cluster
kubeconfig_path: "./kubeconfig"
k3s_version: v1.30.3+k3s1

networking:
  ssh:
    port: 22
    public_key_path: "~/.ssh/id_ed25519.pub"
    private_key_path: "~/.ssh/id_ed25519"
  allowed_networks:
    ssh:
      - 0.0.0.0/0
    api:
      - 0.0.0.0/0
  private_network:
    enabled: true
    subnet: 10.0.0.0/16

masters_pool:
  instance_type: cax11  # 2 vCPU, 4 GB RAM, ~€4/month
  instance_count: 1     # Single master (non-HA)
  locations:
    - fsn1

schedule_workloads_on_masters: true  # Run workloads on the single master

create_load_balancer_for_the_kubernetes_api: false  # Not needed for single master
```

**Architecture:**

```
┌────────────────────────────────────┐
│        Single Master Node          │
│         (cax11 - €4/mo)            │
│                                    │
│  - k3s Control Plane               │
│  - etcd (single instance)          │
│  - Workloads (tolerated)           │
└────────────────────────────────────┘
```

**Monthly Cost:** ~€4
**Limitations:** No HA, not suitable for production

### Step 1: Scale to 3 Masters (Basic HA)

**Use Case:** Small production workloads, basic high availability

```yaml
masters_pool:
  instance_type: cax11  # Still small instances
  instance_count: 3     # HA configuration
  locations:
    - fsn1
    - hel1
    - nbg1

schedule_workloads_on_masters: false  # Stop scheduling on masters

create_load_balancer_for_the_kubernetes_api: true  # Enable API LB

worker_node_pools:
- name: workers
  instance_type: cax11
  instance_count: 2
  location: hel1
```

**Migration Steps:**

1. Update configuration as shown above
2. Run: `hetzner-k3s create --config cluster_config.yaml`
3. Wait for `master2` and `master3` to join
4. Verify etcd cluster: `kubectl get nodes`
5. Drain workloads from `master1` if `schedule_workloads_on_masters` was true

**Architecture:**

```
        ┌─────────────────────┐
        │   API Load Balancer │
        └──────────┬───────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
┌───▼───┐      ┌───▼───┐      ┌──▼────┐
│Master1│◄────►│Master2│◄────►│Master3│
│(fsn1) │ etcd │(hel1) │ etcd │(nbg1) │
│cax11  │      │cax11  │      │cax11  │
└───────┘      └───────┘      └───────┘
                   │
        ┌──────────┴──────────┐
        │                     │
    ┌───▼───┐             ┌───▼───┐
    │Worker1│             │Worker2│
    │(hel1) │             │(hel1) │
    │cax11  │             │cax11  │
    └───────┘             └───────┘
```

**Monthly Cost:** ~€24 (3 masters + 2 workers + LB)
**Benefits:** Survives single node failure, production-ready

### Step 2: Vertical Scale (More Resources)

**Use Case:** Growing workloads, need more resources per node

```yaml
masters_pool:
  instance_type: cax21  # 4 vCPU, 8 GB RAM
  instance_count: 3
  locations:
    - fsn1
    - hel1
    - nbg1

worker_node_pools:
- name: workers
  instance_type: cax21  # Upgraded from cax11
  instance_count: 3     # Increased from 2
  location: hel1
```

**Migration Strategy:**

For each node (one at a time):
1. Drain: `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`
2. Delete from Kubernetes: `kubectl delete node <node>`
3. Remove instance from Hetzner Console
4. Run: `hetzner-k3s create --config cluster_config.yaml`
5. Verify new node joins and is ready
6. Wait for workloads to reschedule
7. Proceed to next node

**Architecture:**

```
        ┌─────────────────────┐
        │   API Load Balancer │
        └──────────┬───────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
┌───▼───┐      ┌───▼───┐      ┌──▼────┐
│Master1│      │Master2│      │Master3│
│(fsn1) │      │(hel1) │      │(nbg1) │
│cax21  │      │cax21  │      │cax21  │
│4vCPU  │      │4vCPU  │      │4vCPU  │
│8GB    │      │8GB    │      │8GB    │
└───────┘      └───────┘      └───────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
    ┌───▼───┐  ┌───▼───┐  ┌──▼────┐
    │Worker1│  │Worker2│  │Worker3│
    │cax21  │  │cax21  │  │cax21  │
    │4vCPU  │  │4vCPU  │  │4vCPU  │
    │8GB    │  │8GB    │  │8GB    │
    └───────┘  └───────┘  └───────┘
```

**Monthly Cost:** ~€60 (3×cax21 masters + 3×cax21 workers + LB)
**Benefits:** More resources, better performance

### Step 3: Add Autoscaling Workers

**Use Case:** Variable workloads, cost optimization, burst capacity

```yaml
masters_pool:
  instance_type: cax21
  instance_count: 3
  locations:
    - fsn1
    - hel1
    - nbg1

worker_node_pools:
# Static baseline for critical workloads
- name: baseline
  instance_type: cax21
  instance_count: 2
  location: hel1
  labels:
    - key: workload-type
      value: critical

# Autoscaled pool for variable workloads
- name: autoscaled
  instance_type: cax31  # 8 vCPU, 16 GB RAM
  location: fsn1
  autoscaling:
    enabled: true
    min_instances: 1
    max_instances: 10
  labels:
    - key: workload-type
      value: batch

cluster_autoscaler:
  scan_interval: "10s"
  scale_down_delay_after_add: "10m"
  max_node_provision_time: "15m"
```

**Architecture:**

```
        ┌─────────────────────┐
        │   API Load Balancer │
        └──────────┬───────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
┌───▼───┐      ┌───▼───┐      ┌──▼────┐
│Master1│      │Master2│      │Master3│
│cax21  │      │cax21  │      │cax21  │
└───────┘      └───────┘      └───────┘
    │              │              │
    ├──────────────┴──────────────┤
    │                             │
    │   Static Workers            │   Autoscaled Workers
    │   (critical)                │   (batch, scale 1-10)
┌───▼─────┐  ┌─────────┐      ┌──▼────────┐
│Baseline1│  │Baseline2│      │Autoscaled │
│cax21    │  │cax21    │      │cax31      │ ◄── Scales up/down
└─────────┘  └─────────┘      └───────────┘     based on demand
```

**Monthly Cost:** Variable
- Base: ~€50 (3 masters + 2 baseline workers + 1 autoscaled + LB)
- Peak: ~€170 (base + 9 additional cax31 instances)
- Average: Depends on workload patterns

**Benefits:**
- Cost-effective (scale to zero capability)
- Handles traffic spikes automatically
- Baseline capacity always available

### Step 4: Production-Ready with Multiple Node Pools

**Use Case:** Large production workloads, diverse requirements

```yaml
masters_pool:
  instance_type: cpx31  # 4 vCPU, 8 GB RAM (dedicated)
  instance_count: 3
  locations:
    - fsn1
    - hel1
    - nbg1

datastore:
  mode: etcd
  etcd:
    snapshot_retention: 24
    snapshot_schedule_cron: "0 * * * *"
    s3_enabled: true
    s3_bucket: "my-cluster-backups"

worker_node_pools:
# Web/API tier
- name: web
  instance_type: cpx31
  location: hel1
  autoscaling:
    enabled: true
    min_instances: 3
    max_instances: 15
  labels:
    - key: tier
      value: web

# Database tier (dedicated CPU)
- name: database
  instance_type: ccx23  # 4 vCPU dedicated, 16 GB RAM
  instance_count: 3
  location: nbg1
  labels:
    - key: tier
      value: database
  taints:
    - key: dedicated
      value: database:NoSchedule

# Batch processing (cost-optimized)
- name: batch
  instance_type: cax41  # 16 vCPU ARM, 32 GB RAM
  location: fsn1
  autoscaling:
    enabled: true
    min_instances: 0
    max_instances: 20
  labels:
    - key: tier
      value: batch

cluster_autoscaler:
  scan_interval: "10s"
  scale_down_delay_after_add: "15m"
  max_node_provision_time: "15m"

k3s_upgrade_concurrency: 3  # Upgrade 3 workers simultaneously

embedded_registry_mirror:
  enabled: true  # Faster pod startup

create_load_balancer_for_the_kubernetes_api: true
protect_against_deletion: true
```

**Architecture:**

```
                 ┌─────────────────────┐
                 │   API Load Balancer │
                 └──────────┬───────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
     ┌───▼───┐          ┌───▼───┐          ┌──▼────┐
     │Master1│          │Master2│          │Master3│
     │cpx31  │◄────────►│cpx31  │◄────────►│cpx31  │
     │(fsn1) │   etcd   │(hel1) │   etcd   │(nbg1) │
     └───────┘          └───────┘          └───────┘
         │                  │                  │
         │                  │                  │
    ┌────┴──────────────────┴──────────────────┴────┐
    │                                                │
    │  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐
    │  │   Web Tier   │  │  Database    │  │  Batch Tier   │
    │  │              │  │    Tier      │  │               │
    │  │ cpx31 (3-15) │  │  ccx23 (3)   │  │  cax41 (0-20) │
    │  │ Autoscaling  │  │  Static      │  │  Autoscaling  │
    │  │              │  │  Dedicated   │  │  ARM          │
    │  └──────────────┘  └──────────────┘  └───────────────┘
    └─────────────────────────────────────────────────────────┘
```

**Monthly Cost:** Variable
- Base: ~€180 (3 cpx31 masters + 3 web + 3 database)
- Peak: ~€600+ (with full autoscaling active)

**Benefits:**
- Production-grade reliability
- Optimized for different workload types
- Cost-efficient with autoscaling
- Automated backups (etcd to S3)
- Fast container pulls (registry mirror)

### Migration Best Practices

**Planning:**
1. **Backup everything**: Databases, volumes, configurations
2. **Test in staging**: Replicate the migration in a test environment
3. **Schedule maintenance**: Inform users of potential disruptions
4. **Document current state**: Instance IDs, IPs, configurations

**Execution:**
1. **One node at a time**: Especially for masters
2. **Verify between steps**: Check node status and pod health
3. **Monitor workloads**: Watch for scheduling issues
4. **Keep rollback plan**: Document how to revert changes

**Validation:**
1. All nodes show `Ready` status
2. All pods are running
3. Applications are accessible
4. Persistent data is intact
5. Monitoring and logging are working

**Post-Migration:**
1. Update documentation
2. Adjust monitoring thresholds
3. Review costs and optimization opportunities
4. Schedule next review/scaling event

---

## Summary

### Scaling Decision Matrix

| Scenario | Approach | Downtime | Complexity |
|----------|----------|----------|------------|
| Need more resources per node | Vertical scaling | Brief (per node) | Medium |
| Need more total capacity | Horizontal scaling | None | Low |
| Variable workloads | Autoscaling | None | Medium |
| Non-HA → HA | Add masters | Brief (API) | Medium |
| Upgrade k3s | Rolling upgrade | None | Low |
| Change instance type | Replace nodes | Brief (per node) | Medium |

### Key Takeaways

1. **Start Small, Scale Up**: Begin with minimal setup, expand as needed
2. **HA is Essential**: 3+ masters for production (odd number)
3. **Autoscaling Saves Money**: Use it for variable workloads
4. **Regional Distribution**: Place masters in different locations
5. **Test Before Production**: Validate scaling strategies in dev/staging
6. **Monitor and Adjust**: Continuously optimize based on actual usage
7. **Backup Regularly**: etcd snapshots to S3 for disaster recovery
8. **Plan Upgrades**: Use rolling upgrades during low-traffic periods

### Quick Reference Commands

```bash
# Create/update cluster
hetzner-k3s create --config cluster_config.yaml

# Upgrade k3s version
hetzner-k3s upgrade --config cluster_config.yaml --new-k3s-version v1.30.3+k3s1

# Monitor upgrades
watch kubectl get nodes -owide
kubectl get jobs,pods -n system-upgrade

# Drain node for maintenance
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Delete node
kubectl delete node <node-name>

# Check cluster health
kubectl get nodes
kubectl get pods --all-namespaces
kubectl top nodes
kubectl top pods --all-namespaces

# Check autoscaler status
kubectl get pods -n kube-system -l app=cluster-autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler
```

---

## Additional Resources

- [Creating a cluster](Creating_a_cluster.md)
- [Maintenance](Maintenance.md)
- [Masters in Different Locations](Masters_in_different_locations.md)
- [Load Balancers](Load_balancers.md)
- [Recommendations](Recommendations.md)
- [Hetzner Cloud Controller Manager](https://github.com/hetznercloud/hcloud-cloud-controller-manager)
- [Rancher System Upgrade Controller](https://github.com/rancher/system-upgrade-controller)
- [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler)
