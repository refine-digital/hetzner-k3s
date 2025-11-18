# hetzner-k3s Architecture Review Documentation
## Multi-Tenant Managed Kubernetes Platform Design Guide

**Last Updated:** 2025-11-17
**Total Documentation:** ~5,700 lines across 15 core documents + feature documentation system

---

## 📖 Documentation Overview

This comprehensive architecture review provides everything needed to design and build a multi-tenant managed Kubernetes platform on Hetzner Cloud ARM servers using Terraform/OpenTofu.

### Quick Start (30 minutes)

1. **[01-executive-summary.md](./01-executive-summary.md)** - Start here! Quick answers to your key questions
2. **[03-implementation-strategy.md](./03-implementation-strategy.md)** - Your platform approach
3. **[terraform-conversion-strategy.md](./terraform-conversion-strategy.md)** - Jump to "Example: Single CAX11 Cluster" section

### Full Review (2-3 hours)

Read all documents in numerical order for complete understanding.

---

## 📚 Document Structure

### Core Documents (11 numbered parts)

Read these in order for a complete understanding of the architecture and implementation strategy:

| # | Document | Description | Lines |
|---|----------|-------------|-------|
| 1 | **[01-executive-summary.md](./01-executive-summary.md)** | Overview, key findings, quick answers | 300 |
| 2 | **[02-how-it-works.md](./02-how-it-works.md)** | Architecture diagrams, design patterns | 120 |
| 3 | **[03-implementation-strategy.md](./03-implementation-strategy.md)** | Single node → HA growth path, multi-tenant pattern | 200 |
| 4 | **[04-technical-details.md](./04-technical-details.md)** | Cloud-init, networking, k3s installation | 250 |
| 5 | **[05-terraform-decisions.md](./05-terraform-decisions.md)** | Template management, state, token handling | 150 |
| 6 | **[06-arm-considerations.md](./06-arm-considerations.md)** | ARM64 compatibility and recommendations | 100 |
| 7 | **[07-implementation-roadmap.md](./07-implementation-roadmap.md)** | 5-phase plan: POC → Production | 200 |
| 8 | **[08-cost-estimation.md](./08-cost-estimation.md)** | Pricing models, margins, ROI | 120 |
| 9 | **[09-pitfalls-and-solutions.md](./09-pitfalls-and-solutions.md)** | 8 common issues and solutions | 300 |
| 10 | **[10-quick-reference.md](./10-quick-reference.md)** | Commands, configs, snippets | 200 |
| 11 | **[11-resources.md](./11-resources.md)** | Links, community, conclusion | 150 |

**Total:** ~2,100 lines

### Deep-Dive Documents (Technical Reference)

Use these for detailed implementation guidance:

| Document | Description | Lines |
|----------|-------------|-------|
| **[provisioning-workflow.md](./provisioning-workflow.md)** | Complete provisioning flow, API calls, timing | 1,133 |
| **[network-architecture.md](./network-architecture.md)** | Network topology, firewall rules, security | 900 |
| **[scaling-and-ha.md](./scaling-and-ha.md)** | Scaling strategies, autoscaler, upgrades | 1,319 |
| **[terraform-conversion-strategy.md](./terraform-conversion-strategy.md)** | Terraform modules, working examples | 1,341 |

**Total:** ~4,700 lines

### Platform Research Documents (20-25)

Multi-tenant platform architecture and implementation:

| # | Document | Description | Lines |
|---|----------|-------------|-------|
| 20 | **[20-email-infrastructure-postfix-dovecot.md](./20-email-infrastructure-postfix-dovecot.md)** | FREE tier email with Postfix + Dovecot | 1,066 |
| 21 | **[21-packer-k3s-image-strategy.md](./21-packer-k3s-image-strategy.md)** | Pre-baked images for <3min deployment | 3,075 |
| 22 | **[22-free-to-premium-migration-automation.md](./22-free-to-premium-migration-automation.md)** | Zero-downtime migration system | 1,740 |
| 23 | **[23-platform-api-architecture.md](./23-platform-api-architecture.md)** | Node.js + TypeScript REST API | 980 |
| 24 | **[24-mobile-ux-patterns.md](./24-mobile-ux-patterns.md)** | Mobile-first UX for non-technical users | 615 |
| 25 | **[25-deployment-system-overview.md](./25-deployment-system-overview.md)** | Complete platform architecture | 682 |

**Total:** ~8,158 lines

### Process & System Documents (30+)

Development processes and workflows:

| # | Document | Description | Lines |
|---|----------|-------------|-------|
| 30 | **[30-feature-documentation-system.md](./30-feature-documentation-system.md)** | Feature docs lifecycle (RFCs, ADRs, Features, Guides) | 2,338 |
| 31 | **[31-github-integration-for-features.md](./31-github-integration-for-features.md)** | GitHub Issues, Projects, Discussions, Actions integration | 1,736 |

**Total:** ~4,074 lines

---

## 🎯 Quick Answers

### Can I deploy a single CAX11 with k3s using Terraform?

**✅ YES** - See [terraform-conversion-strategy.md#example-single-cax11-cluster](./terraform-conversion-strategy.md#example-single-cax11-cluster)

- **Resources:** 5 Terraform resources
- **Time:** 3-5 minutes
- **Cost:** €4/month
- **Code:** 700-line working example

### Can the same IaC work for multiple isolated clients?

**✅ YES** - See [03-implementation-strategy.md#multi-tenant-pattern](./03-implementation-strategy.md#multi-tenant-pattern)

- **Approach:** One module, variable-driven configuration
- **Isolation:** Separate state files per client
- **Security:** Label-based resource targeting

### Can I scale from 1→3 nodes without cluster recreation?

**✅ YES** - See [scaling-and-ha.md#horizontal-scaling](./scaling-and-ha.md#horizontal-scaling)

- **Process:** Idempotent operations
- **Impact:** No disruption to existing nodes
- **Time:** Incremental, same as initial creation

### How do network/firewall patterns work for multi-tenant?

**✅ SOLVED** - See [network-architecture.md#multi-tenant-security](./network-architecture.md#multi-tenant-security)

- **Approach:** Label selectors (not IP-based)
- **Isolation:** Separate firewall per cluster
- **Security:** Three isolation strategies provided

---

## 🚀 Reading Paths

### Path 1: Quick Implementation (Developer)

**Goal:** Deploy first cluster ASAP

1. [01-executive-summary.md](./01-executive-summary.md) - 10 min
2. [terraform-conversion-strategy.md](./terraform-conversion-strategy.md) - Focus on "Example: Single CAX11 Cluster" - 20 min
3. Deploy! - 5 min

**Total:** 35 minutes to working cluster

### Path 2: Platform Architect

**Goal:** Design complete multi-tenant platform

1. [01-executive-summary.md](./01-executive-summary.md) - 15 min
2. [02-how-it-works.md](./02-how-it-works.md) - 10 min
3. [03-implementation-strategy.md](./03-implementation-strategy.md) - 20 min
4. [network-architecture.md](./network-architecture.md) - 30 min
5. [scaling-and-ha.md](./scaling-and-ha.md) - 30 min
6. [terraform-conversion-strategy.md](./terraform-conversion-strategy.md) - 45 min
7. [07-implementation-roadmap.md](./07-implementation-roadmap.md) - 20 min

**Total:** 2.5 hours to complete platform design

### Path 3: Security Review

**Goal:** Validate security and isolation

1. [01-executive-summary.md](./01-executive-summary.md) - 15 min
2. [network-architecture.md](./network-architecture.md) - 45 min (focus on multi-tenant security)
3. [04-technical-details.md](./04-technical-details.md) - 20 min (focus on firewall rules)
4. [09-pitfalls-and-solutions.md](./09-pitfalls-and-solutions.md) - 20 min

**Total:** 1.5 hours for security validation

### Path 4: Cost Analysis

**Goal:** Build business case

1. [01-executive-summary.md](./01-executive-summary.md) - 15 min
2. [08-cost-estimation.md](./08-cost-estimation.md) - 30 min
3. [scaling-and-ha.md](./scaling-and-ha.md) - Focus on "Practical Scaling Path" - 20 min
4. [07-implementation-roadmap.md](./07-implementation-roadmap.md) - 20 min

**Total:** 1.5 hours for complete cost model

---

## 🔍 Topic Index

### Provisioning & Setup
- **Basic Setup:** [terraform-conversion-strategy.md](./terraform-conversion-strategy.md)
- **Complete Flow:** [provisioning-workflow.md](./provisioning-workflow.md)
- **Cloud-Init:** [04-technical-details.md](./04-technical-details.md)

### Networking & Security
- **Network Topology:** [network-architecture.md](./network-architecture.md)
- **Firewall Rules:** [04-technical-details.md](./04-technical-details.md), [network-architecture.md](./network-architecture.md)
- **Multi-Tenant Security:** [network-architecture.md#multi-tenant-security](./network-architecture.md)

### Scaling & HA
- **Horizontal Scaling:** [scaling-and-ha.md](./scaling-and-ha.md)
- **Vertical Scaling:** [scaling-and-ha.md](./scaling-and-ha.md)
- **HA Setup:** [scaling-and-ha.md](./scaling-and-ha.md), [04-technical-details.md](./04-technical-details.md)
- **Autoscaling:** [scaling-and-ha.md](./scaling-and-ha.md)

### Terraform Implementation
- **Module Structure:** [terraform-conversion-strategy.md](./terraform-conversion-strategy.md)
- **Working Examples:** [terraform-conversion-strategy.md](./terraform-conversion-strategy.md)
- **Key Decisions:** [05-terraform-decisions.md](./05-terraform-decisions.md)
- **Multi-Tenant Pattern:** [03-implementation-strategy.md](./03-implementation-strategy.md), [terraform-conversion-strategy.md](./terraform-conversion-strategy.md)

### ARM Architecture
- **Compatibility:** [06-arm-considerations.md](./06-arm-considerations.md)
- **CAX Servers:** [01-executive-summary.md](./01-executive-summary.md), [06-arm-considerations.md](./06-arm-considerations.md)

### Cost & Planning
- **Cost Models:** [08-cost-estimation.md](./08-cost-estimation.md)
- **Implementation Roadmap:** [07-implementation-roadmap.md](./07-implementation-roadmap.md)
- **ROI Analysis:** [08-cost-estimation.md](./08-cost-estimation.md)

### Troubleshooting
- **Common Pitfalls:** [09-pitfalls-and-solutions.md](./09-pitfalls-and-solutions.md)
- **Network Issues:** [09-pitfalls-and-solutions.md](./09-pitfalls-and-solutions.md)
- **State Management:** [09-pitfalls-and-solutions.md](./09-pitfalls-and-solutions.md)

---

## 💡 Key Insights

### Architecture
- **Provisioning Time:** 2-3 minutes for HA cluster
- **Concurrent Limit:** 10 instances simultaneously
- **Network Detection:** MTU-based (1450/1280 for private networks)
- **Retry Strategy:** 30 attempts × 10s for network interfaces

### Terraform
- **Minimal Resources:** 5 for single-node cluster
- **HA Resources:** +5 for production HA (LB + additional masters/workers)
- **Template Strategy:** Inline for simple, module for complex
- **State Management:** Separate backends per client (S3)

### Costs
- **Dev Cluster:** €4/month (single CAX11)
- **Prod HA:** €68/month (3 masters, 2 workers, LB)
- **Platform Margin:** 50-75% on infrastructure costs
- **Scaling:** Variable cost with autoscaling (€82-€181/month)

### Security
- **Isolation Method:** Label selectors + separate state
- **Firewall Limit:** 50 rules max (Hetzner constraint)
- **Network Model:** Private network (10.0.0.0/16) per cluster
- **Multi-Tenant:** 3 strategies (separate clusters, shared cluster, namespace isolation)

---

## 🛠️ Practical Examples

### Deploy Single CAX11 Cluster

```bash
# 1. Clone and navigate
cd terraform/modules/hetzner-k3s-cluster

# 2. Copy example
cp examples/single-cax11/main.tf .

# 3. Configure
cat > terraform.tfvars <<EOF
cluster_name = "my-first-cluster"
hcloud_token = "YOUR_HETZNER_TOKEN"
ssh_public_key_path = "~/.ssh/id_ed25519.pub"
ssh_private_key_path = "~/.ssh/id_ed25519"
EOF

# 4. Deploy
terraform init
terraform plan
terraform apply

# 5. Access cluster
export KUBECONFIG=./kubeconfig
kubectl get nodes
```

**See:** [terraform-conversion-strategy.md](./terraform-conversion-strategy.md) for complete code

### Multi-Tenant Setup

```bash
# 1. Create client environment
mkdir -p environments/client-a-prod
cd environments/client-a-prod

# 2. Reference module
cat > main.tf <<EOF
module "cluster" {
  source = "../../modules/hetzner-k3s-cluster"

  cluster_name = "client-a-prod"
  # ... other variables
}
EOF

# 3. Separate state
cat > backend.tf <<EOF
terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "clients/client-a/prod/terraform.tfstate"
    region = "us-east-1"
  }
}
EOF
```

**See:** [03-implementation-strategy.md](./03-implementation-strategy.md) for complete pattern

---

## 📊 Comparison: hetzner-k3s CLI vs. Terraform

| Aspect | hetzner-k3s CLI | Terraform Approach |
|--------|-----------------|-------------------|
| **Language** | Crystal | HCL |
| **Provisioning** | Direct API calls | Terraform providers |
| **State Management** | None (idempotent checks) | Terraform state |
| **Multi-Tenancy** | Manual (separate configs) | Modules + workspaces |
| **Cloud-Init** | Built-in templates | Custom templates |
| **Complexity** | Low (CLI tool) | Medium (IaC patterns) |
| **Flexibility** | Limited to CLI features | Highly customizable |
| **CI/CD** | Manual scripting | Native Terraform support |
| **Team Collaboration** | Difficult | Built-in (remote state) |

**Recommendation:** Use Terraform for multi-tenant platform (better collaboration, state management, and CI/CD integration)

---

## 🎓 Learning Resources

### Prerequisites
- **Terraform:** https://learn.hashicorp.com/terraform
- **Kubernetes Basics:** https://kubernetes.io/docs/tutorials/
- **k3s Documentation:** https://docs.k3s.io/
- **Hetzner Cloud:** https://docs.hetzner.cloud/

### Related Projects
- **hetzner-k3s GitHub:** https://github.com/vitobotta/hetzner-k3s
- **Terraform Hetzner Provider:** https://registry.terraform.io/providers/hetznercloud/hcloud/latest/docs
- **k3s on ARM64:** https://docs.k3s.io/advanced#additional-preparation-for-alpine-linux-setup

### Community
- **Hetzner Community:** https://community.hetzner.com/
- **k3s Slack:** https://slack.k3s.io/
- **r/kubernetes:** https://reddit.com/r/kubernetes

---

## 📝 Document Metadata

- **Project:** hetzner-k3s Architecture Review
- **Version:** 1.0
- **Last Updated:** 2025-11-17
- **Maintainer:** Architecture Review Team
- **Total Lines:** ~5,700
- **Documents:** 15 (11 core + 4 deep-dive)
- **Status:** Complete

---

## 🤝 How to Use This Documentation

### For Implementation

1. **Start:** Read [01-executive-summary.md](./01-executive-summary.md)
2. **Understand:** Read numbered docs 02-07 in order
3. **Implement:** Follow [terraform-conversion-strategy.md](./terraform-conversion-strategy.md)
4. **Deploy:** Use provided Terraform examples
5. **Scale:** Follow [scaling-and-ha.md](./scaling-and-ha.md)

### For Review

1. **Executive Summary:** [01-executive-summary.md](./01-executive-summary.md)
2. **Cost Analysis:** [08-cost-estimation.md](./08-cost-estimation.md)
3. **Security Review:** [network-architecture.md](./network-architecture.md)
4. **Risk Assessment:** [09-pitfalls-and-solutions.md](./09-pitfalls-and-solutions.md)

### For Reference

- **Quick Lookup:** [10-quick-reference.md](./10-quick-reference.md)
- **API Details:** [provisioning-workflow.md](./provisioning-workflow.md)
- **Network Details:** [network-architecture.md](./network-architecture.md)
- **Code Examples:** [terraform-conversion-strategy.md](./terraform-conversion-strategy.md)

---

## 📊 Feature Status Dashboard

Track features in development, testing, and production across FREE, PREMIUM, and Platform tiers:

| Feature | Status | Tier | Target Release | Owner |
|---------|--------|------|----------------|-------|
| *No features documented yet* | - | - | - | - |

**[View Full Feature Dashboard →](./features/README.md)**

### Quick Stats

| Tier | Total | Production | In Development |
|------|-------|------------|----------------|
| FREE | 0 | 0 | 0 |
| PREMIUM | 0 | 0 | 0 |
| Platform | 0 | 0 | 0 |

**Next Step:** Document existing production features (domain management, email accounts, k3s provisioning, etc.)

---

## 🏗️ Architecture Decisions (ADRs)

Document key technical decisions with context, alternatives, and consequences:

| ADR | Decision | Date | Status |
|-----|----------|------|--------|
| *No ADRs yet* | - | - | - |

**[View All ADRs →](./adrs/README.md)**

### Suggested Retroactive ADRs

Document decisions from existing architecture:
- **ADR-001**: Use k3s over k8s for lightweight Kubernetes
- **ADR-002**: Use PostgreSQL over MySQL for relational database
- **ADR-003**: Use Argon2id over bcrypt for password hashing
- **ADR-004**: Use Maildir over mbox for email storage

**[Learn More →](./adrs/README.md)**

---

## 📋 Feature Documentation System

**New in Document 30!** Comprehensive system for documenting features through their lifecycle:

- **[RFCs](./rfcs/README.md)**: Request for Comments for significant features
- **[ADRs](./adrs/README.md)**: Architecture Decision Records for technical decisions
- **[Features](./features/README.md)**: Feature documentation (FREE/PREMIUM/Platform)
- **[Guides](./guides/README.md)**: Implementation and troubleshooting guides

**[Read Document 30: Feature Documentation System →](./30-feature-documentation-system.md)**

### Feature Lifecycle

```
Idea → RFC → Design (ADRs) → Development → Testing → Deployment → Production
                                                                        ↓
                                                                   Deprecated
```

### Documentation by Stage

| Stage | Badge | Document Type | Example |
|-------|-------|---------------|---------|
| Proposal | 📋 DRAFT | RFC | RFC-001-email-quota-enforcement.md |
| Design | 🎨 DESIGN | ADR | ADR-004-dovecot-maildir-over-mbox.md |
| Development | 🔨 IN DEVELOPMENT | Feature Doc | FEAT-002-email-accounts.md |
| Testing | 🧪 TESTING | Feature Doc + Tests | FEAT-102-auto-scaling.md |
| Production | ✅ PRODUCTION | Feature Doc + Guide | Guide-Managing_Email_Accounts.md |

**[View Complete System →](./30-feature-documentation-system.md)**

---

**Ready to start?** Begin with [01-executive-summary.md](./01-executive-summary.md)
