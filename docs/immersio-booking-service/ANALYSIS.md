# Immersio Booking Service - System Analysis & Design Document

## Table of Contents
1. [Executive Summary](#1-executive-summary)
2. [Use Cases](#2-use-cases)
3. [System Architecture](#3-system-architecture)
4. [Module & Feature Analysis](#4-module--feature-analysis)
5. [Database Schema (ERD)](#5-database-schema-erd)
6. [API Specification](#6-api-specification)
7. [Implementation Roadmap](#7-implementation-roadmap)
8. [Cost Estimation](#8-cost-estimation)
9. [DevOps Plan](#9-devops-plan)

---

## 1. Executive Summary

### 1.1 Project Overview
The Immersio Booking Service is a **multi-tenant SaaS platform** built on top of the open-source **Cal.com** scheduling system. It enables teachers and language centers to:
- Create and manage paid classes and workshops
- Handle student registrations with integrated payments
- Sync with Immersio's existing LMS platform

### 1.2 Key Objectives
| Objective | Target |
|-----------|--------|
| Development Cost Reduction | 60-70% by leveraging Cal.com |
| Time to MVP | 4-6 weeks |
| Multi-tenant Isolation | Complete data isolation per tenant |
| Payment Routing | Direct Stripe integration per tenant |

### 1.3 Cal.com Reusability Analysis

> ⚠️ **Important**: Using Cal.com **Open Source (AGPLv3)** only. Enterprise Edition (EE) features must be built from scratch.

#### Open Source Features (Can Use)

| Cal.com Feature | Reusability | Customization Needed |
|----------------|-------------|---------------------|
| **Event Types** | ✅ 95% | Add workshop-specific fields |
| **Availability/Schedules** | ✅ 100% | None - use as-is |
| **Basic Booking** | ✅ 90% | Add whitelist, Immersio metadata |
| **Calendar Sync** | ✅ 100% | Google/Outlook integration |
| **Basic Webhooks** | ✅ 95% | Add Immersio endpoints |
| **Email Notifications** | ✅ 90% | Custom email templates |
| **Seats/Capacity** | ✅ 100% | Built-in Group Events |

#### Enterprise Features (Must Build from Scratch)

| EE Feature | Build Effort | Description |
|------------|--------------|-------------|
| **Organizations/Multi-tenant** | 🔴 High (20 days) | Tenant isolation, subdomain routing |
| **Stripe Payments** | 🔴 High (27 days) | Payment processing, per-tenant Stripe Connect |
| **SSO/SAML** | 🔴 Medium (12 days) | JWT validation, Immersio SSO |
| **Admin Panel** | 🔴 High (24 days) | Super admin, tenant management |
| **Billing/Subscriptions** | 🔴 Medium (15 days) | Plan management, usage limits |

See [ESTIMATE-DETAILED.md](./ESTIMATE-DETAILED.md) for complete breakdown.

---

## 2. Use Cases

### 2.1 Actor Definitions

```
┌─────────────────────────────────────────────────────────────────┐
│                        ACTORS                                    │
├─────────────────────────────────────────────────────────────────┤
│  Super Admin     │ Platform operator, manages tenants & plans   │
│  Tenant Admin    │ School owner/manager, configures tenant      │
│  Teacher         │ Instructor, manages availability & classes   │
│  Student         │ End-user, books classes & workshops          │
│  Immersio System │ External LMS for data synchronization        │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Use Case Diagrams

#### UC-001: Tenant Management
```
Super Admin
    │
    ├── UC-001.1: Create Tenant
    ├── UC-001.2: Configure Subscription Plan
    ├── UC-001.3: Approve/Reject Tenant
    ├── UC-001.4: View System Logs
    └── UC-001.5: Manage Feature Flags
```

#### UC-002: Tenant Configuration
```
Tenant Admin
    │
    ├── UC-002.1: Setup Stripe API Keys
    ├── UC-002.2: Configure Webhooks
    ├── UC-002.3: Invite Teachers
    ├── UC-002.4: Create Workshop/Live Class
    ├── UC-002.5: Configure Whitelist
    ├── UC-002.6: Setup Landing Page
    ├── UC-002.7: Link Immersio Courses
    └── UC-002.8: Export Schedules
```

#### UC-003: Teacher Operations
```
Teacher
    │
    ├── UC-003.1: Set Availability
    ├── UC-003.2: Sync External Calendar
    ├── UC-003.3: View Bookings
    ├── UC-003.4: Manage Zoom Links
    ├── UC-003.5: Update Profile
    └── UC-003.6: Receive Notifications
```

#### UC-004: Student Booking Flow
```
Student
    │
    ├── UC-004.1: Browse Workshops
    ├── UC-004.2: View Available Slots
    ├── UC-004.3: Register for Class
    ├── UC-004.4: Make Payment
    ├── UC-004.5: Receive Confirmation
    ├── UC-004.6: Add to Calendar (ICS)
    ├── UC-004.7: Cancel Booking
    ├── UC-004.8: Request Refund
    └── UC-004.9: Reschedule
```

#### UC-005: Immersio Integration
```
Immersio System
    │
    ├── UC-005.1: Sync Course Data → Booking Service
    ├── UC-005.2: Sync Student Data → Booking Service
    ├── UC-005.3: Sync Teacher Data → Booking Service
    ├── UC-005.4: Receive Live Class Schedules
    ├── UC-005.5: Receive Booking Confirmations
    └── UC-005.6: Receive Payment Transactions
```

### 2.3 Detailed Use Case Specifications

#### UC-004.3: Register for Class (Happy Path)

| Field | Description |
|-------|-------------|
| **Use Case ID** | UC-004.3 |
| **Name** | Register for Class |
| **Actor** | Student |
| **Preconditions** | 1. Student is authenticated<br>2. Workshop has available slots<br>3. Student meets whitelist criteria (if enabled) |
| **Main Flow** | 1. Student selects workshop from listing<br>2. System displays available time slots<br>3. Student selects preferred slot<br>4. System validates whitelist (if enabled)<br>5. System redirects to payment (if paid event)<br>6. Student completes payment via Stripe<br>7. System creates booking record<br>8. System sends confirmation email<br>9. System generates ICS calendar event |
| **Postconditions** | 1. Booking record created<br>2. Seat count decremented<br>3. Payment recorded<br>4. Webhook triggered to Immersio |
| **Alternative Flows** | A1: Whitelist check fails → Show error<br>A2: Payment fails → Cancel booking<br>A3: No slots available → Show waitlist option |

#### UC-004.7: Cancel Booking with Refund

| Field | Description |
|-------|-------------|
| **Use Case ID** | UC-004.7 |
| **Name** | Cancel Booking |
| **Actor** | Student |
| **Preconditions** | 1. Student has active booking<br>2. Booking not already cancelled |
| **Main Flow** | 1. Student navigates to "My Bookings"<br>2. Student selects booking to cancel<br>3. System checks cancellation window (12 hours)<br>4. If within window: initiate automatic refund<br>5. System updates booking status<br>6. System sends cancellation email<br>7. System increments available seats<br>8. Webhook notifies Immersio |
| **Business Rules** | - Refund only within 12-hour window before event<br>- Record cancellation reason<br>- Notify teacher of cancellation |

---

## 3. System Architecture

### 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           IMMERSIO BOOKING SERVICE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐      │
│   │   Landing Page  │     │   Booking UI    │     │   Admin Panel   │      │
│   │   (NextJS SSR)  │     │ (Cal.com Embed) │     │    (NextJS)     │      │
│   └────────┬────────┘     └────────┬────────┘     └────────┬────────┘      │
│            │                       │                       │                │
│            └───────────────────────┼───────────────────────┘                │
│                                    │                                        │
│                           ┌────────▼────────┐                               │
│                           │   API Gateway   │                               │
│                           │   (Next API)    │                               │
│                           └────────┬────────┘                               │
│                                    │                                        │
│   ┌────────────────────────────────┼────────────────────────────────┐      │
│   │                                │                                 │      │
│   ▼                                ▼                                 ▼      │
│ ┌──────────────┐  ┌──────────────────────┐  ┌──────────────────────┐       │
│ │   Tenant     │  │   Cal.com Core       │  │   Immersio          │       │
│ │   Service    │  │   (Booking Engine)   │  │   Integration       │       │
│ │              │  │                      │  │   Service           │       │
│ │ - CRUD       │  │ - Event Types        │  │                     │       │
│ │ - Stripe Keys│  │ - Availability       │  │ - Course Sync       │       │
│ │ - Plans      │  │ - Bookings           │  │ - Student Sync      │       │
│ │ - Landing    │  │ - Payments           │  │ - Webhooks          │       │
│ └──────┬───────┘  │ - Notifications      │  └──────────┬──────────┘       │
│        │          │ - Webhooks           │             │                   │
│        │          └──────────┬───────────┘             │                   │
│        │                     │                         │                   │
│        └─────────────────────┼─────────────────────────┘                   │
│                              │                                              │
│                     ┌────────▼────────┐                                     │
│                     │   PostgreSQL    │                                     │
│                     │   (Multi-tenant)│                                     │
│                     └────────┬────────┘                                     │
│                              │                                              │
│                     ┌────────▼────────┐                                     │
│                     │     Redis       │                                     │
│                     │ (Cache/Queue)   │                                     │
│                     └─────────────────┘                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
              ┌─────▼─────┐  ┌──────▼──────┐  ┌────▼────┐
              │  Stripe   │  │   Immersio  │  │  Zoom   │
              │  Connect  │  │   LMS API   │  │   API   │
              └───────────┘  └─────────────┘  └─────────┘
```

### 3.2 Multi-Tenant Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-TENANT DATA MODEL                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Cal.com Organization Model → Immersio Tenant                  │
│                                                                 │
│   ┌─────────────────────────────────────────────────────┐      │
│   │                 Team (isOrganization=true)          │      │
│   │                       ↓                             │      │
│   │              Maps to: TENANT                        │      │
│   │                                                     │      │
│   │   • id → tenant_id                                  │      │
│   │   • slug → subdomain (tenant.booking.immersio.com)  │      │
│   │   • metadata → stripe_credentials, plan_info        │      │
│   │   • organizationSettings → tenant_settings          │      │
│   └─────────────────────────────────────────────────────┘      │
│                           │                                     │
│                           ▼                                     │
│   ┌─────────────────────────────────────────────────────┐      │
│   │              Team (parentId=tenant_id)              │      │
│   │                       ↓                             │      │
│   │              Maps to: DEPARTMENT/CLASS_GROUP        │      │
│   │                                                     │      │
│   │   • Can have sub-teams for course categories        │      │
│   │   • Teachers assigned via Membership                │      │
│   └─────────────────────────────────────────────────────┘      │
│                           │                                     │
│                           ▼                                     │
│   ┌─────────────────────────────────────────────────────┐      │
│   │                  EventType                          │      │
│   │                       ↓                             │      │
│   │              Maps to: WORKSHOP/LIVE_CLASS           │      │
│   │                                                     │      │
│   │   • teamId → tenant_id (or sub-team)               │      │
│   │   • metadata.immersio_course_id                    │      │
│   │   • metadata.workshop_type                         │      │
│   │   • metadata.whitelist_enabled                     │      │
│   │   • seatsPerTimeSlot → capacity                    │      │
│   │   • price/currency → payment info                  │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Data Isolation Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                   DATA ISOLATION APPROACH                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   OPTION CHOSEN: Shared Database with tenant_id                 │
│                                                                 │
│   Benefits:                                                     │
│   ✓ Lower infrastructure cost                                   │
│   ✓ Easier maintenance                                          │
│   ✓ Cal.com's existing Organization model                       │
│   ✓ Row-Level Security (RLS) optional for additional safety     │
│                                                                 │
│   Implementation:                                                │
│   1. Every query filtered by organizationId/teamId              │
│   2. Cal.com's Profile model ensures user-org binding           │
│   3. Middleware validates tenant context on every request        │
│                                                                 │
│   Security Layers:                                              │
│   ┌─────────────────────────────────────────────────────┐      │
│   │  Layer 1: API Gateway - Tenant context injection    │      │
│   │  Layer 2: tRPC Middleware - Org validation          │      │
│   │  Layer 3: Repository Layer - tenant_id filtering    │      │
│   │  Layer 4: (Optional) PostgreSQL RLS policies        │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Payment Architecture (Stripe Connect)

```
┌─────────────────────────────────────────────────────────────────┐
│                 STRIPE CONNECT ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Cal.com's PaymentService already supports Stripe Connect!     │
│   Key finding: stripeAccount parameter in API calls             │
│                                                                 │
│   Flow:                                                         │
│                                                                 │
│   1. Tenant Admin connects Stripe account                       │
│      └── Saves: stripe_user_id, stripe_publishable_key          │
│          in: Credential.key (encrypted)                         │
│                                                                 │
│   2. Student makes payment                                      │
│      └── PaymentService.create() uses stripeAccount             │
│          to route payment to tenant's Stripe                    │
│                                                                 │
│   3. Refund processing                                          │
│      └── PaymentService.refund() uses same stripeAccount        │
│                                                                 │
│   Code Reference (packages/app-store/stripepayment/lib/):       │
│   ┌─────────────────────────────────────────────────────┐      │
│   │  const paymentIntent = await stripe.paymentIntents  │      │
│   │    .create({                                        │      │
│   │      amount: params.amount,                         │      │
│   │      currency: params.currency,                     │      │
│   │      ...                                            │      │
│   │    }, {                                             │      │
│   │      stripeAccount: this.credentials.stripe_user_id │ ←──  │
│   │    });                                              │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Module & Feature Analysis

### 4.1 Module Breakdown

```
┌─────────────────────────────────────────────────────────────────┐
│                      MODULE STRUCTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   packages/                                                     │
│   ├── features/                                                 │
│   │   ├── immersio-tenant/          [NEW MODULE]               │
│   │   │   ├── components/                                      │
│   │   │   ├── lib/                                             │
│   │   │   ├── repositories/                                    │
│   │   │   └── services/                                        │
│   │   │                                                        │
│   │   ├── immersio-integration/     [NEW MODULE]               │
│   │   │   ├── sync/                                            │
│   │   │   ├── webhooks/                                        │
│   │   │   └── api/                                             │
│   │   │                                                        │
│   │   ├── immersio-workshops/       [NEW MODULE]               │
│   │   │   ├── components/                                      │
│   │   │   ├── lib/                                             │
│   │   │   └── whitelist/                                       │
│   │   │                                                        │
│   │   ├── immersio-landing/         [NEW MODULE]               │
│   │   │   ├── templates/                                       │
│   │   │   └── components/                                      │
│   │   │                                                        │
│   │   └── ee/                       [EXISTING - Cal.com]       │
│   │       ├── organizations/        → Tenant basis             │
│   │       ├── payments/             → Stripe integration       │
│   │       └── teams/                → Sub-tenant structure     │
│   │                                                            │
│   ├── prisma/                                                  │
│   │   └── schema.prisma            [EXTEND]                    │
│   │                                                            │
│   └── trpc/                                                    │
│       └── server/routers/                                      │
│           ├── immersio/            [NEW ROUTERS]               │
│           │   ├── tenant.ts                                    │
│           │   ├── workshop.ts                                  │
│           │   └── integration.ts                               │
│           └── viewer/              [EXISTING - Extend]         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Feature Matrix

| Feature | Cal.com Base | New Development | Effort |
|---------|--------------|-----------------|--------|
| **Core Scheduling** |
| Teacher Availability | ✅ Schedule model | Config UI | Low |
| Recurring Events | ✅ EventType.recurringEvent | Workshop mapping | Low |
| Calendar Sync | ✅ Google/Outlook | - | None |
| Booking Creation | ✅ Booking model | Whitelist check | Low |
| **Payment** |
| Stripe Integration | ✅ PaymentService | - | None |
| Per-Tenant Stripe | ✅ stripeAccount | Tenant config UI | Medium |
| Refund Automation | ✅ refund() method | Policy config | Low |
| Invoice Email | ✅ Workflows | Custom template | Low |
| **Multi-Tenant** |
| Tenant Model | ✅ Organization | Custom metadata | Medium |
| Data Isolation | ✅ teamId filtering | Additional checks | Medium |
| Subdomain Routing | ✅ orgDomains | Tenant landing | Medium |
| Plan/Subscription | ✅ Billing module | Plan features | Medium |
| **Immersio Integration** |
| Course Sync | ❌ | New API client | High |
| Student Sync | ❌ | New API client | High |
| Webhook Out | ✅ Webhook model | Immersio endpoints | Medium |
| SSO/JWT | ❌ | Auth middleware | Medium |
| **UI/Landing** |
| Template System | ❌ | New module | Medium |
| Template Editor | ❌ | Admin UI | Medium |
| Public Booking Page | ✅ Booker component | Customization | Low |

### 4.3 Detailed Feature Specifications

#### 4.3.1 Workshop (EventType Extension)

```typescript
// Extended EventType metadata for Immersio Workshops
interface WorkshopMetadata {
  // Immersio-specific
  immersio_course_id?: string;
  immersio_tenant_id: string;
  
  // Workshop configuration
  workshop_type: 'live_class' | 'workshop' | 'tutoring';
  topic_category: string;
  
  // Registration control
  whitelist_enabled: boolean;
  whitelist_emails?: string[];
  registration_mode: 'whitelist' | 'public';
  
  // Zoom integration
  zoom_link?: string;
  zoom_auto_create: boolean;
  
  // Pricing
  price_amount: number;
  price_currency: string;
  
  // Policy
  refund_window_hours: number; // default: 12
  cancellation_notice_hours: number; // default: 24
}
```

#### 4.3.2 Tenant Landing Page Configuration

```typescript
interface TenantLandingSettings {
  tenant_id: number;
  template_id: 'education' | 'tutor_personal' | 'center_small' | 'kids_1' | 'kids_2';
  
  // Branding
  primary_color: string;
  secondary_color?: string;
  logo_url?: string;
  
  // Hero Section
  hero_image_url?: string;
  page_title: string;
  page_subtitle: string;
  
  // Content
  teacher_bio?: string;
  teacher_avatar_url?: string;
  
  // Social
  social_links: {
    facebook?: string;
    instagram?: string;
    youtube?: string;
    tiktok?: string;
  };
  
  // CTA
  cta_text: string; // "Book Now", "Join Now", etc.
  
  // SEO
  meta_title?: string;
  meta_description?: string;
  
  // State
  is_published: boolean;
  updated_at: Date;
}
```

---

## 5. Database Schema (ERD)

### 5.1 New Tables (Extend Cal.com Schema)

```sql
-- ============================================================
-- IMMERSIO BOOKING SERVICE - SCHEMA EXTENSIONS
-- Extends Cal.com's existing Prisma schema
-- ============================================================

-- Tenant Stripe Configuration
-- Stores per-tenant Stripe credentials (extends existing Credential model)
-- Note: Cal.com's Credential model already supports this via:
--   - type: 'stripe_payment'
--   - key: JSON containing stripe_user_id, stripe_publishable_key
--   - teamId: links to tenant (Organization)

-- Workshop Whitelist
CREATE TABLE "WorkshopWhitelist" (
    "id" SERIAL PRIMARY KEY,
    "eventTypeId" INTEGER NOT NULL REFERENCES "EventType"("id") ON DELETE CASCADE,
    "email" TEXT NOT NULL,
    "name" TEXT,
    "immersio_student_id" TEXT,
    "added_by" INTEGER REFERENCES "users"("id"),
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE("eventTypeId", "email")
);

CREATE INDEX "idx_whitelist_event_email" ON "WorkshopWhitelist"("eventTypeId", "email");

-- Tenant Landing Page Settings
CREATE TABLE "TenantLandingSettings" (
    "id" SERIAL PRIMARY KEY,
    "tenant_id" INTEGER NOT NULL UNIQUE REFERENCES "Team"("id") ON DELETE CASCADE,
    "template_id" TEXT NOT NULL DEFAULT 'education',
    "primary_color" TEXT NOT NULL DEFAULT '#0066FF',
    "secondary_color" TEXT,
    "logo_url" TEXT,
    "hero_image_url" TEXT,
    "page_title" TEXT NOT NULL,
    "page_subtitle" TEXT,
    "teacher_bio" TEXT,
    "teacher_avatar_url" TEXT,
    "social_links" JSONB DEFAULT '{}',
    "cta_text" TEXT NOT NULL DEFAULT 'Book Now',
    "meta_title" TEXT,
    "meta_description" TEXT,
    "is_published" BOOLEAN NOT NULL DEFAULT false,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Immersio Integration Configuration
CREATE TABLE "ImmersioIntegration" (
    "id" SERIAL PRIMARY KEY,
    "tenant_id" INTEGER NOT NULL UNIQUE REFERENCES "Team"("id") ON DELETE CASCADE,
    "immersio_org_id" TEXT NOT NULL,
    "api_key_encrypted" TEXT NOT NULL,
    "webhook_secret" TEXT,
    "sync_enabled" BOOLEAN NOT NULL DEFAULT true,
    "last_sync_at" TIMESTAMP(3),
    "sync_status" TEXT DEFAULT 'pending',
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Immersio Course Mapping
CREATE TABLE "ImmersioCourseMapping" (
    "id" SERIAL PRIMARY KEY,
    "tenant_id" INTEGER NOT NULL REFERENCES "Team"("id") ON DELETE CASCADE,
    "event_type_id" INTEGER NOT NULL REFERENCES "EventType"("id") ON DELETE CASCADE,
    "immersio_course_id" TEXT NOT NULL,
    "immersio_course_name" TEXT,
    "sync_students" BOOLEAN NOT NULL DEFAULT true,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE("event_type_id"),
    UNIQUE("tenant_id", "immersio_course_id")
);

-- Booking Extensions (for Immersio metadata)
-- Note: Cal.com Booking already has a metadata JSON field
-- We'll store immersio-specific data there:
-- booking.metadata = {
--   immersio_student_id: "...",
--   immersio_course_id: "...",
--   registration_source: "immersio" | "direct",
--   synced_to_immersio: boolean,
--   synced_at: timestamp
-- }

-- Cancellation History (extends Cal.com's booking workflow)
CREATE TABLE "BookingCancellationHistory" (
    "id" SERIAL PRIMARY KEY,
    "booking_id" INTEGER NOT NULL REFERENCES "Booking"("id") ON DELETE CASCADE,
    "cancelled_by" INTEGER REFERENCES "users"("id"),
    "cancellation_reason" TEXT,
    "refund_status" TEXT NOT NULL DEFAULT 'pending', -- pending, processing, completed, failed, not_eligible
    "refund_amount" INTEGER, -- in cents
    "refund_transaction_id" TEXT,
    "within_refund_window" BOOLEAN NOT NULL,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX "idx_cancellation_booking" ON "BookingCancellationHistory"("booking_id");

-- Tenant Subscription Plans (extends Cal.com billing)
CREATE TABLE "TenantSubscription" (
    "id" SERIAL PRIMARY KEY,
    "tenant_id" INTEGER NOT NULL UNIQUE REFERENCES "Team"("id") ON DELETE CASCADE,
    "plan_type" TEXT NOT NULL DEFAULT 'lite', -- lite, starter, growth, pro, enterprise
    "max_teachers" INTEGER NOT NULL DEFAULT 1,
    "max_bookings_per_month" INTEGER NOT NULL DEFAULT 10,
    "max_event_types" INTEGER NOT NULL DEFAULT 1,
    "features" JSONB NOT NULL DEFAULT '{}',
    "stripe_subscription_id" TEXT,
    "billing_period" TEXT DEFAULT 'monthly',
    "current_period_start" TIMESTAMP(3),
    "current_period_end" TIMESTAMP(3),
    "status" TEXT NOT NULL DEFAULT 'active',
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 5.2 ERD Diagram (Mermaid)

```mermaid
erDiagram
    %% Cal.com Core Models (Existing)
    Team ||--o{ Membership : has
    Team ||--o{ EventType : has
    Team ||--o| OrganizationSettings : has
    Team ||--o{ Team : "parentId (sub-teams)"
    
    User ||--o{ Membership : has
    User ||--o{ Profile : has
    User ||--o{ Schedule : has
    User ||--o{ Credential : has
    
    EventType ||--o{ Booking : has
    EventType ||--o{ Schedule : uses
    EventType ||--o| Host : has
    
    Booking ||--o{ Attendee : has
    Booking ||--o| Payment : has
    
    %% Immersio Extensions (New)
    Team ||--o| TenantLandingSettings : has
    Team ||--o| ImmersioIntegration : has
    Team ||--o| TenantSubscription : has
    Team ||--o{ ImmersioCourseMapping : has
    
    EventType ||--o{ WorkshopWhitelist : has
    EventType ||--o| ImmersioCourseMapping : has
    
    Booking ||--o{ BookingCancellationHistory : has

    %% Entity Definitions
    Team {
        int id PK
        string name
        string slug UK
        int parentId FK
        boolean isOrganization
        json metadata
        boolean isPlatform
    }
    
    User {
        int id PK
        string email UK
        string username
        int organizationId FK
        string timeZone
    }
    
    EventType {
        int id PK
        string title
        string slug
        int teamId FK
        int userId FK
        int length
        int seatsPerTimeSlot
        int price
        string currency
        json metadata
    }
    
    Booking {
        int id PK
        string uid UK
        int eventTypeId FK
        int userId FK
        datetime startTime
        datetime endTime
        string status
        boolean paid
        json metadata
    }
    
    Payment {
        int id PK
        int bookingId FK
        int amount
        string currency
        boolean success
        boolean refunded
        string externalId
    }
    
    TenantLandingSettings {
        int id PK
        int tenant_id FK UK
        string template_id
        string primary_color
        string logo_url
        string page_title
        boolean is_published
    }
    
    ImmersioIntegration {
        int id PK
        int tenant_id FK UK
        string immersio_org_id
        string api_key_encrypted
        boolean sync_enabled
        datetime last_sync_at
    }
    
    ImmersioCourseMapping {
        int id PK
        int tenant_id FK
        int event_type_id FK UK
        string immersio_course_id UK
        boolean sync_students
    }
    
    WorkshopWhitelist {
        int id PK
        int eventTypeId FK
        string email
        string immersio_student_id
    }
    
    BookingCancellationHistory {
        int id PK
        int booking_id FK
        string cancellation_reason
        string refund_status
        int refund_amount
        boolean within_refund_window
    }
    
    TenantSubscription {
        int id PK
        int tenant_id FK UK
        string plan_type
        int max_teachers
        int max_bookings_per_month
        string status
    }
```

### 5.3 Key Relationships

| Parent | Child | Relationship | Purpose |
|--------|-------|--------------|---------|
| Team (Org) | Team (Sub) | 1:N | Tenant → Department hierarchy |
| Team | Membership | 1:N | Tenant → Teachers/Admins |
| Team | EventType | 1:N | Tenant → Workshops/Classes |
| Team | TenantLandingSettings | 1:1 | Tenant → Landing page config |
| Team | ImmersioIntegration | 1:1 | Tenant → Immersio connection |
| Team | TenantSubscription | 1:1 | Tenant → Plan features |
| EventType | WorkshopWhitelist | 1:N | Workshop → Allowed students |
| EventType | ImmersioCourseMapping | 1:1 | Workshop → Immersio course |
| Booking | BookingCancellationHistory | 1:N | Booking → Cancellation audit |

---

## 6. API Specification

### 6.1 API Architecture

The API follows Cal.com's existing tRPC pattern with extensions for Immersio:

```
┌─────────────────────────────────────────────────────────────────┐
│                      API STRUCTURE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   /api/trpc/                                                    │
│   ├── viewer.*                    [EXISTING - Cal.com]         │
│   │   ├── viewer.me                                            │
│   │   ├── viewer.bookings.*                                    │
│   │   ├── viewer.eventTypes.*                                  │
│   │   └── viewer.availability.*                                │
│   │                                                            │
│   ├── teams.*                     [EXISTING - Cal.com]         │
│   │   ├── teams.get                                            │
│   │   ├── teams.create                                         │
│   │   └── teams.listMembers                                    │
│   │                                                            │
│   └── immersio.*                  [NEW - Immersio Extension]   │
│       ├── immersio.tenant.*                                    │
│       ├── immersio.workshop.*                                  │
│       ├── immersio.integration.*                               │
│       └── immersio.landing.*                                   │
│                                                                 │
│   /api/v1/                        [REST - Public API]          │
│   ├── /webhooks/immersio                                       │
│   ├── /public/workshops                                        │
│   └── /public/landing/:slug                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 API Endpoints Specification

#### 6.2.1 Tenant Management APIs

```yaml
# immersio.tenant Router

immersio.tenant.get:
  description: Get current tenant details
  input: {}
  output:
    id: number
    name: string
    slug: string
    plan: TenantPlan
    stripeConnected: boolean
    immersioConnected: boolean
  
immersio.tenant.updateStripeCredentials:
  description: Update Stripe API credentials for tenant
  input:
    stripeAccountId: string
    stripePublishableKey: string
  output:
    success: boolean
    
immersio.tenant.getStats:
  description: Get tenant usage statistics
  input:
    period: 'week' | 'month' | 'year'
  output:
    totalBookings: number
    totalRevenue: number
    teacherCount: number
    workshopCount: number
```

#### 6.2.2 Workshop Management APIs

```yaml
# immersio.workshop Router

immersio.workshop.create:
  description: Create a new workshop/live class
  input:
    title: string
    description: string
    duration: number
    price: number
    currency: string
    seats: number
    recurrence: 'weekly' | 'biweekly' | 'none'
    teacherId: number
    zoomLink?: string
    immersioCourseId?: string
    whitelistEnabled: boolean
    whitelistEmails?: string[]
  output:
    id: number
    eventTypeId: number
    bookingUrl: string

immersio.workshop.updateWhitelist:
  description: Update workshop registration whitelist
  input:
    workshopId: number
    emails: string[]
    mode: 'add' | 'replace' | 'remove'
  output:
    success: boolean
    totalWhitelisted: number

immersio.workshop.list:
  description: List all workshops for tenant
  input:
    status?: 'active' | 'draft' | 'archived'
    teacherId?: number
  output:
    workshops: Workshop[]
    totalCount: number
```

#### 6.2.3 Immersio Integration APIs

```yaml
# immersio.integration Router

immersio.integration.connect:
  description: Connect Immersio account to tenant
  input:
    immersioOrgId: string
    apiKey: string
  output:
    success: boolean
    connectionId: number

immersio.integration.syncCourses:
  description: Trigger course sync from Immersio
  input: {}
  output:
    syncedCount: number
    errors: string[]

immersio.integration.mapCourse:
  description: Map Immersio course to workshop
  input:
    workshopId: number
    immersioCourseId: string
    syncStudents: boolean
  output:
    mappingId: number

# Webhook Endpoints (REST)
POST /api/v1/webhooks/immersio/course-updated:
  description: Receive course update notifications from Immersio
  
POST /api/v1/webhooks/immersio/student-enrolled:
  description: Receive student enrollment notifications
```

#### 6.2.4 Landing Page APIs

```yaml
# immersio.landing Router

immersio.landing.getSettings:
  description: Get landing page configuration
  input: {}
  output: TenantLandingSettings

immersio.landing.updateSettings:
  description: Update landing page configuration
  input:
    templateId?: string
    primaryColor?: string
    logoUrl?: string
    heroImageUrl?: string
    pageTitle?: string
    pageSubtitle?: string
    teacherBio?: string
    socialLinks?: SocialLinks
    ctaText?: string
  output:
    success: boolean

immersio.landing.publish:
  description: Publish landing page
  input: {}
  output:
    publicUrl: string

# Public REST API
GET /api/v1/public/landing/:slug:
  description: Get public landing page data
  output:
    settings: TenantLandingSettings
    workshops: PublicWorkshop[]
    tenant: PublicTenantInfo
```

### 6.3 Webhook Events (Outgoing to Immersio)

```yaml
webhook_events:
  - name: BOOKING_CREATED
    payload:
      eventType: 'BOOKING_CREATED'
      booking:
        id: number
        uid: string
        workshopId: number
        studentEmail: string
        startTime: string
        endTime: string
        paid: boolean
        immersioCourseId?: string
      tenant:
        id: number
        slug: string

  - name: BOOKING_CANCELLED
    payload:
      eventType: 'BOOKING_CANCELLED'
      booking:
        id: number
        uid: string
        cancellationReason: string
        refundStatus: string
        refundAmount?: number
      tenant:
        id: number

  - name: PAYMENT_COMPLETED
    payload:
      eventType: 'PAYMENT_COMPLETED'
      payment:
        id: number
        bookingId: number
        amount: number
        currency: string
        stripePaymentId: string
      tenant:
        id: number
```

---

## 7. Implementation Roadmap

> ⚠️ **Single Resource Estimate**: Timeline based on 1 developer working full-time.
> 
> See [ESTIMATE-DETAILED.md](./ESTIMATE-DETAILED.md) for feature-by-feature breakdown.

### 7.1 Phase Overview (1 Developer)

```
┌─────────────────────────────────────────────────────────────────┐
│              IMPLEMENTATION PHASES (1 Developer)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Phase 1: MVP Core (68 days / ~14 weeks)                      │
│   ├── Project setup & foundation (6.5 days)                    │
│   ├── Multi-tenant architecture (20 days) ← BUILD FROM SCRATCH │
│   ├── Teacher availability (7 days)                            │
│   ├── Workshop/Event Type (13 days)                            │
│   ├── Basic booking flow (7 days)                              │
│   ├── Whitelist feature (10 days) ← BUILD FROM SCRATCH         │
│   └── Basic notifications (4.5 days)                           │
│                                                                 │
│   Phase 2: Payments & Policies (61 days / ~12 weeks)           │
│   ├── Stripe integration (12 days) ← BUILD FROM SCRATCH        │
│   ├── Per-tenant Stripe Connect (15 days) ← BUILD FROM SCRATCH │
│   ├── Refund automation (10 days) ← BUILD FROM SCRATCH         │
│   ├── Cancellation policies (8 days)                           │
│   ├── Transaction history (10 days) ← BUILD FROM SCRATCH       │
│   └── Invoice emails (6 days) ← BUILD FROM SCRATCH             │
│                                                                 │
│   Phase 3: Immersio Integration (78 days / ~16 weeks)          │
│   ├── Immersio API client (14 days) ← BUILD FROM SCRATCH       │
│   ├── Course sync - pull (12 days) ← BUILD FROM SCRATCH        │
│   ├── Student sync - pull (10 days) ← BUILD FROM SCRATCH       │
│   ├── Webhook events - push (12 days) ← BUILD FROM SCRATCH     │
│   ├── Incoming webhooks (8 days) ← BUILD FROM SCRATCH          │
│   ├── SSO/JWT integration (12 days) ← BUILD FROM SCRATCH       │
│   └── Embedded UI support (10 days) ← BUILD FROM SCRATCH       │
│                                                                 │
│   Phase 4: White-label & Admin (85 days / ~17 weeks)           │
│   ├── Landing page templates (25 days) ← BUILD FROM SCRATCH    │
│   ├── Template customization UI (13 days) ← BUILD FROM SCRATCH │
│   ├── Subdomain routing (8 days) ← BUILD FROM SCRATCH          │
│   ├── Super Admin panel (24 days) ← BUILD FROM SCRATCH         │
│   └── Tenant subscriptions (15 days) ← BUILD FROM SCRATCH      │
│                                                                 │
│   Base Total: 292 days (~59 weeks / ~14 months)                │
│   With 20% Buffer: 351 days (~71 weeks / ~17 months)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              TIMELINE OPTIONS BY TEAM SIZE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1 Developer:  ~14 months (with buffer: ~17 months)           │
│   2 Developers: ~8 months                                       │
│   3 Developers: ~5-6 months                                     │
│   4 Devs + Lead: ~4 months                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Detailed Task Breakdown (Days per 1 Developer)

#### Phase 1: MVP Core (68 days)

| Task Group | Days | Build Type | Details |
|------------|------|------------|---------|
| Project Setup | 6.5 | Leverage | Fork repo, env setup, schema extensions |
| Multi-tenant Architecture | 20 | **Build** | Tenant model, isolation, subdomain routing |
| Teacher Availability | 7 | Leverage/Extend | Use Cal.com Schedule, add custom fields |
| Workshop/Event Type | 13 | Extend | Custom metadata, creation form, listing |
| Basic Booking Flow | 7 | Leverage/Extend | Booking page, confirmation, ICS |
| Whitelist Feature | 10 | **Build** | Whitelist model, validation, bulk upload |
| Notifications | 4.5 | Leverage | Email templates, reminders |

#### Phase 2: Payments (61 days)

| Task Group | Days | Build Type | Details |
|------------|------|------------|---------|
| Stripe Integration | 12 | **Build** | SDK, payment intents, webhooks |
| Per-tenant Stripe Connect | 15 | **Build** | OAuth flow, credential storage, routing |
| Refund Automation | 10 | **Build** | Policy config, window calc, Stripe refund |
| Cancellation Policies | 8 | Extend/Build | Config, history, audit trail |
| Transaction History | 10 | **Build** | Listing, dashboard, export |
| Invoice Emails | 6 | **Build** | Templates, generation, sending |

#### Phase 3: Immersio Integration (78 days)

| Task Group | Days | Build Type | Details |
|------------|------|------------|---------|
| Immersio API Client | 14 | **Build** | HTTP client, auth, endpoints |
| Course Sync (Pull) | 12 | **Build** | Sync service, mapping, incremental |
| Student Sync (Pull) | 10 | **Build** | Per-course sync, whitelist update |
| Webhook Events (Push) | 12 | **Build** | Config, events, retry logic |
| Incoming Webhooks | 8 | **Build** | Endpoints, signature verification |
| SSO/JWT Integration | 12 | **Build** | JWT validation, user provisioning |
| Embedded UI | 10 | **Build** | iframe, widget, cross-origin |

#### Phase 4: White-label & Admin (85 days)

| Task Group | Days | Build Type | Details |
|------------|------|------------|---------|
| Landing Page Templates | 25 | **Build** | 5 templates, data fetching |
| Template Customization UI | 13 | **Build** | Settings form, preview, publish |
| Subdomain Routing | 8 | **Build** | Middleware, tenant resolution |
| Super Admin Panel | 24 | **Build** | Tenant CRUD, approval, analytics |
| Tenant Subscriptions | 15 | **Build** | Plans, limits, billing |

### 7.3 Dependencies & Critical Path

```
┌─────────────────────────────────────────────────────────────────┐
│                     CRITICAL PATH                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   [Schema Extensions] ──┬──> [Tenant Model] ──> [Booking Flow] │
│                         │                                       │
│                         └──> [Stripe Connect] ──> [Payments]   │
│                                                                 │
│   [Immersio API Client] ──> [Sync Services] ──> [Webhooks]     │
│                                                                 │
│   [Template System] ──> [Landing Pages] ──> [White-label]      │
│                                                                 │
│   Blockers:                                                     │
│   • Immersio API documentation (required before Phase 3)       │
│   • Stripe Connect account setup (required before Phase 2)     │
│   • UI/UX designs from Immersio team                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Cost Estimation

> ⚠️ **Updated**: Estimates based on **single developer** and building **EE features from scratch**.
> 
> See [ESTIMATE-DETAILED.md](./ESTIMATE-DETAILED.md) for complete feature-by-feature breakdown.

### 8.1 Single Resource Estimate (1 Developer)

#### Phase-by-Phase Estimates

| Phase | Days | Weeks | Hours | Cost Range (USD) @ $60-90/hr |
|-------|------|-------|-------|------------------------------|
| **Phase 1: MVP Core** | 68 (+20%) = 82 | 17 | 656 | $39,360 - $59,040 |
| **Phase 2: Payments** | 61 (+20%) = 73 | 15 | 584 | $35,040 - $52,560 |
| **Phase 3: Immersio Integration** | 78 (+20%) = 94 | 19 | 752 | $45,120 - $67,680 |
| **Phase 4: White-label** | 85 (+20%) = 102 | 20 | 816 | $48,960 - $73,440 |
| **Subtotal Development** | **351 days** | **71 weeks** | **2,808** | **$168,480 - $252,720** |

*Note: +20% contingency buffer included*

#### Additional Costs

| Item | Cost Range (USD) |
|------|-----------------|
| Project Management (10%) | $16,848 - $25,272 |
| QA/Testing (15%) | $25,272 - $37,908 |
| Documentation | $5,000 - $8,000 |
| **Total Project Cost (1 Dev)** | **$215,600 - $323,900** |

### 8.2 Multi-Resource Options

| Team Size | Duration | Parallel Work | Total Cost |
|-----------|----------|---------------|------------|
| 1 Developer | ~14 months | None | $215,600 - $323,900 |
| 2 Developers | ~8 months | 50% overlap | $180,000 - $270,000 |
| 3 Developers | ~5-6 months | Full parallel | $200,000 - $300,000 |
| 4 Devs + Lead | ~4 months | Full parallel | $220,000 - $330,000 |

### 8.3 Effort Breakdown by Type

| Category | Days | % of Total |
|----------|------|------------|
| **Leverage Cal.com (Low Effort)** | 10.5 | 4% |
| **Extend Cal.com (Medium Effort)** | 28 | 10% |
| **Build from Scratch (High Effort)** | 250 | 86% |
| **Total** | **288.5** | 100% |

*Most effort is on EE-equivalent features (multi-tenant, payments, SSO, admin)*

### 8.4 Infrastructure Cost (Monthly)

| Component | Service | Monthly Cost |
|-----------|---------|--------------|
| Web Servers | AWS EC2 / Vercel Pro | $100 - $300 |
| Database | AWS RDS PostgreSQL | $100 - $200 |
| Redis Cache | AWS ElastiCache | $50 - $100 |
| CDN & Storage | CloudFront + S3 | $50 - $100 |
| Monitoring | Datadog / New Relic | $50 - $150 |
| Email Service | SendGrid / AWS SES | $30 - $80 |
| **Total Monthly Infra** | | **$380 - $930** |

### 8.5 Cal.com License - Open Source Only

```
┌─────────────────────────────────────────────────────────────────┐
│                  CAL.COM LICENSING                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   USING: AGPLv3 Open Source (FREE)                             │
│                                                                 │
│   What we get for free:                                        │
│   ✅ Core scheduling engine                                    │
│   ✅ Basic booking flow                                        │
│   ✅ Calendar integrations (Google, Outlook)                   │
│   ✅ Basic webhooks                                            │
│   ✅ Email notifications                                       │
│   ✅ Event types & availability                                │
│                                                                 │
│   What we must BUILD (EE features):                            │
│   🔴 Organizations (multi-tenant) → 20 days                    │
│   🔴 Stripe Payments → 27 days                                 │
│   🔴 SSO/SAML → 12 days                                        │
│   🔴 Admin Panel → 24 days                                     │
│   🔴 Billing/Subscriptions → 15 days                           │
│                                                                 │
│   License cost: $0                                             │
│   Development cost for EE features: ~98 days                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.6 Cost Comparison: Build All vs Use Cal.com Open Source

| Feature | Build from Scratch | With Cal.com OSS | Savings |
|---------|-------------------|------------------|---------|
| Scheduling Engine | 40 days | 0 days | 40 days |
| Calendar Sync | 20 days | 0 days | 20 days |
| Basic Booking | 25 days | 2 days | 23 days |
| Event Types | 15 days | 2 days | 13 days |
| Email Notifications | 10 days | 2 days | 8 days |
| UI Components | 30 days | 5 days | 25 days |
| **Total Saved** | | | **~129 days** |

**Without Cal.com**: 288 + 129 = **417 days** (~20 months)  
**With Cal.com OSS**: **288 days** (~14 months)  
**Savings**: ~31% time reduction

*Note: Original 60-70% estimate assumed using EE features*

---

## 9. DevOps Plan

### 9.1 Infrastructure Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Production Environment                                        │
│   ┌─────────────────────────────────────────────────────┐      │
│   │                    CloudFlare                        │      │
│   │              (CDN + DDoS Protection)                │      │
│   └───────────────────────┬─────────────────────────────┘      │
│                           │                                     │
│   ┌───────────────────────▼─────────────────────────────┐      │
│   │              Load Balancer (AWS ALB)                │      │
│   └───────────────────────┬─────────────────────────────┘      │
│                           │                                     │
│   ┌───────────┬───────────┼───────────┬───────────────┐        │
│   │           │           │           │               │        │
│   ▼           ▼           ▼           ▼               ▼        │
│ ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌──────────┐           │
│ │Web 1│   │Web 2│   │Web 3│   │API 1│   │Worker    │           │
│ │     │   │     │   │     │   │     │   │(BullMQ)  │           │
│ └──┬──┘   └──┬──┘   └──┬──┘   └──┬──┘   └─────┬────┘           │
│    │         │         │         │            │                 │
│    └─────────┴─────────┴─────────┴────────────┘                 │
│                           │                                     │
│   ┌───────────────────────▼─────────────────────────────┐      │
│   │           VPC - Private Subnet                      │      │
│   │  ┌─────────────┐   ┌─────────────┐                 │      │
│   │  │  PostgreSQL │   │    Redis    │                 │      │
│   │  │   (RDS)     │   │(ElastiCache)│                 │      │
│   │  │             │   │             │                 │      │
│   │  │ Primary +   │   │  Cluster    │                 │      │
│   │  │ Read Replica│   │  Mode       │                 │      │
│   │  └─────────────┘   └─────────────┘                 │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 CI/CD Pipeline

```yaml
# GitHub Actions Workflow
name: Deploy Immersio Booking Service

on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      - name: Install dependencies
        run: yarn install
      - name: Type check
        run: yarn type-check
      - name: Lint
        run: yarn lint
      - name: Unit tests
        run: TZ=UTC yarn test

  e2e:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: E2E Tests
        run: yarn e2e

  deploy-staging:
    needs: [test, e2e]
    if: github.ref == 'refs/heads/staging'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: |
          # Deploy to staging environment

  deploy-production:
    needs: [test, e2e]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Production
        run: |
          # Blue-green deployment to production
```

### 9.3 Monitoring & Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                    MONITORING STACK                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Metrics (Datadog / Prometheus + Grafana)                     │
│   ├── API response times                                       │
│   ├── Error rates by endpoint                                  │
│   ├── Database query performance                               │
│   ├── Redis hit/miss rates                                     │
│   ├── Booking conversion rates                                 │
│   └── Payment success rates                                    │
│                                                                 │
│   Logging (CloudWatch / ELK Stack)                             │
│   ├── Application logs                                         │
│   ├── Access logs                                              │
│   ├── Error logs with stack traces                             │
│   └── Audit logs (tenant actions)                              │
│                                                                 │
│   Alerting                                                     │
│   ├── PagerDuty / Opsgenie integration                        │
│   ├── Slack notifications                                      │
│   └── Alert thresholds:                                        │
│       • Error rate > 1%                                        │
│       • Response time > 2s (p95)                              │
│       • Payment failure rate > 5%                              │
│       • Database connection pool exhaustion                    │
│                                                                 │
│   Uptime Monitoring (Checkly / Pingdom)                        │
│   ├── Health check endpoints                                   │
│   ├── Synthetic transaction monitoring                         │
│   └── SSL certificate monitoring                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.4 Scaling Strategy

| Component | Horizontal Scaling | Vertical Scaling |
|-----------|-------------------|------------------|
| Web Servers | Auto-scaling group (2-10 instances) | c5.xlarge → c5.2xlarge |
| API Servers | Auto-scaling group (2-8 instances) | c5.large → c5.xlarge |
| Database | Read replicas (up to 5) | db.r5.large → db.r5.2xlarge |
| Redis | Cluster mode (3-6 shards) | cache.r5.large → cache.r5.xlarge |
| Workers | Queue-based scaling | Based on job backlog |

### 9.5 Backup & Disaster Recovery

```
┌─────────────────────────────────────────────────────────────────┐
│                    BACKUP STRATEGY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Database:                                                     │
│   • Automated daily snapshots (RDS)                            │
│   • Point-in-time recovery (PITR) - 7 days                     │
│   • Cross-region replication for DR                            │
│                                                                 │
│   File Storage:                                                 │
│   • S3 versioning enabled                                      │
│   • Cross-region replication                                   │
│                                                                 │
│   Recovery Objectives:                                          │
│   • RPO (Recovery Point Objective): 1 hour                     │
│   • RTO (Recovery Time Objective): 4 hours                     │
│                                                                 │
│   DR Runbook:                                                   │
│   1. Promote read replica to primary                           │
│   2. Update DNS to DR region                                   │
│   3. Scale up DR infrastructure                                │
│   4. Verify all integrations                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.6 Security Measures

| Layer | Measure |
|-------|---------|
| Network | VPC isolation, Security Groups, WAF |
| Application | HTTPS only, CORS, Rate limiting |
| Authentication | JWT validation, Session management |
| Data | Encryption at rest (RDS, S3), TLS in transit |
| Secrets | AWS Secrets Manager / HashiCorp Vault |
| Compliance | SOC 2 readiness, GDPR considerations |
| Audit | CloudTrail, Application audit logs |

---

## 10. Summary & Recommendations

### 10.1 Key Findings

1. **Cal.com provides ~65% of required functionality** out of the box
2. **Organizations model** directly maps to multi-tenant requirement
3. **Stripe Connect** is already implemented and supports per-tenant payments
4. **Main development effort** is in:
   - Immersio integration (API client, sync services)
   - Workshop customizations (whitelist, metadata)
   - Landing page template system
   - Super Admin panel

### 10.2 Recommendations

1. **Contact Cal.com** for enterprise license pricing (Organizations, Payments are EE features)
2. **Start with fork approach** rather than API-only integration for maximum flexibility
3. **Prioritize Immersio API documentation** before Phase 3
4. **Consider phased rollout**: Start with independent SaaS, add Immersio integration later
5. **Plan for 20% buffer** on timeline due to Cal.com learning curve

### 10.3 Next Steps

1. [ ] Finalize Cal.com enterprise licensing
2. [ ] Setup development environment
3. [ ] Create detailed UI/UX mockups for custom components
4. [ ] Document Immersio API integration requirements
5. [ ] Establish communication channels with stakeholders
6. [ ] Begin Phase 1 development

---

*Document Version: 1.0*  
*Last Updated: November 2024*  
*Author: AI System Analyst*
