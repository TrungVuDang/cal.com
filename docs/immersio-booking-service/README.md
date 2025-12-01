# Immersio Booking Service Documentation

## Overview

This directory contains the complete technical documentation for the **Immersio Booking Service** - a multi-tenant SaaS platform built on Cal.com for educational scheduling and booking.

> ⚠️ **Important**: Using Cal.com **Open Source (AGPLv3)** only. Enterprise Edition (EE) features (Organizations, Payments, SSO, Admin Panel) must be built from scratch.

## Documents

| Document | Description |
|----------|-------------|
| [ANALYSIS.md](./ANALYSIS.md) | Comprehensive system analysis including use cases, architecture, module breakdown, timeline, cost estimation, and DevOps plan |
| [ESTIMATE-DETAILED.md](./ESTIMATE-DETAILED.md) | **Detailed per-feature estimates for 1 developer** with days breakdown |
| [ERD.md](./ERD.md) | Entity Relationship Diagram with Mermaid diagrams showing database schema |
| [API-SPECIFICATION.md](./API-SPECIFICATION.md) | Complete API specification for tRPC routers and REST endpoints |

## Quick Navigation

### Business Requirements
- [Executive Summary](./ANALYSIS.md#1-executive-summary)
- [Use Cases](./ANALYSIS.md#2-use-cases)
- [User Roles](./ANALYSIS.md#23-detailed-use-case-specifications)

### Technical Design
- [System Architecture](./ANALYSIS.md#3-system-architecture)
- [Multi-Tenant Design](./ANALYSIS.md#32-multi-tenant-architecture)
- [Payment Architecture](./ANALYSIS.md#34-payment-architecture-stripe-connect)

### Database
- [Schema Extensions](./ERD.md#new-tables-extend-calcom-schema)
- [ERD Diagram](./ERD.md#erd-diagram-mermaid)
- [Entity Relationships](./ERD.md#key-relationships)

### API
- [tRPC Routers](./API-SPECIFICATION.md#1-trpc-routers-immersio-extension)
- [REST Endpoints](./API-SPECIFICATION.md#2-rest-api-endpoints)
- [Webhook Events](./API-SPECIFICATION.md#3-webhook-events-outgoing-to-immersio)

### Implementation
- [Module Breakdown](./ANALYSIS.md#4-module--feature-analysis)
- [Implementation Roadmap](./ANALYSIS.md#7-implementation-roadmap)
- [Sprint Plan](./ANALYSIS.md#72-detailed-sprint-plan)

### Costs & Operations
- [Cost Estimation](./ANALYSIS.md#8-cost-estimation)
- [DevOps Plan](./ANALYSIS.md#9-devops-plan)
- [Infrastructure](./ANALYSIS.md#91-infrastructure-architecture)

## Key Findings

### Cal.com Open Source Features (Can Use)

| Feature | Reusability |
|---------|-------------|
| Event Types | 95% |
| Availability/Schedules | 100% |
| Basic Booking | 90% |
| Calendar Sync | 100% |
| Basic Webhooks | 95% |
| Email Notifications | 90% |

### EE Features (Must Build from Scratch)

| Feature | Days (1 Dev) | Complexity |
|---------|--------------|------------|
| Multi-tenant Architecture | 20 | High |
| Stripe Payments | 27 | High |
| SSO/JWT Integration | 12 | Medium |
| Admin Panel | 24 | High |
| Billing/Subscriptions | 15 | Medium |

### Effort Breakdown

| Category | Days | % of Total |
|----------|------|------------|
| Leverage Cal.com | 10.5 | 4% |
| Extend Cal.com | 28 | 10% |
| **Build from Scratch** | **250** | **86%** |

## Timeline Summary (1 Developer)

| Phase | Days | Weeks | Focus |
|-------|------|-------|-------|
| Phase 1: MVP Core | 68 | 14 | Multi-tenant, booking, whitelist |
| Phase 2: Payments | 61 | 12 | Stripe Connect, refunds |
| Phase 3: Immersio | 78 | 16 | API sync, SSO, webhooks |
| Phase 4: White-label | 85 | 17 | Landing pages, admin panel |
| **Total** | **292** | **59** | (~14 months) |
| **With 20% Buffer** | **351** | **71** | (~17 months) |

### Timeline by Team Size

| Team | Duration |
|------|----------|
| 1 Developer | ~14 months |
| 2 Developers | ~8 months |
| 3 Developers | ~5-6 months |
| 4 Devs + Lead | ~4 months |

## Cost Summary (1 Developer @ $60-90/hr)

| Phase | Hours | Cost Range (USD) |
|-------|-------|------------------|
| Phase 1: MVP | 656 | $39,360 - $59,040 |
| Phase 2: Payments | 584 | $35,040 - $52,560 |
| Phase 3: Immersio | 752 | $45,120 - $67,680 |
| Phase 4: White-label | 816 | $48,960 - $73,440 |
| **Development Subtotal** | **2,808** | **$168,480 - $252,720** |

| Additional Costs | Range (USD) |
|------------------|-------------|
| Project Management (10%) | $16,848 - $25,272 |
| QA/Testing (15%) | $25,272 - $37,908 |
| Documentation | $5,000 - $8,000 |
| **Total Project** | **$215,600 - $323,900** |

### Monthly Infrastructure
- Estimated: $380 - $930/month

### Time Savings from Cal.com OSS
- Without Cal.com: ~417 days (~20 months)
- With Cal.com OSS: ~288 days (~14 months)  
- **Savings: ~31% time reduction**

*Note: Original 60-70% estimate assumed using EE features*

## Next Steps

1. [ ] Review and confirm scope with stakeholders
2. [ ] Decide on team size and timeline
3. [ ] Setup development environment (fork Cal.com OSS)
4. [ ] Create detailed UI/UX mockups for custom components
5. [ ] Document Immersio API integration requirements
6. [ ] Begin Phase 1 development (Multi-tenant architecture first)

## Related Resources

- [Cal.com Documentation](https://cal.com/docs)
- [Cal.com GitHub](https://github.com/calcom/cal.com)
- [Prisma Schema](../../packages/prisma/schema.prisma)
- [App Store Integrations](../../packages/app-store/)

---

*Last Updated: November 2024*
