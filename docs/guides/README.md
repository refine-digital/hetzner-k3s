# Implementation Guides

This directory contains operational and implementation guides for the my.refine.digital platform.

## Guide Categories

### Operations Guides (`operations/`)
Step-by-step guides for operating production systems.

| Guide | Description | Audience |
|-------|-------------|----------|
| - | - | - |

*No operations guides yet*

### Development Guides (`development/`)
Setup and development workflow guides.

| Guide | Description | Audience |
|-------|-------------|----------|
| - | - | - |

*No development guides yet*

### Troubleshooting Guides (`troubleshooting/`)
Common issues and their solutions.

| Guide | Description | Audience |
|-------|-------------|----------|
| - | - | - |

*No troubleshooting guides yet*

---

## Suggested Guides

Based on the platform architecture (docs 20-25), consider creating these guides:

### Operations

| Priority | Guide Name | Purpose |
|----------|------------|---------|
| High | Guide-Monitoring_Email_Queues.md | Monitor Postfix/Dovecot queues |
| High | Guide-K3s_Cluster_Backup.md | Backup and restore k3s clusters |
| Medium | Guide-Managing_DNS_Records.md | Update DNS records for clients |
| Medium | Guide-Certificate_Renewal.md | TLS certificate management |
| Low | Guide-Database_Maintenance.md | PostgreSQL vacuum, reindex, etc. |

### Development

| Priority | Guide Name | Purpose |
|----------|------------|---------|
| High | Guide-Local_Development_Setup.md | Set up local dev environment |
| High | Guide-Testing_Migrations.md | Test FREE → PREMIUM migrations |
| Medium | Guide-API_Development.md | Create new API endpoints |
| Medium | Guide-Database_Migrations.md | Run and create migrations |
| Low | Guide-Frontend_Components.md | Build new UI components |

### Troubleshooting

| Priority | Guide Name | Purpose |
|----------|------------|---------|
| High | Guide-Email_Delivery_Issues.md | Debug email not sending/receiving |
| High | Guide-K3s_Node_Not_Ready.md | Fix k3s node issues |
| Medium | Guide-DNS_Propagation_Delays.md | Understand DNS timing |
| Medium | Guide-Database_Connection_Errors.md | Fix PostgreSQL connection issues |
| Low | Guide-Performance_Debugging.md | Investigate slow responses |

---

## Guide Naming Convention

**Format:** `Guide-Action_Description.md`

- `Action`: Verb (Setting_up, Monitoring, Troubleshooting, Managing, etc.)
- `Description`: What is being acted upon
- Use CamelCase with underscores

**Examples:**
```
Guide-Setting_up_Email_Forwarding.md
Guide-Monitoring_K3s_Clusters.md
Guide-Troubleshooting_DNS_Issues.md
Guide-Managing_User_Accounts.md
```

---

## Guide Template Structure

All guides should include:

1. **Purpose**: What this guide helps you accomplish
2. **Prerequisites**: What you need before starting
3. **Steps**: Numbered steps with code examples
4. **Troubleshooting**: Common issues and solutions
5. **Related Resources**: Links to feature docs, ADRs, etc.

See [Document 30: Feature Documentation System](../30-feature-documentation-system.md) for template.

---

## Creating a New Guide

1. Determine category (operations/development/troubleshooting)
2. Name using convention: `Guide-Action_Description.md`
3. Place in appropriate subdirectory
4. Follow template structure
5. Add to this README
6. Cross-link from related feature docs

---

## Difference Between Guides and Feature Docs

| Aspect | Feature Doc | Guide |
|--------|-------------|-------|
| **Purpose** | Document feature design & implementation | Explain how to perform a task |
| **Audience** | Developers, product team | Operators, developers, users |
| **Lifecycle** | Evolves with feature | Updated as needed |
| **Format** | Comprehensive (design, code, tests) | Task-focused (steps, examples) |
| **Example** | FEAT-002: Email Accounts | Guide-Creating_Email_Accounts.md |

**Rule of thumb:**
- Feature doc = WHY and WHAT
- Guide = HOW

---

## Related Documentation

- [Document 30: Feature Documentation System](../30-feature-documentation-system.md)
- [Feature Status Dashboard](../features/README.md)
- [Architecture Decisions (ADRs)](../adrs/README.md)
