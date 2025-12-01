# Immersio Booking Service - Detailed Estimate (Single Resource)

## ⚠️ Important Notes

1. **Open Source Only**: Using Cal.com AGPLv3 (open source), NOT Enterprise Edition
2. **EE Features Must Be Built**: Organizations, Payments, SSO, Admin Panel are EE features → need custom implementation
3. **Estimates**: Per single developer (1 resource), measured in working days
4. **Buffer**: 20% contingency recommended for each phase

---

## Cal.com Feature Analysis: Open Source vs EE

| Feature | Open Source (Free) | Enterprise Edition (EE) |
|---------|-------------------|------------------------|
| Basic Event Types | ✅ | ✅ |
| Availability/Schedules | ✅ | ✅ |
| Basic Booking | ✅ | ✅ |
| Calendar Sync (Google/Outlook) | ✅ | ✅ |
| Basic Webhooks | ✅ | ✅ |
| Email Notifications | ✅ | ✅ |
| **Organizations/Multi-tenant** | ❌ | ✅ |
| **Stripe Payments** | ❌ | ✅ |
| **SSO/SAML** | ❌ | ✅ |
| **Admin Panel** | ❌ | ✅ |
| **Billing/Subscriptions** | ❌ | ✅ |
| **Managed Event Types** | ❌ | ✅ |
| **Round Robin** | ❌ | ✅ |
| **Directory Sync (SCIM)** | ❌ | ✅ |
| **Impersonation** | ❌ | ✅ |

### Impact on Development

Since we're using open source only, we need to build:
- Multi-tenant architecture from scratch
- Stripe payment integration
- Per-tenant payment routing
- SSO integration
- Admin panel for tenant management

---

## Detailed Feature Estimates (1 Developer)

### Legend
- 🟢 **Leverage**: Can use Cal.com open source as-is or with minor changes
- 🟡 **Extend**: Requires extending Cal.com open source
- 🔴 **Build**: Must build from scratch (EE equivalent)

---

## Phase 1: MVP Core

### 1.1 Project Setup & Foundation

| Task | Type | Days | Notes |
|------|------|------|-------|
| Fork Cal.com repository | 🟢 | 0.5 | Clone, setup git |
| Development environment setup | 🟢 | 1 | Docker, env vars, local DB |
| Database schema extensions | 🟡 | 2 | Prisma migrations for new tables |
| Understanding Cal.com codebase | 🟢 | 3 | Read code, understand patterns |
| **Subtotal** | | **6.5** | |

### 1.2 Multi-Tenant Architecture (Build from scratch - EE equivalent)

| Task | Type | Days | Notes |
|------|------|------|-------|
| Tenant model design | 🔴 | 2 | Design schema, relationships |
| Tenant CRUD operations | 🔴 | 3 | Create, read, update, delete |
| Tenant context middleware | 🔴 | 3 | Request context injection |
| Data isolation layer | 🔴 | 4 | Query filtering by tenant_id |
| Tenant-aware authentication | 🔴 | 3 | JWT with tenant claims |
| Subdomain routing | 🔴 | 3 | tenant.domain.com routing |
| Tenant settings management | 🔴 | 2 | Configuration per tenant |
| **Subtotal** | | **20** | |

### 1.3 Teacher Availability

| Task | Type | Days | Notes |
|------|------|------|-------|
| Availability UI adaptation | 🟢 | 1 | Use Cal.com Schedule component |
| Schedule management API | 🟢 | 1 | Use existing tRPC routes |
| Google Calendar sync | 🟢 | 0.5 | Already in Cal.com |
| Outlook Calendar sync | 🟢 | 0.5 | Already in Cal.com |
| Custom availability form | 🟡 | 2 | Add skill tags, Zoom link fields |
| Teacher profile management | 🟡 | 2 | Extend User model |
| **Subtotal** | | **7** | |

### 1.4 Workshop/Event Type

| Task | Type | Days | Notes |
|------|------|------|-------|
| Workshop EventType extension | 🟡 | 2 | Extend metadata schema |
| Workshop creation form | 🟡 | 3 | Custom fields UI |
| Workshop listing page | 🟡 | 2 | Filter by tenant, teacher |
| Recurrence configuration | 🟢 | 1 | Use Cal.com recurring events |
| Seat/capacity management | 🟢 | 1 | Use seatsPerTimeSlot |
| Workshop categories/topics | 🟡 | 2 | Add topic taxonomy |
| Draft/publish workflow | 🟡 | 2 | Workshop status management |
| **Subtotal** | | **13** | |

### 1.5 Basic Booking Flow

| Task | Type | Days | Notes |
|------|------|------|-------|
| Booking page customization | 🟡 | 2 | Tenant branding on booker |
| Booking creation API | 🟢 | 1 | Use existing booking logic |
| Attendee management | 🟢 | 0.5 | Use Attendee model |
| Booking confirmation email | 🟢 | 1 | Use existing email templates |
| ICS calendar invite | 🟢 | 0.5 | Already in Cal.com |
| Booking list view | 🟡 | 2 | Teacher/admin dashboard |
| **Subtotal** | | **7** | |

### 1.6 Whitelist Feature

| Task | Type | Days | Notes |
|------|------|------|-------|
| Whitelist database model | 🔴 | 1 | WorkshopWhitelist table |
| Whitelist CRUD API | 🔴 | 2 | Add/remove/list emails |
| Whitelist check in booking | 🔴 | 2 | Validate before booking |
| Bulk email upload | 🔴 | 2 | CSV/Excel import |
| Whitelist management UI | 🔴 | 3 | Admin interface |
| **Subtotal** | | **10** | |

### 1.7 Basic Notifications

| Task | Type | Days | Notes |
|------|------|------|-------|
| Email template customization | 🟡 | 2 | Tenant branding |
| Booking reminder emails | 🟢 | 1 | Use Cal.com workflows |
| Cancellation email | 🟢 | 0.5 | Existing template |
| Teacher notification | 🟡 | 1 | New booking alert |
| **Subtotal** | | **4.5** | |

### Phase 1 Total: 68 days (~14 weeks for 1 developer)

---

## Phase 2: Payments & Policies

### 2.1 Stripe Integration (Build from scratch - EE equivalent)

| Task | Type | Days | Notes |
|------|------|------|-------|
| Stripe SDK integration | 🔴 | 2 | Setup, configuration |
| Payment Intent creation | 🔴 | 3 | Create payment flow |
| Payment confirmation handling | 🔴 | 2 | Webhook handling |
| Payment record storage | 🔴 | 2 | Payment model, history |
| Payment UI components | 🔴 | 3 | Checkout, payment form |
| **Subtotal** | | **12** | |

### 2.2 Per-Tenant Stripe Connect

| Task | Type | Days | Notes |
|------|------|------|-------|
| Stripe Connect OAuth flow | 🔴 | 4 | Account connection |
| Tenant Stripe credentials storage | 🔴 | 2 | Secure key storage |
| Payment routing to tenant account | 🔴 | 3 | stripeAccount parameter |
| Stripe Connect webhook per tenant | 🔴 | 3 | Multi-tenant webhooks |
| Stripe setup UI for tenant admin | 🔴 | 3 | Connection wizard |
| **Subtotal** | | **15** | |

### 2.3 Refund Automation

| Task | Type | Days | Notes |
|------|------|------|-------|
| Refund policy configuration | 🔴 | 2 | Per-workshop settings |
| Refund window calculation | 🔴 | 2 | Time-based eligibility |
| Automatic refund processing | 🔴 | 3 | Stripe refund API |
| Refund status tracking | 🔴 | 2 | Status updates, history |
| Refund notification emails | 🔴 | 1 | Email templates |
| **Subtotal** | | **10** | |

### 2.4 Cancellation Policies

| Task | Type | Days | Notes |
|------|------|------|-------|
| Cancellation window config | 🟡 | 2 | Use Cal.com + extend |
| Cancellation request flow | 🟡 | 2 | Student self-service |
| Cancellation history/audit | 🔴 | 2 | BookingCancellationHistory |
| Admin cancellation override | 🔴 | 2 | Force cancel with/without refund |
| **Subtotal** | | **8** | |

### 2.5 Transaction History

| Task | Type | Days | Notes |
|------|------|------|-------|
| Transaction listing API | 🔴 | 2 | Filter, pagination |
| Transaction dashboard UI | 🔴 | 3 | Admin view |
| Export to CSV/Excel | 🔴 | 2 | Report generation |
| Revenue analytics | 🔴 | 3 | Charts, summaries |
| **Subtotal** | | **10** | |

### 2.6 Invoice Emails

| Task | Type | Days | Notes |
|------|------|------|-------|
| Invoice template design | 🔴 | 2 | PDF/HTML template |
| Invoice generation | 🔴 | 2 | Auto-generate on payment |
| Invoice email sending | 🔴 | 1 | Email integration |
| Invoice history | 🔴 | 1 | Storage, retrieval |
| **Subtotal** | | **6** | |

### Phase 2 Total: 61 days (~12 weeks for 1 developer)

---

## Phase 3: Immersio Integration

### 3.1 Immersio API Client

| Task | Type | Days | Notes |
|------|------|------|-------|
| API client architecture | 🔴 | 2 | HTTP client, error handling |
| Authentication handling | 🔴 | 2 | API key, token refresh |
| Course API integration | 🔴 | 3 | List, get course details |
| Student API integration | 🔴 | 3 | List, get student details |
| Teacher API integration | 🔴 | 2 | List, get teacher details |
| Rate limiting & retry logic | 🔴 | 2 | Resilient API calls |
| **Subtotal** | | **14** | |

### 3.2 Course Sync (Pull)

| Task | Type | Days | Notes |
|------|------|------|-------|
| Course sync service | 🔴 | 3 | Scheduled sync job |
| Course-to-Workshop mapping | 🔴 | 3 | ImmersioCourseMapping table |
| Incremental sync logic | 🔴 | 3 | Delta updates |
| Sync status tracking | 🔴 | 2 | ImmersioSyncLog |
| Manual sync trigger | 🔴 | 1 | Admin UI button |
| **Subtotal** | | **12** | |

### 3.3 Student Sync (Pull)

| Task | Type | Days | Notes |
|------|------|------|-------|
| Student sync service | 🔴 | 3 | Per-course sync |
| Auto-whitelist update | 🔴 | 3 | Sync to WorkshopWhitelist |
| Student metadata storage | 🔴 | 2 | immersio_student_id |
| Sync conflict resolution | 🔴 | 2 | Handle duplicates |
| **Subtotal** | | **10** | |

### 3.4 Webhook Events (Push to Immersio)

| Task | Type | Days | Notes |
|------|------|------|-------|
| Webhook configuration UI | 🔴 | 2 | Endpoint, secret setup |
| BOOKING_CREATED event | 🔴 | 2 | Payload, delivery |
| BOOKING_CANCELLED event | 🔴 | 2 | Include refund info |
| PAYMENT_COMPLETED event | 🔴 | 2 | Financial data |
| Webhook retry logic | 🔴 | 2 | Failed delivery handling |
| Webhook delivery logs | 🔴 | 2 | Debug, monitoring |
| **Subtotal** | | **12** | |

### 3.5 Incoming Webhooks (From Immersio)

| Task | Type | Days | Notes |
|------|------|------|-------|
| Webhook endpoint setup | 🔴 | 1 | REST API route |
| Signature verification | 🔴 | 2 | HMAC validation |
| course.updated handler | 🔴 | 2 | Update workshop |
| student.enrolled handler | 🔴 | 2 | Update whitelist |
| student.unenrolled handler | 🔴 | 1 | Remove from whitelist |
| **Subtotal** | | **8** | |

### 3.6 SSO/JWT Integration (Build from scratch - EE equivalent)

| Task | Type | Days | Notes |
|------|------|------|-------|
| JWT validation middleware | 🔴 | 3 | Verify Immersio tokens |
| User auto-provisioning | 🔴 | 3 | Create user on first login |
| Session management | 🔴 | 2 | NextAuth.js adapter |
| SSO login flow | 🔴 | 3 | Redirect, callback |
| SSO logout handling | 🔴 | 1 | Session cleanup |
| **Subtotal** | | **12** | |

### 3.7 Embedded UI Support

| Task | Type | Days | Notes |
|------|------|------|-------|
| iframe embed configuration | 🔴 | 2 | CSP, X-Frame-Options |
| Embedded booking widget | 🔴 | 4 | Standalone React component |
| Cross-origin communication | 🔴 | 2 | postMessage API |
| Deep linking support | 🔴 | 2 | Direct workshop URLs |
| **Subtotal** | | **10** | |

### Phase 3 Total: 78 days (~16 weeks for 1 developer)

---

## Phase 4: White-label & Admin

### 4.1 Landing Page Template System

| Task | Type | Days | Notes |
|------|------|------|-------|
| Template architecture | 🔴 | 3 | React component system |
| Template 1: Education | 🔴 | 4 | Design + implementation |
| Template 2: Tutor Personal | 🔴 | 4 | Design + implementation |
| Template 3: Center Small | 🔴 | 4 | Design + implementation |
| Template 4: Kids 1 | 🔴 | 4 | Design + implementation |
| Template 5: Kids 2 | 🔴 | 4 | Design + implementation |
| Template data fetching | 🔴 | 2 | API integration |
| **Subtotal** | | **25** | |

### 4.2 Template Customization UI

| Task | Type | Days | Notes |
|------|------|------|-------|
| Settings form UI | 🔴 | 4 | Color, logo, text inputs |
| Image upload handling | 🔴 | 3 | Logo, hero, avatar |
| Live preview | 🔴 | 4 | Real-time preview |
| Save & publish flow | 🔴 | 2 | Draft vs published |
| **Subtotal** | | **13** | |

### 4.3 Subdomain Routing

| Task | Type | Days | Notes |
|------|------|------|-------|
| Subdomain detection | 🔴 | 2 | Next.js middleware |
| Tenant resolution by subdomain | 🔴 | 2 | DB lookup, caching |
| DNS configuration docs | 🔴 | 1 | Wildcard DNS setup |
| Custom domain support | 🔴 | 3 | CNAME, SSL |
| **Subtotal** | | **8** | |

### 4.4 Super Admin Panel (Build from scratch - EE equivalent)

| Task | Type | Days | Notes |
|------|------|------|-------|
| Admin authentication | 🔴 | 2 | Super admin role |
| Tenant listing & management | 🔴 | 4 | CRUD, search, filter |
| Tenant approval workflow | 🔴 | 2 | Review, approve/reject |
| Subscription plan management | 🔴 | 4 | Plan CRUD, limits |
| Tenant plan assignment | 🔴 | 2 | Assign plan to tenant |
| System-wide analytics | 🔴 | 5 | Dashboard, charts |
| Activity logs viewer | 🔴 | 3 | Audit trail |
| Feature flags management | 🔴 | 2 | Toggle features |
| **Subtotal** | | **24** | |

### 4.5 Tenant Subscription & Billing (Build from scratch - EE equivalent)

| Task | Type | Days | Notes |
|------|------|------|-------|
| Subscription plan model | 🔴 | 2 | Plan features, limits |
| Usage tracking | 🔴 | 3 | Bookings/month counter |
| Plan limit enforcement | 🔴 | 3 | Block when exceeded |
| Upgrade/downgrade flow | 🔴 | 3 | Plan changes |
| Subscription billing (Stripe) | 🔴 | 4 | Recurring charges |
| **Subtotal** | | **15** | |

### Phase 4 Total: 85 days (~17 weeks for 1 developer)

---

## Summary: Total Estimate

| Phase | Days | Weeks |
|-------|------|-------|
| Phase 1: MVP Core | 68 | 14 |
| Phase 2: Payments & Policies | 61 | 12 |
| Phase 3: Immersio Integration | 78 | 16 |
| Phase 4: White-label & Admin | 85 | 17 |
| **Total** | **292** | **59** |

### With 20% Contingency Buffer

| Phase | Days | Weeks |
|-------|------|-------|
| Phase 1: MVP Core | 82 | 17 |
| Phase 2: Payments & Policies | 73 | 15 |
| Phase 3: Immersio Integration | 94 | 19 |
| Phase 4: White-label & Admin | 102 | 20 |
| **Total** | **351** | **71** (~14 months) |

---

## Feature-by-Feature Breakdown

### Features Leveraging Cal.com Open Source (Low Effort)

| Feature | Days | Cal.com Component Used |
|---------|------|----------------------|
| Development environment | 1 | Docker, scripts |
| Teacher availability UI | 1 | Schedule component |
| Schedule management API | 1 | viewer.availability routes |
| Google Calendar sync | 0.5 | google-calendar app |
| Outlook Calendar sync | 0.5 | office365-calendar app |
| Recurrence configuration | 1 | EventType.recurringEvent |
| Seat management | 1 | seatsPerTimeSlot |
| Booking creation | 1 | Booking logic |
| Attendee management | 0.5 | Attendee model |
| Confirmation email | 1 | Email templates |
| ICS calendar invite | 0.5 | ICS generator |
| Booking reminder | 1 | Workflows |
| Cancellation email | 0.5 | Email templates |
| **Total (Leverage)** | **10.5** | |

### Features Extending Cal.com (Medium Effort)

| Feature | Days | Extension Type |
|---------|------|---------------|
| Database schema extensions | 2 | Prisma migrations |
| Custom availability form | 2 | Add fields to UI |
| Teacher profile | 2 | Extend User model |
| Workshop EventType | 2 | Extend metadata |
| Workshop creation form | 3 | Custom UI |
| Workshop listing | 2 | New page |
| Workshop categories | 2 | New taxonomy |
| Draft/publish workflow | 2 | Status field |
| Booking page customization | 2 | Tenant branding |
| Booking list view | 2 | Dashboard |
| Email customization | 2 | Templates |
| Teacher notification | 1 | New email type |
| Cancellation window | 2 | Extend config |
| Cancellation flow | 2 | Self-service |
| **Total (Extend)** | **28** | |

### Features Built from Scratch (High Effort - EE Equivalents)

| Feature | Days | EE Feature Replaced |
|---------|------|-------------------|
| Multi-tenant architecture | 20 | Organizations |
| Whitelist feature | 10 | Custom |
| Stripe integration | 12 | Payments |
| Per-tenant Stripe Connect | 15 | Payments |
| Refund automation | 10 | Payments |
| Cancellation history | 4 | Custom |
| Transaction history | 10 | Custom |
| Invoice emails | 6 | Custom |
| Immersio API client | 14 | Custom |
| Course sync | 12 | Custom |
| Student sync | 10 | Custom |
| Outgoing webhooks | 12 | Custom |
| Incoming webhooks | 8 | Custom |
| SSO/JWT | 12 | SSO |
| Embedded UI | 10 | Custom |
| Landing page templates | 25 | Custom |
| Template customization | 13 | Custom |
| Subdomain routing | 8 | Organizations |
| Super Admin panel | 24 | Admin |
| Tenant subscriptions | 15 | Billing |
| **Total (Build)** | **250** | |

---

## Resource Scaling Options

### Option A: 1 Developer (As Estimated)
- **Duration**: ~14 months (with buffer)
- **Best for**: Budget-constrained, no time pressure
- **Risk**: Key person dependency

### Option B: 2 Developers
- **Duration**: ~8 months
- **Split**: 
  - Dev 1: Multi-tenant, Payments, Admin
  - Dev 2: Workshops, Immersio, Landing Pages
- **Overlap**: Database, APIs, Testing

### Option C: 3 Developers
- **Duration**: ~5-6 months
- **Split**:
  - Dev 1: Multi-tenant architecture, Admin panel
  - Dev 2: Payments, Subscriptions, Billing
  - Dev 3: Immersio integration, Landing pages
- **Need**: Tech Lead for coordination

### Option D: 4 Developers + Tech Lead
- **Duration**: ~4 months
- **Split**:
  - Tech Lead: Architecture, code review
  - Dev 1: Multi-tenant, data layer
  - Dev 2: Payments (Stripe Connect, refunds)
  - Dev 3: Immersio integration, SSO
  - Dev 4: UI (Landing pages, Admin, Booking)

---

## Cost Estimate (Updated)

### Single Developer Rate: $60-90/hour

| Phase | Days | Hours (8hr/day) | Cost Range (USD) |
|-------|------|-----------------|------------------|
| Phase 1: MVP | 82 | 656 | $39,360 - $59,040 |
| Phase 2: Payments | 73 | 584 | $35,040 - $52,560 |
| Phase 3: Immersio | 94 | 752 | $45,120 - $67,680 |
| Phase 4: White-label | 102 | 816 | $48,960 - $73,440 |
| **Total** | **351** | **2,808** | **$168,480 - $252,720** |

### Additional Costs

| Item | Cost Range (USD) |
|------|-----------------|
| Project Management (10%) | $16,848 - $25,272 |
| QA/Testing (15%) | $25,272 - $37,908 |
| Documentation | $5,000 - $8,000 |
| **Total Project** | **$215,600 - $323,900** |

### Note on Cost Difference from Previous Estimate

Previous estimate assumed:
- Team of developers working in parallel
- Using Cal.com Enterprise features
- Shorter timeline

Updated estimate reflects:
- Single developer working sequentially
- Building EE features from scratch
- More realistic complexity

---

## Risk Factors

| Risk | Impact | Mitigation |
|------|--------|------------|
| Cal.com breaking changes | High | Pin version, minimal core changes |
| Immersio API changes | Medium | Abstract API client layer |
| Single developer dependency | High | Documentation, code reviews |
| Stripe API complexity | Medium | Use existing patterns |
| Multi-tenant security gaps | High | Security audit, penetration testing |
| Performance at scale | Medium | Load testing, optimization |

---

## Recommendations

1. **Start with Phase 1 + Phase 2** together for a complete paid booking MVP
2. **Consider 2 developers minimum** to reduce timeline to 8 months
3. **Prioritize multi-tenant architecture** - it's foundational
4. **Build Stripe integration early** - critical for revenue
5. **Immersio integration can be phased** - start with one-way sync
6. **Landing pages can use simpler approach** - maybe just 2-3 templates initially

---

*Estimate Version: 2.0*  
*Assumptions: 1 developer, 8 hours/day, 5 days/week*  
*Does not include: DevOps, infrastructure, deployment, maintenance*
