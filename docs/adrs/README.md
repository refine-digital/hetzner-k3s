# ADRs (Architecture Decision Records)

This directory contains Architecture Decision Records documenting key technical decisions.

## Active ADRs

| ADR | Decision | Date | Status | Related RFC |
|-----|----------|------|--------|-------------|
| - | - | - | - | - |

*No ADRs yet - document retroactive decisions from existing architecture*

## Suggested Retroactive ADRs

Based on the existing platform architecture (docs 01-25), consider documenting these decisions:

| Priority | Suggested ADR | Decision to Document |
|----------|---------------|---------------------|
| High | ADR-001 | Use k3s over k8s for lightweight Kubernetes |
| High | ADR-002 | Use PostgreSQL over MySQL for relational database |
| High | ADR-003 | Use Argon2id over bcrypt for password hashing |
| High | ADR-004 | Use Maildir over mbox for email storage |
| Medium | ADR-005 | Use Dovecot over Cyrus IMAP for IMAP server |
| Medium | ADR-006 | Use Postfix over Exim for SMTP server |
| Medium | ADR-007 | Use Packer for pre-baked images over cloud-init only |
| Medium | ADR-008 | Use BullMQ over other job queue systems |
| Low | ADR-009 | Use Next.js over Create React App for frontend |
| Low | ADR-010 | Use Redis over Memcached for caching |

## Superseded ADRs

When an ADR is replaced by a new decision, it's marked as "Superseded" and moved to `superseded/` directory.

| ADR | Decision | Superseded By | Date |
|-----|----------|---------------|------|
| - | - | - | - |

*No superseded ADRs*

## Creating a New ADR

1. Copy the [template](./template.md)
2. Name it `ADR-NNN-decision-statement.md` (use next sequential number)
3. Document context, options considered, and decision
4. Include consequences (positive, negative, neutral)
5. Commit to repository

## ADR Lifecycle

```
Proposed → Accepted → [Active]
                        ↓
            Superseded by ADR-XXX → Moved to superseded/
```

See [Document 30: Feature Documentation System](../30-feature-documentation-system.md) for detailed guidance.

## Why ADRs?

ADRs provide:
- **Context**: Why was this decision made?
- **Alternatives**: What other options were considered?
- **Consequences**: What are the trade-offs?
- **History**: Institutional knowledge that survives team changes

"If you don't know the history, you're doomed to repeat it."
