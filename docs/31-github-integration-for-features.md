# GitHub Integration for Feature Documentation System

**Document 31** | Development Process & Tooling
**Last Updated:** 2025-11-17
**Status:** ✅ Production Ready
**Reading Time:** 30 minutes

---

## Table of Contents

1. [Overview](#overview)
2. [GitHub Features Mapping](#github-features-mapping)
3. [Repository Setup](#repository-setup)
4. [GitHub Issues for Feature Tracking](#github-issues-for-feature-tracking)
5. [GitHub Projects for Dashboards](#github-projects-for-dashboards)
6. [GitHub Discussions for RFCs](#github-discussions-for-rfcs)
7. [Labels and Milestones](#labels-and-milestones)
8. [Pull Request Templates](#pull-request-templates)
9. [GitHub Actions Automation](#github-actions-automation)
10. [Workflow Examples](#workflow-examples)
11. [Integration Patterns](#integration-patterns)
12. [Best Practices](#best-practices)
13. [Quick Reference](#quick-reference)

---

## Overview

Yes, **GitHub is an excellent platform for the Feature Documentation System!** This document shows you how to integrate GitHub's native features (Issues, Projects, Discussions, Actions) with the documentation system from [Document 30](./30-feature-documentation-system.md).

### Why GitHub?

1. **Single Source of Truth**: Code, docs, and feature tracking in one place
2. **Native Integration**: Issues, PRs, and docs are automatically linked
3. **Automation**: GitHub Actions can automate documentation updates
4. **Collaboration**: Built-in review, comments, and notifications
5. **Visibility**: Public or private tracking based on your needs
6. **Free for Teams**: GitHub Free includes unlimited private repos and collaborators

### What You'll Get

- **GitHub Issues** → Feature tracking with status labels
- **GitHub Projects** → Visual kanban boards for features by tier
- **GitHub Discussions** → Async RFC reviews and architectural debates
- **GitHub Milestones** → Group features by release version
- **GitHub Actions** → Auto-update feature dashboard, validate docs
- **Pull Requests** → RFC/ADR review workflow with templates

---

## GitHub Features Mapping

Here's how GitHub features map to the Feature Documentation System:

| Documentation System | GitHub Feature | Purpose |
|---------------------|----------------|---------|
| **RFC (Request for Comments)** | GitHub Discussion | Async review, comments, voting |
| **ADR (Architecture Decision)** | GitHub Issue + Label `adr` | Track decision, link to PR |
| **Feature Doc (FEAT-NNN)** | GitHub Issue + Label `feature` | Track feature through lifecycle |
| **Implementation Guide** | Wiki or `/docs/guides/` | How-to documentation |
| **Feature Status Dashboard** | GitHub Project Board | Visual kanban by status/tier |
| **Status Badges** | GitHub Issue Labels | Visual status indicators |
| **Lifecycle Stages** | GitHub Project Workflow | Automated status transitions |
| **Feature Numbering** | GitHub Issue Number + Template | Auto-increment feature IDs |
| **Cross-References** | GitHub Auto-linking | `#123`, `FEAT-042` auto-link |

---

## Repository Setup

### Option 1: Single Repository (Recommended for Small Teams)

```
refine-digital/hetzner-k3s/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── rfc.yml
│   │   ├── adr.yml
│   │   ├── feature-free-tier.yml
│   │   ├── feature-premium-tier.yml
│   │   ├── feature-platform.yml
│   │   └── bug.yml
│   ├── PULL_REQUEST_TEMPLATE.md
│   ├── DISCUSSION_TEMPLATE/
│   │   └── rfc-discussion.yml
│   └── workflows/
│       ├── update-feature-dashboard.yml
│       ├── validate-docs.yml
│       └── rfc-reminder.yml
├── docs/
│   ├── rfcs/
│   ├── adrs/
│   ├── features/
│   └── guides/
└── src/
```

**Pros:**
- Everything in one place
- Simple cross-referencing
- Single project board

**Cons:**
- Mixes infrastructure code with features
- Can get crowded with many issues

### Option 2: Separate Repositories (Recommended for Large Teams)

**Main Repo:** `refine-digital/hetzner-k3s` (infrastructure code)
**Docs Repo:** `refine-digital/platform-docs` (RFCs, ADRs, Features)
**Private Repo:** `refine-digital/my-refine-digital` (platform API/frontend)

**Pros:**
- Clean separation of concerns
- Can have public docs, private code
- Easier permissions management

**Cons:**
- More complex cross-repo linking
- Multiple project boards

### Recommendation

**Start with Option 1** (single repo) and split later if needed. You can always move issues/discussions to a new repo.

---

## GitHub Issues for Feature Tracking

### Issue Templates

Create `.github/ISSUE_TEMPLATE/` directory with these templates:

#### 1. RFC Template (`rfc.yml`)

```yaml
name: 🎯 RFC - Request for Comments
description: Propose a significant feature or architectural change
title: "[RFC] "
labels: ["rfc", "needs-review"]
body:
  - type: markdown
    attributes:
      value: |
        ## RFC Process
        1. Fill out this template
        2. Team reviews (5 business days)
        3. Discussion via comments or GitHub Discussion
        4. Decision: Accepted/Rejected
        5. If accepted, create Feature issue and ADRs

  - type: input
    id: rfc-number
    attributes:
      label: RFC Number
      description: Next sequential number (check docs/rfcs/)
      placeholder: "RFC-001"
    validations:
      required: true

  - type: dropdown
    id: tier
    attributes:
      label: Tier
      options:
        - FREE
        - PREMIUM
        - Platform
        - Infrastructure
    validations:
      required: true

  - type: textarea
    id: summary
    attributes:
      label: Summary
      description: One paragraph explaining what this RFC proposes
      placeholder: "This RFC proposes..."
    validations:
      required: true

  - type: textarea
    id: motivation
    attributes:
      label: Motivation
      description: |
        **Problem Statement:** What problem are we solving?
        **User Impact:** Who benefits and how?
        **Business Value:** Why is this important?
      placeholder: |
        Problem Statement:

        User Impact:

        Business Value:
    validations:
      required: true

  - type: textarea
    id: proposal
    attributes:
      label: Proposal
      description: High-level design and technical approach
      placeholder: |
        ## High-Level Design

        ## User Experience

        ## Technical Approach
    validations:
      required: true

  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives Considered
      description: What other approaches did you consider and why were they rejected?
      placeholder: |
        ### Alternative 1: [Name]
        **Pros:**
        **Cons:**
        **Reason for rejection:**

  - type: textarea
    id: success-metrics
    attributes:
      label: Success Metrics
      description: How will we measure success?
      placeholder: |
        | Metric | Target | Measurement Method |
        |--------|--------|--------------------|
        | ... | ... | ... |

  - type: textarea
    id: timeline
    attributes:
      label: Timeline
      description: Proposed review period and implementation timeline
      placeholder: |
        - Review Period: YYYY-MM-DD to YYYY-MM-DD
        - Implementation: YYYY-MM-DD to YYYY-MM-DD

  - type: checkboxes
    id: checklist
    attributes:
      label: Pre-submission Checklist
      options:
        - label: I've searched for existing RFCs covering this topic
          required: true
        - label: I've discussed this informally with relevant team members
          required: true
        - label: I've created the RFC document in docs/rfcs/
          required: false
        - label: I've reviewed Document 30 (Feature Documentation System)
          required: true
```

#### 2. Feature Template (`feature-free-tier.yml`)

```yaml
name: 🆓 Feature - FREE Tier
description: New feature for FREE tier users
title: "[FEAT-0XX] "
labels: ["feature", "tier:free", "status:draft"]
body:
  - type: markdown
    attributes:
      value: |
        ## Feature Tracking
        Use this template for FREE tier features (FEAT-001 to FEAT-099).

        **Note:** Feature number will be assigned based on next available in docs/features/free-tier/

  - type: input
    id: feature-number
    attributes:
      label: Feature Number
      description: Next available FREE tier number (001-099)
      placeholder: "FEAT-042"
    validations:
      required: true

  - type: input
    id: feature-name
    attributes:
      label: Feature Name
      description: Short descriptive name (lowercase-with-hyphens)
      placeholder: "domain-verification-wizard"
    validations:
      required: true

  - type: textarea
    id: one-line-summary
    attributes:
      label: One-Line Summary
      description: Single sentence describing what this feature does
      placeholder: "Allow users to verify domain ownership via DNS TXT records"
    validations:
      required: true

  - type: textarea
    id: problem-statement
    attributes:
      label: Problem Statement
      description: What problem does this solve? What pain point does it address?
    validations:
      required: true

  - type: textarea
    id: user-stories
    attributes:
      label: User Stories
      description: As a [persona], I want to [action], so that [benefit]
      placeholder: |
        **As a** FREE tier user
        **I want to** verify my domain ownership easily
        **So that** I can start receiving email immediately
    validations:
      required: true

  - type: dropdown
    id: target-release
    attributes:
      label: Target Release
      options:
        - v1.2.0
        - v1.3.0
        - v1.4.0
        - v2.0.0
        - Backlog
    validations:
      required: true

  - type: input
    id: owner
    attributes:
      label: Feature Owner
      description: GitHub username of feature owner
      placeholder: "@alice"
    validations:
      required: true

  - type: textarea
    id: technical-design
    attributes:
      label: Technical Design (High-Level)
      description: Brief overview of implementation approach
      placeholder: |
        **Components:**
        - Frontend: Domain verification wizard
        - Backend: DNS query API endpoint
        - Database: domain_verifications table

        **Integration Points:**
        - DNS resolver
        - Domain model

  - type: textarea
    id: success-criteria
    attributes:
      label: Success Criteria
      description: What does "done" look like?
      placeholder: |
        - [ ] User can add TXT record instructions
        - [ ] DNS verification succeeds within 60 seconds
        - [ ] Clear error messages for failed verification
        - [ ] 80% test coverage

  - type: input
    id: related-rfc
    attributes:
      label: Related RFC
      description: "RFC issue number (if applicable)"
      placeholder: "#123"

  - type: input
    id: related-adr
    attributes:
      label: Related ADRs
      description: "ADR issue numbers (comma-separated)"
      placeholder: "#145, #167"

  - type: checkboxes
    id: checklist
    attributes:
      label: Pre-submission Checklist
      options:
        - label: I've checked next available feature number in docs/features/free-tier/
          required: true
        - label: I've reviewed similar features to avoid duplication
          required: true
        - label: RFC created and accepted (if significant feature)
          required: false
```

#### 3. ADR Template (`adr.yml`)

```yaml
name: 🏗️ ADR - Architecture Decision Record
description: Document an important technical decision
title: "[ADR] "
labels: ["adr", "architecture"]
body:
  - type: markdown
    attributes:
      value: |
        ## ADR Process
        1. Propose decision in this issue
        2. Discuss alternatives in comments
        3. Create PR with ADR document in docs/adrs/
        4. Merge PR when decision is accepted

  - type: input
    id: adr-number
    attributes:
      label: ADR Number
      description: Next sequential number (check docs/adrs/)
      placeholder: "ADR-004"
    validations:
      required: true

  - type: input
    id: decision-statement
    attributes:
      label: Decision Statement
      description: "Format: Use X over Y for Z"
      placeholder: "Use Dovecot over Cyrus IMAP for email server"
    validations:
      required: true

  - type: textarea
    id: context
    attributes:
      label: Context
      description: |
        What is the problem we're solving?
        What constraints or assumptions exist?
      placeholder: |
        ## Problem
        We need an IMAP server for FREE tier email.

        ## Constraints
        - Must support virtual users (PostgreSQL backend)
        - Must handle 1000+ concurrent users
        - Must integrate with Postfix
    validations:
      required: true

  - type: textarea
    id: decision-drivers
    attributes:
      label: Decision Drivers
      description: What factors influenced this decision?
      placeholder: |
        - Performance requirements (> 1000 concurrent IMAP connections)
        - Cost constraints (must be open-source)
        - Developer experience (ease of configuration)
        - Community support (active development)

  - type: textarea
    id: options
    attributes:
      label: Options Considered
      description: List all options with pros/cons
      placeholder: |
        ### Option 1: Dovecot
        **Pros:**
        - Industry standard (90%+ deployments)
        - Excellent PostgreSQL integration
        - High performance (10k+ connections)

        **Cons:**
        - Configuration complexity
        - Large feature set (may not need all)

        ### Option 2: Cyrus IMAP
        **Pros:**
        - Enterprise-grade
        - Good documentation

        **Cons:**
        - Harder to configure
        - Smaller community
    validations:
      required: true

  - type: dropdown
    id: chosen-option
    attributes:
      label: Recommended Option
      description: Which option do you recommend?
      options:
        - Option 1
        - Option 2
        - Option 3
        - Other (specify in rationale)
    validations:
      required: true

  - type: textarea
    id: rationale
    attributes:
      label: Rationale
      description: Why is this the best choice?
      placeholder: |
        Dovecot is the clear choice because:
        1. Industry standard with proven scalability
        2. Excellent PostgreSQL virtual user support
        3. Active community and extensive documentation
        4. Performance benchmarks show 3x better concurrent connection handling
    validations:
      required: true

  - type: textarea
    id: consequences
    attributes:
      label: Consequences
      description: What are the implications of this decision?
      placeholder: |
        **Positive:**
        - Fast IMAP access for FREE tier users
        - Easy to find community support/examples

        **Negative:**
        - Initial configuration complexity (mitigated by templates)
        - Need to learn Dovecot-specific syntax

        **Neutral:**
        - Standard implementation (no custom code)

  - type: input
    id: related-rfc
    attributes:
      label: Related RFC
      description: "RFC issue number (if applicable)"
      placeholder: "#123"

  - type: checkboxes
    id: checklist
    attributes:
      label: Checklist
      options:
        - label: I've researched all viable alternatives
          required: true
        - label: I've consulted with relevant team members
          required: true
        - label: I'll create ADR document in docs/adrs/ after approval
          required: true
```

### Creating Issues

**From GitHub UI:**
1. Go to repository → Issues → New Issue
2. Select template (RFC / Feature / ADR)
3. Fill out form
4. Submit

**From CLI:**
```bash
gh issue create --template rfc.yml
gh issue create --template feature-free-tier.yml
gh issue create --template adr.yml
```

---

## GitHub Projects for Dashboards

> **⚠️ Important Decision:** Should you use ONE project for all features, or separate projects per tier?
>
> **See:** [31-addendum-projects-structure.md](./31-addendum-projects-structure.md) for detailed analysis.
>
> **Quick Answer:** Start with **one project with multiple views** (recommended for teams < 15 people). Split later if needed.

### Create Feature Board

**GitHub Projects (Beta) - Recommended for 2025:**

1. Go to repository → Projects → New Project
2. Choose "Board" view
3. Name it "Feature Development Pipeline"

### Board Configuration

#### Columns (Statuses)

Create columns matching lifecycle stages:

| Column | Status Badge | GitHub Label |
|--------|-------------|--------------|
| 📋 Backlog | DRAFT | `status:draft` |
| 🎨 Design | DESIGN | `status:design` |
| 🔨 In Development | IN DEVELOPMENT | `status:in-development` |
| 🧪 Testing | TESTING | `status:testing` |
| 🚀 Deploying | DEPLOYING | `status:deploying` |
| ✅ Production | PRODUCTION | `status:production` |
| ⚠️ Deprecated | DEPRECATED | `status:deprecated` |

#### Views

Create multiple views for different perspectives:

**View 1: By Status (Default Board)**
- Group by: Status
- Sort by: Priority (high → low)
- Filter: None

**View 2: By Tier**
- Group by: Tier (`tier:free`, `tier:premium`, `tier:platform`)
- Sort by: Target Release
- Show: Features only

**View 3: By Release**
- Group by: Milestone (v1.2.0, v1.3.0, etc.)
- Sort by: Priority
- Filter: `status:in-development` OR `status:testing`

**View 4: Roadmap (Table)**
- Layout: Table
- Columns: Title, Status, Tier, Owner, Target Release, Updated
- Sort by: Target Release (ascending)
- Filter: NOT `status:deprecated`

### Automation

GitHub Projects (Beta) supports workflow automation:

**Auto-add to Project:**
```yaml
# .github/workflows/add-to-project.yml
name: Add to Project
on:
  issues:
    types: [opened]

jobs:
  add-to-project:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/add-to-project@v0.5.0
        with:
          project-url: https://github.com/orgs/refine-digital/projects/1
          github-token: ${{ secrets.ADD_TO_PROJECT_PAT }}
          labeled: feature, rfc, adr
```

**Auto-move based on labels:**
- When label `status:in-development` added → Move to "🔨 In Development"
- When label `status:production` added → Move to "✅ Production"

### Custom Fields

Add custom fields to track additional metadata:

| Field Name | Type | Values |
|------------|------|--------|
| Feature Number | Text | FEAT-042 |
| Tier | Single Select | FREE, PREMIUM, Platform |
| Complexity | Single Select | Low, Medium, High |
| Estimated Effort | Number | (story points or hours) |
| Documentation Status | Single Select | Not Started, In Progress, Complete |

---

## GitHub Discussions for RFCs

### Why Discussions for RFCs?

- **Better than Issues** for long-form discussions
- **Voting** via reactions (👍 👎)
- **Threaded comments** for organized debates
- **Mark answer** when decision is reached
- **Categories** for organization (RFCs, Q&A, Ideas)

### Setup

1. Go to repository → Settings → Features
2. Enable "Discussions"
3. Create categories:

| Category | Purpose | Format |
|----------|---------|--------|
| 🎯 RFCs | Architecture proposals | Announcement |
| 💡 Ideas | Feature ideas (pre-RFC) | Open-ended |
| ❓ Q&A | Technical questions | Q&A |
| 📢 Announcements | Decisions, releases | Announcement |

### Discussion Template

Create `.github/DISCUSSION_TEMPLATE/rfc-discussion.yml`:

```yaml
title: "[RFC] "
labels: [rfc]
body: |
  ## RFC: [Feature Name]

  **Status:** Draft | In Review | Accepted | Rejected
  **Author:** @username
  **Review Period:** YYYY-MM-DD to YYYY-MM-DD
  **Related Issue:** #XXX (RFC issue)
  **Document:** [docs/rfcs/RFC-XXX-name.md](link)

  ---

  ## Summary

  [Paste RFC summary here]

  ## Key Questions for Review

  1. Does this solve the right problem?
  2. Are the alternatives adequately considered?
  3. Is the proposed solution feasible?
  4. Are there any risks we haven't identified?

  ## How to Provide Feedback

  - **👍** if you support this RFC
  - **👎** if you have concerns
  - **Comment** with specific feedback
  - **React** to comments you agree with

  ---

  **Review Deadline:** YYYY-MM-DD
```

### RFC Review Process

**Week 1:**
1. Author creates RFC document in `docs/rfcs/RFC-NNN-name.md`
2. Author creates GitHub Discussion (link to doc)
3. Author notifies team via Slack/email
4. Team members read and react/comment

**Week 2:**
5. Author addresses feedback, updates RFC document
6. Team reaches consensus via voting (👍/👎 reactions)
7. Decision maker makes final call (mark discussion as "Answered")
8. Author updates RFC status to "Accepted" or "Rejected"
9. If accepted, create Feature issue and ADRs

### Example Discussion

**Title:** [RFC] Email Quota Enforcement for FREE Tier

**Body:**
```markdown
## RFC: Email Quota Enforcement for FREE Tier

**Status:** In Review
**Author:** @alice
**Review Period:** 2025-11-01 to 2025-11-08
**Related Issue:** #123
**Document:** [docs/rfcs/RFC-001-email-quota-enforcement.md](link)

---

## Summary

Implement automatic email quota enforcement for FREE tier users (5GB limit) to prevent abuse and manage infrastructure costs.

**Key Points:**
- 15% of FREE tier users currently exceed documented quota
- Proposed: 80% warning, 90% final warning, 100% hard limit
- Phased rollout: monitoring → warnings → enforcement
- Expected savings: $200/month + 5% conversion increase

## Open Questions

1. Should we offer one-time quota extensions for long-time users?
2. What's the ideal upgrade flow from quota warning?
3. Grace period: allow +10% temporary overage?

## How to Provide Feedback

Please review the [full RFC document](link) and:
- **👍** if you support moving forward
- **👎** if you have blocking concerns
- **Comment** with specific feedback on approach, timeline, or metrics

---

**Review Deadline:** November 8, 2025
```

**Comments:**
- @bob: "Suggest adding Redis caching for quota lookups to reduce DB load" (10 👍)
- @carol: "Include upgrade CTA in warning emails for better conversion" (8 👍)
- @dave: "What about bulk delete UI to help users clean up?" (5 👍)

**Resolution:**
- Author marks @bob's comment as "Answer" (Redis caching will be added)
- Updates RFC based on @carol's feedback
- Creates follow-up issue for @dave's suggestion

---

## Labels and Milestones

### Label Strategy

#### Status Labels (Lifecycle Stages)

| Label | Color | Description |
|-------|-------|-------------|
| `status:draft` | `#d4c5f9` (light purple) | Initial draft, not yet approved |
| `status:design` | `#bfdadc` (light blue) | Architecture/design in progress |
| `status:in-development` | `#fbca04` (yellow) | Active development |
| `status:testing` | `#0e8a16` (green) | In QA/testing |
| `status:deploying` | `#1d76db` (blue) | Staged rollout |
| `status:production` | `#0e8a16` (dark green) | Fully deployed |
| `status:deprecated` | `#e99695` (red) | End-of-life |

#### Type Labels

| Label | Color | Description |
|-------|-------|-------------|
| `type:rfc` | `#5319e7` (purple) | Request for Comments |
| `type:adr` | `#0052cc` (blue) | Architecture Decision Record |
| `type:feature` | `#1d76db` (blue) | Feature implementation |
| `type:bug` | `#d73a4a` (red) | Bug fix |
| `type:guide` | `#0e8a16` (green) | Implementation guide |

#### Tier Labels

| Label | Color | Description |
|-------|-------|-------------|
| `tier:free` | `#c5def5` (light blue) | FREE tier feature |
| `tier:premium` | `#ffd700` (gold) | PREMIUM tier feature |
| `tier:platform` | `#5319e7` (purple) | Platform-wide feature |

#### Priority Labels

| Label | Color | Description |
|-------|-------|-------------|
| `priority:critical` | `#d73a4a` (red) | Critical priority |
| `priority:high` | `#ff9900` (orange) | High priority |
| `priority:medium` | `#fbca04` (yellow) | Medium priority |
| `priority:low` | `#c5def5` (light blue) | Low priority |

#### Review Labels

| Label | Color | Description |
|-------|-------|-------------|
| `needs-review` | `#fbca04` (yellow) | Awaiting review |
| `needs-feedback` | `#d876e3` (pink) | Needs team feedback |
| `blocked` | `#d73a4a` (red) | Blocked by dependency |
| `approved` | `#0e8a16` (green) | Approved for implementation |

### Creating Labels

**Via GitHub UI:**
1. Repository → Issues → Labels → New Label

**Via CLI:**
```bash
# Create status labels
gh label create "status:draft" --color "d4c5f9" --description "Initial draft"
gh label create "status:design" --color "bfdadc" --description "Design phase"
gh label create "status:in-development" --color "fbca04" --description "Active development"
gh label create "status:testing" --color "0e8a16" --description "Testing phase"
gh label create "status:deploying" --color "1d76db" --description "Deploying"
gh label create "status:production" --color "0e8a16" --description "Production"
gh label create "status:deprecated" --color "e99695" --description "Deprecated"

# Create tier labels
gh label create "tier:free" --color "c5def5" --description "FREE tier"
gh label create "tier:premium" --color "ffd700" --description "PREMIUM tier"
gh label create "tier:platform" --color "5319e7" --description "Platform"

# Create type labels
gh label create "type:rfc" --color "5319e7" --description "Request for Comments"
gh label create "type:adr" --color "0052cc" --description "Architecture Decision"
gh label create "type:feature" --color "1d76db" --description "Feature"
```

### Milestones

Create milestones for each release:

| Milestone | Due Date | Description |
|-----------|----------|-------------|
| v1.2.0 | 2025-12-01 | Q4 2025 Release |
| v1.3.0 | 2025-12-15 | Year-end Release |
| v1.4.0 | 2026-01-15 | Q1 2026 Release |
| Backlog | - | Future features |

**Create via CLI:**
```bash
gh api repos/refine-digital/hetzner-k3s/milestones \
  -f title="v1.2.0" \
  -f due_on="2025-12-01T00:00:00Z" \
  -f description="Q4 2025 Release"
```

---

## Pull Request Templates

Create `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Description

<!-- Brief description of changes -->

## Type of Change

- [ ] 🎯 RFC (Request for Comments)
- [ ] 🏗️ ADR (Architecture Decision Record)
- [ ] ✨ Feature implementation
- [ ] 🐛 Bug fix
- [ ] 📝 Documentation update
- [ ] 🔧 Configuration change

## Related Issues

<!-- Link to related issues -->

- Closes #XXX
- Related to #YYY

## Documentation Checklist

- [ ] RFC document created/updated in `docs/rfcs/`
- [ ] ADR document created/updated in `docs/adrs/`
- [ ] Feature document created/updated in `docs/features/`
- [ ] Implementation guide created/updated in `docs/guides/`
- [ ] README.md updated (if applicable)
- [ ] Feature dashboard updated (docs/features/README.md)
- [ ] Status badge updated in document header

## Feature Lifecycle (if feature PR)

**Current Status:** [Draft | Design | Development | Testing | Deploying | Production]

- [ ] Feature doc created (FEAT-NNN-name.md)
- [ ] Technical design complete
- [ ] Unit tests written (target: 80% coverage)
- [ ] Integration tests written
- [ ] E2E tests written (if applicable)
- [ ] Documentation complete
- [ ] Code review completed
- [ ] QA signoff received

## Testing

<!-- Describe testing performed -->

- [ ] Unit tests pass (`npm test`)
- [ ] Integration tests pass
- [ ] E2E tests pass (if applicable)
- [ ] Manual testing completed

## Deployment Notes

<!-- Any special deployment considerations -->

- [ ] Database migrations (attach migration file)
- [ ] Environment variables updated (.env.example)
- [ ] Feature flags configured
- [ ] Rollback procedure documented

## Screenshots (if UI changes)

<!-- Add screenshots here -->

## Reviewer Notes

<!-- Anything specific for reviewers to focus on -->

---

**Merge Checklist:**
- [ ] All CI checks passing
- [ ] Code review approved (minimum 1 approval)
- [ ] Documentation review approved
- [ ] No merge conflicts
- [ ] Target branch is correct
```

---

## GitHub Actions Automation

### 1. Auto-Update Feature Dashboard

Create `.github/workflows/update-feature-dashboard.yml`:

```yaml
name: Update Feature Dashboard

on:
  push:
    branches: [main]
    paths:
      - 'docs/features/**/*.md'
  workflow_dispatch:

jobs:
  update-dashboard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: |
          npm install -g @octokit/core js-yaml

      - name: Update Dashboard
        run: |
          node scripts/update-feature-dashboard.js

      - name: Commit changes
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'
          git add docs/features/README.md
          git diff --quiet && git diff --staged --quiet || \
            git commit -m "chore: auto-update feature dashboard [skip ci]"
          git push
```

**Script:** `scripts/update-feature-dashboard.js`

```javascript
const fs = require('fs');
const path = require('path');
const yaml = require('js-yaml');

// Scan all feature docs
const tiers = ['free-tier', 'premium-tier', 'platform'];
const features = [];

tiers.forEach(tier => {
  const tierPath = path.join('docs', 'features', tier);
  if (!fs.existsSync(tierPath)) return;

  const files = fs.readdirSync(tierPath).filter(f => f.endsWith('.md'));

  files.forEach(file => {
    const content = fs.readFileSync(path.join(tierPath, file), 'utf8');

    // Extract front matter (if using front matter)
    const match = content.match(/^---\n([\s\S]+?)\n---/);
    if (match) {
      const frontMatter = yaml.load(match[1]);
      features.push({
        file: `${tier}/${file}`,
        ...frontMatter
      });
    } else {
      // Parse from markdown headers
      const statusMatch = content.match(/\*\*Status:\*\* (.+)/);
      const tierMatch = content.match(/\*\*Tier:\*\* (.+)/);
      const ownerMatch = content.match(/\*\*Owner:\*\* (.+)/);
      const releaseMatch = content.match(/\*\*Target Release:\*\* (.+)/);

      features.push({
        file: `${tier}/${file}`,
        status: statusMatch ? statusMatch[1].trim() : 'Unknown',
        tier: tierMatch ? tierMatch[1].trim() : tier,
        owner: ownerMatch ? ownerMatch[1].trim() : 'Unassigned',
        targetRelease: releaseMatch ? releaseMatch[1].trim() : 'TBD',
        title: file.replace(/^FEAT-\d+-/, '').replace(/-/g, ' ').replace('.md', '')
      });
    }
  });
});

// Generate dashboard
let dashboard = `# Feature Status Dashboard\n\n`;
dashboard += `Last Updated: ${new Date().toISOString().split('T')[0]} (Auto-generated)\n\n`;

// By Status
dashboard += `## By Status\n\n`;
const statuses = [...new Set(features.map(f => f.status))];
statuses.forEach(status => {
  const statusFeatures = features.filter(f => f.status === status);
  dashboard += `### ${status} (${statusFeatures.length})\n`;
  statusFeatures.forEach(f => {
    dashboard += `- [${f.title}](../${f.file}) - ${f.tier} - Target: ${f.targetRelease}\n`;
  });
  dashboard += `\n`;
});

// By Tier
dashboard += `## By Tier\n\n`;
dashboard += `| Tier | Total | Production | In Development |\n`;
dashboard += `|------|-------|------------|----------------|\n`;
tiers.forEach(tier => {
  const tierFeatures = features.filter(f => f.tier === tier || f.file.startsWith(tier));
  const production = tierFeatures.filter(f => f.status.includes('PRODUCTION')).length;
  const inDev = tierFeatures.filter(f => f.status.includes('DEVELOPMENT')).length;
  dashboard += `| ${tier} | ${tierFeatures.length} | ${production} | ${inDev} |\n`;
});

// Write to file
fs.writeFileSync('docs/features/README.md', dashboard);
console.log('✅ Feature dashboard updated!');
```

### 2. Validate Documentation

Create `.github/workflows/validate-docs.yml`:

```yaml
name: Validate Documentation

on:
  pull_request:
    paths:
      - 'docs/**/*.md'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check RFC Numbering
        run: |
          # Ensure RFC numbers are sequential
          cd docs/rfcs
          numbers=$(ls RFC-*.md 2>/dev/null | sed 's/RFC-\([0-9]*\)-.*/\1/' | sort -n)
          expected=1
          for num in $numbers; do
            if [ "$num" -ne "$expected" ]; then
              echo "❌ RFC numbering gap: expected RFC-$(printf '%03d' $expected), found RFC-$num"
              exit 1
            fi
            expected=$((expected + 1))
          done
          echo "✅ RFC numbering is sequential"

      - name: Check Feature Numbering
        run: |
          # Ensure feature numbers match tier ranges
          cd docs/features

          # FREE tier: 001-099
          free=$(ls free-tier/FEAT-*.md 2>/dev/null | sed 's/.*FEAT-\([0-9]*\)-.*/\1/')
          for num in $free; do
            if [ "$num" -lt 1 ] || [ "$num" -gt 99 ]; then
              echo "❌ FREE tier feature out of range: FEAT-$num (should be 001-099)"
              exit 1
            fi
          done

          # PREMIUM tier: 100-199
          premium=$(ls premium-tier/FEAT-*.md 2>/dev/null | sed 's/.*FEAT-\([0-9]*\)-.*/\1/')
          for num in $premium; do
            if [ "$num" -lt 100 ] || [ "$num" -gt 199 ]; then
              echo "❌ PREMIUM tier feature out of range: FEAT-$num (should be 100-199)"
              exit 1
            fi
          done

          echo "✅ Feature numbering follows tier conventions"

      - name: Check Required Sections
        run: |
          # Ensure feature docs have required sections
          for file in docs/features/**/FEAT-*.md; do
            [ -f "$file" ] || continue

            required=("Overview" "Technical Design" "Implementation Plan" "Testing" "Deployment")
            for section in "${required[@]}"; do
              if ! grep -q "## $section" "$file"; then
                echo "❌ Missing required section '$section' in $file"
                exit 1
              fi
            done
          done
          echo "✅ All feature docs have required sections"

      - name: Check Links
        uses: gaurav-nelson/github-action-markdown-link-check@v1
        with:
          use-quiet-mode: 'yes'
          config-file: '.github/markdown-link-check-config.json'
```

### 3. RFC Review Reminder

Create `.github/workflows/rfc-reminder.yml`:

```yaml
name: RFC Review Reminder

on:
  schedule:
    - cron: '0 9 * * 1'  # Every Monday at 9am UTC

jobs:
  remind:
    runs-on: ubuntu-latest
    steps:
      - name: Find RFCs needing review
        uses: actions/github-script@v7
        with:
          script: |
            const issues = await github.rest.issues.listForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              labels: 'rfc,needs-review',
              state: 'open'
            });

            if (issues.data.length === 0) {
              console.log('✅ No RFCs pending review');
              return;
            }

            let message = '🎯 **RFC Review Reminder**\n\n';
            message += `There are ${issues.data.length} RFC(s) awaiting review:\n\n`;

            issues.data.forEach(issue => {
              const daysOpen = Math.floor((Date.now() - new Date(issue.created_at)) / (1000 * 60 * 60 * 24));
              message += `- [${issue.title}](${issue.html_url}) (${daysOpen} days open)\n`;
            });

            message += '\nPlease review and provide feedback!';

            // Post to Slack (if webhook configured)
            if (process.env.SLACK_WEBHOOK) {
              await fetch(process.env.SLACK_WEBHOOK, {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({text: message})
              });
            }

            console.log(message);
```

---

## Workflow Examples

### Example 1: New RFC → Feature → Production

**Step 1: Create RFC**
```bash
# Create RFC issue from template
gh issue create --template rfc.yml

# Fill out:
# - RFC Number: RFC-001
# - Title: Email Quota Enforcement
# - Tier: FREE
# - Summary: Implement 5GB quota for FREE tier email
```

**Step 2: Create RFC Document**
```bash
# Create RFC document
cp docs/rfcs/template.md docs/rfcs/RFC-001-email-quota-enforcement.md

# Edit document with full details
# Commit and push
git add docs/rfcs/RFC-001-email-quota-enforcement.md
git commit -m "docs: add RFC-001 email quota enforcement"
git push
```

**Step 3: Start Discussion**
```bash
# Create GitHub Discussion
# Category: RFCs
# Title: [RFC] Email Quota Enforcement
# Link to RFC document
# Set review period: 5 business days
```

**Step 4: Gather Feedback**
- Team members read RFC
- Comment with feedback
- Author updates RFC based on feedback
- Team votes via reactions (👍 👎)

**Step 5: Decision**
```bash
# Update RFC issue
gh issue edit 123 --add-label "approved" --remove-label "needs-review"

# Update RFC document status to "Accepted"
# Close discussion with decision summary
```

**Step 6: Create ADR(s)**
```bash
# Create ADR issue for technical decision
gh issue create --template adr.yml

# Fill out:
# - ADR Number: ADR-004
# - Decision: Use Maildir over mbox format
# - Related RFC: #123

# Create ADR document
cp docs/adrs/template.md docs/adrs/ADR-004-dovecot-maildir-over-mbox.md
# Edit and commit
```

**Step 7: Create Feature Issue**
```bash
# Create feature tracking issue
gh issue create --template feature-free-tier.yml

# Fill out:
# - Feature Number: FEAT-002
# - Name: email-quota-enforcement
# - Related RFC: #123
# - Related ADR: #145
# - Target Release: v1.2.0
# - Owner: @alice

# Add to milestone
gh issue edit 156 --milestone "v1.2.0"

# Add to project
gh project item-add 1 --owner refine-digital --url https://github.com/refine-digital/hetzner-k3s/issues/156
```

**Step 8: Development**
```bash
# Create feature branch
git checkout -b feat/002-email-quota-enforcement

# Create feature document
cp docs/features/template.md docs/features/free-tier/FEAT-002-email-quota-enforcement.md
# Fill out complete feature documentation

# Update issue status
gh issue edit 156 --add-label "status:in-development" --remove-label "status:draft"

# Implement feature
# Write tests
# Update docs

# Commit regularly
git commit -am "feat: implement email quota tracking"
```

**Step 9: Testing**
```bash
# Update issue status
gh issue edit 156 --add-label "status:testing" --remove-label "status:in-development"

# Run tests
npm test
npm run test:integration
npm run test:e2e

# QA signoff
# Update feature doc with test results
```

**Step 10: Create Pull Request**
```bash
# Push feature branch
git push origin feat/002-email-quota-enforcement

# Create PR
gh pr create \
  --title "feat: email quota enforcement (FEAT-002)" \
  --body "Closes #156. Implements RFC-001 email quota enforcement for FREE tier." \
  --label "type:feature,tier:free"

# Link to feature issue
gh pr edit 157 --add-label "status:deploying"
```

**Step 11: Deploy**
```bash
# Merge PR
gh pr merge 157 --squash

# Update feature issue
gh issue edit 156 --add-label "status:deploying" --remove-label "status:testing"

# Deploy to staging
# Gradual rollout to production (1% → 10% → 50% → 100%)

# Monitor metrics
```

**Step 12: Production**
```bash
# After successful 100% rollout
gh issue edit 156 --add-label "status:production" --remove-label "status:deploying"

# Update feature doc with production metrics
# Create implementation guide
cp docs/guides/template.md docs/guides/operations/Guide-Managing_Email_Quotas.md

# Archive RFC
git mv docs/rfcs/RFC-001-email-quota-enforcement.md docs/rfcs/archived/
git commit -m "docs: archive RFC-001 (implemented as FEAT-002)"

# Close RFC issue
gh issue close 123 --comment "Implemented in FEAT-002, now in production"

# Close feature issue (or keep open for ongoing maintenance)
gh issue close 156 --comment "Successfully deployed to production ✅"
```

**Total Timeline: 6-8 weeks**
- RFC review: 1 week
- ADR creation: 1 week
- Development: 2-4 weeks
- Testing: 1 week
- Deployment: 1 week

---

## Integration Patterns

### Pattern 1: Documentation-First

```
1. Create RFC document (docs/rfcs/)
2. Create RFC issue (link to document)
3. Create GitHub Discussion (link to both)
4. Team reviews → Decision
5. Create Feature issue
6. Create Feature document (docs/features/)
7. Implement → Test → Deploy
8. Update Feature document status
```

**Pros:**
- Documents drive the process
- Full context in markdown files
- Git history of decisions

**Cons:**
- Requires discipline to keep docs updated
- Multiple places to check (docs + issues + discussions)

### Pattern 2: Issue-First

```
1. Create RFC issue (full details in issue)
2. Discussion happens in issue comments
3. Decision recorded in issue
4. Create Feature issue
5. Implementation details in feature issue
6. After production, create docs for reference
```

**Pros:**
- Everything in GitHub (no external docs)
- Natural discussion flow
- Easy to search

**Cons:**
- Less structured than markdown docs
- Hard to maintain long-term (issue gets long)
- Difficult to version control

### Pattern 3: Hybrid (Recommended)

```
1. Create RFC issue (lightweight template)
2. Create RFC document for details (docs/rfcs/)
3. Discussion in GitHub Discussion
4. Decision → Update both issue and document
5. Create Feature issue (link to RFC)
6. Create Feature document (link to issue)
7. Implement with PR (link to feature issue)
8. Update Feature document with results
```

**Pros:**
- Best of both worlds
- Documents for structured info
- Issues/Discussions for collaboration
- Clear audit trail

**Cons:**
- Most overhead
- Requires automation to stay in sync

**Recommendation:** Start with Pattern 3 (Hybrid) and use GitHub Actions to automate synchronization.

---

## Best Practices

### 1. Link Everything

**In Issues:**
```markdown
Related RFC: #123
Related ADR: #145, #167
Related Feature: #156
Implementation PR: #157
Documentation: [FEAT-002-email-quota.md](link)
```

**In Commits:**
```bash
git commit -m "feat: add quota tracking

Implements FEAT-002 email quota enforcement.
Part of RFC-001, uses ADR-004 (Maildir format).

Closes #156"
```

**In Documents:**
```markdown
## Related Issues
- **RFC Issue:** [#123](link)
- **Feature Issue:** [#156](link)
- **Implementation PR:** [#157](link)
```

### 2. Use Consistent Naming

| Document Type | Issue Title | File Name |
|---------------|-------------|-----------|
| RFC | `[RFC] Email Quota Enforcement` | `RFC-001-email-quota-enforcement.md` |
| ADR | `[ADR] Use Maildir over mbox` | `ADR-004-dovecot-maildir-over-mbox.md` |
| Feature | `[FEAT-002] Email Quota Enforcement` | `FEAT-002-email-quota-enforcement.md` |
| Guide | `Guide: Managing Email Quotas` | `Guide-Managing_Email_Quotas.md` |

### 3. Automate What You Can

**Auto-label:**
- PR opened → Add `needs-review`
- PR approved → Add `approved`
- PR merged → Add `status:production` to feature issue

**Auto-assign:**
- RFC issue → Assign to RFC author
- Feature issue → Assign to feature owner

**Auto-close:**
- PR merged with "Closes #156" → Close feature issue

**Auto-notify:**
- RFC needs review → Slack notification
- Feature deployed → Slack announcement

### 4. Keep Issues Updated

**Bad:**
```
Issue created → No updates → Closed after 6 months
```

**Good:**
```
Issue created
↓
Weekly updates on progress
↓
Status labels updated as work progresses
↓
Linked to PRs as they're created
↓
Final comment summarizing outcome
↓
Closed with clear resolution
```

### 5. Archive Completed Work

**RFCs:**
- Implemented RFCs → Move to `docs/rfcs/archived/`
- Keep issue open for reference or close with "implemented" comment

**ADRs:**
- Superseded ADRs → Move to `docs/adrs/superseded/`
- Update with "Superseded by ADR-XXX" note

**Features:**
- Deprecated features → Move to `docs/features/deprecated/`
- Update status to `⚠️ DEPRECATED`

### 6. Use GitHub Search

**Find all RFCs needing review:**
```
is:issue is:open label:rfc label:needs-review
```

**Find all features in development for v1.2.0:**
```
is:issue is:open label:feature label:status:in-development milestone:v1.2.0
```

**Find all FREE tier features:**
```
is:issue label:feature label:tier:free
```

**Find all ADRs:**
```
is:issue label:adr
```

### 7. Review Regularly

**Weekly:**
- Review RFC/ADR discussions needing feedback
- Update feature issue statuses
- Triage new issues

**Monthly:**
- Review feature dashboard for stale items
- Archive completed RFCs
- Clean up old branches

**Quarterly:**
- Review all documentation for accuracy
- Update templates based on learnings
- Retrospective on documentation process

---

## Quick Reference

### Create New Items

```bash
# RFC
gh issue create --template rfc.yml
cp docs/rfcs/template.md docs/rfcs/RFC-NNN-name.md

# ADR
gh issue create --template adr.yml
cp docs/adrs/template.md docs/adrs/ADR-NNN-name.md

# Feature (FREE tier)
gh issue create --template feature-free-tier.yml
cp docs/features/template.md docs/features/free-tier/FEAT-NNN-name.md

# Feature (PREMIUM tier)
gh issue create --template feature-premium-tier.yml
cp docs/features/template.md docs/features/premium-tier/FEAT-NNN-name.md
```

### Update Status

```bash
# Update issue labels
gh issue edit 156 --add-label "status:testing" --remove-label "status:in-development"

# Update milestone
gh issue edit 156 --milestone "v1.2.0"

# Add to project
gh project item-add 1 --owner refine-digital --url https://github.com/refine-digital/hetzner-k3s/issues/156
```

### Search

```bash
# All RFCs needing review
gh issue list --label "rfc,needs-review"

# All features in v1.2.0
gh issue list --milestone "v1.2.0" --label "feature"

# All FREE tier features
gh issue list --label "tier:free,feature"
```

### Automation

```bash
# Trigger feature dashboard update
gh workflow run update-feature-dashboard.yml

# Validate documentation
gh workflow run validate-docs.yml
```

---

## Next Steps

1. **Set up repository structure:**
   ```bash
   mkdir -p .github/{ISSUE_TEMPLATE,DISCUSSION_TEMPLATE,workflows}
   mkdir -p scripts
   ```

2. **Create issue templates:**
   - Copy RFC, ADR, and Feature templates from this document
   - Customize for your team's needs

3. **Set up GitHub Project:**
   - Create "Feature Development Pipeline" board
   - Configure columns and views

4. **Create labels:**
   - Run label creation commands
   - Customize colors/descriptions

5. **Enable GitHub Discussions:**
   - Go to Settings → Features → Enable Discussions
   - Create RFC category

6. **Set up GitHub Actions:**
   - Create workflow files
   - Add automation scripts
   - Test workflows

7. **Create first RFC:**
   - Use templates to create RFC-001
   - Practice the full workflow
   - Refine process based on learnings

8. **Document retroactive decisions:**
   - Create ADRs for existing architecture
   - Link to implementation code
   - Build institutional knowledge

---

**Related Documents:**
- [Document 30: Feature Documentation System](./30-feature-documentation-system.md)
- [GitHub Issues Documentation](https://docs.github.com/en/issues)
- [GitHub Projects Documentation](https://docs.github.com/en/issues/planning-and-tracking-with-projects)
- [GitHub Discussions Documentation](https://docs.github.com/en/discussions)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
