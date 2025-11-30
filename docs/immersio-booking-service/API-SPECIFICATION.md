# Immersio Booking Service - API Specification

## Overview

This document provides the complete API specification for the Immersio Booking Service, 
following Cal.com's tRPC patterns with REST endpoints for public/webhook APIs.

## API Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       API LAYERS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Layer 1: tRPC (Internal/Authenticated)                       │
│   └── /api/trpc/*                                              │
│       ├── viewer.* (Cal.com existing)                          │
│       ├── teams.* (Cal.com existing)                           │
│       └── immersio.* (New extension)                           │
│                                                                 │
│   Layer 2: REST (Public/External)                              │
│   └── /api/v1/*                                                │
│       ├── /webhooks/* (Incoming from Immersio)                 │
│       ├── /public/* (Public landing pages, workshop listing)   │
│       └── /oauth/* (SSO integration)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. tRPC Routers (Immersio Extension)

### 1.1 Tenant Router (`immersio.tenant`)

#### `immersio.tenant.get`
Get current tenant details for authenticated user.

```typescript
// Input
input: z.object({})

// Output
output: z.object({
  id: z.number(),
  name: z.string(),
  slug: z.string(),
  logoUrl: z.string().nullable(),
  plan: z.object({
    type: z.enum(['lite', 'starter', 'growth', 'pro', 'enterprise']),
    maxTeachers: z.number(),
    maxBookingsPerMonth: z.number(),
    maxEventTypes: z.number(),
    features: z.record(z.boolean()),
  }),
  stripeConnected: z.boolean(),
  immersioConnected: z.boolean(),
  landingPagePublished: z.boolean(),
  createdAt: z.date(),
})

// Example Response
{
  "id": 1,
  "name": "Language Academy",
  "slug": "language-academy",
  "logoUrl": "https://cdn.example.com/logo.png",
  "plan": {
    "type": "growth",
    "maxTeachers": 5,
    "maxBookingsPerMonth": 200,
    "maxEventTypes": -1,
    "features": {
      "whitelist": true,
      "immersioSync": true,
      "customEmails": true
    }
  },
  "stripeConnected": true,
  "immersioConnected": true,
  "landingPagePublished": true,
  "createdAt": "2024-01-15T10:00:00Z"
}
```

#### `immersio.tenant.updateStripeCredentials`
Configure Stripe Connect credentials for the tenant.

```typescript
// Input
input: z.object({
  stripeAccountId: z.string(),
  stripePublishableKey: z.string(),
})

// Output
output: z.object({
  success: z.boolean(),
  credentialId: z.number(),
})

// Example
// Input:
{
  "stripeAccountId": "acct_1234567890",
  "stripePublishableKey": "pk_live_..."
}
// Output:
{
  "success": true,
  "credentialId": 42
}
```

#### `immersio.tenant.getStats`
Get usage statistics for the tenant.

```typescript
// Input
input: z.object({
  period: z.enum(['week', 'month', 'year']),
  startDate: z.date().optional(),
  endDate: z.date().optional(),
})

// Output
output: z.object({
  totalBookings: z.number(),
  completedBookings: z.number(),
  cancelledBookings: z.number(),
  totalRevenue: z.number(), // in cents
  averageBookingValue: z.number(),
  teacherCount: z.number(),
  workshopCount: z.number(),
  studentCount: z.number(),
  topWorkshops: z.array(z.object({
    id: z.number(),
    title: z.string(),
    bookingCount: z.number(),
    revenue: z.number(),
  })),
})
```

#### `immersio.tenant.inviteTeacher`
Invite a teacher to join the tenant.

```typescript
// Input
input: z.object({
  email: z.string().email(),
  name: z.string().optional(),
  role: z.enum(['ADMIN', 'MEMBER']).default('MEMBER'),
})

// Output
output: z.object({
  success: z.boolean(),
  inviteId: z.number(),
  inviteLink: z.string(),
})
```

---

### 1.2 Workshop Router (`immersio.workshop`)

#### `immersio.workshop.create`
Create a new workshop or live class.

```typescript
// Input
input: z.object({
  title: z.string().min(3).max(100),
  description: z.string().max(2000).optional(),
  duration: z.number().min(15).max(480), // minutes
  
  // Pricing
  price: z.number().min(0), // in cents, 0 for free
  currency: z.string().default('USD'),
  
  // Capacity
  seats: z.number().min(1).max(100),
  
  // Scheduling
  scheduleId: z.number().optional(),
  locations: z.array(z.object({
    type: z.enum(['zoom', 'google_meet', 'in_person', 'link']),
    link: z.string().optional(),
    address: z.string().optional(),
  })),
  
  // Recurrence
  recurrence: z.object({
    frequency: z.enum(['weekly', 'biweekly', 'monthly', 'none']),
    count: z.number().optional(), // number of occurrences
    endDate: z.date().optional(),
  }).optional(),
  
  // Assignment
  teacherId: z.number(),
  
  // Registration
  whitelistEnabled: z.boolean().default(false),
  whitelistEmails: z.array(z.string().email()).optional(),
  
  // Immersio
  immersioCourseId: z.string().optional(),
  syncStudentsFromImmersio: z.boolean().default(false),
  
  // Policy
  refundWindowHours: z.number().default(12),
  cancellationNoticeHours: z.number().default(24),
  
  // Category
  topicCategory: z.string().optional(),
  workshopType: z.enum(['live_class', 'workshop', 'tutoring']).default('workshop'),
})

// Output
output: z.object({
  id: z.number(),
  eventTypeId: z.number(),
  slug: z.string(),
  bookingUrl: z.string(),
  createdAt: z.date(),
})

// Example
// Input:
{
  "title": "Business English Workshop",
  "description": "Advanced business communication skills",
  "duration": 60,
  "price": 1500, // $15.00
  "currency": "USD",
  "seats": 8,
  "locations": [
    { "type": "zoom", "link": "https://zoom.us/j/123456" }
  ],
  "recurrence": {
    "frequency": "weekly",
    "count": 8
  },
  "teacherId": 5,
  "whitelistEnabled": true,
  "immersioCourseId": "course_abc123",
  "syncStudentsFromImmersio": true,
  "workshopType": "workshop"
}

// Output:
{
  "id": 42,
  "eventTypeId": 156,
  "slug": "business-english-workshop",
  "bookingUrl": "https://tenant.booking.immersio.com/teacher-name/business-english-workshop",
  "createdAt": "2024-11-30T10:00:00Z"
}
```

#### `immersio.workshop.list`
List all workshops for the tenant.

```typescript
// Input
input: z.object({
  status: z.enum(['active', 'draft', 'archived']).optional(),
  teacherId: z.number().optional(),
  immersioCourseId: z.string().optional(),
  page: z.number().default(1),
  limit: z.number().default(20).max(100),
})

// Output
output: z.object({
  workshops: z.array(z.object({
    id: z.number(),
    eventTypeId: z.number(),
    title: z.string(),
    slug: z.string(),
    description: z.string().nullable(),
    duration: z.number(),
    price: z.number(),
    currency: z.string(),
    seats: z.number(),
    bookedSeats: z.number(),
    teacher: z.object({
      id: z.number(),
      name: z.string(),
      avatarUrl: z.string().nullable(),
    }),
    whitelistEnabled: z.boolean(),
    immersioCourseId: z.string().nullable(),
    status: z.enum(['active', 'draft', 'archived']),
    nextOccurrence: z.date().nullable(),
    createdAt: z.date(),
  })),
  totalCount: z.number(),
  page: z.number(),
  totalPages: z.number(),
})
```

#### `immersio.workshop.updateWhitelist`
Update workshop registration whitelist.

```typescript
// Input
input: z.object({
  workshopId: z.number(),
  mode: z.enum(['add', 'replace', 'remove']),
  emails: z.array(z.object({
    email: z.string().email(),
    name: z.string().optional(),
    immersioStudentId: z.string().optional(),
  })),
})

// Output
output: z.object({
  success: z.boolean(),
  totalWhitelisted: z.number(),
  added: z.number(),
  removed: z.number(),
  duplicatesSkipped: z.number(),
})
```

#### `immersio.workshop.getBookings`
Get bookings for a specific workshop.

```typescript
// Input
input: z.object({
  workshopId: z.number(),
  status: z.enum(['accepted', 'pending', 'cancelled']).optional(),
  startDate: z.date().optional(),
  endDate: z.date().optional(),
  page: z.number().default(1),
  limit: z.number().default(50),
})

// Output
output: z.object({
  bookings: z.array(z.object({
    id: z.number(),
    uid: z.string(),
    title: z.string(),
    startTime: z.date(),
    endTime: z.date(),
    status: z.string(),
    paid: z.boolean(),
    attendees: z.array(z.object({
      email: z.string(),
      name: z.string(),
      timeZone: z.string(),
    })),
    payment: z.object({
      amount: z.number(),
      currency: z.string(),
      status: z.string(),
    }).nullable(),
    metadata: z.object({
      immersioStudentId: z.string().optional(),
      registrationSource: z.string().optional(),
    }),
    createdAt: z.date(),
  })),
  totalCount: z.number(),
})
```

---

### 1.3 Integration Router (`immersio.integration`)

#### `immersio.integration.connect`
Connect Immersio account to tenant.

```typescript
// Input
input: z.object({
  immersioOrgId: z.string(),
  apiKey: z.string(),
  webhookUrl: z.string().url().optional(),
})

// Output
output: z.object({
  success: z.boolean(),
  integrationId: z.number(),
  testResult: z.object({
    connectionValid: z.boolean(),
    orgName: z.string().optional(),
    courseCount: z.number().optional(),
    error: z.string().optional(),
  }),
})
```

#### `immersio.integration.syncCourses`
Trigger manual course sync from Immersio.

```typescript
// Input
input: z.object({
  fullSync: z.boolean().default(false), // true = sync all, false = incremental
})

// Output
output: z.object({
  syncId: z.number(),
  status: z.enum(['started', 'completed', 'failed']),
  coursesFound: z.number(),
  coursesUpdated: z.number(),
  coursesCreated: z.number(),
  errors: z.array(z.object({
    courseId: z.string(),
    error: z.string(),
  })),
})
```

#### `immersio.integration.mapCourse`
Map an Immersio course to a workshop.

```typescript
// Input
input: z.object({
  workshopId: z.number(),
  immersioCourseId: z.string(),
  syncStudents: z.boolean().default(true),
})

// Output
output: z.object({
  mappingId: z.number(),
  courseName: z.string(),
  studentCount: z.number(),
  whitelistUpdated: z.boolean(),
})
```

#### `immersio.integration.getSyncHistory`
Get sync operation history.

```typescript
// Input
input: z.object({
  syncType: z.enum(['courses', 'students', 'all']).optional(),
  limit: z.number().default(20),
})

// Output
output: z.object({
  logs: z.array(z.object({
    id: z.number(),
    syncType: z.string(),
    direction: z.enum(['pull', 'push']),
    status: z.string(),
    recordsProcessed: z.number(),
    recordsFailed: z.number(),
    startedAt: z.date(),
    completedAt: z.date().nullable(),
    errorDetails: z.any().nullable(),
  })),
})
```

---

### 1.4 Landing Page Router (`immersio.landing`)

#### `immersio.landing.getSettings`
Get landing page configuration.

```typescript
// Input
input: z.object({})

// Output
output: z.object({
  id: z.number(),
  templateId: z.string(),
  primaryColor: z.string(),
  secondaryColor: z.string().nullable(),
  logoUrl: z.string().nullable(),
  heroImageUrl: z.string().nullable(),
  pageTitle: z.string(),
  pageSubtitle: z.string().nullable(),
  teacherBio: z.string().nullable(),
  teacherAvatarUrl: z.string().nullable(),
  socialLinks: z.object({
    facebook: z.string().optional(),
    instagram: z.string().optional(),
    youtube: z.string().optional(),
    tiktok: z.string().optional(),
  }),
  ctaText: z.string(),
  metaTitle: z.string().nullable(),
  metaDescription: z.string().nullable(),
  isPublished: z.boolean(),
  publicUrl: z.string().nullable(),
  updatedAt: z.date(),
})
```

#### `immersio.landing.updateSettings`
Update landing page configuration.

```typescript
// Input
input: z.object({
  templateId: z.enum([
    'education',
    'tutor_personal', 
    'center_small',
    'kids_1',
    'kids_2'
  ]).optional(),
  primaryColor: z.string().regex(/^#[0-9A-Fa-f]{6}$/).optional(),
  secondaryColor: z.string().regex(/^#[0-9A-Fa-f]{6}$/).nullable().optional(),
  logoUrl: z.string().url().nullable().optional(),
  heroImageUrl: z.string().url().nullable().optional(),
  pageTitle: z.string().max(100).optional(),
  pageSubtitle: z.string().max(300).nullable().optional(),
  teacherBio: z.string().max(1000).nullable().optional(),
  teacherAvatarUrl: z.string().url().nullable().optional(),
  socialLinks: z.object({
    facebook: z.string().url().optional(),
    instagram: z.string().url().optional(),
    youtube: z.string().url().optional(),
    tiktok: z.string().url().optional(),
  }).optional(),
  ctaText: z.string().max(50).optional(),
  metaTitle: z.string().max(60).nullable().optional(),
  metaDescription: z.string().max(160).nullable().optional(),
})

// Output
output: z.object({
  success: z.boolean(),
  updatedAt: z.date(),
})
```

#### `immersio.landing.publish`
Publish landing page.

```typescript
// Input
input: z.object({})

// Output
output: z.object({
  success: z.boolean(),
  publicUrl: z.string(),
  publishedAt: z.date(),
})
```

#### `immersio.landing.uploadAsset`
Upload image asset for landing page.

```typescript
// Input
input: z.object({
  assetType: z.enum(['logo', 'hero', 'teacher_avatar']),
  file: z.any(), // File upload
})

// Output
output: z.object({
  success: z.boolean(),
  url: z.string(),
  assetId: z.string(),
})
```

---

## 2. REST API Endpoints

### 2.1 Public Endpoints

#### `GET /api/v1/public/landing/:slug`
Get public landing page data.

```yaml
Path Parameters:
  slug: string (required) - Tenant slug

Response 200:
  tenant:
    name: string
    logoUrl: string | null
  settings:
    templateId: string
    primaryColor: string
    secondaryColor: string | null
    heroImageUrl: string | null
    pageTitle: string
    pageSubtitle: string | null
    teacherBio: string | null
    teacherAvatarUrl: string | null
    socialLinks: object
    ctaText: string
    metaTitle: string | null
    metaDescription: string | null
  workshops:
    - id: number
      title: string
      slug: string
      description: string | null
      duration: number
      price: number
      currency: string
      availableSeats: number
      teacher:
        name: string
        avatarUrl: string | null
      nextOccurrence: string | null
      bookingUrl: string

Response 404:
  error: "Tenant not found"
  code: "TENANT_NOT_FOUND"

Example:
GET /api/v1/public/landing/language-academy

Response:
{
  "tenant": {
    "name": "Language Academy",
    "logoUrl": "https://cdn.example.com/logo.png"
  },
  "settings": {
    "templateId": "education",
    "primaryColor": "#0066FF",
    "pageTitle": "Learn Languages Online",
    "ctaText": "Book Now"
  },
  "workshops": [
    {
      "id": 1,
      "title": "Business English",
      "duration": 60,
      "price": 1500,
      "currency": "USD",
      "availableSeats": 5,
      "bookingUrl": "https://language-academy.booking.immersio.com/john-doe/business-english"
    }
  ]
}
```

#### `GET /api/v1/public/workshops/:slug`
Get public workshop details.

```yaml
Path Parameters:
  slug: string (required) - Workshop slug

Query Parameters:
  tenantSlug: string (required) - Tenant slug

Response 200:
  id: number
  title: string
  slug: string
  description: string | null
  duration: number
  price: number
  currency: string
  totalSeats: number
  availableSeats: number
  locations:
    - type: string
      displayValue: string
  teacher:
    name: string
    bio: string | null
    avatarUrl: string | null
  schedule:
    - date: string
      startTime: string
      endTime: string
      availableSeats: number
  bookingUrl: string
  requiresPayment: boolean
  whitelistEnabled: boolean
```

### 2.2 Webhook Endpoints (Incoming from Immersio)

#### `POST /api/v1/webhooks/immersio`
Receive webhook events from Immersio.

```yaml
Headers:
  X-Webhook-Signature: string (required) - HMAC signature
  X-Webhook-Timestamp: string (required) - Unix timestamp
  Content-Type: application/json

Request Body:
  event: string - Event type
  data: object - Event payload
  timestamp: string - ISO timestamp
  webhookId: string - Unique webhook ID

Events Supported:
  - course.created
  - course.updated
  - course.deleted
  - student.enrolled
  - student.unenrolled
  - student.updated

Example - course.updated:
{
  "event": "course.updated",
  "data": {
    "courseId": "course_abc123",
    "name": "Business English Level 2",
    "description": "Updated description",
    "status": "active"
  },
  "timestamp": "2024-11-30T10:00:00Z",
  "webhookId": "wh_xyz789"
}

Example - student.enrolled:
{
  "event": "student.enrolled",
  "data": {
    "courseId": "course_abc123",
    "studentId": "student_456",
    "email": "student@example.com",
    "name": "John Doe",
    "enrolledAt": "2024-11-30T10:00:00Z"
  },
  "timestamp": "2024-11-30T10:00:00Z",
  "webhookId": "wh_xyz790"
}

Response 200:
  success: true
  processed: true

Response 400:
  error: "Invalid signature"
  code: "INVALID_SIGNATURE"
```

### 2.3 OAuth/SSO Endpoints

#### `POST /api/v1/oauth/immersio/callback`
Handle Immersio SSO callback.

```yaml
Request Body:
  token: string - JWT token from Immersio
  redirectUrl: string - Where to redirect after auth

Response 200:
  success: true
  sessionToken: string
  user:
    id: number
    email: string
    name: string
    tenantId: number
    role: string

Response 401:
  error: "Invalid token"
  code: "INVALID_TOKEN"
```

---

## 3. Webhook Events (Outgoing to Immersio)

### Event: `BOOKING_CREATED`

```json
{
  "event": "BOOKING_CREATED",
  "timestamp": "2024-11-30T10:00:00Z",
  "data": {
    "booking": {
      "id": 123,
      "uid": "booking_abc123",
      "title": "Business English Workshop",
      "workshopId": 42,
      "eventTypeId": 156,
      "startTime": "2024-12-01T14:00:00Z",
      "endTime": "2024-12-01T15:00:00Z",
      "status": "accepted",
      "attendee": {
        "email": "student@example.com",
        "name": "John Doe",
        "timeZone": "America/New_York"
      },
      "payment": {
        "amount": 1500,
        "currency": "USD",
        "paid": true,
        "stripePaymentId": "pi_abc123"
      },
      "metadata": {
        "immersioCourseId": "course_abc123",
        "immersioStudentId": "student_456"
      }
    },
    "tenant": {
      "id": 1,
      "slug": "language-academy",
      "immersioOrgId": "org_xyz"
    }
  }
}
```

### Event: `BOOKING_CANCELLED`

```json
{
  "event": "BOOKING_CANCELLED",
  "timestamp": "2024-11-30T11:00:00Z",
  "data": {
    "booking": {
      "id": 123,
      "uid": "booking_abc123",
      "workshopId": 42,
      "cancelledAt": "2024-11-30T11:00:00Z",
      "cancelledBy": "student",
      "cancellationReason": "Schedule conflict"
    },
    "refund": {
      "eligible": true,
      "withinWindow": true,
      "status": "completed",
      "amount": 1500,
      "currency": "USD",
      "stripeRefundId": "re_xyz789"
    },
    "tenant": {
      "id": 1,
      "slug": "language-academy"
    }
  }
}
```

### Event: `PAYMENT_COMPLETED`

```json
{
  "event": "PAYMENT_COMPLETED",
  "timestamp": "2024-11-30T10:00:00Z",
  "data": {
    "payment": {
      "id": 456,
      "bookingId": 123,
      "bookingUid": "booking_abc123",
      "amount": 1500,
      "fee": 0,
      "currency": "USD",
      "stripePaymentId": "pi_abc123",
      "stripeCustomerId": "cus_xyz"
    },
    "workshop": {
      "id": 42,
      "title": "Business English Workshop",
      "immersioCourseId": "course_abc123"
    },
    "attendee": {
      "email": "student@example.com",
      "immersioStudentId": "student_456"
    },
    "tenant": {
      "id": 1,
      "slug": "language-academy",
      "stripeAccountId": "acct_123"
    }
  }
}
```

### Event: `WORKSHOP_CREATED`

```json
{
  "event": "WORKSHOP_CREATED",
  "timestamp": "2024-11-30T10:00:00Z",
  "data": {
    "workshop": {
      "id": 42,
      "eventTypeId": 156,
      "title": "Business English Workshop",
      "slug": "business-english-workshop",
      "duration": 60,
      "price": 1500,
      "currency": "USD",
      "seats": 8,
      "immersioCourseId": "course_abc123",
      "teacher": {
        "id": 5,
        "name": "Jane Smith",
        "email": "jane@example.com"
      }
    },
    "tenant": {
      "id": 1,
      "slug": "language-academy"
    }
  }
}
```

---

## 4. Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Authentication required |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `TENANT_NOT_FOUND` | 404 | Tenant does not exist |
| `WORKSHOP_NOT_FOUND` | 404 | Workshop does not exist |
| `BOOKING_NOT_FOUND` | 404 | Booking does not exist |
| `VALIDATION_ERROR` | 400 | Invalid input data |
| `WHITELIST_DENIED` | 403 | Email not in whitelist |
| `NO_SEATS_AVAILABLE` | 409 | Workshop is fully booked |
| `PAYMENT_FAILED` | 402 | Payment processing failed |
| `REFUND_NOT_ELIGIBLE` | 400 | Outside refund window |
| `STRIPE_NOT_CONFIGURED` | 400 | Tenant has no Stripe setup |
| `IMMERSIO_NOT_CONNECTED` | 400 | Immersio not configured |
| `SYNC_IN_PROGRESS` | 409 | Sync already running |
| `INVALID_WEBHOOK_SIGNATURE` | 401 | Webhook signature mismatch |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Server error |

---

## 5. Rate Limits

| Endpoint Type | Rate Limit | Window |
|---------------|------------|--------|
| Public APIs | 100 requests | 1 minute |
| Authenticated APIs | 1000 requests | 1 minute |
| Webhook endpoints | 500 requests | 1 minute |
| File uploads | 10 requests | 1 minute |
| Sync operations | 5 requests | 1 hour |

---

## 6. Authentication

### tRPC Endpoints
- Require valid session cookie (NextAuth.js)
- Tenant context derived from user's organization membership

### REST Public Endpoints
- No authentication required
- Tenant identified by slug in URL

### REST Webhook Endpoints
- HMAC signature verification using `X-Webhook-Signature` header
- Timestamp validation to prevent replay attacks

### SSO Flow
1. User clicks "Login with Immersio" on booking page
2. Redirect to Immersio with tenant context
3. Immersio authenticates and returns JWT
4. Booking service validates JWT and creates session
