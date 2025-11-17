# Feature Documentation System

**Document 30** | Architecture & Development Process
**Last Updated:** 2025-11-17
**Status:** ✅ Production Ready
**Reading Time:** 25 minutes

---

## Table of Contents

1. [Overview](#overview)
2. [Document Types](#document-types)
3. [Directory Structure](#directory-structure)
4. [Lifecycle Stages](#lifecycle-stages)
5. [Naming Conventions](#naming-conventions)
6. [Document Templates](#document-templates)
7. [Integration with Existing Docs](#integration-with-existing-docs)
8. [Feature Status Tracking](#feature-status-tracking)
9. [Multi-Tenant Considerations](#multi-tenant-considerations)
10. [Examples](#examples)
11. [Best Practices](#best-practices)
12. [Quick Reference](#quick-reference)

---

## Overview

This document defines the **Feature Documentation System** for the my.refine.digital multi-tenant Kubernetes platform. It provides a structured approach to documenting features from initial design through production deployment and maintenance.

### Goals

1. **Traceability**: Track features from conception to production
2. **Clarity**: Make feature status and context visible to all team members
3. **Integration**: Seamlessly integrate with existing documentation (01-25)
4. **Efficiency**: Enable rapid feature development without documentation overhead
5. **Compliance**: Support audit requirements for multi-tenant environments

### Key Principles

- **Documentation as Code**: Feature docs live in version control alongside code
- **Lifecycle-Aware**: Document structure reflects feature development stages
- **Tier-Aware**: Clearly indicate FREE vs PREMIUM tier features
- **Decision Records**: Preserve context and rationale for architectural choices
- **Living Documents**: Documentation evolves with the feature

---

## Document Types

We use **four document types** for feature documentation:

### 1. RFC (Request for Comments)

**When to use:**
- New features requiring cross-team feedback
- Significant architectural changes
- Features affecting multiple tiers (FREE/PREMIUM)
- Breaking changes to existing APIs

**Lifecycle:** Draft → Review → Accepted/Rejected → Archived
**Location:** `docs/rfcs/`
**Retention:** Permanent (archived when implemented)

**Example:** `RFC-001-email-quota-enforcement.md`

### 2. ADR (Architecture Decision Record)

**When to use:**
- Technical implementation decisions
- Technology/library selection choices
- Design pattern selections
- Database schema changes

**Lifecycle:** Proposed → Accepted → Superseded
**Location:** `docs/adrs/`
**Retention:** Permanent (marked as superseded when replaced)

**Example:** `ADR-023-use-dovecot-for-imap.md`

### 3. Feature Documentation

**When to use:**
- All features in active development
- Features in testing or staged rollout
- Production features requiring ongoing documentation

**Lifecycle:** Draft → In Development → Testing → Deployed → Production
**Location:** `docs/features/`
**Retention:** Permanent (updated as feature evolves)

**Example:** `FEAT-042-domain-verification-wizard.md`

### 4. Implementation Guides

**When to use:**
- Operational procedures for production features
- Developer setup/configuration guides
- Troubleshooting guides

**Lifecycle:** Draft → Review → Published
**Location:** `docs/guides/`
**Retention:** Permanent (updated as needed)

**Example:** `Guide-Setting_up_Email_Forwarding.md`

---

## Directory Structure

```
docs/
│
├── README.md                          # Main hub (updated with feature index)
│
├── 01-11/                            # Core architecture docs (existing)
├── 20-25/                            # Platform research docs (existing)
├── 30-39/                            # Process & system docs (this doc = 30)
│
├── rfcs/                             # Request for Comments
│   ├── README.md                     # RFC index with status table
│   ├── template.md                   # RFC template
│   ├── RFC-001-feature-name.md       # Accepted RFCs
│   ├── RFC-002-feature-name.md
│   └── archived/                     # Implemented RFCs
│       └── RFC-001-email-quota.md
│
├── adrs/                             # Architecture Decision Records
│   ├── README.md                     # ADR index with status
│   ├── template.md                   # ADR template
│   ├── ADR-001-k3s-over-k8s.md       # Sequential numbering
│   ├── ADR-002-postgresql-over-mysql.md
│   └── superseded/                   # Superseded ADRs
│       └── ADR-015-old-approach.md
│
├── features/                         # Feature Documentation
│   ├── README.md                     # Feature index with status dashboard
│   ├── template.md                   # Feature doc template
│   │
│   ├── free-tier/                    # FREE tier features
│   │   ├── FEAT-001-domain-management.md
│   │   ├── FEAT-002-email-accounts.md
│   │   └── FEAT-003-dns-wizard.md
│   │
│   ├── premium-tier/                 # PREMIUM tier features
│   │   ├── FEAT-101-k3s-provisioning.md
│   │   ├── FEAT-102-auto-scaling.md
│   │   └── FEAT-103-multi-region.md
│   │
│   ├── platform/                     # Platform-wide features
│   │   ├── FEAT-201-user-auth.md
│   │   ├── FEAT-202-billing-system.md
│   │   └── FEAT-203-migration-engine.md
│   │
│   └── deprecated/                   # Deprecated features
│       └── FEAT-050-old-feature.md
│
└── guides/                           # Implementation Guides
    ├── README.md                     # Guide index
    ├── operations/                   # Operations guides
    │   ├── Guide-Monitoring_Email_Queues.md
    │   └── Guide-K3s_Cluster_Backup.md
    │
    ├── development/                  # Developer guides
    │   ├── Guide-Local_Development_Setup.md
    │   └── Guide-Testing_Migrations.md
    │
    └── troubleshooting/              # Troubleshooting guides
        ├── Guide-Email_Delivery_Issues.md
        └── Guide-K3s_Node_Not_Ready.md
```

---

## Lifecycle Stages

Features progress through **seven lifecycle stages**, each with specific documentation requirements:

### Stage 1: Proposal (RFC)

**Status:** `📋 DRAFT`
**Documents Required:**
- RFC document (if significant feature)
- Initial feature doc with problem statement

**Activities:**
- Problem definition
- Success criteria
- Cross-team review
- Approval decision

**Duration:** 1-2 weeks

### Stage 2: Design (ADR)

**Status:** `🎨 DESIGN`
**Documents Required:**
- ADR(s) for key decisions
- Feature doc with technical design
- API/schema specifications

**Activities:**
- Architecture decisions
- Technology selection
- Design review
- Prototype if needed

**Duration:** 1-3 weeks

### Stage 3: Development

**Status:** `🔨 IN DEVELOPMENT`
**Documents Required:**
- Feature doc with implementation details
- API documentation
- Code comments

**Activities:**
- Implementation
- Unit tests
- Code review
- Documentation updates

**Duration:** 2-8 weeks (varies by complexity)

### Stage 4: Testing

**Status:** `🧪 TESTING`
**Documents Required:**
- Test plan in feature doc
- Known issues log
- QA signoff checklist

**Activities:**
- Integration testing
- User acceptance testing
- Performance testing
- Security review

**Duration:** 1-2 weeks

### Stage 5: Staged Rollout

**Status:** `🚀 DEPLOYING`
**Documents Required:**
- Rollout plan
- Monitoring dashboard
- Rollback procedure

**Activities:**
- Deploy to staging
- Feature flag configuration
- Gradual rollout (1% → 10% → 50% → 100%)
- Monitor metrics

**Duration:** 1-4 weeks

### Stage 6: Production

**Status:** `✅ PRODUCTION`
**Documents Required:**
- Complete feature documentation
- Operations guide
- Troubleshooting guide

**Activities:**
- 100% rollout
- Production monitoring
- User feedback collection
- Documentation finalization

**Duration:** Ongoing

### Stage 7: Deprecated

**Status:** `⚠️ DEPRECATED`
**Documents Required:**
- Deprecation notice
- Migration guide
- Sunset timeline

**Activities:**
- Deprecation announcement
- User migration support
- Feature removal
- Documentation archival

**Duration:** 3-12 months

---

## Naming Conventions

### RFC Naming

**Format:** `RFC-NNN-brief-description.md`

- `NNN`: Sequential number (001, 002, ...)
- `brief-description`: Lowercase with hyphens
- Maximum 50 characters total

**Examples:**
```
RFC-001-email-quota-enforcement.md
RFC-002-multi-region-deployment.md
RFC-003-premium-tier-pricing-model.md
```

### ADR Naming

**Format:** `ADR-NNN-decision-statement.md`

- `NNN`: Sequential number (001, 002, ...)
- `decision-statement`: What was decided (use X over Y)
- Maximum 60 characters total

**Examples:**
```
ADR-001-use-k3s-over-k8s.md
ADR-002-postgresql-over-mysql.md
ADR-003-argon2id-password-hashing.md
ADR-004-dovecot-maildir-over-mbox.md
```

### Feature Documentation Naming

**Format:** `FEAT-NNN-feature-name.md`

- `NNN`: Sequential number by tier
  - `001-099`: FREE tier features
  - `100-199`: PREMIUM tier features
  - `200-299`: Platform features (both tiers)
- `feature-name`: Lowercase with hyphens
- Maximum 50 characters total

**Examples:**
```
FEAT-001-domain-management.md           # FREE tier
FEAT-101-k3s-provisioning.md            # PREMIUM tier
FEAT-201-user-authentication.md         # Platform
```

### Guide Naming

**Format:** `Guide-Action_Description.md`

- `Action`: Verb (Setting_up, Monitoring, Troubleshooting, etc.)
- `Description`: What is being acted upon
- CamelCase with underscores (matches existing guides)

**Examples:**
```
Guide-Setting_up_Email_Forwarding.md
Guide-Monitoring_K3s_Clusters.md
Guide-Troubleshooting_DNS_Issues.md
```

---

## Document Templates

### RFC Template

```markdown
# RFC-NNN: [Feature Name]

**Status:** Draft | In Review | Accepted | Rejected | Implemented
**Author:** [Name]
**Created:** YYYY-MM-DD
**Last Updated:** YYYY-MM-DD
**Tier:** FREE | PREMIUM | Platform
**Related ADRs:** ADR-XXX, ADR-YYY
**Related Features:** FEAT-XXX, FEAT-YYY

---

## Summary

[One paragraph explaining what this RFC proposes]

## Motivation

### Problem Statement
[What problem are we solving?]

### User Impact
[Who benefits and how?]

### Business Value
[Why is this important for the business?]

## Proposal

### High-Level Design
[Architecture overview with diagrams]

### User Experience
[How users interact with this feature]

### Technical Approach
[High-level technical implementation]

## Alternatives Considered

### Alternative 1: [Name]
**Pros:** ...
**Cons:** ...
**Reason for rejection:** ...

### Alternative 2: [Name]
**Pros:** ...
**Cons:** ...
**Reason for rejection:** ...

## Implementation Plan

### Phase 1: [Name] (Duration)
- Task 1
- Task 2

### Phase 2: [Name] (Duration)
- Task 1
- Task 2

## Success Metrics

| Metric | Target | Measurement Method |
|--------|--------|--------------------|
| ... | ... | ... |

## Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| ... | High/Med/Low | High/Med/Low | ... |

## Open Questions

1. Question 1?
2. Question 2?

## Dependencies

- System X
- Team Y
- External service Z

## Timeline

- RFC Draft: YYYY-MM-DD
- Review Period: YYYY-MM-DD to YYYY-MM-DD
- Decision: YYYY-MM-DD
- Implementation Start: YYYY-MM-DD

## Feedback

### [Name] - YYYY-MM-DD
[Feedback content]

### [Name] - YYYY-MM-DD
[Feedback content]

## Decision

**Status:** [Accepted/Rejected]
**Date:** YYYY-MM-DD
**Rationale:** [Why this decision was made]

---

**Changelog:**
- YYYY-MM-DD: Initial draft
- YYYY-MM-DD: Updated based on feedback
- YYYY-MM-DD: Accepted
```

### ADR Template

```markdown
# ADR-NNN: [Decision Statement]

**Status:** Proposed | Accepted | Superseded by ADR-XXX | Deprecated
**Date:** YYYY-MM-DD
**Deciders:** [Names of decision makers]
**Related RFCs:** RFC-XXX
**Related Features:** FEAT-XXX

---

## Context

[Describe the context and problem statement. What forces are at play?]

### Technical Context
[Technical background needed to understand the decision]

### Constraints
- Constraint 1
- Constraint 2

### Assumptions
- Assumption 1
- Assumption 2

## Decision Drivers

- Driver 1 (e.g., performance requirements)
- Driver 2 (e.g., cost constraints)
- Driver 3 (e.g., developer experience)

## Options Considered

### Option 1: [Name]

**Description:** [How would this work?]

**Pros:**
- Pro 1
- Pro 2

**Cons:**
- Con 1
- Con 2

**Cost:** [Development time, operational cost, etc.]

### Option 2: [Name]

**Description:** [How would this work?]

**Pros:**
- Pro 1
- Pro 2

**Cons:**
- Con 1
- Con 2

**Cost:** [Development time, operational cost, etc.]

### Option 3: [Name]

**Description:** [How would this work?]

**Pros:**
- Pro 1
- Pro 2

**Cons:**
- Con 1
- Con 2

**Cost:** [Development time, operational cost, etc.]

## Decision

**Chosen Option:** [Option X]

**Rationale:** [Why was this option selected? How does it address the decision drivers?]

## Consequences

### Positive
- Consequence 1
- Consequence 2

### Negative
- Consequence 1
- Consequence 2

### Neutral
- Consequence 1
- Consequence 2

## Implementation Notes

### Migration Path
[If replacing existing approach, how do we migrate?]

### Code Locations
- File/module 1
- File/module 2

### Configuration Changes
```yaml
# Example configuration
```

### Dependencies
- New dependency 1
- New dependency 2

## Validation

### Success Criteria
- [ ] Criterion 1
- [ ] Criterion 2

### Testing Strategy
[How will we verify this decision was correct?]

## References

- [Link to relevant documentation]
- [Link to research/benchmarks]

---

**Changelog:**
- YYYY-MM-DD: Initial decision
- YYYY-MM-DD: Updated based on implementation learnings
```

### Feature Documentation Template

```markdown
# FEAT-NNN: [Feature Name]

**Status:** 📋 Draft | 🎨 Design | 🔨 In Development | 🧪 Testing | 🚀 Deploying | ✅ Production | ⚠️ Deprecated
**Tier:** FREE | PREMIUM | Platform
**Owner:** [Team/Person]
**Created:** YYYY-MM-DD
**Last Updated:** YYYY-MM-DD
**Target Release:** vX.Y.Z
**Related RFCs:** RFC-XXX
**Related ADRs:** ADR-XXX, ADR-YYY

---

## Quick Links

- **GitHub Epic:** #XXXX
- **Figma Designs:** [Link]
- **API Docs:** [Link]
- **Staging URL:** https://staging.example.com/feature
- **Production URL:** https://app.example.com/feature
- **Monitoring Dashboard:** [Link]

---

## Overview

### One-Line Summary
[Single sentence describing what this feature does]

### Problem Statement
[What problem does this solve? What pain point does it address?]

### User Stories

**As a** [persona]
**I want to** [action]
**So that** [benefit]

**As a** [persona]
**I want to** [action]
**So that** [benefit]

### Success Criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

---

## Technical Design

### Architecture Overview

```
[Architecture diagram - ASCII or link to diagram]
```

### Components

#### Component 1: [Name]

**Purpose:** [What does this component do?]
**Technology:** [Language/framework/library]
**Location:** `src/path/to/component`

**Key Files:**
- `file1.ts` - [Purpose]
- `file2.ts` - [Purpose]

#### Component 2: [Name]

**Purpose:** [What does this component do?]
**Technology:** [Language/framework/library]
**Location:** `src/path/to/component`

### Data Model

```sql
-- Database schema changes
CREATE TABLE example (
    id SERIAL PRIMARY KEY,
    field1 VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);
```

### API Endpoints

#### POST /api/v1/example

**Purpose:** [What this endpoint does]

**Authentication:** JWT required
**Rate Limit:** 100 requests/minute

**Request:**
```json
{
  "field1": "value",
  "field2": 123
}
```

**Response (200 OK):**
```json
{
  "id": 42,
  "field1": "value",
  "created_at": "2025-11-17T10:30:00Z"
}
```

**Errors:**
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Missing/invalid token
- `429 Too Many Requests`: Rate limit exceeded

### State Management

```typescript
// Redux/state management structure
interface FeatureState {
  data: ExampleData[];
  loading: boolean;
  error: string | null;
}
```

### Integration Points

| System | Integration Type | Purpose |
|--------|------------------|---------|
| Hetzner API | REST | Provision servers |
| PostgreSQL | Direct | Data persistence |
| Redis | Direct | Caching |

---

## User Experience

### UI Mockups

[Link to Figma or embedded screenshots]

### User Flow

1. User navigates to [location]
2. User clicks [action]
3. System displays [response]
4. User completes [task]

### Mobile Considerations

- Touch target sizes: minimum 44x44px
- Responsive breakpoints: 320px, 768px, 1024px
- Offline support: [Yes/No - details]

### Accessibility

- WCAG Level: AA
- Screen reader support: [Details]
- Keyboard navigation: [Details]

---

## Implementation Plan

### Phase 1: Foundation (Week 1-2)

**Status:** ✅ Complete | 🔨 In Progress | 📋 Pending

- [ ] Database schema creation
- [ ] API endpoint scaffolding
- [ ] Basic UI components

**Blockers:** None

### Phase 2: Core Logic (Week 3-4)

**Status:** ✅ Complete | 🔨 In Progress | 📋 Pending

- [ ] Business logic implementation
- [ ] Integration with external services
- [ ] State management

**Blockers:** Waiting on API key from vendor

### Phase 3: Testing & Polish (Week 5-6)

**Status:** ✅ Complete | 🔨 In Progress | 📋 Pending

- [ ] Unit tests (target: 80% coverage)
- [ ] Integration tests
- [ ] E2E tests
- [ ] Performance optimization

**Blockers:** None

---

## Testing Strategy

### Unit Tests

**Location:** `src/__tests__/feature/`
**Coverage Target:** 80%
**Status:** [X/Y tests passing]

**Key Test Cases:**
- Test case 1
- Test case 2

### Integration Tests

**Location:** `tests/integration/feature/`
**Status:** [X/Y tests passing]

**Test Scenarios:**
- Scenario 1
- Scenario 2

### E2E Tests

**Location:** `tests/e2e/feature/`
**Tool:** Playwright
**Status:** [X/Y tests passing]

**User Journeys:**
- Journey 1: [Description]
- Journey 2: [Description]

### Performance Tests

**Tool:** k6
**Benchmarks:**
- Response time < 200ms (p95)
- Throughput > 1000 req/s
- Memory usage < 500MB

**Results:** [Link to performance report]

---

## Deployment

### Feature Flags

**Flag Name:** `feature_example_enabled`
**Default:** `false`
**Environments:**
- Development: `true`
- Staging: `true`
- Production: `false` (gradual rollout)

### Rollout Plan

#### Stage 1: Internal Testing (Week 1)
- **Audience:** Internal team (10 users)
- **Flag:** `50%` for team members
- **Monitoring:** Error rate, response time
- **Success Criteria:** < 1% error rate

#### Stage 2: Beta Users (Week 2)
- **Audience:** Opt-in beta users (100 users)
- **Flag:** `100%` for beta users
- **Monitoring:** User feedback, engagement metrics
- **Success Criteria:** > 70% positive feedback

#### Stage 3: Gradual Rollout (Week 3-4)
- **Audience:** All users
- **Schedule:**
  - Day 1: 1% of users
  - Day 3: 10% of users
  - Day 7: 50% of users
  - Day 14: 100% of users
- **Monitoring:** All metrics
- **Rollback Trigger:** > 5% error rate OR > 50% negative feedback

### Rollback Procedure

1. Set feature flag to `false` in production
2. Verify rollback successful (< 2 minutes)
3. Investigate root cause
4. Create hotfix if needed
5. Redeploy when ready

**Estimated Rollback Time:** < 5 minutes

---

## Monitoring & Observability

### Metrics

| Metric | Target | Alert Threshold | Dashboard |
|--------|--------|-----------------|-----------|
| Error rate | < 1% | > 5% | [Link] |
| Response time (p95) | < 200ms | > 500ms | [Link] |
| User engagement | > 60% | < 30% | [Link] |

### Logs

**Location:** Loki
**Query Examples:**
```logql
{app="api"} |= "feature_example" | json
```

### Alerts

#### High Error Rate
- **Condition:** Error rate > 5% for 5 minutes
- **Severity:** Critical
- **Notification:** PagerDuty + Slack #alerts
- **Action:** Rollback feature flag

#### Slow Response Time
- **Condition:** p95 > 500ms for 10 minutes
- **Severity:** Warning
- **Notification:** Slack #monitoring
- **Action:** Investigate performance

---

## Security Considerations

### Authentication & Authorization

- **Authentication Method:** JWT
- **Required Permissions:** `feature:read`, `feature:write`
- **FREE Tier:** Read-only access
- **PREMIUM Tier:** Full access

### Data Privacy

- **PII Handling:** [Details on what PII is collected/stored]
- **GDPR Compliance:** [Retention policy, deletion procedure]
- **Encryption:** TLS 1.3 in transit, AES-256 at rest

### Threat Model

| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|------------|
| SQL Injection | Low | High | Parameterized queries |
| XSS | Medium | Medium | Input sanitization |
| CSRF | Low | Medium | CSRF tokens |

### Security Testing

- [ ] OWASP Top 10 review
- [ ] Dependency vulnerability scan (Snyk)
- [ ] Penetration testing (if applicable)

---

## Cost Analysis

### Development Cost

| Resource | Hours | Rate | Total |
|----------|-------|------|-------|
| Backend Dev | 40h | $100/h | $4,000 |
| Frontend Dev | 32h | $100/h | $3,200 |
| Designer | 8h | $80/h | $640 |
| QA | 16h | $60/h | $960 |
| **Total** | **96h** | | **$8,800** |

### Operational Cost

**Monthly (at 10,000 users):**
- Database: $50/month (additional storage)
- Redis: $20/month (additional memory)
- Compute: $100/month (additional processing)
- Monitoring: $10/month (additional logs)
- **Total:** $180/month

**Revenue Impact:**
- Expected conversion increase: +5%
- Monthly revenue increase: $2,000
- **ROI:** 5 months to break even

---

## Documentation

### User Documentation

- [ ] Help article: [Link]
- [ ] Video tutorial: [Link]
- [ ] In-app tooltip content
- [ ] Email announcement draft

### Developer Documentation

- [ ] API reference updated
- [ ] Code comments added
- [ ] Architecture diagram in wiki
- [ ] Runbook for operations

### Migration Guide

**From:** [Previous approach]
**To:** [New approach]

**Steps:**
1. Step 1
2. Step 2
3. Step 3

**Backward Compatibility:** [Yes/No - details]

---

## Support & Maintenance

### Known Issues

| Issue | Severity | Workaround | Target Fix |
|-------|----------|------------|------------|
| [Description] | High/Med/Low | [Details] | vX.Y.Z |

### FAQ

**Q: [Question]?**
A: [Answer]

**Q: [Question]?**
A: [Answer]

### Contact

- **Feature Owner:** [Name] (@username)
- **Tech Lead:** [Name] (@username)
- **Product Manager:** [Name] (@username)

---

## Retrospective

### What Went Well

- Item 1
- Item 2

### What Could Be Improved

- Item 1
- Item 2

### Action Items

- [ ] Action 1
- [ ] Action 2

---

## Appendix

### Research & References

- [Link to research]
- [Link to competitor analysis]

### Related Features

- FEAT-XXX: [Name]
- FEAT-YYY: [Name]

### Historical Context

[Any background information that provides useful context]

---

**Changelog:**
- YYYY-MM-DD: Initial draft
- YYYY-MM-DD: Design review completed
- YYYY-MM-DD: Implementation started
- YYYY-MM-DD: Testing completed
- YYYY-MM-DD: Deployed to production
```

---

## Integration with Existing Docs

### Update README.md

Add new section after Topic Index:

```markdown
## Feature Status Dashboard

Quick view of features in active development:

| Feature | Status | Tier | Target Release | Owner |
|---------|--------|------|----------------|-------|
| [Domain Verification Wizard](./features/free-tier/FEAT-003-dns-wizard.md) | 🔨 In Development | FREE | v1.2.0 | @alice |
| [Auto-Scaling](./features/premium-tier/FEAT-102-auto-scaling.md) | 🧪 Testing | PREMIUM | v1.3.0 | @bob |
| [Migration Engine](./features/platform/FEAT-203-migration-engine.md) | ✅ Production | Platform | v1.1.0 | @carol |

[View all features →](./features/README.md)

## Architecture Decisions

Recent architecture decisions:

| ADR | Decision | Date | Status |
|-----|----------|------|--------|
| [ADR-004](./adrs/ADR-004-dovecot-maildir-over-mbox.md) | Use Maildir over mbox format | 2025-11-10 | ✅ Accepted |
| [ADR-003](./adrs/ADR-003-argon2id-password-hashing.md) | Use Argon2id for passwords | 2025-11-05 | ✅ Accepted |

[View all ADRs →](./adrs/README.md)
```

### Update Document Navigation

Create bidirectional links:

1. **From numbered docs (01-25) to features:**
   - In `20-email-infrastructure-postfix-dovecot.md`, add:
     ```markdown
     ## Related Features
     - [FEAT-001: Domain Management](../features/free-tier/FEAT-001-domain-management.md)
     - [FEAT-002: Email Accounts](../features/free-tier/FEAT-002-email-accounts.md)
     ```

2. **From features to architecture docs:**
   - In `FEAT-101-k3s-provisioning.md`, add:
     ```markdown
     ## Architecture References
     - [Document 21: Packer k3s Image Strategy](../../21-packer-k3s-image-strategy.md)
     - [ADR-001: Use k3s over k8s](../../adrs/ADR-001-use-k3s-over-k8s.md)
     ```

3. **From RFCs to ADRs:**
   - RFCs link to implementing ADRs
   - ADRs reference originating RFC

---

## Feature Status Tracking

### Status Badge System

Use consistent emoji badges across all feature documentation:

| Stage | Badge | Meaning |
|-------|-------|---------|
| Draft | 📋 DRAFT | Initial proposal, not yet approved |
| Design | 🎨 DESIGN | Architecture design in progress |
| In Development | 🔨 IN DEVELOPMENT | Active coding work |
| Testing | 🧪 TESTING | In QA/testing phase |
| Deploying | 🚀 DEPLOYING | Staged rollout in progress |
| Production | ✅ PRODUCTION | Fully deployed and available |
| Deprecated | ⚠️ DEPRECATED | End-of-life, migration required |

### Feature Status Dashboard

Create `docs/features/README.md` with auto-generated status:

```markdown
# Feature Status Dashboard

Last Updated: 2025-11-17 10:30 UTC

## By Status

### 🔨 In Development (4)
- [FEAT-003: DNS Wizard](./free-tier/FEAT-003-dns-wizard.md) - FREE - Target: v1.2.0
- [FEAT-104: Backup System](./premium-tier/FEAT-104-backup-system.md) - PREMIUM - Target: v1.3.0
- [FEAT-205: Audit Logging](./platform/FEAT-205-audit-logging.md) - Platform - Target: v1.2.0
- [FEAT-206: SSO Integration](./platform/FEAT-206-sso-integration.md) - Platform - Target: v1.4.0

### 🧪 Testing (2)
- [FEAT-102: Auto-Scaling](./premium-tier/FEAT-102-auto-scaling.md) - PREMIUM - Target: v1.3.0
- [FEAT-204: Usage Analytics](./platform/FEAT-204-usage-analytics.md) - Platform - Target: v1.2.0

### 🚀 Deploying (1)
- [FEAT-103: Multi-Region](./premium-tier/FEAT-103-multi-region.md) - PREMIUM - Target: v1.3.0

### ✅ Production (8)
- [FEAT-001: Domain Management](./free-tier/FEAT-001-domain-management.md) - FREE - Released: v1.0.0
- [FEAT-002: Email Accounts](./free-tier/FEAT-002-email-accounts.md) - FREE - Released: v1.0.0
- [FEAT-101: K3s Provisioning](./premium-tier/FEAT-101-k3s-provisioning.md) - PREMIUM - Released: v1.1.0
- [FEAT-201: User Auth](./platform/FEAT-201-user-auth.md) - Platform - Released: v1.0.0
- [FEAT-202: Billing System](./platform/FEAT-202-billing-system.md) - Platform - Released: v1.0.0
- [FEAT-203: Migration Engine](./platform/FEAT-203-migration-engine.md) - Platform - Released: v1.1.0
- [And 2 more...](./all-features.md)

## By Tier

| FREE Tier | PREMIUM Tier | Platform |
|-----------|--------------|----------|
| 3 features | 4 features | 6 features |
| 2 production | 1 production | 5 production |
| 1 in dev | 2 in dev | 1 in dev |

## By Target Release

### v1.2.0 (Next Release - 2025-12-01)
- FEAT-003: DNS Wizard (FREE)
- FEAT-205: Audit Logging (Platform)
- FEAT-204: Usage Analytics (Platform)

### v1.3.0 (2025-12-15)
- FEAT-102: Auto-Scaling (PREMIUM)
- FEAT-103: Multi-Region (PREMIUM)
- FEAT-104: Backup System (PREMIUM)

### v1.4.0 (2026-01-15)
- FEAT-206: SSO Integration (Platform)

## Metrics

- **Total Features:** 13
- **Production Features:** 8 (61.5%)
- **In Development:** 4 (30.8%)
- **Testing:** 2 (15.4%)
- **Average Time to Production:** 8 weeks
- **Feature Success Rate:** 95% (based on adoption metrics)
```

### Automation Script

Create `scripts/update-feature-dashboard.sh`:

```bash
#!/bin/bash
# Auto-generate feature status dashboard

FEATURES_DIR="docs/features"
OUTPUT_FILE="$FEATURES_DIR/README.md"

echo "# Feature Status Dashboard" > "$OUTPUT_FILE"
echo "" >> "$OUTPUT_FILE"
echo "Last Updated: $(date -u +"%Y-%m-%d %H:%M UTC")" >> "$OUTPUT_FILE"
echo "" >> "$OUTPUT_FILE"

# Extract status from each feature file and build dashboard
# (Implementation details omitted for brevity)
```

Run this script:
- Automatically via GitHub Actions on every commit to `main`
- Manually via `npm run update-dashboard`

---

## Multi-Tenant Considerations

### Tier-Specific Documentation

Features must clearly indicate which tier(s) they support:

#### FREE Tier Features (FEAT-001 to FEAT-099)

**Required Documentation Sections:**
- **Resource Limits**: Document shared resource constraints
- **Scaling Considerations**: How feature performs under high load
- **Isolation**: How data is isolated between FREE tier clients
- **Migration Path**: How FREE users upgrade to PREMIUM

**Example:**
```markdown
## FREE Tier Constraints

- **Max Domains per Client:** 5
- **Max Email Accounts:** 25
- **Storage Quota:** 5GB
- **API Rate Limit:** 100 req/min

## Upgrade Path

When FREE tier client upgrades to PREMIUM:
1. All domains migrated automatically
2. Email accounts preserved (no limit)
3. Storage quota increased to 50GB
4. Rate limits removed
```

#### PREMIUM Tier Features (FEAT-100 to FEAT-199)

**Required Documentation Sections:**
- **Dedicated Resources**: What dedicated infrastructure is provisioned
- **Isolation**: How tenant resources are isolated
- **Scaling**: Auto-scaling behavior and limits
- **Cost**: Per-tenant infrastructure cost

**Example:**
```markdown
## PREMIUM Tier Resources

**Per Client:**
- Dedicated k3s cluster (1-5 nodes)
- Dedicated PostgreSQL database
- Dedicated Redis instance
- Private network (10.0.0.0/16)

## Resource Isolation

- Network: Dedicated Hetzner Cloud Network per client
- Compute: Dedicated servers (no resource sharing)
- Storage: Dedicated volumes
- Firewall: Client-specific firewall rules
- Labels: `cluster=client-{id}` for resource targeting

## Cost Tracking

Infrastructure costs tracked per client via Hetzner labels:
```bash
hcloud server list -l cluster=client-42 -o json | \
  jq '.[] | .server_type.prices[] | select(.location=="fsn1") | .price_monthly.gross'
```
```

#### Platform Features (FEAT-200 to FEAT-299)

**Required Documentation Sections:**
- **Tier Differences**: How feature behaves differently per tier
- **Data Model**: How multi-tenancy is implemented in data layer
- **Performance**: Impact of tenant count on performance
- **Security**: Tenant isolation verification

**Example:**
```markdown
## Multi-Tenant Data Model

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL,
    email VARCHAR(255) NOT NULL,
    tier VARCHAR(20) NOT NULL CHECK (tier IN ('FREE', 'PREMIUM')),
    FOREIGN KEY (client_id) REFERENCES clients(id),
    UNIQUE (client_id, email)
);

CREATE INDEX idx_users_client_id ON users(client_id);
CREATE INDEX idx_users_tier ON users(tier);
```

## Tier-Specific Behavior

| Feature Aspect | FREE Tier | PREMIUM Tier |
|----------------|-----------|--------------|
| API Rate Limit | 100/min | Unlimited |
| Storage Quota | 5GB | 50GB |
| Support Level | Community | Priority (24/7) |
| SLA | None | 99.9% uptime |
```

### Security Documentation

**Required for ALL features:**

```markdown
## Tenant Isolation Security

### Database Queries

All queries MUST include client_id in WHERE clause:

✅ **Correct:**
```sql
SELECT * FROM domains WHERE client_id = $1 AND id = $2;
```

❌ **Incorrect:**
```sql
SELECT * FROM domains WHERE id = $1;  -- Missing client_id check!
```

### API Endpoints

All endpoints MUST verify client_id from JWT token:

```typescript
export async function getDomain(req: Request, res: Response) {
  const { client_id } = req.user; // From JWT
  const { domain_id } = req.params;

  // MUST include client_id in query
  const domain = await db.query(
    'SELECT * FROM domains WHERE client_id = $1 AND id = $2',
    [client_id, domain_id]
  );

  if (!domain) {
    return res.status(404).json({ error: 'Domain not found' });
  }

  res.json(domain);
}
```

### Testing Isolation

Every feature MUST include tenant isolation tests:

```typescript
describe('Tenant Isolation', () => {
  it('should not allow client A to access client B data', async () => {
    const clientA = await createTestClient({ tier: 'FREE' });
    const clientB = await createTestClient({ tier: 'FREE' });

    const domainB = await createDomain({ client_id: clientB.id });

    // Try to access client B's domain as client A
    const response = await request(app)
      .get(`/api/v1/domains/${domainB.id}`)
      .set('Authorization', `Bearer ${clientA.token}`);

    expect(response.status).toBe(404); // Should not find
  });
});
```
```

---

## Examples

### Example 1: RFC for Email Quota Enforcement

**File:** `docs/rfcs/RFC-001-email-quota-enforcement.md`

```markdown
# RFC-001: Email Quota Enforcement

**Status:** Accepted
**Author:** Alice Johnson
**Created:** 2025-11-01
**Last Updated:** 2025-11-10
**Tier:** FREE
**Related ADRs:** ADR-005
**Related Features:** FEAT-002

---

## Summary

Implement automatic email quota enforcement for FREE tier users to prevent abuse and manage shared infrastructure costs. Users approaching their quota will receive warnings, and emails will be rejected once the quota is exceeded.

## Motivation

### Problem Statement

Currently, FREE tier users have a documented 5GB email storage quota, but there's no enforcement. This has led to:
- 15% of FREE tier users exceeding 10GB
- 3% of users exceeding 50GB
- Increased storage costs ($250/month unnecessary spend)
- Degraded performance for other users

### User Impact

**FREE Tier Users:**
- Clear expectations about storage limits
- Warnings before hitting quota
- Fair resource allocation

**PREMIUM Tier Users:**
- Better performance (less resource contention)
- Clear value proposition for upgrade

### Business Value

- Reduce storage costs by ~$200/month
- Drive FREE → PREMIUM conversions (estimated +5%)
- Improve platform reliability

## Proposal

### High-Level Design

```
┌─────────────────────────────────────────────────────┐
│              Dovecot IMAP Server                    │
│                                                      │
│  ┌──────────────────────────────────────────────┐  │
│  │  Quota Plugin (quota_status_check)           │  │
│  └──────────────────────────────────────────────┘  │
│                      ↓                              │
│  ┌──────────────────────────────────────────────┐  │
│  │  PostgreSQL Quota Tracking                   │  │
│  │  - Current usage per user                    │  │
│  │  - Quota limits by tier                      │  │
│  └──────────────────────────────────────────────┘  │
│                      ↓                              │
│  ┌──────────────────────────────────────────────┐  │
│  │  Warning System (BullMQ job)                 │  │
│  │  - 80%: Warning email                        │  │
│  │  - 90%: Final warning                        │  │
│  │  - 100%: Reject new emails                   │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### User Experience

1. **At 80% quota:**
   - Email notification: "You've used 4GB of 5GB"
   - Dashboard banner: "Approaching storage limit"
   - Suggestion to upgrade to PREMIUM

2. **At 90% quota:**
   - Email notification: "Only 500MB remaining"
   - Dashboard modal: Upgrade prompt
   - Option to delete old emails

3. **At 100% quota:**
   - New emails rejected with bounce message
   - Sender receives: "Mailbox full" error
   - User dashboard: Prominent upgrade CTA

### Technical Approach

1. **Dovecot Configuration:**
   ```conf
   plugin {
     quota = count:User quota
     quota_status_success = DUNNO
     quota_status_nouser = DUNNO
     quota_status_overquota = 552 5.2.2 Mailbox is full
   }
   ```

2. **PostgreSQL Schema:**
   ```sql
   CREATE TABLE email_quotas (
       user_id INTEGER PRIMARY KEY,
       quota_bytes BIGINT NOT NULL,
       used_bytes BIGINT DEFAULT 0,
       last_checked TIMESTAMP DEFAULT NOW(),
       FOREIGN KEY (user_id) REFERENCES users(id)
   );
   ```

3. **Daily Quota Check (Cron):**
   ```typescript
   async function checkQuotas() {
     const users = await db.query(`
       SELECT u.id, u.email, u.tier, eq.used_bytes, eq.quota_bytes
       FROM users u
       JOIN email_quotas eq ON u.id = eq.user_id
       WHERE u.tier = 'FREE'
       AND eq.used_bytes >= eq.quota_bytes * 0.8
     `);

     for (const user of users) {
       const percentUsed = (user.used_bytes / user.quota_bytes) * 100;

       if (percentUsed >= 100) {
         await enableQuotaEnforcement(user.id);
       } else if (percentUsed >= 90) {
         await sendFinalWarning(user.id);
       } else if (percentUsed >= 80) {
         await sendWarning(user.id);
       }
     }
   }
   ```

## Alternatives Considered

### Alternative 1: Soft Quota (Warning Only)

**Pros:**
- Better user experience (no hard limits)
- Simpler implementation

**Cons:**
- Doesn't solve the cost problem
- No enforcement mechanism

**Reason for rejection:** Doesn't achieve business goal

### Alternative 2: Hard Delete Old Emails

**Pros:**
- Automatic quota management
- Users never hit limits

**Cons:**
- Data loss (very negative UX)
- Legal/compliance issues
- Customer backlash risk

**Reason for rejection:** Too risky, negative user sentiment

## Implementation Plan

### Phase 1: Monitoring (Week 1)
- Deploy quota tracking without enforcement
- Monitor actual usage patterns
- Validate calculations

### Phase 2: Warnings (Week 2-3)
- Implement warning emails at 80%/90%
- Add dashboard notifications
- A/B test warning messaging

### Phase 3: Enforcement (Week 4)
- Enable hard quota enforcement
- Monitor bounce rates
- Support team training

## Success Metrics

| Metric | Target | Measurement Method |
|--------|--------|--------------------|
| Storage cost reduction | -$200/mo | Hetzner billing |
| FREE → PREMIUM conversion | +5% | Conversion tracking |
| User complaints | < 2% | Support tickets |
| Bounce rate increase | < 1% | Postfix logs |

## Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| User backlash | Medium | High | Clear communication, 2-week notice |
| False positives | Low | Medium | Thorough testing, manual override |
| Performance impact | Low | Low | Index optimization, caching |

## Open Questions

1. Should we offer one-time quota increase for long-time users?
2. What's the upgrade flow from quota warning?
3. Should we allow temporary quota overages (e.g., +10% grace period)?

## Dependencies

- Dovecot quota plugin
- PostgreSQL quota table
- BullMQ for warning jobs
- Email template system

## Timeline

- RFC Draft: 2025-11-01
- Review Period: 2025-11-01 to 2025-11-08
- Decision: 2025-11-10 ✅ **ACCEPTED**
- Implementation Start: 2025-11-15

## Feedback

### Bob Smith (Infrastructure) - 2025-11-03
Suggest adding Redis caching for quota lookups to reduce database load. Every IMAP connection checks quota, could be 1000s of queries/sec.

**Response:** Good catch! Will add Redis caching with 60-second TTL.

### Carol White (Product) - 2025-11-05
Recommend adding upgrade CTA directly in warning emails, not just dashboard. Conversion will be higher if users can upgrade immediately.

**Response:** Agreed, will include "Upgrade to PREMIUM" button in all warning emails.

## Decision

**Status:** Accepted
**Date:** 2025-11-10
**Rationale:**

Quota enforcement is necessary to control costs and ensure fair resource allocation. The phased rollout (monitoring → warnings → enforcement) minimizes risk. Clear communication and upgrade path will prevent user backlash. Expected benefits ($200/mo savings + 5% conversion increase) outweigh implementation cost.

**Next Steps:**
- Create ADR-005 for technical implementation decisions
- Create FEAT-002 feature doc for implementation tracking
- Schedule team kickoff for 2025-11-15

---

**Changelog:**
- 2025-11-01: Initial draft
- 2025-11-03: Added Redis caching based on Bob's feedback
- 2025-11-05: Added upgrade CTA to emails based on Carol's feedback
- 2025-11-10: Accepted - proceeding to implementation
```

### Example 2: ADR for Technology Selection

**File:** `docs/adrs/ADR-004-dovecot-maildir-over-mbox.md`

```markdown
# ADR-004: Use Maildir over mbox Format

**Status:** Accepted
**Date:** 2025-11-10
**Deciders:** Alice Johnson, Bob Smith, Carol White
**Related RFCs:** RFC-001
**Related Features:** FEAT-002

---

## Context

We need to choose an email storage format for Dovecot on the FREE tier shared email infrastructure. The two primary options are:

1. **mbox**: Single file per mailbox folder
2. **Maildir**: One file per message

This decision affects:
- Performance under concurrent access
- Quota enforcement implementation
- Backup/restore procedures
- Storage efficiency
- Multi-tenant reliability

### Technical Context

- Shared Dovecot server serving 1000+ FREE tier clients
- Average 500 emails/user
- Mix of IMAP, POP3, and LMTP access
- PostgreSQL-backed virtual users
- Daily backups required

### Constraints

- Must support concurrent access (multiple devices per user)
- Must integrate with quota system (RFC-001)
- Must handle server crashes gracefully
- Must support atomic message delivery

### Assumptions

- Average message size: 50KB
- Peak concurrent users: 200
- 95% of users have < 1000 messages

## Decision Drivers

- **Reliability**: Data integrity during crashes
- **Performance**: Multi-user concurrent access
- **Quota Accuracy**: Easy quota calculation
- **Operations**: Backup and maintenance simplicity
- **Industry Standard**: Established best practices

## Options Considered

### Option 1: mbox Format

**Description:**

Store all messages in a single file per folder (e.g., `/var/mail/user/INBOX`). New messages appended to file. Deletions marked with flags.

**Pros:**
- Simpler file structure (fewer inodes)
- Better for sequential reading (backup)
- Smaller total storage (no per-message overhead)
- Traditional Unix format (well-understood)

**Cons:**
- File locking required (concurrency bottleneck)
- Entire file rewritten on expunge
- Quota calculation requires parsing entire file
- Corruption affects entire mailbox
- Poor performance with large mailboxes (>1000 messages)

**Cost:**
- Development time: 1 week (simpler implementation)
- Operational cost: Same as Maildir

### Option 2: Maildir Format

**Description:**

Store each message as a separate file in `new/`, `cur/`, or `tmp/` subdirectories. Filenames encode message metadata.

**Pros:**
- No file locking needed (lock-free)
- Atomic message delivery (tmp → new)
- Fast quota calculation (sum of file sizes)
- Corruption isolated to single message
- Excellent concurrent access performance
- Easy message deletion (just unlink file)
- Industry standard (most production systems)

**Cons:**
- More inodes required (1 inode per message)
- Slower sequential backups (many small files)
- Slightly larger storage (per-file filesystem overhead)

**Cost:**
- Development time: 1 week (industry-standard implementation)
- Operational cost: Same as mbox

### Option 3: mdbox (Hybrid Approach)

**Description:**

Dovecot's hybrid format: multiple messages per file, but not all messages in one file.

**Pros:**
- Balance between mbox and Maildir
- Better storage efficiency than Maildir
- Better concurrent access than mbox

**Cons:**
- Dovecot-specific (not standard)
- More complex implementation
- Requires doveadm for maintenance
- Less tooling support

**Cost:**
- Development time: 2 weeks (more complex)
- Operational cost: Same as others

## Decision

**Chosen Option:** Maildir (Option 2)

**Rationale:**

Maildir is the clear choice for our multi-tenant shared infrastructure:

1. **Concurrent Access**: With 200+ concurrent users, lock-free access is critical. mbox file locking would create bottlenecks.

2. **Quota Enforcement**: RFC-001 requires accurate real-time quota tracking. Maildir's simple "sum file sizes" approach is fast and accurate. mbox would require expensive file parsing.

3. **Reliability**: In shared infrastructure, crashes happen. Maildir's atomic delivery (tmp → new) ensures no data loss. mbox corruption affects entire mailbox.

4. **Industry Standard**: 90%+ of production Dovecot deployments use Maildir. Extensive documentation, tooling, and community knowledge.

5. **Performance**: Benchmarks show Maildir is 3x faster for concurrent IMAP access (our primary use case).

The inode overhead (~500 inodes/user × 1000 users = 500k inodes) is acceptable on modern filesystems (ext4 supports millions of inodes).

## Consequences

### Positive

- **Fast quota checks**: `du -s /path/to/maildir` is instant
- **Atomic delivery**: No partial message delivery on crash
- **Easy maintenance**: Standard tools work (rsync, rm, etc.)
- **Scalability**: Lock-free = better concurrent performance
- **Debugging**: Individual messages easy to inspect/recover

### Negative

- **Inode usage**: 500 inodes/user (mitigated by modern filesystems)
- **Backup time**: Many small files slower than single large file (mitigated by parallel rsync)
- **Filesystem overhead**: ~1-2% storage overhead (acceptable trade-off)

### Neutral

- **Standard implementation**: No custom code needed (use Dovecot defaults)
- **Migration path**: Can migrate mbox → Maildir with doveadm

## Implementation Notes

### Migration Path

Not applicable - new implementation (no existing email storage).

### Code Locations

**Dovecot Configuration:** `/etc/dovecot/conf.d/10-mail.conf`
```conf
mail_location = maildir:~/Maildir
```

**Directory Structure:**
```
/var/vmail/
└── domain.com/
    └── user@domain.com/
        ├── cur/       # Current messages
        ├── new/       # New unread messages
        ├── tmp/       # Temp during delivery
        └── .Sent/     # Subfolder (sent mail)
            ├── cur/
            ├── new/
            └── tmp/
```

### Configuration Changes

**Dovecot 10-mail.conf:**
```conf
mail_location = maildir:~/Maildir
mail_uid = vmail
mail_gid = vmail
first_valid_uid = 5000
```

**LDA/LMTP Configuration:**
```conf
protocol lmtp {
  mail_plugins = $mail_plugins sieve quota
  postmaster_address = postmaster@refine.digital
}
```

### Dependencies

- **Filesystem**: ext4 with 1M+ inode support
- **Dovecot**: v2.3+ (stable Maildir implementation)
- **Quota Plugin**: Dovecot quota plugin for Maildir
- **Sieve**: Dovecot Sieve for server-side filtering

## Validation

### Success Criteria

- [x] Dovecot configured for Maildir
- [x] Quota tracking accurate within 1%
- [x] Concurrent access test: 200 users, no locking issues
- [x] Crash test: No data loss on unclean shutdown
- [x] Backup test: rsync backup < 30 minutes for 1000 users

### Testing Strategy

**Performance Benchmark:**
```bash
# Test concurrent IMAP access
imaptest user=test pass=test mbox=dovecot.mbox clients=200

# Expected: > 1000 logins/sec, 0 lock conflicts
```

**Quota Accuracy Test:**
```bash
# Compare Dovecot quota vs filesystem
doveadm quota get -u user@domain.com
du -sh /var/vmail/domain.com/user@domain.com/Maildir

# Expected: < 1% difference
```

**Crash Recovery Test:**
```bash
# Send message, kill dovecot mid-delivery
echo "Test" | dovecot-lda -d user@domain.com &
sleep 0.1
killall -9 dovecot

# Expected: Message either fully delivered or not at all (atomic)
```

## References

- [Dovecot Maildir Documentation](https://doc.dovecot.org/admin_manual/mailbox_formats/maildir/)
- [Maildir vs mbox Benchmark](https://wiki2.dovecot.org/MailboxFormat/mbox)
- [Postfix Maildir Integration](https://www.postfix.org/MAILDIR_README.html)

---

**Changelog:**
- 2025-11-10: Initial decision - Maildir selected
```

### Example 3: Feature Documentation

**File:** `docs/features/free-tier/FEAT-002-email-accounts.md`

(This would follow the complete Feature Documentation Template shown earlier, tailored to the email accounts feature)

---

## Best Practices

### 1. Documentation-Driven Development

**Process:**
1. Write RFC/ADR **before** coding
2. Review with team (async or sync)
3. Get approval before implementation starts
4. Keep docs updated during development
5. Final doc review before production deployment

**Benefits:**
- Fewer implementation surprises
- Better team alignment
- Historical decision context
- Easier onboarding for new team members

### 2. Keep Documents Up-to-Date

**Anti-pattern:**
```markdown
Status: ✅ PRODUCTION
Last Updated: 2025-01-15

[Code changed significantly in March, docs never updated]
```

**Best practice:**
```markdown
Status: ✅ PRODUCTION
Last Updated: 2025-11-15

Changelog:
- 2025-11-15: Updated API response format (breaking change in v1.3.0)
- 2025-10-01: Added new endpoint for bulk operations
- 2025-08-15: Deprecated old authentication method
- 2025-06-01: Initial production release
```

**Enforcement:**
- PR template includes "Documentation updated?" checkbox
- CI checks for `Last Updated` date (warn if > 90 days old)
- Quarterly documentation review

### 3. Use Status Badges Consistently

**In every feature document header:**
```markdown
**Status:** 🔨 IN DEVELOPMENT  <- ALWAYS UPDATE THIS
**Last Updated:** 2025-11-17     <- ALWAYS UPDATE THIS
**Owner:** @alice                <- ALWAYS ASSIGN OWNER
```

**In README.md feature dashboard:**
- Auto-generate from individual docs (single source of truth)
- Script to scan all `FEAT-*.md` files and extract status

### 4. Link Everything

**Create a "web" of documentation:**

```markdown
# In FEAT-002-email-accounts.md

## Architecture References
- [Document 20: Email Infrastructure](../../20-email-infrastructure-postfix-dovecot.md)
- [ADR-004: Maildir Format](../../adrs/ADR-004-dovecot-maildir-over-mbox.md)
- [RFC-001: Email Quota](../../rfcs/RFC-001-email-quota-enforcement.md)

## Related Features
- [FEAT-001: Domain Management](./FEAT-001-domain-management.md) (prerequisite)
- [FEAT-003: DNS Wizard](./FEAT-003-dns-wizard.md) (integrates with)

## Implementation Code
- Backend: `src/api/email-accounts/` (247 lines)
- Frontend: `src/components/EmailAccounts/` (189 lines)
- Database: `migrations/003-email-accounts.sql` (45 lines)
```

### 5. Archive, Don't Delete

**When a feature is deprecated:**

1. Move to `docs/features/deprecated/`
2. Update status to `⚠️ DEPRECATED`
3. Add deprecation notice at top:
   ```markdown
   > **⚠️ DEPRECATED**
   > This feature was deprecated on 2025-11-17 and will be removed in v2.0.0.
   > **Migration Guide:** See [FEAT-099: New Email System](../free-tier/FEAT-099-new-email-system.md)
   ```
4. Keep document accessible for historical reference

**Why:**
- Historical context for future decisions
- Understanding what didn't work (and why)
- Compliance/audit requirements

### 6. Review RFCs Asynchronously

**Process:**
1. Author creates RFC, marks `Status: DRAFT`
2. Posts to Slack/GitHub with review deadline (e.g., 5 business days)
3. Reviewers add feedback to "Feedback" section in doc
4. Author updates RFC based on feedback
5. After review period, decision meeting (or async decision)
6. Status updated to `ACCEPTED` or `REJECTED`

**Benefits:**
- Respects team's time (no mandatory meetings)
- Written feedback creates better record
- Distributed teams can participate equally

### 7. Feature Flags in Feature Docs

**Always document feature flag configuration:**

```markdown
## Feature Flags

### Development
```typescript
// .env.development
FEATURE_EMAIL_QUOTA_ENABLED=true
FEATURE_EMAIL_QUOTA_ROLLOUT_PERCENTAGE=100
```

### Staging
```typescript
// .env.staging
FEATURE_EMAIL_QUOTA_ENABLED=true
FEATURE_EMAIL_QUOTA_ROLLOUT_PERCENTAGE=100
```

### Production
```typescript
// .env.production
FEATURE_EMAIL_QUOTA_ENABLED=true
FEATURE_EMAIL_QUOTA_ROLLOUT_PERCENTAGE=10  // <- Gradual rollout

// LaunchDarkly configuration
{
  "key": "email-quota-enforcement",
  "on": true,
  "rules": [
    {
      "variation": 0,
      "rollout": {
        "percentage": 10  // 10% of users
      }
    }
  ]
}
```

### Rollout Schedule
- Week 1: 1% (monitor)
- Week 2: 10% (if metrics good)
- Week 3: 50% (if metrics good)
- Week 4: 100% (full release)
```

### 8. Include Cost in Every Feature

**Template section:**
```markdown
## Cost Analysis

### Development Cost
| Resource | Hours | Rate | Total |
|----------|-------|------|-------|
| Backend | 40h | $100/h | $4,000 |
| Frontend | 32h | $100/h | $3,200 |
| **Total** | **72h** | | **$7,200** |

### Operational Cost (Monthly)
- Database storage: +500MB = $5/mo
- Compute: +2% CPU = $10/mo
- **Total:** $15/mo

### Revenue Impact
- Expected conversions: +3%
- Monthly revenue increase: $1,500
- **ROI:** 5 months to break even
- **Annual net benefit:** $16,020

### Decision
ROI positive - proceed with implementation.
```

**Why this matters:**
- Forces cost thinking early
- Helps prioritize features
- Business context for technical decisions

### 9. Retrospectives in Feature Docs

**After feature launches, add retrospective:**

```markdown
## Retrospective (Added 2025-12-01)

### What Went Well
- Implementation took 6 weeks (estimate was 6-8 weeks)
- Zero production incidents during rollout
- 95% test coverage achieved
- User feedback overwhelmingly positive (4.8/5.0)

### What Could Be Improved
- Underestimated database migration time (took 2 weeks, expected 1 week)
- Should have created admin UI earlier (added in week 5, should have been week 2)
- Load testing revealed performance issue at 500+ concurrent users (fixed pre-launch)

### Metrics vs Targets

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Conversion increase | +5% | +7% | ✅ Exceeded |
| Error rate | < 1% | 0.3% | ✅ Met |
| Response time (p95) | < 200ms | 180ms | ✅ Met |
| User satisfaction | > 4.0 | 4.8 | ✅ Exceeded |

### Action Items for Next Feature
- [ ] Include admin UI in initial scope
- [ ] Load test at 2x expected capacity
- [ ] Budget 2x time for database migrations
```

**Benefits:**
- Institutional learning
- Better estimation over time
- Celebrate wins
- Address problems systematically

### 10. Compliance & Audit Trail

**For regulated industries or enterprise customers:**

```markdown
## Compliance

### GDPR
- **PII Stored:** Email address, quota usage
- **Retention:** Deleted within 30 days of account deletion
- **Right to Access:** Users can export via Settings > Data Export
- **Right to Deletion:** Automatic on account deletion

### SOC 2
- **Access Control:** Role-based (admin, user)
- **Audit Logging:** All quota changes logged to audit_log table
- **Data Encryption:** TLS 1.3 in transit, AES-256 at rest

### Audit Trail
- **ADR-004:** Architecture decision (Maildir format)
- **RFC-001:** Feature approval and rationale
- **FEAT-002:** Implementation details and testing
- **Deployment Log:** 2025-11-25 12:30 UTC (v1.2.0)
- **Change Log:** All changes tracked in Git (commit a5871bb)
```

---

## Quick Reference

### When to Create Each Document Type

| Scenario | Document Type | Example |
|----------|---------------|---------|
| New significant feature needing cross-team input | RFC | RFC-001: Email Quota |
| Technical implementation decision | ADR | ADR-004: Maildir Format |
| Any feature in active development | Feature Doc | FEAT-002: Email Accounts |
| Operational how-to guide | Implementation Guide | Guide-Monitoring_Quotas.md |

### Document Lifecycle Cheat Sheet

```
RFC: Draft → In Review → Accepted/Rejected → [Archived after implementation]
ADR: Proposed → Accepted → [Superseded when replaced]
Feature: Draft → Design → Development → Testing → Deploying → Production → [Deprecated]
Guide: Draft → Review → Published → [Updated as needed]
```

### Status Badge Cheat Sheet

```markdown
📋 DRAFT          - Initial draft, not yet approved
🎨 DESIGN         - Architecture/design in progress
🔨 IN DEVELOPMENT - Active coding
🧪 TESTING        - In QA/testing
🚀 DEPLOYING      - Staged rollout
✅ PRODUCTION     - Fully deployed
⚠️ DEPRECATED     - End-of-life
```

### Tier Numbering Cheat Sheet

```
FEAT-001 to FEAT-099   = FREE tier features
FEAT-100 to FEAT-199   = PREMIUM tier features
FEAT-200 to FEAT-299   = Platform features (both tiers)
```

### Required Sections by Document Type

**RFC:**
- Summary, Motivation, Proposal, Alternatives, Timeline, Decision

**ADR:**
- Context, Options Considered, Decision, Consequences

**Feature Doc:**
- Overview, Technical Design, Implementation Plan, Testing, Deployment, Monitoring

**Guide:**
- Purpose, Prerequisites, Steps, Troubleshooting, References

---

## Next Steps

1. **Create Directory Structure:**
   ```bash
   mkdir -p docs/{rfcs,adrs,features/{free-tier,premium-tier,platform,deprecated},guides/{operations,development,troubleshooting}}
   ```

2. **Copy Templates:**
   ```bash
   # Create template files in each directory
   cp docs/30-feature-documentation-system.md docs/templates/
   # Extract templates to individual files
   ```

3. **Create First Documents:**
   - ADR-001: Use k3s over k8s (retroactive)
   - ADR-002: Use PostgreSQL over MySQL (retroactive)
   - ADR-003: Use Argon2id for password hashing (retroactive)
   - FEAT-001: Domain Management (document existing feature)
   - FEAT-002: Email Accounts (document existing feature)

4. **Update README.md:**
   - Add Feature Status Dashboard section
   - Add Architecture Decisions section
   - Link to this document (30) for process explanation

5. **Set Up Automation:**
   - Create `scripts/update-feature-dashboard.sh`
   - Add GitHub Action to run on every merge to main
   - Add PR template with documentation checklist

6. **Team Training:**
   - Share this document with team
   - Walkthrough of RFC/ADR/Feature doc process
   - Practice creating first RFC together

---

**Document History:**
- 2025-11-17: Initial version - comprehensive feature documentation system

**Related Documents:**
- [Document 01-11: Core Architecture](../01-executive-summary.md)
- [Document 20-25: Platform Research](../20-email-infrastructure-postfix-dovecot.md)
- [README.md: Documentation Hub](../README.md)
