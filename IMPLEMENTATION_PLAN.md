# KẾ HOẠCH TRIỂN KHAI CHI TIẾT - BOOKING SERVICE

## MỤC LỤC
1. [Setup & Infrastructure](#1-setup--infrastructure)
2. [Phase 1: MVP Core](#2-phase-1-mvp-core)
3. [Phase 2: Payments](#3-phase-2-payments)
4. [Phase 3: Immersio Integration](#4-phase-3-immersio-integration)
5. [Phase 4: Advanced Features](#5-phase-4-advanced-features)
6. [Testing Strategy](#6-testing-strategy)
7. [Deployment Plan](#7-deployment-plan)

---

## 1. SETUP & INFRASTRUCTURE

### 1.1. Project Setup

#### 1.1.1. Repository Structure
```
booking-service/
├── apps/
│   ├── web/              # Next.js web app
│   └── api/              # API server (if separate)
├── packages/
│   ├── prisma/           # Database schema
│   ├── trpc/             # tRPC routers
│   ├── features/
│   │   ├── bookings/     # Extended from Cal.com
│   │   ├── multi-tenant/ # NEW: Multi-tenant logic
│   │   ├── immersio/     # NEW: Immersio integration
│   │   ├── workshops/    # NEW: Workshop management
│   │   └── landing/      # NEW: Landing pages
│   ├── ui/               # Shared UI components
│   └── lib/              # Shared utilities
├── .env.example
├── docker-compose.yml
└── README.md
```

#### 1.1.2. Environment Variables
```bash
# Database
DATABASE_URL=postgresql://...
DATABASE_DIRECT_URL=postgresql://...

# Multi-tenant
NEXT_PUBLIC_APP_URL=https://booking-service.com
ENABLE_MULTI_TENANT=true

# Stripe (global, but per-tenant keys in DB)
STRIPE_WEBHOOK_SECRET=whsec_...

# Immersio
IMMERSIO_API_BASE_URL=https://api.immersio.com
IMMERSIO_DEFAULT_TIMEOUT=30000

# Redis
REDIS_URL=redis://...

# Auth
NEXTAUTH_SECRET=...
NEXTAUTH_URL=https://booking-service.com
```

### 1.2. Infrastructure Setup

#### 1.2.1. Database
- **PostgreSQL 14+** với connection pooling (PgBouncer)
- **Read replicas** cho scaling
- **Backup**: Daily automated backups

#### 1.2.2. Redis
- **Caching**: Tenant settings, event types
- **Queue**: Background jobs (sync, emails)
- **Rate limiting**: API rate limits per tenant

#### 1.2.3. Hosting
- **Vercel/Railway** cho Next.js app
- **AWS RDS** cho PostgreSQL
- **AWS ElastiCache** cho Redis
- **Cloudflare** cho CDN và DDoS protection

---

## 2. PHASE 1: MVP CORE

### 2.1. Week 1-2: Foundation & Multi-Tenant

#### Task 1.1: Database Schema Setup
**Owner**: Backend Dev  
**Effort**: 3 days

```typescript
// packages/prisma/schema.prisma
// Add Tenant model and extend existing models
```

**Deliverables**:
- [ ] Tenant model created
- [ ] Migration scripts ready
- [ ] Seed data for testing

#### Task 1.2: Multi-Tenant Middleware
**Owner**: Backend Dev  
**Effort**: 5 days

```typescript
// packages/features/multi-tenant/lib/tenant-context.ts
export function getTenantFromRequest(req: Request): Promise<Tenant> {
  // Extract tenant from subdomain or custom domain
  // Verify tenant exists and is active
  // Set tenant context for request
}
```

**Deliverables**:
- [ ] Subdomain routing middleware
- [ ] Tenant context provider
- [ ] Tenant isolation in queries

#### Task 1.3: Tenant CRUD APIs
**Owner**: Backend Dev  
**Effort**: 4 days

```typescript
// packages/trpc/server/routers/tenant.ts
export const tenantRouter = router({
  create: protectedProcedure
    .input(z.object({ name, subdomain, adminEmail }))
    .mutation(async ({ input, ctx }) => {
      // Create tenant
      // Create admin user
      // Send activation email
    }),
  // ... other CRUD operations
});
```

**Deliverables**:
- [ ] Create tenant API
- [ ] Update tenant API
- [ ] Get tenant API
- [ ] List tenants API (super admin)

### 2.2. Week 3-4: Cal.com Integration

#### Task 2.1: Extend Booking Module
**Owner**: Backend Dev  
**Effort**: 5 days

```typescript
// packages/features/bookings/lib/handleNewBooking.ts
// Add tenant_id to all booking queries
// Filter by tenant in availability checks
```

**Changes needed**:
1. Add `tenantId` to booking creation
2. Filter availability by tenant
3. Update email templates with tenant branding

**Deliverables**:
- [ ] Booking creation works with tenant
- [ ] Availability filtered by tenant
- [ ] Email notifications include tenant info

#### Task 2.2: Extend EventType Module
**Owner**: Backend Dev  
**Effort**: 4 days

```typescript
// packages/features/eventtypes/lib/eventTypeService.ts
// Add tenant filtering
// Add workshop-specific fields
```

**Deliverables**:
- [ ] EventType CRUD with tenant
- [ ] Workshop flag support
- [ ] Basic seat limitation

### 2.3. Week 5-6: Teacher & Availability

#### Task 3.1: Teacher Availability UI
**Owner**: Frontend Dev  
**Effort**: 5 days

```typescript
// apps/web/app/(use-page-wrapper)/availability/page.tsx
// Reuse Cal.com availability UI
// Add tenant context
```

**Deliverables**:
- [ ] Availability calendar UI
- [ ] Time slot selection
- [ ] Save availability

#### Task 3.2: Calendar Sync
**Owner**: Backend Dev  
**Effort**: 3 days

**Reuse**: Cal.com calendar sync module  
**Customization**: Add tenant context

**Deliverables**:
- [ ] Google Calendar sync
- [ ] Outlook sync
- [ ] Sync status UI

#### Task 3.3: Teacher Assignment
**Owner**: Full-stack Dev  
**Effort**: 4 days

```typescript
// packages/features/workshops/lib/teacherAssignment.ts
export async function assignTeacherToCourse(
  tenantId: number,
  eventTypeId: number,
  teacherId: number,
  scheduleId: number
) {
  // Create assignment
  // Create series events
  // Validate conflicts
}
```

**Deliverables**:
- [ ] Assignment UI
- [ ] Series event creation
- [ ] Conflict detection

### 2.4. Week 7-8: Basic Workshop

#### Task 4.1: Workshop EventType Extension
**Owner**: Backend Dev  
**Effort**: 4 days

```typescript
// Extend EventType with workshop fields
interface WorkshopEventType extends EventType {
  isWorkshop: true;
  seatLimit: number;
  price: number;
  currency: string;
  whitelistEnabled: boolean;
}
```

**Deliverables**:
- [ ] Workshop creation API
- [ ] Workshop listing API
- [ ] Workshop detail API

#### Task 4.2: Basic Booking Flow
**Owner**: Full-stack Dev  
**Effort**: 5 days

**Reuse**: Cal.com booking flow  
**Customization**: 
- Seat availability check
- Basic whitelist check (if enabled)

**Deliverables**:
- [ ] Booking page for workshop
- [ ] Seat availability display
- [ ] Booking confirmation

---

## 3. PHASE 2: PAYMENTS

### 3.1. Week 9-10: Multi-Stripe Integration

#### Task 5.1: Stripe Multi-Account Setup
**Owner**: Backend Dev  
**Effort**: 5 days

```typescript
// packages/features/payments/lib/stripe-multi-account.ts
export async function getStripeClientForTenant(
  tenantId: number
): Promise<Stripe> {
  const tenant = await getTenant(tenantId);
  return new Stripe(tenant.stripeSecretKey, {
    apiVersion: '2023-10-16',
  });
}
```

**Deliverables**:
- [ ] Per-tenant Stripe client
- [ ] Key encryption/decryption
- [ ] Webhook routing per tenant

#### Task 5.2: Payment Flow Integration
**Owner**: Full-stack Dev  
**Effort**: 5 days

**Reuse**: Cal.com payment module  
**Customization**:
- Route to correct Stripe account
- Handle per-tenant webhooks

**Deliverables**:
- [ ] Payment checkout flow
- [ ] Webhook handling
- [ ] Payment status updates

### 3.2. Week 11-12: Refund & Cancellation

#### Task 6.1: Refund Window Logic
**Owner**: Backend Dev  
**Effort**: 4 days

```typescript
// packages/features/bookings/lib/refund.ts
export async function canRefundBooking(
  booking: Booking,
  eventType: EventType
): Promise<boolean> {
  const refundWindow = eventType.refundWindowHours * 60 * 60 * 1000;
  const bookingTime = booking.startTime.getTime();
  const now = Date.now();
  return (now - bookingTime) <= refundWindow;
}
```

**Deliverables**:
- [ ] Refund window validation
- [ ] Automatic refund processing
- [ ] Refund history tracking

#### Task 6.2: Cancellation UI
**Owner**: Frontend Dev  
**Effort**: 3 days

**Deliverables**:
- [ ] Cancel booking UI
- [ ] Refund status display
- [ ] Cancellation confirmation

### 3.3. Week 13: Reschedule

#### Task 7.1: Reschedule Logic
**Owner**: Backend Dev  
**Effort**: 3 days

**Reuse**: Cal.com reschedule module  
**Customization**: Add tenant context

**Deliverables**:
- [ ] Reschedule API
- [ ] Availability check for new slot
- [ ] Email notifications

---

## 4. PHASE 3: IMMERSIO INTEGRATION

### 4.1. Week 14-15: Immersio API Client

#### Task 8.1: API Client Library
**Owner**: Backend Dev  
**Effort**: 5 days

```typescript
// packages/features/immersio/lib/immersio-client.ts
export class ImmersioClient {
  constructor(
    private apiKey: string,
    private apiUrl: string
  ) {}

  async getCourses(tenantId: string): Promise<Course[]> {
    // Call Immersio API
  }

  async getStudents(courseId: string): Promise<Student[]> {
    // Call Immersio API
  }

  async createEnrollment(data: EnrollmentData): Promise<Enrollment> {
    // Push to Immersio
  }
}
```

**Deliverables**:
- [ ] Immersio API client
- [ ] Error handling & retries
- [ ] Rate limiting

#### Task 8.2: Authentication & Credentials
**Owner**: Backend Dev  
**Effort**: 3 days

**Deliverables**:
- [ ] Credential storage (encrypted)
- [ ] Token refresh logic
- [ ] Auth error handling

### 4.2. Week 16-17: Course & Student Sync

#### Task 9.1: Pull Courses
**Owner**: Backend Dev  
**Effort**: 4 days

```typescript
// packages/features/immersio/lib/sync-courses.ts
export async function syncCoursesFromImmersio(
  tenantId: number
): Promise<SyncResult> {
  const client = getImmersioClient(tenantId);
  const courses = await client.getCourses();
  
  for (const course of courses) {
    await upsertEventTypeFromCourse(tenantId, course);
  }
}
```

**Deliverables**:
- [ ] Course sync job
- [ ] Conflict resolution
- [ ] Sync status tracking

#### Task 9.2: Pull Students
**Owner**: Backend Dev  
**Effort**: 4 days

**Deliverables**:
- [ ] Student sync job
- [ ] User creation/update
- [ ] Whitelist auto-population

#### Task 9.3: Sync Scheduler
**Owner**: Backend Dev  
**Effort**: 2 days

**Deliverables**:
- [ ] Cron job setup
- [ ] Retry logic
- [ ] Error notifications

### 4.3. Week 18: Booking Push & SSO

#### Task 10.1: Push Bookings
**Owner**: Backend Dev  
**Effort**: 4 days

```typescript
// packages/features/immersio/lib/push-booking.ts
export async function pushBookingToImmersio(
  booking: Booking
): Promise<void> {
  if (!booking.tenant.immersioApiKey) return;
  
  const client = getImmersioClient(booking.tenantId);
  const enrollment = await client.createEnrollment({
    courseId: booking.eventType.immersioCourseId,
    studentId: booking.user.immersioUserId,
    bookingId: booking.id,
  });
  
  await updateBookingWithImmersioId(booking.id, enrollment.id);
}
```

**Deliverables**:
- [ ] Booking push on creation
- [ ] Enrollment creation in Immersio
- [ ] Error handling & retries

#### Task 10.2: SSO Implementation
**Owner**: Backend Dev  
**Effort**: 3 days

```typescript
// apps/web/app/api/auth/immersio/callback/route.ts
export async function GET(request: Request) {
  const token = request.searchParams.get('token');
  const tenantId = request.searchParams.get('tenant_id');
  
  // Verify JWT token from Immersio
  const payload = await verifyImmersioToken(token, tenantId);
  
  // Create or get user
  const user = await getOrCreateUserFromImmersio(payload);
  
  // Create session
  await signIn(user);
}
```

**Deliverables**:
- [ ] SSO endpoint
- [ ] JWT verification
- [ ] Auto-login flow

---

## 5. PHASE 4: ADVANCED FEATURES

### 5.1. Week 19-20: Whitelist & Advanced Workshop

#### Task 11.1: Whitelist Management
**Owner**: Full-stack Dev  
**Effort**: 4 days

**Deliverables**:
- [ ] Whitelist UI (add/remove emails)
- [ ] Bulk import from CSV
- [ ] Auto-sync from Immersio students

#### Task 11.2: Whitelist Validation
**Owner**: Backend Dev  
**Effort**: 3 days

```typescript
// packages/features/bookings/lib/whitelist-check.ts
export async function checkWhitelist(
  eventTypeId: number,
  email: string
): Promise<boolean> {
  const entry = await prisma.whitelistEntry.findUnique({
    where: { eventTypeId_email: { eventTypeId, email } }
  });
  return !!entry;
}
```

**Deliverables**:
- [ ] Whitelist check in booking flow
- [ ] Error messages for non-whitelisted users
- [ ] Admin override option

### 5.2. Week 21-22: Landing Pages

#### Task 12.1: Template System
**Owner**: Frontend Dev  
**Effort**: 5 days

```typescript
// packages/features/landing/lib/templates.ts
export const templates = {
  1: EducationCenterTemplate,
  2: PersonalTutorTemplate,
  // ... 3 more
};
```

**Deliverables**:
- [ ] 5 template components
- [ ] Template preview
- [ ] Template selection UI

#### Task 12.2: Customization UI
**Owner**: Frontend Dev  
**Effort**: 4 days

**Deliverables**:
- [ ] Settings form
- [ ] Image upload
- [ ] Color picker
- [ ] Live preview

#### Task 12.3: Public Page Rendering
**Owner**: Full-stack Dev  
**Effort**: 3 days

```typescript
// apps/web/app/[subdomain]/page.tsx
export default async function LandingPage({
  params
}: {
  params: { subdomain: string }
}) {
  const tenant = await getTenantBySubdomain(params.subdomain);
  const settings = await getLandingSettings(tenant.id);
  const Template = templates[settings.templateId];
  
  return <Template settings={settings} tenant={tenant} />;
}
```

**Deliverables**:
- [ ] Public page route
- [ ] SEO optimization
- [ ] Performance optimization

### 5.3. Week 23: Admin Panel

#### Task 13.1: Super Admin Dashboard
**Owner**: Full-stack Dev  
**Effort**: 5 days

**Deliverables**:
- [ ] Tenant list & management
- [ ] Subscription management
- [ ] Usage statistics
- [ ] System logs

---

## 6. TESTING STRATEGY

### 6.1. Unit Tests
- **Coverage target**: >80%
- **Tools**: Vitest
- **Focus**: Business logic, utilities

### 6.2. Integration Tests
- **Focus**: API endpoints, database operations
- **Tools**: Vitest + Test DB
- **Coverage**: Critical flows

### 6.3. E2E Tests
- **Focus**: User journeys
- **Tools**: Playwright
- **Scenarios**:
  - Tenant creation → Booking → Payment
  - Immersio sync → Booking → Push back
  - Whitelist booking flow

### 6.4. Performance Tests
- **Load testing**: 1000 concurrent bookings
- **Database**: Query performance với 10k tenants
- **API**: Response time <200ms (p95)

---

## 7. DEPLOYMENT PLAN

### 7.1. Staging Environment
- **Purpose**: Pre-production testing
- **Database**: Separate staging DB
- **Deploy**: Auto-deploy on merge to `develop`

### 7.2. Production Deployment
- **Strategy**: Blue-green deployment
- **Rollback**: Instant rollback capability
- **Monitoring**: Real-time alerts

### 7.3. Post-Launch
- **Week 1**: Monitor closely, fix critical bugs
- **Week 2-4**: Iterate based on user feedback
- **Ongoing**: Feature enhancements

---

**Document Version**: 1.0  
**Last Updated**: 2024-03-XX
