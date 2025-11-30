# Immersio Booking Service - Entity Relationship Diagram

## Overview

This document provides a detailed Entity Relationship Diagram (ERD) for the Immersio Booking Service, 
showing both the existing Cal.com models and the new extensions required for Immersio integration.

## ERD Diagram (Mermaid)

```mermaid
erDiagram
    %% ============================================================
    %% CAL.COM CORE MODELS (Existing)
    %% ============================================================
    
    Team {
        int id PK
        string name
        string slug UK
        int parentId FK "Self-reference for hierarchy"
        boolean isOrganization "true = Tenant"
        boolean isPlatform
        boolean isPrivate
        json metadata "Extended tenant config"
        string logoUrl
        string bio
        string brandColor
        string bannerUrl
        datetime createdAt
        datetime updatedAt
    }
    
    OrganizationSettings {
        int id PK
        int organizationId FK UK
        boolean isAdminReviewed
        boolean isOrganizationVerified
        boolean isOrganizationConfigured
        string orgAutoAcceptEmail
        boolean lockEventTypeCreation
        boolean adminGetsNoSlotsNotification
        boolean allowSEOIndexing
    }
    
    User {
        int id PK
        string uuid UK
        string email UK
        string username
        string name
        string password
        string timeZone
        string locale
        int organizationId FK
        boolean completedOnboarding
        json metadata
        datetime createdAt
        datetime updatedAt
    }
    
    Profile {
        int id PK
        int userId FK
        int organizationId FK
        string username
        string uid UK
    }
    
    Membership {
        int id PK
        int userId FK
        int teamId FK
        string role "OWNER|ADMIN|MEMBER"
        boolean accepted
        boolean disableImpersonation
        datetime createdAt
    }
    
    EventType {
        int id PK
        string title
        string slug
        string description
        int length "Duration in minutes"
        boolean hidden
        int userId FK "Personal event type"
        int teamId FK "Team event type"
        int profileId FK
        int scheduleId FK
        json locations "Meeting locations config"
        string periodType "UNLIMITED|ROLLING|RANGE"
        datetime periodStartDate
        datetime periodEndDate
        int periodDays
        int periodCountCalendarDays
        int minimumBookingNotice
        int beforeEventBuffer
        int afterEventBuffer
        int seatsPerTimeSlot "Capacity for group events"
        boolean seatsShowAvailabilityCount
        boolean seatsShowAttendees
        json bookingFields "Custom booking form fields"
        boolean disableGuests
        int slotInterval
        json metadata "Immersio workshop config"
        int price "Price in cents"
        string currency
        boolean requiresConfirmation
        json recurringEvent "Recurrence config"
        datetime createdAt
        datetime updatedAt
    }
    
    Schedule {
        int id PK
        int userId FK
        string name
        string timeZone
        datetime createdAt
        datetime updatedAt
    }
    
    Availability {
        int id PK
        int userId FK
        int eventTypeId FK
        int scheduleId FK
        int days "Bitmask for days"
        datetime startTime
        datetime endTime
        datetime date "For specific date overrides"
    }
    
    Booking {
        int id PK
        string uid UK
        int userId FK "Host user"
        int eventTypeId FK
        string title
        string description
        datetime startTime
        datetime endTime
        string status "accepted|pending|cancelled|rejected"
        boolean paid
        json metadata "Immersio booking data"
        string location
        string cancellationReason
        string rejectionReason
        boolean rescheduled
        int fromReschedule FK "Original booking ID"
        datetime createdAt
        datetime updatedAt
    }
    
    Attendee {
        int id PK
        int bookingId FK
        string email
        string name
        string timeZone
        string locale
    }
    
    Payment {
        int id PK
        string uid UK
        int bookingId FK
        int appId FK "Payment app (Stripe)"
        int amount "Amount in cents"
        int fee "Platform fee"
        string currency
        boolean success
        boolean refunded
        json data "Stripe-specific data"
        string externalId "Stripe payment ID"
        datetime createdAt
        datetime updatedAt
    }
    
    Credential {
        int id PK
        string type "stripe_payment|google_calendar|etc"
        json key "Encrypted credentials"
        int userId FK
        int teamId FK
        int appId FK
        boolean invalid
        datetime createdAt
        datetime updatedAt
    }
    
    App {
        string slug PK
        string dirName
        json keys "App config"
        json categories
        boolean enabled
        datetime createdAt
        datetime updatedAt
    }
    
    Webhook {
        int id PK
        int userId FK
        int teamId FK
        int eventTypeId FK
        string subscriberUrl
        boolean active
        json eventTriggers
        int appId FK
        string secret
        datetime createdAt
    }
    
    Workflow {
        int id PK
        string name
        int userId FK
        int teamId FK
        string trigger
        datetime time
        string timeUnit
        boolean active
        datetime createdAt
        datetime updatedAt
    }
    
    WorkflowStep {
        int id PK
        int workflowId FK
        int stepNumber
        string action
        string template
        string sendTo
        boolean reminderBody
        string emailSubject
        string sender
        datetime createdAt
        datetime updatedAt
    }
    
    %% ============================================================
    %% IMMERSIO EXTENSION MODELS (New)
    %% ============================================================
    
    TenantLandingSettings {
        int id PK
        int tenant_id FK UK "References Team.id"
        string template_id "education|tutor_personal|center_small|kids_1|kids_2"
        string primary_color
        string secondary_color
        string logo_url
        string hero_image_url
        string page_title
        string page_subtitle
        string teacher_bio
        string teacher_avatar_url
        json social_links "facebook|instagram|youtube|tiktok"
        string cta_text
        string meta_title
        string meta_description
        boolean is_published
        datetime created_at
        datetime updated_at
    }
    
    ImmersioIntegration {
        int id PK
        int tenant_id FK UK "References Team.id"
        string immersio_org_id
        string api_key_encrypted
        string webhook_secret
        boolean sync_enabled
        datetime last_sync_at
        string sync_status "pending|syncing|success|error"
        string sync_error_message
        datetime created_at
        datetime updated_at
    }
    
    ImmersioCourseMapping {
        int id PK
        int tenant_id FK "References Team.id"
        int event_type_id FK UK "References EventType.id"
        string immersio_course_id UK
        string immersio_course_name
        boolean sync_students
        datetime last_student_sync
        datetime created_at
        datetime updated_at
    }
    
    WorkshopWhitelist {
        int id PK
        int event_type_id FK "References EventType.id"
        string email
        string name
        string immersio_student_id
        int added_by FK "References User.id"
        datetime created_at
    }
    
    BookingCancellationHistory {
        int id PK
        int booking_id FK "References Booking.id"
        int cancelled_by FK "References User.id"
        string cancellation_reason
        string refund_status "pending|processing|completed|failed|not_eligible"
        int refund_amount "Amount in cents"
        string refund_transaction_id
        boolean within_refund_window
        datetime created_at
    }
    
    TenantSubscription {
        int id PK
        int tenant_id FK UK "References Team.id"
        string plan_type "lite|starter|growth|pro|enterprise"
        int max_teachers
        int max_bookings_per_month
        int max_event_types
        json features "Feature flags"
        string stripe_subscription_id
        string billing_period "monthly|annually"
        datetime current_period_start
        datetime current_period_end
        string status "active|past_due|cancelled|trialing"
        datetime created_at
        datetime updated_at
    }
    
    ImmersioSyncLog {
        int id PK
        int tenant_id FK "References Team.id"
        string sync_type "courses|students|teachers"
        string direction "pull|push"
        int records_processed
        int records_failed
        string status "success|partial|failed"
        json error_details
        datetime started_at
        datetime completed_at
    }
    
    %% ============================================================
    %% RELATIONSHIPS
    %% ============================================================
    
    %% Cal.com Core Relationships
    Team ||--o{ Team : "parentId (sub-teams)"
    Team ||--o| OrganizationSettings : "organizationSettings"
    Team ||--o{ Membership : "members"
    Team ||--o{ EventType : "eventTypes"
    Team ||--o{ Credential : "credentials"
    Team ||--o{ Webhook : "webhooks"
    Team ||--o{ Workflow : "workflows"
    
    User ||--o{ Membership : "teams"
    User ||--o{ Profile : "profiles"
    User ||--o{ Schedule : "schedules"
    User ||--o{ Credential : "credentials"
    User ||--o{ EventType : "eventTypes"
    User ||--o{ Booking : "bookings (as host)"
    User ||--o{ Webhook : "webhooks"
    User ||--o{ Workflow : "workflows"
    
    Profile }o--|| User : "user"
    Profile }o--|| Team : "organization"
    
    EventType ||--o{ Booking : "bookings"
    EventType ||--o{ Availability : "availability"
    EventType }o--o| Schedule : "schedule"
    EventType ||--o{ Webhook : "webhooks"
    
    Schedule ||--o{ Availability : "availability"
    
    Booking ||--o{ Attendee : "attendees"
    Booking ||--o| Payment : "payment"
    Booking ||--o{ Booking : "fromReschedule"
    
    Workflow ||--o{ WorkflowStep : "steps"
    
    App ||--o{ Credential : "credentials"
    App ||--o{ Payment : "payments"
    App ||--o{ Webhook : "webhooks"
    
    %% Immersio Extension Relationships
    Team ||--o| TenantLandingSettings : "landingSettings"
    Team ||--o| ImmersioIntegration : "immersioIntegration"
    Team ||--o| TenantSubscription : "subscription"
    Team ||--o{ ImmersioCourseMapping : "courseMappings"
    Team ||--o{ ImmersioSyncLog : "syncLogs"
    
    EventType ||--o{ WorkshopWhitelist : "whitelist"
    EventType ||--o| ImmersioCourseMapping : "courseMapping"
    
    Booking ||--o{ BookingCancellationHistory : "cancellationHistory"
    
    User ||--o{ WorkshopWhitelist : "addedWhitelistEntries"
    User ||--o{ BookingCancellationHistory : "cancelledBookings"
```

## Entity Descriptions

### Cal.com Core Entities

| Entity | Description | Key Fields |
|--------|-------------|------------|
| **Team** | Represents both tenants (organizations) and sub-teams | `isOrganization=true` for tenants |
| **User** | System users (admins, teachers, students) | `organizationId` links to tenant |
| **Profile** | User's profile within an organization | Ensures username uniqueness per org |
| **Membership** | Links users to teams with roles | OWNER, ADMIN, MEMBER |
| **EventType** | Workshop/class definitions | `seatsPerTimeSlot` for capacity |
| **Schedule** | Teacher availability schedule | Named schedules per user |
| **Availability** | Time slots within a schedule | Day/time ranges |
| **Booking** | Student registrations | Status tracking, payment flag |
| **Attendee** | Booking participants | Email, timezone |
| **Payment** | Payment transactions | Stripe integration |
| **Credential** | OAuth/API credentials | Stripe keys per tenant |
| **Webhook** | Event notifications | Immersio endpoint config |
| **Workflow** | Automated actions | Email reminders |

### Immersio Extension Entities

| Entity | Description | Key Fields |
|--------|-------------|------------|
| **TenantLandingSettings** | Landing page configuration | Template selection, branding |
| **ImmersioIntegration** | Immersio API connection | API keys, sync status |
| **ImmersioCourseMapping** | Links workshops to Immersio courses | Course ID, student sync flag |
| **WorkshopWhitelist** | Email whitelist for workshops | Per-event whitelist |
| **BookingCancellationHistory** | Cancellation audit trail | Refund status tracking |
| **TenantSubscription** | Subscription plan details | Plan limits, features |
| **ImmersioSyncLog** | Sync operation history | Pull/push tracking |

## Metadata Schema Details

### EventType.metadata (Workshop Configuration)

```json
{
  "immersio_course_id": "course_abc123",
  "immersio_tenant_id": "tenant_xyz789",
  "workshop_type": "live_class",
  "topic_category": "Business English",
  "whitelist_enabled": true,
  "registration_mode": "whitelist",
  "zoom_auto_create": true,
  "refund_window_hours": 12,
  "cancellation_notice_hours": 24
}
```

### Booking.metadata (Immersio Booking Data)

```json
{
  "immersio_student_id": "student_123",
  "immersio_course_id": "course_abc123",
  "registration_source": "immersio",
  "synced_to_immersio": true,
  "synced_at": "2024-11-30T10:00:00Z",
  "booking_context": {
    "referrer": "course_page",
    "promo_code": null
  }
}
```

### Team.metadata (Tenant Configuration)

```json
{
  "isPlatform": false,
  "orgSeats": 5,
  "orgPricePerSeat": 1500,
  "billingPeriod": "MONTHLY",
  "subscriptionId": "sub_xxx",
  "subscriptionItemId": "si_xxx",
  "immersio_config": {
    "auto_sync_enabled": true,
    "webhook_url": "https://immersio.com/webhooks/booking"
  }
}
```

## Index Recommendations

```sql
-- Performance indexes for common queries

-- Tenant queries
CREATE INDEX idx_team_is_org ON "Team"("isOrganization") WHERE "isOrganization" = true;
CREATE INDEX idx_team_parent ON "Team"("parentId");

-- User queries
CREATE INDEX idx_user_org ON "users"("organizationId");
CREATE INDEX idx_profile_user_org ON "Profile"("userId", "organizationId");

-- Event type queries
CREATE INDEX idx_eventtype_team ON "EventType"("teamId");
CREATE INDEX idx_eventtype_user ON "EventType"("userId");

-- Booking queries
CREATE INDEX idx_booking_eventtype ON "Booking"("eventTypeId");
CREATE INDEX idx_booking_user ON "Booking"("userId");
CREATE INDEX idx_booking_status ON "Booking"("status");
CREATE INDEX idx_booking_time ON "Booking"("startTime", "endTime");

-- Whitelist queries
CREATE INDEX idx_whitelist_email ON "WorkshopWhitelist"("eventTypeId", "email");

-- Course mapping queries
CREATE INDEX idx_coursemap_tenant ON "ImmersioCourseMapping"("tenant_id");
CREATE INDEX idx_coursemap_course ON "ImmersioCourseMapping"("immersio_course_id");

-- Sync log queries
CREATE INDEX idx_synclog_tenant ON "ImmersioSyncLog"("tenant_id", "started_at" DESC);
```

## Data Flow Diagrams

### Booking Flow with Whitelist Check

```
Student                    Booking API                   Database
   │                           │                            │
   │  1. Select Workshop       │                            │
   │─────────────────────────> │                            │
   │                           │  2. Check EventType        │
   │                           │──────────────────────────> │
   │                           │                            │
   │                           │  3. If whitelist_enabled   │
   │                           │  Check WorkshopWhitelist   │
   │                           │──────────────────────────> │
   │                           │                            │
   │                           │  4. Check seat availability│
   │                           │  (seatsPerTimeSlot)        │
   │                           │──────────────────────────> │
   │                           │                            │
   │  5. Redirect to payment   │                            │
   │<───────────────────────── │                            │
   │                           │                            │
   │  6. Payment completed     │                            │
   │─────────────────────────> │                            │
   │                           │  7. Create Booking         │
   │                           │  with metadata             │
   │                           │──────────────────────────> │
   │                           │                            │
   │                           │  8. Create Payment record  │
   │                           │──────────────────────────> │
   │                           │                            │
   │                           │  9. Trigger Webhook        │
   │                           │  (to Immersio)             │
   │                           │──────────────────────────> │
   │                           │                            │
   │  10. Confirmation         │                            │
   │<───────────────────────── │                            │
```

### Immersio Sync Flow

```
Scheduler              ImmersioIntegration              Immersio API
    │                          │                             │
    │  1. Trigger sync job     │                             │
    │─────────────────────────>│                             │
    │                          │  2. Get sync config         │
    │                          │  for enabled tenants        │
    │                          │──────────────────────────>  │
    │                          │                             │
    │                          │  3. Fetch courses           │
    │                          │─────────────────────────────│
    │                          │<────────────────────────────│
    │                          │                             │
    │                          │  4. Fetch students          │
    │                          │  for each course            │
    │                          │─────────────────────────────│
    │                          │<────────────────────────────│
    │                          │                             │
    │                          │  5. Update EventType        │
    │                          │  metadata                   │
    │                          │──────────────────────────>  │
    │                          │                             │
    │                          │  6. Update Whitelist        │
    │                          │──────────────────────────>  │
    │                          │                             │
    │                          │  7. Log sync result         │
    │                          │  (ImmersioSyncLog)          │
    │                          │──────────────────────────>  │
    │                          │                             │
    │  8. Job complete         │                             │
    │<─────────────────────────│                             │
```
