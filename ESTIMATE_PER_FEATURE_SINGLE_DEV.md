# Estimate Per Feature (Single Resource / Single Developer)

**Assumptions**

- 1 resource = 1 full‑time senior full‑stack engineer (can do FE/BE on top of Cal.com OSS).
- Working day = 1 dev‑day ≈ 8 hours.
- Only using **Cal.com open‑source (AGPL)** parts; **no EE**. Any EE‑like feature (payments, admin, organizations, etc.) must be implemented by us.
- Estimates include basic implementation + unit/integration tests, but not heavy E2E automation.

---

## 1. Core & Multi‑tenant Layer

| Feature | Scope (relative to Cal.com OSS) | Est. days / 1 dev |
|--------|----------------------------------|-------------------|
| **Base Cal.com OSS setup & customization baseline** | Fork, env setup, CI, deploy, configure basic booking & availability on a single tenant; understand codebase | **8–10 days** |
| **Tenant model & multi‑tenant isolation** | New `Tenant` model, add `tenantId` to core entities (User, EventType, Booking, Payment shim), tenant‑scoped queries, Prisma migrations | **12–15 days** |
| **Subdomain / domain routing per tenant** | Resolve tenant by subdomain/custom domain, middleware, error states, SEO‑safe redirects | **5–7 days** |
| **Tenant settings (plans, limits, feature flags)** | Plan enum, per‑tenant limits (teachers, bookings/month, event types), soft enforcement hooks | **7–9 days** |

**Subtotal (core multi‑tenant)**: **32–41 dev‑days**

---

## 2. Teacher & Scheduling

| Feature | Scope | Est. days / 1 dev |
|--------|-------|-------------------|
| **Teacher role & profile** | Map Cal.com `User` to roles (Teacher/Student/TenantAdmin), minimal profile UI | **4–5 days** |
| **Teacher availability (reuse OSS availability)** | Wire existing availability UI to tenant context, minor UX tweaks, ensure it works with multi‑tenant & Google/Outlook sync | **5–7 days** |
| **Teacher assignment to course/event type** | New `TeacherCourseAssignment` model, UI for CS/Tenant admin to pick teacher + schedule, conflict detection (no double‑booking teacher across tenants), reusing Cal.com recurrence where possible | **9–12 days** |

**Subtotal (teacher & scheduling)**: **18–24 dev‑days**

---

## 3. Workshop / Event Types (Pay‑per‑session, whitelist, seats)

| Feature | Scope | Est. days / 1 dev |
|--------|-------|-------------------|
| **Workshop event type extension** | Extend `EventType` with `isWorkshop`, price, `seatLimit`, `refundWindowHours`, recurrence; CRUD UI + validations | **8–10 days** |
| **Seat limitation (group events)** | Reuse Cal.com group booking logic where possible, adapt to multi‑tenant & per‑event seat counts, display remaining seats | **6–8 days** |
| **Whitelist mode (per workshop)** | Whitelist model, CSV upload, per‑event whitelist settings (Immersio integration will populate later), validation in booking flow | **8–10 days** |
| **Public vs whitelist visibility / booking rules** | Config & UI (public, whitelist‑only), error states for non‑whitelisted students | **4–5 days** |

**Subtotal (workshop logic)**: **26–33 dev‑days**

---

## 4. Booking Flow (Student side) – extended

| Feature | Scope | Est. days / 1 dev |
|--------|-------|-------------------|
| **Student booking UI integration** | Use Cal.com OSS booking components with tenant context + Immersio SSO; embed/standalone booking page | **6–8 days** |
| **Booking flow with whitelist & seat checks** | Inject whitelist + seat availability checks into booking creation path, handle error flows, UX messages | **6–8 days** |
| **Notifications (email templates per tenant)** | Reuse Cal.com email service, add tenant branding context + a few custom templates for workshop/payments | **5–7 days** |

**Subtotal (booking UX & flow)**: **17–23 dev‑days**

---

## 5. Payments – No EE, Self‑Implemented

> Cal.com EE handles payments; since we cannot use EE, we must implement our own payment layer using Stripe directly and only reuse minor OSS bits.

| Feature | Scope | Est. days / 1 dev |
|--------|-------|-------------------|
| **Stripe per tenant – credential storage & config UI** | Per‑tenant Stripe key storage (encrypted), settings page, validation ping | **7–9 days** |
| **Checkout & payment flow** | Build new payment service (server‑side), Stripe Checkout or PaymentIntents, link to booking, handle success/cancel, idempotency | **10–14 days** |
| **Multi‑account routing (Stripe account per tenant)** | Decide between Stripe Connect vs separate accounts + API keys; implement selection of correct Stripe client per booking | **6–8 days** |
| **Webhooks per tenant** | Single webhook endpoint multiplexed by tenant, verify signature, update Payment + Booking status, handle retries | **7–9 days** |
| **Refund logic (12h window)** | Refund policy evaluation (event type specific), Stripe refunds, state tracking in Payment, integration with cancellation flow | **6–8 days** |
| **Basic transaction history per tenant** | Simple listing of transactions (tenant scoped, filter by status/date) | **4–5 days** |

**Subtotal (payments, fully custom)**: **40–53 dev‑days**

---

## 6. Cancellation, Refund, Reschedule

> Reuse Cal.com OSS booking status/reschedule logic, combined with our custom payment/refund rules.

| Feature | Scope | Est. days / 1 dev |
|--------|-------|-------------------|
| **Cancellation windows (X hours, default 24h)** | EventType config, enforcement in booking cancel flow, UI warnings, edge cases (after window) | **5–7 days** |
| **Automatic refund on cancel (within window)** | Integrate cancellation with payment service (above), handle partial failures, audit trail | **5–7 days** |
| **Reschedule rules** | Reuse Cal.com reschedule where possible, adapt to seat limits and whitelist; handle re‑payment where needed (usually no extra payment) | **6–8 days** |
| **Audit & history (cancel / refund log)** | Simple logs of who cancelled, when, reason, refund status, exposed in admin UI | **4–5 days** |

**Subtotal (cancel/refund/reschedule)**: **20–27 dev‑days**

---

## 7. Immersio Integration

| Feature | Scope | Est. days / 1 dev |
|--------|-------|-------------------|
| **Immersio API client & config** | HTTP client, error handling, env/tenant credentials, basic monitoring | **5–6 days** |
| **Pull courses → EventTypes** | Map Immersio course → EventType (including workshop mapping), sync job, upsert logic, conflict handling | **7–9 days** |
| **Pull students/teachers → Users** | Map Immersio users/roles to our User/role + whitelist population | **7–9 days** |
| **Push bookings & payments → Immersio** | On booking success, push enrollment & payment info; retry & error log | **6–8 days** |
| **SSO (JWT) between Immersio ↔ Booking Service** | JWT verification, user provision, session creation, tenant scoping | **6–8 days** |

**Subtotal (Immersio integration)**: **31–40 dev‑days**

---

## 8. Landing Pages (Per Tenant)

| Feature | Scope | Est. days / 1 dev |
|--------|-------|-------------------|
| **Template system (5 fixed templates)** | Implement templates as React components, config model, preview | **7–9 days** |
| **Landing settings UI** | Admin form: logo upload, colors, hero image, copy, social links; validation and autosave | **6–8 days** |
| **Public landing routing & SEO** | `tenant.slug` routing, SSR, SEO meta tags, performance tuning | **5–7 days** |
| **Integration with Booking Core** | Show list of workshops/courses with CTA → booking modal/Cal embed | **5–7 days** |

**Subtotal (landing pages)**: **23–31 dev‑days**

---

## 9. Admin & Tenant Management

> Cal.com EE has some Admin/Org features; we need to build lightweight equivalents.

| Feature | Scope | Est. days / 1 dev |
|--------|-------|-------------------|
| **Super Admin dashboard** | Tenant list, status, basic metrics, enable/disable, impersonate/preview | **8–10 days** |
| **Tenant subscription management** | Plan selection (LITE/STARTER/GROWTH/PRO/ENTERPRISE), enforce limits, metrics per tenant | **7–9 days** |
| **Logs & monitoring view (simple)** | Show recent sync errors, payment errors, booking anomalies per tenant | **6–8 days** |

**Subtotal (admin & subscription)**: **21–27 dev‑days**

---

## 10. DevOps, QA & Hardening

| Feature | Scope | Est. days / 1 dev |
|--------|-------|-------------------|
| **Env, CI/CD, staging & prod** | Pipelines, DB migrations automation, secrets management | **7–9 days** |
| **Observability & monitoring** | Basic metrics, logs, alerts on failures (payments, sync, high error rate) | **5–7 days** |
| **Test suite (critical paths)** | Unit + integration for core flows (booking, payment, sync); a few E2E happy‑paths | **10–15 days** |
| **Performance passes & security hardening** | Load tests for booking/payment flows, basic security review | **7–10 days** |

**Subtotal (DevOps & hardening)**: **29–41 dev‑days**

---

## 11. Overall Re‑estimation (Single Developer)

Summing lower and upper bounds:

- Core multi‑tenant: **32–41**
- Teacher & scheduling: **18–24**
- Workshops & whitelist: **26–33**
- Booking flow UX: **17–23**
- Payments (fully custom): **40–53**
- Cancel / Refund / Reschedule: **20–27**
- Immersio integration: **31–40**
- Landing pages: **23–31**
- Admin & subscription: **21–27**
- DevOps & hardening: **29–41**

**Total (1 dev)** ≈ **257–340 dev‑days**

At 5 working days/week:

- **Lower bound**: 257 / 5 ≈ **51 weeks (~12 months)**
- **Upper bound**: 340 / 5 ≈ **68 weeks (~16 months)**

With **3–4 engineers** in parallel, this lines up with a **5–7 month** calendar timeline as previously discussed.
