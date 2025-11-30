# DATABASE SCHEMA CHI TIẾT - BOOKING SERVICE

## 1. MIGRATION STRATEGY

### 1.1. Approach
- **Extend existing Cal.com schema** thay vì tạo mới hoàn toàn
- Thêm `tenant_id` vào các bảng hiện có
- Tạo các bảng mới cho features riêng của Immersio
- Sử dụng Prisma migrations

### 1.2. Migration Order
1. Tạo bảng `Tenant` (foundation)
2. Thêm `tenant_id` vào các bảng core (User, EventType, Booking, Payment)
3. Tạo các bảng mới (LandingPage, TeacherAssignment, etc.)
4. Tạo indexes và constraints
5. Data migration (nếu có data cũ)

---

## 2. SCHEMA DEFINITIONS

### 2.1. Tenant Table

```prisma
model Tenant {
  id                        Int       @id @default(autoincrement())
  name                      String
  subdomain                 String    @unique
  customDomain              String?   @unique
  stripePublishableKey      String?
  stripeSecretKeyEncrypted  String?   @db.Text
  stripeWebhookSecret       String?
  stripeAccountId           String?
  immersioApiKey            String?   @db.Text
  immersioApiUrl            String?
  immersioApiSecret         String?   @db.Text
  subscriptionPlan           String    @default("LITE") // LITE, STARTER, GROWTH, PRO, ENTERPRISE
  subscriptionStatus        String    @default("ACTIVE") // ACTIVE, SUSPENDED, CANCELLED
  subscriptionStartDate     DateTime?
  subscriptionEndDate       DateTime?
  maxTeachers               Int       @default(1)
  maxBookingsPerMonth       Int?
  maxEventTypes             Int?
  features                  Json?     // Feature flags per plan
  settings                  Json?     // General tenant settings
  createdAt                 DateTime  @default(now())
  updatedAt                 DateTime  @updatedAt
  
  // Relations
  users                     User[]
  eventTypes                EventType[]
  bookings                  Booking[]
  payments                  Payment[]
  landingPage               TenantLandingSettings?
  teacherAssignments        TeacherCourseAssignment[]
  whitelistEntries          WhitelistEntry[]
  immersioSyncLogs          ImmersioSyncLog[]
  
  @@index([subdomain])
  @@index([customDomain])
  @@index([subscriptionStatus])
}
```

### 2.2. Extended User Model

```prisma
// Add to existing User model
model User {
  // ... existing fields ...
  
  // New fields for multi-tenant
  tenantId                  Int?
  tenant                    Tenant?   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  role                      String    @default("STUDENT") // STUDENT, TEACHER, TENANT_ADMIN, SUPER_ADMIN
  immersioUserId            String?
  immersioSyncStatus        String?   // SYNCED, PENDING, FAILED
  immersioLastSyncedAt      DateTime?
  
  // Relations
  teacherAssignments        TeacherCourseAssignment[]
  whitelistEntries          WhitelistEntry[]
  
  @@index([tenantId])
  @@index([role])
  @@index([immersioUserId])
}
```

### 2.3. Extended EventType Model

```prisma
// Add to existing EventType model
model EventType {
  // ... existing fields ...
  
  // New fields for multi-tenant and workshop
  tenantId                  Int?
  tenant                    Tenant?   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  immersioCourseId          String?
  isWorkshop                Boolean   @default(false)
  whitelistEnabled          Boolean   @default(false)
  seatLimit                 Int?
  refundWindowHours         Int       @default(12)
  workshopPrice             Decimal?  @db.Decimal(10, 2)
  workshopCurrency          String?   @default("USD")
  workshopRecurrence        Json?     // { frequency: "WEEKLY", interval: 1, endDate: ... }
  isPublished               Boolean   @default(true)
  
  // Relations
  teacherAssignments        TeacherCourseAssignment[]
  whitelistEntries          WhitelistEntry[]
  
  @@index([tenantId])
  @@index([immersioCourseId])
  @@index([isWorkshop])
}
```

### 2.4. Extended Booking Model

```prisma
// Add to existing Booking model
model Booking {
  // ... existing fields ...
  
  // New fields for multi-tenant and Immersio
  tenantId                  Int?
  tenant                    Tenant?   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  immersioEnrollmentId      String?
  workshopSessionNumber     Int?
  whitelistVerified         Boolean   @default(false)
  
  @@index([tenantId])
  @@index([immersioEnrollmentId])
}
```

### 2.5. Extended Payment Model

```prisma
// Add to existing Payment model
model Payment {
  // ... existing fields ...
  
  // New fields for multi-tenant Stripe
  tenantId                  Int?
  tenant                    Tenant?   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  stripeAccountId           String?   // Stripe account ID for this tenant
  
  @@index([tenantId])
  @@index([stripeAccountId])
}
```

### 2.6. Tenant Landing Settings

```prisma
model TenantLandingSettings {
  id                        Int       @id @default(autoincrement())
  tenantId                  Int       @unique
  tenant                    Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  templateId                Int       @default(1) // 1-5
  primaryColor              String?   @default("#3B82F6")
  secondaryColor            String?
  logoUrl                   String?
  heroImageUrl              String?
  pageTitle                 String?
  pageSubtitle              String?   @db.Text
  teacherBio                String?   @db.Text
  teacherImageUrl           String?
  socialLinks               Json?     // { facebook, instagram, youtube, tiktok }
  ctaButtonText             String    @default("Book Now")
  published                 Boolean   @default(false)
  seoSettings               Json?     // { metaTitle, metaDescription, ogImage }
  createdAt                 DateTime  @default(now())
  updatedAt                 DateTime  @updatedAt
  
  @@index([tenantId])
  @@index([published])
}
```

### 2.7. Teacher Course Assignment

```prisma
model TeacherCourseAssignment {
  id                        Int       @id @default(autoincrement())
  tenantId                  Int
  tenant                    Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  eventTypeId               Int
  eventType                 EventType @relation(fields: [eventTypeId], references: [id], onDelete: Cascade)
  teacherId                 Int
  teacher                   User      @relation(fields: [teacherId], references: [id], onDelete: Cascade)
  scheduleId                Int?
  schedule                  Schedule? @relation(fields: [scheduleId], references: [id])
  recurrenceRule            Json?     // Cal.com recurrence format
  isPublished               Boolean   @default(false)
  createdAt                 DateTime  @default(now())
  updatedAt                 DateTime  @updatedAt
  
  @@unique([eventTypeId, teacherId])
  @@index([tenantId])
  @@index([eventTypeId])
  @@index([teacherId])
}
```

### 2.8. Whitelist Entry

```prisma
model WhitelistEntry {
  id                        Int       @id @default(autoincrement())
  tenantId                  Int
  tenant                    Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  eventTypeId               Int
  eventType                 EventType @relation(fields: [eventTypeId], references: [id], onDelete: Cascade)
  email                     String
  immersioCourseId          String?
  addedBy                   Int?      // User ID who added this
  addedAt                   DateTime  @default(now())
  
  @@unique([eventTypeId, email])
  @@index([tenantId])
  @@index([eventTypeId])
  @@index([email])
}
```

### 2.9. Immersio Sync Log

```prisma
model ImmersioSyncLog {
  id                        Int       @id @default(autoincrement())
  tenantId                  Int
  tenant                    Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  syncType                  String    // COURSE, STUDENT, BOOKING, ENROLLMENT
  direction                 String    // PULL, PUSH
  resourceId                String    // ID from Immersio or Booking Service
  resourceType              String    // course, student, booking, enrollment
  status                    String    // SUCCESS, FAILED, PENDING
  errorMessage              String?   @db.Text
  errorCode                 String?
  retryCount                Int       @default(0)
  syncedAt                  DateTime  @default(now())
  metadata                  Json?     // Additional sync data
  
  @@index([tenantId])
  @@index([syncType])
  @@index([status])
  @@index([syncedAt])
}
```

### 2.10. Subscription Usage Tracking

```prisma
model TenantUsage {
  id                        Int       @id @default(autoincrement())
  tenantId                  Int       @unique
  tenant                    Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  month                     Int       // 1-12
  year                      Int
  bookingsCount             Int       @default(0)
  teachersCount             Int       @default(0)
  eventTypesCount           Int       @default(0)
  lastResetAt               DateTime  @default(now())
  createdAt                 DateTime  @default(now())
  updatedAt                 DateTime  @updatedAt
  
  @@unique([tenantId, year, month])
  @@index([tenantId])
}
```

---

## 3. INDEXES & PERFORMANCE

### 3.1. Critical Indexes

```sql
-- Multi-tenant queries
CREATE INDEX idx_booking_tenant_status ON "Booking"(tenant_id, status);
CREATE INDEX idx_eventtype_tenant_workshop ON "EventType"(tenant_id, is_workshop);
CREATE INDEX idx_user_tenant_role ON "User"(tenant_id, role);

-- Immersio sync
CREATE INDEX idx_synclog_tenant_status ON "ImmersioSyncLog"(tenant_id, status, synced_at);

-- Whitelist lookups
CREATE INDEX idx_whitelist_event_email ON "WhitelistEntry"(event_type_id, email);

-- Payment tracking
CREATE INDEX idx_payment_tenant_booking ON "Payment"(tenant_id, booking_id);
```

### 3.2. Row-Level Security (Optional)

```sql
-- Enable RLS on tenant-scoped tables
ALTER TABLE "Booking" ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_booking ON "Booking"
  FOR ALL
  USING (tenant_id = current_setting('app.current_tenant_id')::int);
```

---

## 4. DATA MIGRATION SCRIPTS

### 4.1. Migrate Existing Data (if any)

```sql
-- If migrating from single-tenant to multi-tenant
-- Assign all existing data to a default tenant

INSERT INTO "Tenant" (name, subdomain, subscription_plan)
VALUES ('Default Tenant', 'default', 'PRO');

-- Update all existing records
UPDATE "User" SET tenant_id = 1 WHERE tenant_id IS NULL;
UPDATE "EventType" SET tenant_id = 1 WHERE tenant_id IS NULL;
UPDATE "Booking" SET tenant_id = 1 WHERE tenant_id IS NULL;
UPDATE "Payment" SET tenant_id = 1 WHERE tenant_id IS NULL;
```

---

## 5. SEED DATA

### 5.1. Default Templates

```typescript
// Seed landing page templates
const templates = [
  {
    id: 1,
    name: "Education Center",
    description: "Professional template for language centers",
    previewImage: "/templates/education-center.jpg"
  },
  {
    id: 2,
    name: "Personal Tutor",
    description: "Simple template for individual teachers",
    previewImage: "/templates/personal-tutor.jpg"
  },
  // ... 3 more templates
];
```

---

## 6. CONSTRAINTS & VALIDATIONS

### 6.1. Database Constraints

```sql
-- Ensure tenant_id is set for tenant-scoped records
ALTER TABLE "Booking" 
  ADD CONSTRAINT booking_tenant_required 
  CHECK (tenant_id IS NOT NULL);

-- Ensure whitelist emails are valid
ALTER TABLE "WhitelistEntry"
  ADD CONSTRAINT valid_email_format
  CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

-- Ensure refund window is positive
ALTER TABLE "EventType"
  ADD CONSTRAINT positive_refund_window
  CHECK (refund_window_hours >= 0);
```

---

## 7. BACKUP & RECOVERY

### 7.1. Backup Strategy

- **Daily full backups** of PostgreSQL
- **Point-in-time recovery** enabled
- **Tenant-level backups** for critical tenants
- **Encrypted backups** stored in S3/GCS

### 7.2. Disaster Recovery

- RTO (Recovery Time Objective): < 4 hours
- RPO (Recovery Point Objective): < 1 hour
- Multi-region replication for critical data

---

**Document Version**: 1.0  
**Last Updated**: 2024-03-XX
