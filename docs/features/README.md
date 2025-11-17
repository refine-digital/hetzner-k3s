# Feature Status Dashboard

Last Updated: 2025-11-17 (Automated updates coming soon)

## Overview

This directory contains documentation for all features across the my.refine.digital platform.

## Feature Organization

- **free-tier/**: Features available on FREE tier (FEAT-001 to FEAT-099)
- **premium-tier/**: Features available on PREMIUM tier (FEAT-100 to FEAT-199)
- **platform/**: Platform features available to both tiers (FEAT-200 to FEAT-299)
- **deprecated/**: End-of-life features (for historical reference)

## Feature Status

### By Status

#### 📋 DRAFT (0)
*No features in draft stage*

#### 🎨 DESIGN (0)
*No features in design stage*

#### 🔨 IN DEVELOPMENT (0)
*No features in active development*

#### 🧪 TESTING (0)
*No features in testing*

#### 🚀 DEPLOYING (0)
*No features deploying*

#### ✅ PRODUCTION (0)
*No features documented yet*

> **Next Steps:** Document existing production features (domain management, email accounts, k3s provisioning, etc.)

#### ⚠️ DEPRECATED (0)
*No deprecated features*

---

## By Tier

| Tier | Total | Production | In Development | Testing | Deploying |
|------|-------|------------|----------------|---------|-----------|
| FREE | 0 | 0 | 0 | 0 | 0 |
| PREMIUM | 0 | 0 | 0 | 0 | 0 |
| Platform | 0 | 0 | 0 | 0 | 0 |
| **Total** | **0** | **0** | **0** | **0** | **0** |

---

## By Target Release

### v1.2.0 (Next Release - TBD)
*No features scheduled*

### v1.3.0 (TBD)
*No features scheduled*

### Backlog
*No features in backlog*

---

## Suggested Features to Document

Based on the architecture documentation (docs 20-25), these production features should be documented:

### FREE Tier (FEAT-001 to FEAT-099)

| Suggested ID | Feature Name | Description | Priority |
|--------------|--------------|-------------|----------|
| FEAT-001 | Domain Management | Add/remove domains, DNS configuration | High |
| FEAT-002 | Email Accounts | Create email accounts, manage passwords | High |
| FEAT-003 | DNS Wizard | Guided DNS setup for domains | Medium |
| FEAT-004 | Email Forwarding | Forward emails to external addresses | Medium |
| FEAT-005 | Webmail Access | IMAP/POP3 access configuration | Medium |
| FEAT-006 | User Dashboard | Main user interface for FREE tier | High |
| FEAT-007 | Billing & Payments | Stripe integration for upgrades | High |

### PREMIUM Tier (FEAT-100 to FEAT-199)

| Suggested ID | Feature Name | Description | Priority |
|--------------|--------------|-------------|----------|
| FEAT-101 | K3s Provisioning | Single-node k3s cluster deployment | High |
| FEAT-102 | Auto-Scaling | Vertical & horizontal scaling | Medium |
| FEAT-103 | Multi-Region | Deploy clusters in multiple regions | Low |
| FEAT-104 | Backup & Restore | Automated cluster backups | Medium |
| FEAT-105 | Monitoring Dashboard | Prometheus + Grafana integration | Medium |
| FEAT-106 | Custom Domains | Custom domains for k3s workloads | Low |

### Platform (FEAT-200 to FEAT-299)

| Suggested ID | Feature Name | Description | Priority |
|--------------|--------------|-------------|----------|
| FEAT-201 | User Authentication | JWT + session management | High |
| FEAT-202 | Billing System | Subscription management, invoicing | High |
| FEAT-203 | Migration Engine | FREE → PREMIUM migration automation | High |
| FEAT-204 | Usage Analytics | Track usage metrics per client | Medium |
| FEAT-205 | Audit Logging | Security & compliance logging | Medium |
| FEAT-206 | SSO Integration | Single sign-on support | Low |
| FEAT-207 | API Management | Rate limiting, API keys | Medium |
| FEAT-208 | Mobile App | Mobile-first responsive UI | High |

---

## Metrics

- **Total Features:** 0 (documentation pending)
- **Production Features:** 0
- **In Development:** 0
- **Average Time to Production:** TBD
- **Feature Success Rate:** TBD

---

## Creating a New Feature Document

1. Copy the [template](./template.md)
2. Determine tier and assign number:
   - FREE tier: FEAT-001 to FEAT-099
   - PREMIUM tier: FEAT-100 to FEAT-199
   - Platform: FEAT-200 to FEAT-299
3. Name it `FEAT-NNN-feature-name.md`
4. Place in appropriate tier directory
5. Fill in all sections
6. Update this dashboard (manual for now, automated later)

---

## Feature Documentation Process

```
Idea → RFC (if significant) → Design → ADRs → Feature Doc
                                                    ↓
                    Draft → Development → Testing → Deploy → Production
                                                                  ↓
                                                            Deprecated
```

See [Document 30: Feature Documentation System](../30-feature-documentation-system.md) for complete guidance.

---

## Automation (Coming Soon)

Planning to automate:
- [ ] Auto-generate this dashboard from feature docs
- [ ] CI/CD checks for required sections
- [ ] Slack notifications for status changes
- [ ] GitHub Project board integration

**Script location:** `scripts/update-feature-dashboard.sh`
