# Executive Summary
## hetzner-k3s Architecture Review for Multi-Tenant Platform

**Date:** 2025-11-17
**Purpose:** Inform Infrastructure-as-Code design for multi-tenant managed k3s platform on Hetzner Cloud ARM servers

---

## Overview

**hetzner-k3s** is a production-ready, 2,166-line Crystal CLI tool that provisions k3s clusters on Hetzner Cloud in 2-3 minutes. It provides a battle-tested approach to deploying Kubernetes on Hetzner infrastructure that can be adapted to Terraform/OpenTofu for your multi-tenant platform.

---

## Key Findings

| Aspect | Finding | Impact on Your Platform |
|--------|---------|------------------------|
| **Provisioning** | Direct Hetzner API orchestration with concurrent instance creation | Easily translatable to Terraform resources |
| **Architecture** | Cloud-init based setup with templated k3s installation | Can reuse cloud-init templates in Terraform |
| **Networking** | Private network (10.0.0.0/16) + firewall via Hetzner API | Native Terraform support available |
| **Storage** | Hetzner CSI Driver for block volumes | Installed via kubectl after cluster creation |
| **Scaling** | Idempotent operations, cluster autoscaler, rolling upgrades | Supports 1→3→5 node growth path |
| **HA** | Multi-master with embedded etcd + load balancer | Proven pattern for production use |
| **Multi-Tenancy** | Label-based isolation per cluster | Perfect for isolated client clusters |

---

## Quick Answers to Your Key Questions

### 1. Single CAX11 with k3s via Terraform?

**✅ YES** - Complete working example available

- **Terraform Resources Required:** 5 resources (SSH key, network, subnet, firewall, server)
- **Provisioning Time:** ~3-5 minutes
- **Monthly Cost:** ~€4
- **Code:** 700-line example in [terraform-conversion-strategy.md](./terraform-conversion-strategy.md#example-single-cax11-cluster)

### 2. Same IaC for multiple clients?

**✅ YES** - Variable-driven with separate state files per client

**Approach:**
```
terraform/
├── modules/hetzner-k3s-cluster/    # Reusable module (same code)
└── environments/
    ├── client-a-prod/              # Client A state
    ├── client-b-prod/              # Client B state
    └── client-c-dev/               # Client C state
```

**Isolation:**
- Separate Terraform state per client
- Variable-driven configuration (tfvars files)
- Label-based resource targeting
- Client-provided Hetzner API tokens

### 3. Scale 1→3 nodes without recreation?

**✅ YES** - The create command is idempotent and incremental

**Process:**
1. Update configuration: `master_count: 1` → `master_count: 3`
2. Run `terraform apply` (or CLI equivalent)
3. Tool detects existing resources, adds 2 new masters
4. No disruption to existing nodes

**See:** [scaling-and-ha.md](./scaling-and-ha.md#horizontal-scaling)

### 4. Network/firewall for multi-tenant?

**✅ YES** - Label selectors provide isolation, separate firewalls per cluster

**Security Model:**
- Each cluster has unique labels: `cluster=client-a-prod`
- Firewall applies via label selector (not IP-based)
- Private networks are cluster-specific
- No cross-cluster communication possible

**See:** [network-architecture.md](./network-architecture.md#multi-tenant-security)

---

## Architecture at a Glance

```
Crystal CLI Tool (hetzner-k3s)
        ↓
Hetzner Cloud API
        ↓
Cloud Servers (CAX11/21/31/41)
        ↓
k3s Installation (via get.k3s.io)
        ↓
Kubernetes Cluster
        ├─ Hetzner Cloud Controller Manager (LoadBalancers)
        ├─ Hetzner CSI Driver (Volumes)
        ├─ Cluster Autoscaler (Auto-scaling)
        └─ System Upgrade Controller (k3s upgrades)
```

**Provisioning Flow:**
1. Create SSH key
2. Create private network (10.0.0.0/16)
3. Create firewall with label selector
4. Create instances (concurrent, max 10)
5. Cloud-init installs k3s
6. Install Hetzner add-ons
7. Cluster ready (2-3 minutes)

---

## Key Design Patterns

### 1. Cloud-Init Templating
Inject k3s installation scripts via user-data, eliminating manual SSH configuration.

### 2. Concurrent Provisioning
Create instances in parallel (10 max concurrency) using semaphore limiting.

### 3. Network Detection
Use MTU-based interface detection (1450/1280) for Hetzner private networks.

### 4. Label-Based Targeting
Hetzner labels enable firewall/LB targeting without IP hardcoding.

### 5. Idempotent Operations
Check for existing resources before creating, enabling incremental updates.

### 6. Retry Logic
Exponential backoff for API calls (10 attempts, 5s interval) handles transient failures.

---

## Multi-Tenant Platform Feasibility

### Single CAX11 Deployment
- **Resources:** 5 Terraform resources
- **Time:** 3-5 minutes
- **Cost:** €4/month
- **Complexity:** Low

### Multi-Client Management
- **Module:** One reusable Terraform module
- **Isolation:** Separate state files + label selectors
- **Scaling:** Variable-driven configuration
- **Onboarding:** < 1 hour per new client

### Production HA Cluster
- **Resources:** +5 additional resources (LB, 2 more masters, 2 workers)
- **Time:** 5-8 minutes
- **Cost:** €68/month
- **Complexity:** Medium

---

## ARM64 (CAX Servers) Compatibility

**✅ Fully Supported** - No special configuration required

**Verified Components:**
- k3s v1.30+ (native ARM64 builds)
- Containerd runtime
- Flannel and Cilium CNI
- Hetzner Cloud Controller Manager
- Hetzner CSI Driver
- Cluster Autoscaler
- System Upgrade Controller

**Potential Issues:**
- Third-party container images may not have ARM64 builds
- Check with `docker manifest inspect <image>`

**Recommendation:** Start with ARM (CAX), add x86 node pool if specific workloads require it

---

## Documentation Structure

This architecture review consists of **11 numbered documents** + supporting materials:

### Core Documents (Read in Order)

1. **01-executive-summary.md** (this document) - Overview and quick answers
2. **[02-how-it-works.md](./02-how-it-works.md)** - Architecture and design patterns
3. **[03-implementation-strategy.md](./03-implementation-strategy.md)** - Your platform approach
4. **[04-technical-details.md](./04-technical-details.md)** - Cloud-init, networking, k3s
5. **[05-terraform-decisions.md](./05-terraform-decisions.md)** - Key Terraform choices
6. **[06-arm-considerations.md](./06-arm-considerations.md)** - ARM64 compatibility
7. **[07-implementation-roadmap.md](./07-implementation-roadmap.md)** - 5-phase plan
8. **[08-cost-estimation.md](./08-cost-estimation.md)** - Pricing models
9. **[09-pitfalls-and-solutions.md](./09-pitfalls-and-solutions.md)** - Common issues
10. **[10-quick-reference.md](./10-quick-reference.md)** - Commands and configs
11. **[11-resources.md](./11-resources.md)** - Links and community

### Deep-Dive Documents (Reference)

- **[provisioning-workflow.md](./provisioning-workflow.md)** - Complete provisioning flow (1,133 lines)
- **[network-architecture.md](./network-architecture.md)** - Network topology and security (36KB)
- **[scaling-and-ha.md](./scaling-and-ha.md)** - Scaling and HA patterns (1,319 lines)
- **[terraform-conversion-strategy.md](./terraform-conversion-strategy.md)** - Terraform modules (1,341 lines)

---

## Recommended Reading Path

### For Quick Start (30 minutes)
1. This document (01-executive-summary.md)
2. [03-implementation-strategy.md](./03-implementation-strategy.md)
3. [terraform-conversion-strategy.md](./terraform-conversion-strategy.md) - Jump to "Example: Single CAX11 Cluster"

### For Full Understanding (2-3 hours)
1. Read all 11 numbered documents in order
2. Skim the 4 deep-dive documents
3. Review code examples in terraform-conversion-strategy.md

### For Specific Topics
- **Networking:** [network-architecture.md](./network-architecture.md)
- **Scaling:** [scaling-and-ha.md](./scaling-and-ha.md)
- **Provisioning Details:** [provisioning-workflow.md](./provisioning-workflow.md)
- **Terraform Code:** [terraform-conversion-strategy.md](./terraform-conversion-strategy.md)

---

## Key Takeaways

1. ✅ **Single CAX11 deployment is straightforward** - 5 Terraform resources, cloud-init template, done
2. ✅ **Scaling path is clear** - Start small, grow incrementally without recreation
3. ✅ **Multi-tenancy is native** - Label-based isolation, separate state files
4. ✅ **ARM64 works great** - No special considerations needed
5. ✅ **Terraform conversion is feasible** - Direct mapping from CLI operations to Terraform resources

---

## Next Steps

1. **Read [03-implementation-strategy.md](./03-implementation-strategy.md)** - Understand your platform approach
2. **Deploy POC** - Use single-CAX11 Terraform example
3. **Build module** - Follow recommended structure in [terraform-conversion-strategy.md](./terraform-conversion-strategy.md)
4. **Set up multi-tenant** - Use environment layout pattern
5. **Implement CI/CD** - GitHub Actions examples provided

---

**Document Version:** 1.0
**Last Updated:** 2025-11-17
**Part:** 1 of 11
