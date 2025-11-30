# ERD DIAGRAM - BOOKING SERVICE

## Entity Relationship Diagram

```mermaid
erDiagram
    Tenant ||--o{ User : "has many"
    Tenant ||--o{ EventType : "has many"
    Tenant ||--o{ Booking : "has many"
    Tenant ||--o{ Payment : "has many"
    Tenant ||--|| TenantLandingSettings : "has one"
    Tenant ||--o{ TeacherCourseAssignment : "has many"
    Tenant ||--o{ WhitelistEntry : "has many"
    Tenant ||--o{ ImmersioSyncLog : "has many"
    Tenant ||--o{ TenantUsage : "has one"

    User ||--o{ Booking : "creates"
    User ||--o{ EventType : "owns"
    User ||--o{ Schedule : "has"
    User ||--o{ Availability : "has"
    User ||--o{ TeacherCourseAssignment : "assigned to"
    User ||--o{ WhitelistEntry : "added by"

    EventType ||--o{ Booking : "booked for"
    EventType ||--o{ Availability : "has"
    EventType ||--o{ TeacherCourseAssignment : "assigned"
    EventType ||--o{ WhitelistEntry : "has whitelist"
    EventType ||--o{ Host : "hosted by"

    Booking ||--o{ Payment : "has"
    Booking ||--o{ Attendee : "has"
    Booking ||--o{ BookingSeat : "has seats"
    Booking ||--|| ImmersioEnrollment : "linked to"

    Schedule ||--o{ Availability : "contains"
    Schedule ||--o{ TeacherCourseAssignment : "uses"

    Payment }o--|| StripeAccount : "processed by"

    ImmersioCourse ||--o{ ImmersioEnrollment : "has"
    ImmersioStudent ||--o{ ImmersioEnrollment : "enrolled in"

    Tenant {
        int id PK
        string name
        string subdomain UK
        string customDomain UK
        string stripePublishableKey
        string stripeSecretKeyEncrypted
        string stripeWebhookSecret
        string immersioApiKey
        string subscriptionPlan
        string subscriptionStatus
        json settings
        datetime createdAt
        datetime updatedAt
    }

    User {
        int id PK
        int tenantId FK
        string email UK
        string name
        string role
        string immersioUserId
        datetime createdAt
    }

    EventType {
        int id PK
        int tenantId FK
        string title
        string slug
        int length
        boolean isWorkshop
        boolean whitelistEnabled
        int seatLimit
        int refundWindowHours
        decimal workshopPrice
        string immersioCourseId
        json workshopRecurrence
        datetime createdAt
    }

    Booking {
        int id PK
        string uid UK
        int tenantId FK
        int userId FK
        int eventTypeId FK
        string title
        datetime startTime
        datetime endTime
        string status
        boolean paid
        string immersioEnrollmentId
        int workshopSessionNumber
        datetime createdAt
    }

    Payment {
        int id PK
        string uid UK
        int tenantId FK
        int bookingId FK
        int amount
        string currency
        boolean success
        boolean refunded
        string stripeAccountId
        datetime createdAt
    }

    TenantLandingSettings {
        int id PK
        int tenantId FK
        int templateId
        string primaryColor
        string logoUrl
        string heroImageUrl
        string pageTitle
        string pageSubtitle
        string teacherBio
        json socialLinks
        boolean published
        datetime updatedAt
    }

    TeacherCourseAssignment {
        int id PK
        int tenantId FK
        int eventTypeId FK
        int teacherId FK
        int scheduleId FK
        json recurrenceRule
        boolean isPublished
        datetime createdAt
    }

    WhitelistEntry {
        int id PK
        int tenantId FK
        int eventTypeId FK
        string email
        string immersioCourseId
        datetime addedAt
    }

    ImmersioSyncLog {
        int id PK
        int tenantId FK
        string syncType
        string direction
        string resourceId
        string status
        string errorMessage
        datetime syncedAt
    }

    TenantUsage {
        int id PK
        int tenantId FK
        int month
        int year
        int bookingsCount
        int teachersCount
        int eventTypesCount
        datetime lastResetAt
    }

    Schedule {
        int id PK
        int userId FK
        string name
        string timeZone
    }

    Availability {
        int id PK
        int userId FK
        int eventTypeId FK
        int scheduleId FK
        int[] days
        time startTime
        time endTime
    }

    Host {
        int userId PK
        int eventTypeId PK
        int scheduleId FK
    }
```

## Key Relationships

### 1. Tenant-Centric Architecture
- **Tenant** là root entity, tất cả data đều thuộc về một tenant
- Mọi bảng đều có `tenant_id` để đảm bảo data isolation

### 2. Booking Flow
- **User** (Student) tạo **Booking** cho **EventType**
- **Booking** có thể có **Payment** (nếu workshop trả phí)
- **Booking** được link với **ImmersioEnrollment** (nếu sync)

### 3. Teacher Assignment
- **TeacherCourseAssignment** kết nối **EventType** với **User** (Teacher)
- Sử dụng **Schedule** và **Availability** để xác định lịch rảnh

### 4. Whitelist System
- **WhitelistEntry** giới hạn ai có thể book **EventType**
- Có thể sync từ Immersio students

### 5. Immersio Integration
- **ImmersioSyncLog** track tất cả sync operations
- **EventType.immersioCourseId** link với Immersio course
- **Booking.immersioEnrollmentId** link với Immersio enrollment

---

## Index Strategy

### Critical Indexes

```sql
-- Multi-tenant queries (most common)
CREATE INDEX idx_booking_tenant_status_time 
  ON "Booking"(tenant_id, status, start_time);

CREATE INDEX idx_eventtype_tenant_workshop 
  ON "EventType"(tenant_id, is_workshop);

CREATE INDEX idx_user_tenant_role 
  ON "User"(tenant_id, role);

-- Whitelist lookups (performance critical)
CREATE INDEX idx_whitelist_event_email 
  ON "WhitelistEntry"(event_type_id, email);

-- Payment tracking
CREATE INDEX idx_payment_tenant_booking 
  ON "Payment"(tenant_id, booking_id, success);

-- Immersio sync
CREATE INDEX idx_synclog_tenant_status_time 
  ON "ImmersioSyncLog"(tenant_id, status, synced_at);
```

---

## Data Flow Diagrams

### Booking Creation Flow

```mermaid
sequenceDiagram
    participant S as Student
    participant API as Booking API
    participant DB as Database
    participant W as Whitelist Check
    participant P as Payment Service
    participant I as Immersio API

    S->>API: Create Booking Request
    API->>DB: Get EventType
    API->>W: Check Whitelist (if enabled)
    W-->>API: Allowed/Denied
    
    alt Whitelist Denied
        API-->>S: Error: Not in whitelist
    else Allowed
        API->>DB: Check Seat Availability
        alt No Seats Available
            API-->>S: Error: Fully booked
        else Seats Available
            API->>DB: Create Booking (PENDING)
            
            alt Payment Required
                API->>P: Create Payment Intent
                P-->>API: Payment URL
                API-->>S: Redirect to Payment
                S->>P: Complete Payment
                P->>API: Webhook: Payment Success
                API->>DB: Update Booking (CONFIRMED)
            else Free Event
                API->>DB: Update Booking (CONFIRMED)
            end
            
            alt Immersio Sync Enabled
                API->>I: Push Enrollment
                I-->>API: Enrollment ID
                API->>DB: Update Booking with Enrollment ID
            end
            
            API->>S: Send Confirmation Email
        end
    end
```

### Immersio Sync Flow

```mermaid
sequenceDiagram
    participant Scheduler as Sync Scheduler
    participant SyncService as Sync Service
    participant ImmersioAPI as Immersio API
    participant DB as Database
    participant Tenant as Tenant

    Scheduler->>SyncService: Trigger Sync (every X minutes)
    SyncService->>DB: Get Active Tenants with Immersio
    
    loop For Each Tenant
        SyncService->>ImmersioAPI: Get Courses (with API key)
        ImmersioAPI-->>SyncService: Courses List
        
        loop For Each Course
            SyncService->>DB: Upsert EventType
            SyncService->>DB: Log Sync (SUCCESS)
        end
        
        SyncService->>ImmersioAPI: Get Students
        ImmersioAPI-->>SyncService: Students List
        
        loop For Each Student
            SyncService->>DB: Upsert User
            SyncService->>DB: Update Whitelist (if needed)
        end
    end
```

---

**Document Version**: 1.0  
**Last Updated**: 2024-03-XX
