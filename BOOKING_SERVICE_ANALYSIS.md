# PHÂN TÍCH HỆ THỐNG BOOKING SERVICE - TÍCH HỢP IMMERSIO

## MỤC LỤC
1. [Tổng quan dự án](#1-tổng-quan-dự-án)
2. [Use Cases](#2-use-cases)
3. [Kiến trúc hệ thống](#3-kiến-trúc-hệ-thống)
4. [Phân tích Module/Feature](#4-phân-tích-modulefeature)
5. [Database Schema (ERD)](#5-database-schema-erd)
6. [API Specification](#6-api-specification)
7. [Kế hoạch triển khai](#7-kế-hoạch-triển-khai)
8. [Ước lượng thời gian và chi phí](#8-ước-lượng-thời-gian-và-chi-phí)

---

## 1. TỔNG QUAN DỰ ÁN

### 1.1. Mục tiêu
Xây dựng **Booking Service** dựa trên Cal.com open-source với các đặc điểm:
- **Multi-tenant SaaS**: Mỗi giáo viên/trung tâm là một tenant độc lập
- **Tích hợp Immersio**: Đồng bộ 2 chiều với hệ thống LMS Immersio
- **Tận dụng tối đa Cal.com**: Reuse 60-70% codebase hiện có

### 1.2. Tech Stack
- **Frontend**: ReactJS + Next.js (App Router)
- **Backend**: Node.js (Next.js API Routes + tRPC)
- **Database**: PostgreSQL (multi-tenant với `tenant_id`)
- **ORM**: Prisma
- **Payment**: Stripe (multi-account per tenant)
- **Video**: Zoom API
- **Cache**: Redis

### 1.3. User Roles
1. **Super Admin**: Quản lý toàn hệ thống
2. **Tenant Admin**: Quản lý trung tâm/giáo viên chủ
3. **Teacher**: Quản lý lịch và nhận booking
4. **Student**: Đăng ký và thanh toán

---

## 2. USE CASES

### 2.1. Use Cases - Tenant Admin

#### UC-ADMIN-01: Tạo Tenant mới
**Actor**: Super Admin  
**Precondition**: Super Admin đã đăng nhập  
**Flow**:
1. Super Admin tạo tenant mới
2. Hệ thống tạo subdomain/domain cho tenant
3. Gửi email kích hoạt cho Tenant Admin
4. Tenant Admin đăng ký tài khoản
5. Hệ thống tạo workspace cho tenant

**Postcondition**: Tenant được tạo và sẵn sàng sử dụng

#### UC-ADMIN-02: Cấu hình Stripe per Tenant
**Actor**: Tenant Admin  
**Precondition**: Tenant Admin đã đăng nhập  
**Flow**:
1. Tenant Admin vào Settings > Payment
2. Nhập Stripe API keys (publishable key + secret key)
3. Hệ thống verify keys với Stripe
4. Lưu keys vào database (encrypted)
5. Cấu hình webhook endpoint cho tenant

**Postcondition**: Tenant có thể nhận thanh toán qua Stripe riêng

#### UC-ADMIN-03: Tích hợp khóa học từ Immersio
**Actor**: Tenant Admin  
**Precondition**: Tenant đã kết nối với Immersio  
**Flow**:
1. Tenant Admin vào Settings > Immersio Integration
2. Nhập Immersio API credentials
3. Hệ thống pull danh sách khóa học từ Immersio
4. Tenant Admin chọn khóa học muốn sync
5. Hệ thống tạo EventType tương ứng trong Booking Service
6. Đồng bộ học viên từ Immersio sang Booking Service

**Postcondition**: Khóa học và học viên được đồng bộ

#### UC-ADMIN-04: Gán giáo viên vào khóa học
**Actor**: Tenant Admin  
**Precondition**: Đã có khóa học và giáo viên trong hệ thống  
**Flow**:
1. Tenant Admin chọn khóa học
2. Chọn giáo viên từ danh sách
3. Chọn lịch rảnh của giáo viên (từ Cal.com availability)
4. Hệ thống tạo series event (recurrence) với lịch đã chọn
5. Event được ẩn cho đến khi admin publish

**Postcondition**: Giáo viên được gán vào khóa học với lịch cụ thể

#### UC-ADMIN-05: Tạo Workshop trả phí
**Actor**: Tenant Admin  
**Precondition**: Tenant Admin đã đăng nhập  
**Flow**:
1. Tenant Admin tạo EventType mới (workshop)
2. Cấu hình:
   - Tên, mô tả, giá ($7-15)
   - Số chỗ (5-10)
   - Gán giáo viên
   - Lặp lại (weekly/bi-weekly)
   - Zoom link
   - Refund policy (12 giờ)
3. Chọn chế độ đăng ký:
   - Whitelist (chỉ học viên đã mua khóa học)
   - Public (ai cũng đăng ký được)
4. Publish workshop

**Postcondition**: Workshop được tạo và hiển thị trên booking page

#### UC-ADMIN-06: Cấu hình Landing Page
**Actor**: Tenant Admin  
**Precondition**: Tenant Admin đã đăng nhập  
**Flow**:
1. Tenant Admin vào Settings > Landing Page
2. Chọn template (1 trong 5 templates)
3. Upload logo, chọn màu chủ đạo
4. Upload ảnh hero, nhập tiêu đề + mô tả
5. Nhập giới thiệu giáo viên
6. Thêm social links
7. Preview và Publish

**Postcondition**: Landing page được tạo và accessible qua subdomain

### 2.2. Use Cases - Teacher

#### UC-TEACHER-01: Cập nhật lịch trống (Availability)
**Actor**: Teacher  
**Precondition**: Teacher đã đăng nhập  
**Flow**:
1. Teacher vào My Schedule
2. Chọn ngày/tuần cần cập nhật
3. Đánh dấu các slot rảnh
4. Hệ thống sync với Google Calendar (nếu có)
5. Lưu availability vào database

**Postcondition**: Lịch trống được cập nhật và hiển thị cho học viên

#### UC-TEACHER-02: Nhận thông báo booking mới
**Actor**: Teacher  
**Precondition**: Teacher đã có event type và availability  
**Flow**:
1. Học viên đặt lịch thành công
2. Hệ thống gửi email/SMS thông báo cho Teacher
3. Teacher nhận thông báo với thông tin:
   - Tên học viên
   - Thời gian
   - Zoom link (nếu có)
   - Custom fields (nếu có)

**Postcondition**: Teacher nhận được thông báo booking

#### UC-TEACHER-03: Quản lý Zoom link
**Actor**: Teacher  
**Precondition**: Teacher đã tích hợp Zoom  
**Flow**:
1. Teacher tạo event type với Zoom integration
2. Hệ thống tự động tạo Zoom meeting khi booking được confirm
3. Zoom link được gửi cho học viên qua email
4. Teacher có thể xem/edit Zoom link trong booking detail

**Postcondition**: Zoom link được tự động tạo và gửi

### 2.3. Use Cases - Student

#### UC-STUDENT-01: Xem lịch học và đăng ký
**Actor**: Student  
**Precondition**: Student đã có link booking page  
**Flow**:
1. Student truy cập booking page của tenant
2. Xem danh sách workshop/live class
3. Chọn workshop muốn đăng ký
4. Chọn thời gian phù hợp
5. Điền thông tin (tên, email, custom fields)
6. Nếu workshop trả phí → chuyển đến thanh toán
7. Nếu workshop whitelist → kiểm tra email trong whitelist
8. Xác nhận booking
9. Nhận email confirmation với Zoom link

**Postcondition**: Booking được tạo và student nhận confirmation

#### UC-STUDENT-02: Thanh toán workshop
**Actor**: Student  
**Precondition**: Student đã chọn workshop trả phí  
**Flow**:
1. Sau khi điền thông tin booking
2. Hệ thống redirect đến Stripe Checkout (của tenant)
3. Student thanh toán
4. Stripe webhook gửi payment success
5. Hệ thống cập nhật booking status = CONFIRMED
6. Gửi email confirmation cho student và teacher

**Postcondition**: Thanh toán thành công, booking được confirm

#### UC-STUDENT-03: Hủy booking và hoàn tiền
**Actor**: Student  
**Precondition**: Student đã có booking đã thanh toán  
**Flow**:
1. Student vào booking detail
2. Click "Cancel Booking"
3. Hệ thống kiểm tra cancellation window (12 giờ)
4. Nếu trong window → cho phép hủy và refund
5. Nếu ngoài window → không cho hủy hoặc không refund
6. Nếu được refund → tự động refund qua Stripe
7. Gửi email xác nhận hủy

**Postcondition**: Booking được hủy, tiền được refund (nếu trong window)

#### UC-STUDENT-04: Reschedule booking
**Actor**: Student  
**Precondition**: Student đã có booking  
**Flow**:
1. Student vào booking detail
2. Click "Reschedule"
3. Hệ thống hiển thị các slot còn trống
4. Student chọn slot mới
5. Hệ thống cập nhật booking với thời gian mới
6. Gửi email thông báo cho student và teacher

**Postcondition**: Booking được reschedule thành công

### 2.4. Use Cases - Immersio Integration

#### UC-INTEGRATION-01: Pull khóa học từ Immersio
**Actor**: System (Scheduled Job)  
**Precondition**: Tenant đã cấu hình Immersio API  
**Flow**:
1. Scheduled job chạy mỗi X phút
2. Gọi Immersio API để lấy danh sách khóa học mới/cập nhật
3. So sánh với database hiện tại
4. Tạo/cập nhật EventType tương ứng
5. Log kết quả sync

**Postcondition**: Khóa học được đồng bộ từ Immersio

#### UC-INTEGRATION-02: Push booking sang Immersio
**Actor**: System (Webhook/Event)  
**Precondition**: Booking được tạo thành công  
**Flow**:
1. Booking được tạo trong Booking Service
2. Hệ thống trigger webhook/event
3. Gọi Immersio API để tạo/update enrollment
4. Immersio xác nhận thành công
5. Lưu Immersio enrollment ID vào booking metadata

**Postcondition**: Booking được sync sang Immersio

#### UC-INTEGRATION-03: SSO giữa Immersio và Booking Service
**Actor**: Student/Teacher  
**Precondition**: Tenant đã cấu hình SSO  
**Flow**:
1. User đăng nhập vào Immersio
2. Click link đến Booking Service
3. Immersio redirect với JWT token
4. Booking Service verify token
5. Tự động đăng nhập user vào Booking Service
6. User có thể đặt lịch mà không cần login lại

**Postcondition**: User được đăng nhập tự động qua SSO

---

## 3. KIẾN TRÚC HỆ THỐNG

### 3.1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                          │
├─────────────────────────────────────────────────────────────┤
│  Web App (Next.js)  │  Mobile App (Future)  │  Embed Widget  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      API GATEWAY LAYER                       │
├─────────────────────────────────────────────────────────────┤
│  Next.js API Routes  │  tRPC Router  │  Webhook Endpoints   │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Booking     │  │  Payment     │  │  Immersio    │
│  Service     │  │  Service     │  │  Integration │
└──────────────┘  └──────────────┘  └──────────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    BUSINESS LOGIC LAYER                     │
├─────────────────────────────────────────────────────────────┤
│  Cal.com Core  │  Multi-tenant  │  Custom Business Rules    │
│  Modules       │  Logic         │  (Workshop, Whitelist)   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      DATA LAYER                              │
├─────────────────────────────────────────────────────────────┤
│  PostgreSQL (Multi-tenant)  │  Redis (Cache/Queue)          │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Stripe     │  │  Zoom API    │  │  Immersio    │
│  (Multi)    │  │              │  │  API         │
└──────────────┘  └──────────────┘  └──────────────┘
```

### 3.2. Multi-Tenant Architecture

**Strategy**: Shared Database với Row-Level Security

```
┌─────────────────────────────────────────────────────────┐
│              PostgreSQL Database                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Tenant Table                                     │  │
│  │  - id, name, subdomain, stripe_keys, settings    │  │
│  └──────────────────────────────────────────────────┘  │
│                          │                              │
│        ┌─────────────────┼─────────────────┐            │
│        ▼                 ▼                 ▼            │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐       │
│  │ EventType│      │ Booking │      │  User   │       │
│  │ +tenant_id│      │+tenant_id│      │+tenant_id│      │
│  └─────────┘      └─────────┘      └─────────┘       │
│                                                          │
│  All tables have tenant_id for data isolation           │
└─────────────────────────────────────────────────────────┘
```

### 3.3. Service Layer Breakdown

#### 3.3.1. Booking Service
- **Responsibility**: Core booking logic từ Cal.com
- **Reuse**: 70% code từ `packages/features/bookings`
- **Customization**: 
  - Multi-tenant filtering
  - Workshop-specific rules
  - Whitelist validation

#### 3.3.2. Payment Service
- **Responsibility**: Stripe integration per tenant
- **Reuse**: Cal.com payment module
- **Customization**:
  - Multi-Stripe account routing
  - Refund automation (12h window)
  - Per-tenant webhook handling

#### 3.3.3. Immersio Integration Service
- **Responsibility**: 2-way sync với Immersio
- **New Development**: 100% custom
- **Features**:
  - Pull courses/students
  - Push bookings/enrollments
  - SSO authentication

#### 3.3.4. Tenant Service
- **Responsibility**: Tenant management
- **New Development**: 100% custom
- **Features**:
  - Tenant CRUD
  - Subdomain management
  - Settings management

#### 3.3.5. Landing Page Service
- **Responsibility**: Template-based landing pages
- **New Development**: 100% custom
- **Features**:
  - Template rendering
  - Customization UI
  - Public page serving

---

## 4. PHÂN TÍCH MODULE/FEATURE

### 4.1. Module từ Cal.com (Reuse)

| Module | Reuse % | Customization Needed |
|--------|---------|---------------------|
| **Availability** | 80% | Thêm tenant_id filter |
| **Event Types** | 70% | Thêm workshop fields, whitelist logic |
| **Booking Core** | 75% | Multi-tenant, workshop rules |
| **Payment (Stripe)** | 60% | Multi-account routing, per-tenant webhooks |
| **Cancellation/Refund** | 85% | Custom 12h window rule |
| **Reschedule** | 80% | Thêm tenant context |
| **Calendar Sync** | 90% | Minimal changes |
| **Email Notifications** | 85% | Template customization |
| **Webhooks** | 70% | Immersio-specific webhooks |

### 4.2. Module mới cần phát triển

#### 4.2.1. Multi-Tenant Module
**Priority**: P0 (Critical)  
**Complexity**: High  
**Components**:
- Tenant model & CRUD
- Tenant context middleware
- Row-level security policies
- Subdomain routing
- Tenant isolation middleware

**Dependencies**: None  
**Estimated Effort**: 3-4 weeks

#### 4.2.2. Immersio Integration Module
**Priority**: P0 (Critical)  
**Complexity**: High  
**Components**:
- Immersio API client
- Course sync service
- Student sync service
- Booking push service
- SSO authentication
- Sync job scheduler

**Dependencies**: Multi-tenant module  
**Estimated Effort**: 4-5 weeks

#### 4.2.3. Workshop Management Module
**Priority**: P0 (Critical)  
**Complexity**: Medium  
**Components**:
- Workshop EventType extension
- Whitelist management
- Pay-per-session logic
- Seat limitation
- Recurrence rules

**Dependencies**: Event Types, Payment  
**Estimated Effort**: 2-3 weeks

#### 4.2.4. Landing Page Module
**Priority**: P1 (High)  
**Complexity**: Medium  
**Components**:
- Template system (5 templates)
- Template customization UI
- Public page rendering
- SEO optimization

**Dependencies**: Tenant module  
**Estimated Effort**: 2-3 weeks

#### 4.2.5. Teacher Assignment Module
**Priority**: P0 (Critical)  
**Complexity**: Medium  
**Components**:
- Teacher-course assignment
- Availability selection UI
- Series event creation
- Conflict detection

**Dependencies**: Availability, Event Types  
**Estimated Effort**: 2 weeks

#### 4.2.6. Admin Panel Module
**Priority**: P1 (High)  
**Complexity**: Medium  
**Components**:
- Super Admin dashboard
- Tenant management UI
- Subscription management
- System logs & monitoring

**Dependencies**: Multi-tenant module  
**Estimated Effort**: 3-4 weeks

### 4.3. Feature Matrix

| Feature | Cal.com Reuse | New Dev | Total Effort |
|---------|--------------|---------|--------------|
| Multi-tenant | 0% | 100% | 3-4 weeks |
| Booking Core | 75% | 25% | 1 week |
| Payment (Multi-Stripe) | 60% | 40% | 2 weeks |
| Immersio Integration | 0% | 100% | 4-5 weeks |
| Workshop Management | 30% | 70% | 2-3 weeks |
| Landing Pages | 0% | 100% | 2-3 weeks |
| Teacher Assignment | 50% | 50% | 2 weeks |
| Admin Panel | 20% | 80% | 3-4 weeks |
| **TOTAL** | **~40%** | **~60%** | **19-26 weeks** |

---

## 5. DATABASE SCHEMA (ERD)

### 5.1. Core Tables (Extended from Cal.com)

#### Tenant Table (NEW)
```sql
CREATE TABLE "Tenant" (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  subdomain VARCHAR(100) UNIQUE NOT NULL,
  custom_domain VARCHAR(255),
  stripe_publishable_key VARCHAR(255),
  stripe_secret_key_encrypted TEXT,
  stripe_webhook_secret VARCHAR(255),
  immersio_api_key_encrypted TEXT,
  immersio_api_url VARCHAR(255),
  subscription_plan VARCHAR(50) DEFAULT 'LITE',
  subscription_status VARCHAR(50) DEFAULT 'ACTIVE',
  settings JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### EventType (Extended)
```sql
-- Add columns to existing EventType table
ALTER TABLE "EventType" ADD COLUMN tenant_id INTEGER REFERENCES "Tenant"(id);
ALTER TABLE "EventType" ADD COLUMN immersio_course_id VARCHAR(255);
ALTER TABLE "EventType" ADD COLUMN is_workshop BOOLEAN DEFAULT FALSE;
ALTER TABLE "EventType" ADD COLUMN whitelist_enabled BOOLEAN DEFAULT FALSE;
ALTER TABLE "EventType" ADD COLUMN whitelist_emails TEXT[];
ALTER TABLE "EventType" ADD COLUMN seat_limit INTEGER;
ALTER TABLE "EventType" ADD COLUMN refund_window_hours INTEGER DEFAULT 12;
```

#### Booking (Extended)
```sql
-- Add columns to existing Booking table
ALTER TABLE "Booking" ADD COLUMN tenant_id INTEGER REFERENCES "Tenant"(id);
ALTER TABLE "Booking" ADD COLUMN immersio_enrollment_id VARCHAR(255);
ALTER TABLE "Booking" ADD COLUMN workshop_session_number INTEGER;
```

#### Payment (Extended)
```sql
-- Add columns to existing Payment table
ALTER TABLE "Payment" ADD COLUMN tenant_id INTEGER REFERENCES "Tenant"(id);
ALTER TABLE "Payment" ADD COLUMN stripe_account_id VARCHAR(255);
```

#### User (Extended)
```sql
-- Add columns to existing User table
ALTER TABLE "User" ADD COLUMN tenant_id INTEGER REFERENCES "Tenant"(id);
ALTER TABLE "User" ADD COLUMN role VARCHAR(50) DEFAULT 'STUDENT'; -- STUDENT, TEACHER, TENANT_ADMIN
ALTER TABLE "User" ADD COLUMN immersio_user_id VARCHAR(255);
```

### 5.2. New Tables

#### TenantLandingSettings
```sql
CREATE TABLE "TenantLandingSettings" (
  id SERIAL PRIMARY KEY,
  tenant_id INTEGER UNIQUE REFERENCES "Tenant"(id) ON DELETE CASCADE,
  template_id INTEGER NOT NULL,
  primary_color VARCHAR(7),
  logo_url VARCHAR(500),
  hero_image_url VARCHAR(500),
  page_title VARCHAR(255),
  page_subtitle TEXT,
  teacher_bio TEXT,
  social_links JSONB,
  published BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### TeacherCourseAssignment
```sql
CREATE TABLE "TeacherCourseAssignment" (
  id SERIAL PRIMARY KEY,
  tenant_id INTEGER REFERENCES "Tenant"(id) ON DELETE CASCADE,
  event_type_id INTEGER REFERENCES "EventType"(id) ON DELETE CASCADE,
  teacher_id INTEGER REFERENCES "User"(id) ON DELETE CASCADE,
  schedule_id INTEGER REFERENCES "Schedule"(id),
  recurrence_rule JSONB,
  is_published BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(event_type_id, teacher_id)
);
```

#### ImmersioSyncLog
```sql
CREATE TABLE "ImmersioSyncLog" (
  id SERIAL PRIMARY KEY,
  tenant_id INTEGER REFERENCES "Tenant"(id) ON DELETE CASCADE,
  sync_type VARCHAR(50), -- COURSE, STUDENT, BOOKING
  direction VARCHAR(50), -- PULL, PUSH
  resource_id VARCHAR(255),
  status VARCHAR(50), -- SUCCESS, FAILED
  error_message TEXT,
  synced_at TIMESTAMP DEFAULT NOW()
);
```

#### WhitelistEntry
```sql
CREATE TABLE "WhitelistEntry" (
  id SERIAL PRIMARY KEY,
  tenant_id INTEGER REFERENCES "Tenant"(id) ON DELETE CASCADE,
  event_type_id INTEGER REFERENCES "EventType"(id) ON DELETE CASCADE,
  email VARCHAR(255) NOT NULL,
  immersio_course_id VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(event_type_id, email)
);
```

### 5.3. ERD Diagram (Text Representation)

```
┌─────────────┐
│   Tenant    │
│─────────────│
│ id (PK)     │
│ name        │
│ subdomain   │
│ stripe_keys │
└──────┬──────┘
       │ 1
       │
       │ N
┌──────▼──────────┐         ┌──────────────┐
│    User        │         │  EventType   │
│────────────────│         │──────────────│
│ id (PK)        │         │ id (PK)      │
│ tenant_id (FK) │◄────────┤ tenant_id(FK)│
│ role           │    N    │ is_workshop  │
│                │         │ whitelist    │
└──────┬─────────┘         └──────┬───────┘
       │                          │
       │ N                        │ N
       │                          │
┌──────▼──────────┐         ┌─────▼──────────┐
│   Booking       │         │TeacherCourse   │
│─────────────────│         │ Assignment    │
│ id (PK)         │         │───────────────│
│ tenant_id (FK)  │         │ id (PK)       │
│ event_type_id   │◄────────┤ event_type_id │
│ teacher_id      │         │ teacher_id    │
│ status          │         │ schedule_id   │
└──────┬──────────┘         └───────────────┘
       │
       │ N
┌──────▼──────────┐
│    Payment     │
│────────────────│
│ id (PK)        │
│ tenant_id (FK) │
│ booking_id (FK)│
│ stripe_account │
└────────────────┘
```

---

## 6. API SPECIFICATION

### 6.1. Tenant Management APIs

#### Create Tenant
```
POST /api/v1/admin/tenants
Authorization: Bearer <super_admin_token>

Request:
{
  "name": "ABC Language Center",
  "subdomain": "abc-center",
  "adminEmail": "admin@abc-center.com"
}

Response:
{
  "id": 1,
  "name": "ABC Language Center",
  "subdomain": "abc-center",
  "status": "PENDING_ACTIVATION"
}
```

#### Update Tenant Stripe Keys
```
PUT /api/v1/tenant/stripe
Authorization: Bearer <tenant_admin_token>

Request:
{
  "publishableKey": "pk_live_...",
  "secretKey": "sk_live_..."
}

Response:
{
  "success": true,
  "webhookUrl": "https://api.booking-service.com/webhooks/stripe/{tenant_id}"
}
```

### 6.2. Immersio Integration APIs

#### Sync Courses from Immersio
```
POST /api/v1/tenant/immersio/sync-courses
Authorization: Bearer <tenant_admin_token>

Request:
{
  "courseIds": ["course_123", "course_456"] // Optional, empty = sync all
}

Response:
{
  "synced": 5,
  "created": 3,
  "updated": 2,
  "courses": [...]
}
```

#### Push Booking to Immersio
```
POST /api/v1/tenant/immersio/push-booking
Authorization: Bearer <system_token>

Request:
{
  "bookingId": 123,
  "immersioCourseId": "course_123"
}

Response:
{
  "success": true,
  "immersioEnrollmentId": "enrollment_456"
}
```

### 6.3. Workshop Management APIs

#### Create Workshop
```
POST /api/v1/tenant/workshops
Authorization: Bearer <tenant_admin_token>

Request:
{
  "title": "Advanced English Conversation",
  "description": "...",
  "price": 10.00,
  "currency": "USD",
  "seatLimit": 8,
  "teacherId": 5,
  "recurrence": {
    "frequency": "WEEKLY",
    "interval": 1
  },
  "whitelistEnabled": true,
  "whitelistEmails": ["student1@email.com"],
  "refundWindowHours": 12
}

Response:
{
  "id": 10,
  "slug": "advanced-english-conversation",
  "bookingUrl": "https://abc-center.booking-service.com/workshop/advanced-english-conversation"
}
```

#### Add to Whitelist
```
POST /api/v1/tenant/workshops/{workshopId}/whitelist
Authorization: Bearer <tenant_admin_token>

Request:
{
  "emails": ["student1@email.com", "student2@email.com"]
}

Response:
{
  "added": 2,
  "total": 5
}
```

### 6.4. Booking APIs (Extended from Cal.com)

#### Create Booking (with whitelist check)
```
POST /api/v1/bookings
Authorization: Bearer <student_token> (or public)

Request:
{
  "eventTypeId": 10,
  "startTime": "2024-03-15T10:00:00Z",
  "attendeeEmail": "student@email.com",
  "attendeeName": "John Doe",
  "customInputs": {...}
}

Response:
{
  "id": 123,
  "uid": "booking_abc123",
  "status": "ACCEPTED",
  "paymentRequired": true,
  "paymentUrl": "https://checkout.stripe.com/..."
}
```

### 6.5. Landing Page APIs

#### Get Landing Page
```
GET /api/v1/public/landing/{subdomain}
No authentication required

Response:
{
  "tenant": {
    "name": "ABC Language Center",
    "subdomain": "abc-center"
  },
  "landingPage": {
    "templateId": 1,
    "primaryColor": "#3B82F6",
    "logoUrl": "...",
    "heroImageUrl": "...",
    "pageTitle": "Learn English with ABC",
    "pageSubtitle": "...",
    "teacherBio": "...",
    "socialLinks": {...}
  },
  "workshops": [...],
  "courses": [...]
}
```

---

## 7. KẾ HOẠCH TRIỂN KHAI

### 7.1. Phase 1: MVP Core Booking (6-8 weeks)

#### Week 1-2: Foundation & Multi-Tenant
- [ ] Setup project structure
- [ ] Implement Tenant model & CRUD
- [ ] Multi-tenant middleware
- [ ] Subdomain routing
- [ ] Database migration scripts

#### Week 3-4: Cal.com Integration
- [ ] Fork/extend Cal.com booking module
- [ ] Add tenant_id to all queries
- [ ] Test booking flow với multi-tenant
- [ ] Email notifications với tenant context

#### Week 5-6: Teacher & Availability
- [ ] Teacher availability UI
- [ ] Calendar sync (Google/Outlook)
- [ ] Teacher assignment to courses
- [ ] Series event creation

#### Week 7-8: Basic Workshop
- [ ] Workshop EventType extension
- [ ] Seat limitation
- [ ] Basic booking flow
- [ ] Testing & bug fixes

**Deliverables**:
- ✅ Multi-tenant system working
- ✅ Basic booking flow
- ✅ Teacher availability
- ✅ Workshop creation & booking

### 7.2. Phase 2: Payments & Policies (4-5 weeks)

#### Week 9-10: Multi-Stripe Integration
- [ ] Stripe multi-account setup
- [ ] Per-tenant webhook handling
- [ ] Payment flow integration
- [ ] Test payments end-to-end

#### Week 11-12: Refund & Cancellation
- [ ] 12-hour refund window logic
- [ ] Automatic refund processing
- [ ] Cancellation UI
- [ ] Refund history tracking

#### Week 13: Reschedule
- [ ] Reschedule logic
- [ ] UI for rescheduling
- [ ] Email notifications

**Deliverables**:
- ✅ Stripe payments per tenant
- ✅ Refund automation
- ✅ Cancellation & reschedule

### 7.3. Phase 3: Immersio Integration (4-5 weeks)

#### Week 14-15: Immersio API Client
- [ ] Immersio API client library
- [ ] Authentication & credentials
- [ ] Error handling & retries

#### Week 16-17: Course & Student Sync
- [ ] Pull courses from Immersio
- [ ] Pull students from Immersio
- [ ] Sync job scheduler
- [ ] Conflict resolution

#### Week 18: Booking Push & SSO
- [ ] Push bookings to Immersio
- [ ] SSO implementation
- [ ] Testing integration

**Deliverables**:
- ✅ 2-way sync với Immersio
- ✅ SSO authentication
- ✅ Automated sync jobs

### 7.4. Phase 4: Advanced Features (4-5 weeks)

#### Week 19-20: Whitelist & Advanced Workshop
- [ ] Whitelist management UI
- [ ] Whitelist validation in booking
- [ ] Workshop recurrence rules
- [ ] Public vs whitelist modes

#### Week 21-22: Landing Pages
- [ ] Template system (5 templates)
- [ ] Customization UI
- [ ] Public page rendering
- [ ] SEO optimization

#### Week 23: Admin Panel
- [ ] Super Admin dashboard
- [ ] Tenant management UI
- [ ] Subscription management
- [ ] System monitoring

**Deliverables**:
- ✅ Landing pages
- ✅ Whitelist functionality
- ✅ Admin panel

### 7.5. Phase 5: Polish & Launch (2-3 weeks)

#### Week 24-25: Testing & Optimization
- [ ] End-to-end testing
- [ ] Performance optimization
- [ ] Security audit
- [ ] Documentation

#### Week 26: Launch Preparation
- [ ] Production deployment
- [ ] Monitoring setup
- [ ] Support documentation
- [ ] User training

**Deliverables**:
- ✅ Production-ready system
- ✅ Documentation
- ✅ Monitoring & support

---

## 8. ƯỚC LƯỢNG THỜI GIAN VÀ CHI PHÍ

### 8.1. Effort Breakdown by Phase

| Phase | Duration | Team Size | Total Person-Weeks |
|-------|----------|-----------|-------------------|
| Phase 1: MVP | 6-8 weeks | 3-4 devs | 18-32 |
| Phase 2: Payments | 4-5 weeks | 2-3 devs | 8-15 |
| Phase 3: Immersio | 4-5 weeks | 2-3 devs | 8-15 |
| Phase 4: Advanced | 4-5 weeks | 2-3 devs | 8-15 |
| Phase 5: Polish | 2-3 weeks | 2 devs | 4-6 |
| **TOTAL** | **20-26 weeks** | | **46-83 person-weeks** |

### 8.2. Team Composition

**Recommended Team**:
- **1 Tech Lead** (Full-time)
- **2-3 Backend Developers** (Full-time)
- **1-2 Frontend Developers** (Full-time)
- **1 DevOps Engineer** (Part-time, 50%)
- **1 QA Engineer** (Part-time, 50%)

**Total**: 5-7 FTE

### 8.3. Cost Estimation (Rough)

**Assumptions**:
- Development rate: $X per developer per week
- Project duration: 20-26 weeks
- Team size: 5-7 FTE

**Breakdown**:
- Development: 46-83 person-weeks × rate
- Infrastructure (AWS/GCP): ~$500-1000/month
- Third-party services (Stripe, Zoom): Pay-per-use
- Testing tools: ~$200/month

**Total Development Cost**: [To be calculated based on actual rates]

### 8.4. Risk Factors

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Cal.com version updates | High | Lock version, create abstraction layer |
| Immersio API changes | Medium | Version API, maintain compatibility layer |
| Multi-tenant performance | High | Load testing, database optimization |
| Stripe multi-account complexity | Medium | Use Stripe Connect or separate accounts |
| Timeline delays | Medium | Buffer time, phased delivery |

### 8.5. Success Metrics

**Technical Metrics**:
- System uptime: >99.5%
- API response time: <200ms (p95)
- Booking success rate: >99%
- Payment success rate: >98%

**Business Metrics**:
- Number of active tenants
- Monthly recurring revenue (MRR)
- Booking conversion rate
- Customer satisfaction score

---

## 9. KẾT LUẬN

### 9.1. Tóm tắt

Dự án Booking Service dựa trên Cal.com là khả thi và có thể tận dụng được **60-70% codebase hiện có** của Cal.com, giúp giảm đáng kể thời gian và chi phí phát triển.

**Điểm mạnh**:
- Cal.com đã có sẵn booking engine mạnh mẽ
- Multi-tenant architecture có thể implement với shared database
- Stripe integration đã có sẵn, chỉ cần customize cho multi-account

**Thách thức**:
- Immersio integration cần phát triển từ đầu
- Multi-tenant logic cần được implement cẩn thận để đảm bảo data isolation
- Performance optimization cho multi-tenant queries

### 9.2. Khuyến nghị

1. **Bắt đầu với MVP**: Focus vào core booking flow trước, sau đó mở rộng
2. **Phased delivery**: Deliver từng phase để có feedback sớm
3. **Reuse tối đa**: Tận dụng Cal.com modules, chỉ customize khi cần
4. **Testing sớm**: Test multi-tenant và integration sớm trong quá trình phát triển
5. **Documentation**: Maintain documentation tốt cho team và future maintenance

### 9.3. Next Steps

1. **Review và approve** document này
2. **Setup development environment**
3. **Kickoff meeting** với team
4. **Begin Phase 1** development

---

**Document Version**: 1.0  
**Last Updated**: 2024-03-XX  
**Author**: AI Assistant  
**Status**: Draft - Pending Review
