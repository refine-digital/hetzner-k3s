# GitHub Projects Structure: Single vs Multiple Boards

**Addendum to Document 31**
**Last Updated:** 2025-11-17

---

## Quick Answer

**It depends on your team size and structure.**

Here are the recommendations:

| Team Size | Recommendation | Why |
|-----------|---------------|-----|
| **1-5 people** | ✅ Single Project with Views | Simple, everyone sees everything |
| **6-15 people** | ✅ Single Project with Views | Still manageable, views provide focus |
| **16-30 people** | ⚠️ Single or Multiple (see below) | Depends on team structure |
| **30+ people** | ✅ Multiple Projects | Better separation, team autonomy |

---

## Option 1: Single GitHub Project (with Multiple Views)

### What It Looks Like

**One Project:** "Feature Development Pipeline"

**Multiple Views filtered by tier:**
- View 1: "All Features" (default - shows everything)
- View 2: "FREE Tier Only" (filter: `tier:free`)
- View 3: "PREMIUM Tier Only" (filter: `tier:premium`)
- View 4: "Platform Features" (filter: `tier:platform`)
- View 5: "My Features" (filter: `assignee:@me`)
- View 6: "Next Release" (filter: `milestone:v1.2.0`)

### Example Configuration

```yaml
Project: Feature Development Pipeline
├── View: All Features (Board)
│   ├── Columns: Draft → Design → Dev → Test → Deploy → Production
│   ├── Filter: None
│   └── Sort: Priority (high → low)
│
├── View: FREE Tier (Board)
│   ├── Columns: Same as above
│   ├── Filter: label:tier:free
│   └── Sort: Target Release
│
├── View: PREMIUM Tier (Board)
│   ├── Columns: Same as above
│   ├── Filter: label:tier:premium
│   └── Sort: Target Release
│
├── View: Platform (Board)
│   ├── Columns: Same as above
│   ├── Filter: label:tier:platform
│   └── Sort: Priority
│
├── View: Roadmap (Table)
│   ├── Columns: Feature, Status, Tier, Owner, Release, Effort
│   ├── Filter: NOT status:deprecated
│   └── Sort: Target Release (ascending)
│
└── View: My Work (Board)
    ├── Columns: Same as above
    ├── Filter: assignee:@me
    └── Sort: Priority
```

### Pros

✅ **Single source of truth** - Everyone knows where to look
✅ **Easy cross-tier visibility** - See how FREE/PREMIUM/Platform features relate
✅ **Unified metrics** - Total features in flight, team capacity
✅ **Simpler management** - One board to configure/maintain
✅ **Better for small teams** - Everyone sees everything (good communication)
✅ **Easy switching** - Click between views instantly
✅ **Shared workflows** - Same lifecycle stages for all tiers
✅ **No duplication** - Features can span multiple tiers (tagged with multiple labels)

### Cons

❌ **Can get cluttered** - With 50+ features, "All Features" view overwhelming
❌ **Mixed priorities** - FREE vs PREMIUM priorities might conflict
❌ **One workflow** - All tiers must use same lifecycle stages
❌ **Permissions** - Can't give tier-specific access (all or nothing)
❌ **Performance** - Very large boards (200+ items) can slow down

### Best For

- **Small teams (1-15 people)** working across all tiers
- **Unified teams** where everyone needs visibility into everything
- **Similar workflows** where FREE/PREMIUM/Platform follow same lifecycle
- **Early stage** when you have < 50 total features

### Setup

```bash
# Create single project
gh project create --owner refine-digital --title "Feature Development Pipeline"

# Add multiple views (via GitHub UI)
# 1. Go to project
# 2. Click "+" next to views
# 3. Create "FREE Tier" view
# 4. Add filter: label:tier:free
# 5. Repeat for PREMIUM, Platform, etc.
```

---

## Option 2: Multiple GitHub Projects (One per Tier)

### What It Looks Like

**Project 1:** "FREE Tier Features"
**Project 2:** "PREMIUM Tier Features"
**Project 3:** "Platform Features"

Each with same column structure:
- Draft → Design → Dev → Test → Deploy → Production

### Example Configuration

```yaml
Project 1: FREE Tier Features
├── Auto-add rule: label:tier:free
├── Columns: Standard lifecycle
├── Custom fields: Feature Number, Complexity, Effort
└── Team: @free-tier-team

Project 2: PREMIUM Tier Features
├── Auto-add rule: label:tier:premium
├── Columns: Standard lifecycle
├── Custom fields: Feature Number, Infrastructure Cost, Complexity
└── Team: @premium-tier-team

Project 3: Platform Features
├── Auto-add rule: label:tier:platform
├── Columns: Standard lifecycle
├── Custom fields: Feature Number, Affects Tiers, Complexity
└── Team: @platform-team
```

### Pros

✅ **Focused view** - Each team sees only their tier
✅ **Team autonomy** - Each team manages their own board
✅ **Different workflows** - Can customize lifecycle per tier
✅ **Cleaner boards** - Fewer items per board (less clutter)
✅ **Tier-specific metrics** - Track FREE vs PREMIUM separately
✅ **Better permissions** - Can restrict access per project
✅ **Custom fields** - Different fields per tier (e.g., Infrastructure Cost only for PREMIUM)
✅ **Scalable** - Works well with 100+ features across all tiers

### Cons

❌ **Fragmented view** - Hard to see overall picture
❌ **Multiple places to check** - Need to visit 3 boards
❌ **Cross-tier features** - Hard to track features affecting multiple tiers
❌ **More overhead** - Configure and maintain 3 boards
❌ **Duplicate effort** - Same setup repeated 3 times
❌ **Team silos** - Teams might not see what others are doing
❌ **Harder to rebalance** - Can't easily move work between teams

### Best For

- **Large teams (30+ people)** with dedicated teams per tier
- **Separate teams** where FREE/PREMIUM/Platform have different owners
- **Different workflows** where tiers have different processes
- **Many features** (100+ total) making single board unwieldy
- **Strict permissions** where some people should only see certain tiers

### Setup

```bash
# Create three projects
gh project create --owner refine-digital --title "FREE Tier Features"
gh project create --owner refine-digital --title "PREMIUM Tier Features"
gh project create --owner refine-digital --title "Platform Features"

# Set up auto-add automation (via GitHub UI for each project)
# Project 1: Auto-add issues with label:tier:free
# Project 2: Auto-add issues with label:tier:premium
# Project 3: Auto-add issues with label:tier:platform
```

---

## Option 3: Hybrid (Multiple Projects + Overview)

### What It Looks Like

**Project 1:** "FREE Tier Features" (team board)
**Project 2:** "PREMIUM Tier Features" (team board)
**Project 3:** "Platform Features" (team board)
**Project 4:** "Executive Overview" (leadership view)

Teams work in their tier-specific boards. Leadership uses overview board.

### Example Configuration

```yaml
Project 1-3: Tier-specific boards (as in Option 2)
└── Teams work here daily

Project 4: Executive Overview (Table view only)
├── Auto-add: All features (all tiers)
├── View: Table with key metrics
├── Columns: Feature, Tier, Status, Owner, Target Release, Business Value
├── Filter: Only critical/high priority
└── Audience: Leadership, product managers
```

### Pros

✅ **Best of both worlds** - Team focus + executive visibility
✅ **Team autonomy** - Teams manage their own boards
✅ **Leadership visibility** - High-level view without clutter
✅ **Flexible** - Teams can use detailed boards, leadership uses summary
✅ **Scalable** - Works from 15 to 100+ people

### Cons

❌ **Most overhead** - 4 boards to maintain
❌ **Complexity** - Most complex to set up
❌ **Manual curation** - Overview board requires manual filtering

### Best For

- **Medium to large teams (15-50 people)** with leadership oversight
- **Organizations** with separate teams but need executive visibility
- **Balanced approach** when you need both focus and overview

---

## Option 4: By Product Area (Alternative Organization)

### What It Looks Like

Instead of organizing by tier, organize by product area:

**Project 1:** "Email & Domains" (covers FREE email + PREMIUM email features)
**Project 2:** "Infrastructure & K3s" (covers PREMIUM k3s + Platform features)
**Project 3:** "User Management & Billing" (covers Platform features)
**Project 4:** "Migration & Upgrades" (covers FREE → PREMIUM migration)

### Pros

✅ **Natural grouping** - Features grouped by what they affect
✅ **Domain expertise** - Teams organized by technical domain
✅ **End-to-end ownership** - One team owns email from FREE to PREMIUM

### Cons

❌ **Cross-cutting concerns** - Some features affect multiple areas
❌ **Uneven distribution** - Some areas might have way more features

### Best For

- **Product-oriented teams** where teams own specific domains
- **Technical specialization** where expertise matters more than tier

---

## Decision Matrix

Use this table to decide:

| Question | Single Project | Multiple Projects |
|----------|---------------|-------------------|
| How many people on the team? | 1-15 | 15+ |
| Separate teams per tier? | No | Yes |
| Need cross-tier visibility? | Yes | No |
| More than 50 active features? | No | Yes |
| Different workflows per tier? | No | Yes |
| Need tier-specific permissions? | No | Yes |
| Want simplicity? | Yes | No |

**Scoring:**
- **Mostly "Single"**: Use Option 1 (Single Project with Views)
- **Mostly "Multiple"**: Use Option 2 (Multiple Projects)
- **Mixed**: Use Option 3 (Hybrid)

---

## Real-World Examples

### Example 1: Startup (5 people)

**Team:** 2 backend, 2 frontend, 1 product manager
**Features:** 20 total (8 FREE, 7 PREMIUM, 5 Platform)

**Recommendation:** ✅ **Single Project with Views**

**Why:**
- Everyone needs to see everything (small team)
- People work across tiers
- Simple is better at this stage

**Setup:**
```yaml
Project: "All Features"
Views:
  - All Features (board)
  - By Tier (board, grouped by tier label)
  - Next Sprint (board, filtered by milestone)
  - My Work (board, filtered by assignee)
```

### Example 2: Growing Company (25 people)

**Teams:**
- FREE Tier Team (5 people) - Email/domains
- PREMIUM Tier Team (8 people) - K3s/infrastructure
- Platform Team (7 people) - Auth/billing
- Product/Design (5 people) - Work across all tiers

**Features:** 60 total (20 FREE, 25 PREMIUM, 15 Platform)

**Recommendation:** ⚠️ **Hybrid (Tier boards + Overview)**

**Why:**
- Large enough for team separation
- Product/design need cross-tier visibility
- Each team can focus on their tier

**Setup:**
```yaml
Project 1: "FREE Tier Features"
  Team: @free-tier-team
  Access: Team only

Project 2: "PREMIUM Tier Features"
  Team: @premium-tier-team
  Access: Team only

Project 3: "Platform Features"
  Team: @platform-team
  Access: Team only

Project 4: "Product Roadmap"
  Team: Product managers, leadership
  Access: Public (read-only)
  View: High-level table view
```

### Example 3: Enterprise (100 people)

**Teams:**
- Multiple teams per tier
- Dedicated infrastructure team
- Security team
- DevOps team

**Features:** 200+ total

**Recommendation:** ✅ **Multiple Projects (with rollups)**

**Why:**
- Too many features for single board
- Need team autonomy
- Need tier-specific permissions
- Different workflows per tier

**Setup:**
```yaml
Project 1: "FREE Tier Q4 2025"
Project 2: "FREE Tier Q1 2026"
Project 3: "PREMIUM Tier Q4 2025"
Project 4: "PREMIUM Tier Q1 2026"
Project 5: "Platform Core"
Project 6: "Platform Security"
Project 7: "Executive Dashboard" (rollup)
```

---

## Migration Path

### Start Small, Grow Later

**Phase 1 (Day 1):**
- Start with Single Project with Views
- Add all features to one board
- Create tier-specific views

**Phase 2 (At ~30 features or ~15 people):**
- Evaluate if single board is getting cluttered
- If yes, consider splitting

**Phase 3 (At ~60 features or ~25 people):**
- Split into multiple projects if needed
- Move issues using GitHub's bulk operations
- Set up auto-add rules

**Migration Script:**
```bash
# Move FREE tier features to new project
gh project item-list 1 --owner refine-digital --format json | \
  jq -r '.items[] | select(.content.labels[] | .name == "tier:free") | .content.url' | \
  xargs -I {} gh project item-add 2 --owner refine-digital --url {}
```

---

## My Recommendation for You

Based on the platform you're building (my.refine.digital):

### **Start with Option 1: Single Project with Views**

**Why:**
1. **You're likely starting small** - 1-10 people initially
2. **Platform is unified** - FREE and PREMIUM are the same product
3. **Need cross-tier visibility** - Migration features span both tiers
4. **Simpler to start** - Less overhead, faster to set up
5. **Easy to split later** - Can always split when you grow

### **Split later if:**
- You have 3+ dedicated teams (one per tier)
- You have 50+ active features (getting cluttered)
- Teams want more autonomy
- Different workflows per tier emerge

### Setup for Your Platform

```yaml
Project: "my.refine.digital Features"

Views:
  1. All Features (Board)
     - Columns: Draft → Design → Dev → Test → Deploy → Production
     - Filter: None
     - Sort: Priority

  2. FREE Tier (Board)
     - Filter: label:tier:free
     - Focus: Email, domains, DNS

  3. PREMIUM Tier (Board)
     - Filter: label:tier:premium
     - Focus: K3s, auto-scaling, backups

  4. Platform (Board)
     - Filter: label:tier:platform
     - Focus: Auth, billing, migration

  5. Current Sprint (Board)
     - Filter: milestone:v1.2.0
     - Focus: What we're building NOW

  6. Roadmap (Table)
     - Columns: Feature, Status, Tier, Owner, Release, Effort, Business Value
     - Filter: NOT status:deprecated
     - Sort: Target Release

  7. My Work (Board)
     - Filter: assignee:@me
     - Focus: Personal task list
```

### When to Split

**Triggers to split into multiple projects:**
1. ✅ You have 15+ people on the team
2. ✅ You have 50+ active features (board feels cluttered)
3. ✅ You have dedicated teams (e.g., FREE Team, PREMIUM Team)
4. ✅ Teams want different workflows (e.g., Platform uses 2-week sprints, PREMIUM uses kanban)
5. ✅ You need tier-specific permissions (e.g., contractors only see FREE tier)

**If 3+ triggers are true:** Split into multiple projects

---

## Summary

| Approach | Best For | Team Size | Feature Count | Complexity |
|----------|----------|-----------|---------------|------------|
| **Single Project** | Small unified teams | 1-15 | < 50 | Low ⭐ |
| **Multiple Projects** | Large teams, autonomy | 30+ | 100+ | High ⭐⭐⭐ |
| **Hybrid** | Medium teams, oversight | 15-30 | 50-100 | Very High ⭐⭐⭐⭐ |

**My recommendation for my.refine.digital:**
- ✅ Start with **Single Project with Views**
- ✅ Review every quarter
- ✅ Split when you hit 15+ people or 50+ features

---

**Related:**
- [Document 31: GitHub Integration for Features](./31-github-integration-for-features.md)
- [Document 30: Feature Documentation System](./30-feature-documentation-system.md)
