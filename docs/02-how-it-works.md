# How hetzner-k3s Works
**Part:** 2 of 11

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

**Next:** [03-implementation-strategy.md](./03-implementation-strategy.md)
