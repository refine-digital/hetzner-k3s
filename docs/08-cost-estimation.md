# Cost Estimation
**Part:** 8 of 11

## Cost Estimation

### Single Client Examples

**Dev/Test: Single CAX11**
- 1× CAX11: €3.85/month
- Network: €0
- Firewall: €0
- **Total: ~€4/month**

**Production: HA Cluster**
- 3× CAX21: 3 × €12.39 = €37.17/month
- 2× CAX21 workers: 2 × €12.39 = €24.78/month
- Load balancer (lb11): €5.95/month
- Network: €0
- **Total: ~€68/month**

**Production: With Autoscaling**
- 3× CAX21 masters: €37.17/month
- 2-10× CAX21 workers: €24.78 - €123.90/month (variable)
- Load balancer: €5.95/month
- Volumes (100GB each): 3 × €4.76 = €14.28/month
- **Total: €82 - €181/month**

### Multi-Tenant Platform (10 Clients)

**Scenario 1: All Dev Clusters**
- 10× single CAX11: €38.50/month
- Platform overhead: €50/month (monitoring, CI/CD)
- **Total: ~€90/month**

**Scenario 2: Mixed (5 Dev, 5 Prod)**
- 5× dev (CAX11): €19.25/month
- 5× prod (HA): 5 × €68 = €340/month
- Platform overhead: €100/month
- **Total: ~€460/month**

**Scenario 3: All Production HA**
- 10× prod (HA): 10 × €68 = €680/month
- Platform overhead: €150/month
- **Total: ~€830/month**

**Per-Client Pricing (Your Revenue):**
- Dev cluster: €10-20/month (2.5-5x markup)
- Prod HA: €100-150/month (1.5-2x markup)
- Margin: 50-75% on infrastructure costs

---

## Navigation

**Previous:** [07-implementation-roadmap.md](./07-implementation-roadmap.md)
**Next:** [09-pitfalls-and-solutions.md](./09-pitfalls-and-solutions.md)
**Back to:** [ARCHITECTURE-REVIEW.md](./ARCHITECTURE-REVIEW.md)
