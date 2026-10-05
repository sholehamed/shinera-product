# Shinera — Product Backlog

**Product:** Shinera  
**Document Role:** Execution Source of Truth  
**Backlog Model:** Epic → Story → Tasks → Acceptance Criteria → Priority

---

## 1. Goal

این Backlog برای رساندن Shinera از وضعیت فعلی به یک MVP قابل استفاده توسط یک سالن واقعی طراحی شده است.

Golden path:

```text
Register Business
      ↓
Create Workspace
      ↓
Configure Business
      ↓
Create Services
      ↓
Create Staff
      ↓
Configure Schedules
      ↓
Create Customers
      ↓
Create Appointment
      ↓
Manage Appointment
      ↓
Complete Appointment
      ↓
Register Payment
      ↓
View Dashboard
```

---

## 2. Priority Definition

| Priority | Meaning |
|---|---|
| P0 | بدون آن MVP قابل استفاده نیست |
| P1 | برای Launch بسیار مهم است |
| P2 | بعد از MVP |
| P3 | Future / Nice to Have |

---

## 3. Releases

### MVP Core

- Authentication
- Tenant
- Branch
- Registration
- Business Profile
- Services
- Staff
- Schedule
- Customers
- Appointments
- Payments
- Permissions
- Basic Subscription
- Dashboard

### MVP Launch

- Public Booking
- Notifications
- Better Onboarding
- Audit UI
- Production Monitoring
- Subscription Billing

### Release 1.1

- VIP
- Force Appointment
- Online Payment
- Advanced Notifications
- Advanced Reports

---

# EPIC 01 — Platform Foundation

**ID:** SHN-E01  
**Priority:** P0

## SHN-001 — Solution Architecture

### User Story

به عنوان توسعه‌دهنده می‌خواهم ساختار پروژه استاندارد باشد تا Featureها مستقل و قابل نگهداری توسعه داده شوند.

### Backend

- Domain
- Application
- Infrastructure
- API
- Application.Tests
- IntegrationTests

### Architecture

```text
Modular Monolith
Vertical Slice
CQRS
Minimal API
```

### Acceptance Criteria

- Domain به Infrastructure وابسته نباشد
- Application به API وابسته نباشد
- Infrastructure implementationهای Application را ارائه دهد
- API Composition Root باشد

---

## SHN-002 — Result Pattern

**Priority:** P0

ایجاد:

```text
Result
Result<T>
Error
ApiResponse
ApiResponse<T>
```

### Acceptance Criteria

Endpointها برای خطاهای Business از Exception استفاده نکنند.

---

## SHN-003 — Global Exception Handling

**Priority:** P0

پشتیبانی از:

- Validation
- NotFound
- Unauthorized
- Forbidden
- Conflict
- Unexpected Error

نمونه خروجی:

```json
{
  "success": false,
  "error": {
    "code": "appointment.conflict",
    "message": "..."
  }
}
```

---

## SHN-004 — Validation Pipeline

**Priority:** P0

- Required
- Length
- Range
- Business Validation

---

## SHN-005 — Database Infrastructure

**Priority:** P0

- EF Core
- Migrations
- Entity configurations
- SaveChanges abstraction
- Transaction support

---

## SHN-006 — Audit Fields

Entityهای لازم:

```text
CreatedAt
CreatedBy
UpdatedAt
UpdatedBy
```

در صورت نیاز:

```text
DeletedAt
DeletedBy
```

---

## SHN-007 — Soft Delete

**Priority:** P1

برای:

- Customer
- Service
- Staff
- Branch

Global Query Filter اعمال شود.

---

# EPIC 02 — Authentication & Identity

**ID:** SHN-E02  
**Priority:** P0

## SHN-010 — User Entity

```text
Id
FirstName
LastName
Phone
Email
PasswordHash
IsActive
LastLoginAt
```

---

## SHN-011 — OpenIddict Integration

- Access token
- Refresh token
- Token revocation
- Authorization infrastructure

---

## SHN-012 — Login

### Backend

```text
POST /auth/login
```

### Frontend

Login Page

### Acceptance Criteria

- User غیرفعال نتواند وارد شود
- Credential اشتباه خطای مناسب برگرداند
- Access token ایجاد شود
- Refresh token ایجاد شود

---

## SHN-013 — Refresh Token

### Acceptance Criteria

- Token معتبر refresh شود
- Token revoked پذیرفته نشود
- Refresh token rotation انجام شود

---

## SHN-014 — Logout

Refresh token جاری revoke شود.

---

## SHN-015 — Current User

```text
GET /auth/me
```

خروجی:

```text
User
Tenant
Branch
Roles
Permissions
```

---

# EPIC 03 — Tenant & Workspace

**ID:** SHN-E03  
**Priority:** P0

## SHN-020 — Tenant Entity

```text
Id
Name
Slug
Status
IsActive
CreatedAt
```

---

## SHN-021 — Create Tenant

### Backend

```text
CreateTenantCommand
CreateTenantHandler
```

### Acceptance Criteria

- Name required
- Tenant unique identity داشته باشد
- Tenant Active ایجاد شود

---

## SHN-022 — Update Tenant

```text
PUT /api/tenants/{id}
```

---

## SHN-023 — Tenant Status

فعال/غیرفعال کردن Tenant.

---

## SHN-024 — Current Tenant Context

ایجاد:

```text
ICurrentTenant
```

Context:

```text
TenantId
BranchId?
UserId
```

---

## SHN-025 — Tenant Query Isolation

**Critical Security Story**

### Acceptance Criteria

هیچ Query مربوط به Tenant A نباید اطلاعات Tenant B را برگرداند.

### Tests

Integration Test الزامی.

---

# EPIC 04 — Branch Management

**ID:** SHN-E04  
**Priority:** P0

## SHN-030 — Branch Entity

```text
Id
TenantId
Name
Phone
Address
IsMain
IsActive
```

---

## SHN-031 — Main Branch

هنگام ایجاد Tenant یک Main Branch ساخته شود.

---

## SHN-032 — Branch CRUD

### Backend

- List
- Details
- Create
- Update
- Disable

### Frontend

Branch Management Page

---

## SHN-033 — Set Main Branch

در هر Tenant فقط یک Main Branch مجاز است.

---

## SHN-034 — Current Branch

کاربر بتواند Branch جاری را تغییر دهد.

---

# EPIC 05 — Registration

**ID:** SHN-E05  
**Priority:** P0

## SHN-040 — Registration Wizard

```text
Plan
 ↓
Business
 ↓
Owner
 ↓
Branch
 ↓
Review
 ↓
Subscription
 ↓
Workspace
```

---

## SHN-041 — Plan Selection

```text
/register?plan=salon-pro
```

---

## SHN-042 — Business Registration

- Business Name
- Type
- Phone
- Address
- Mode

---

## SHN-043 — Owner Registration

- First Name
- Last Name
- Phone
- Email
- Password

---

## SHN-044 — Registration Transaction

عملیات باید Atomic باشد:

```text
Create User
Create Tenant
Create BusinessProfile
Create Main Branch
Create Membership
Create Subscription
Assign Owner Role
```

در صورت شکست هر مرحله Rollback شود.

---

## SHN-045 — Registration Success

```text
Auto Login
    ↓
Onboarding
```

---

# EPIC 06 — Business Profile

**ID:** SHN-E06  
**Priority:** P0

## SHN-050 — Business Profile Entity

```text
TenantId
DisplayName
BusinessType
Phone
Email
Address
Logo
Description
```

---

## SHN-051 — Business Profile Update

Owner بتواند اطلاعات کسب‌وکار را ویرایش کند.

---

## SHN-052 — Business Logo

**Priority:** P1

آپلود Logo.

---

# EPIC 07 — Plans & Subscription

**ID:** SHN-E07  
**Priority:** P0

## SHN-060 — Plan Entity

```text
Id
Key
Name
Description
Price
BillingPeriod
IsActive
```

---

## SHN-061 — Feature Entity

```text
Id
Key
Name
Description
```

نمونه:

```text
multiple-branches
advanced-reports
force-appointment
vip-customers
```

---

## SHN-062 — Plan Feature

```text
Plan
 ↕
PlanFeature
 ↕
Feature
```

---

## SHN-063 — Subscription

```text
TenantId
PlanId
StartDate
EndDate
Status
```

Status:

```text
Trial
Active
Expired
Cancelled
Suspended
```

---

## SHN-064 — Feature Gating

- Backend باید Feature را enforce کند
- Frontend باید UI مربوط را hide/disable کند
- Frontend به تنهایی security boundary نیست

---

# EPIC 08 — Authorization

**ID:** SHN-E08  
**Priority:** P0

## SHN-070 — Permission Catalog

```text
services.view
services.create
services.update
services.delete

customers.view
customers.create
customers.update

staff.view
staff.manage

appointments.view
appointments.create
appointments.update
appointments.cancel

payments.view
payments.create

reports.view

settings.manage
```

---

## SHN-071 — Roles

```text
Owner
Admin
Receptionist
Staff
```

---

## SHN-072 — Permission Scope

```text
Tenant
Branch
Own
```

Child برای بعد از MVP.

---

## SHN-073 — Endpoint Authorization

Authorization بر اساس:

```text
Resource
Action
Scope
```

---

## SHN-074 — Frontend Permission Guard/Directive

Angular باید امکان کنترل UI بر اساس Permission را داشته باشد.

---

# EPIC 09 — Service Categories

**ID:** SHN-E09  
**Priority:** P0

## SHN-080 — Service Category CRUD

```text
Id
TenantId
Name
Description
SortOrder
IsActive
```

---

## SHN-081 — Category Ordering

**Priority:** P1

امکان مرتب‌سازی دسته‌ها.

---

# EPIC 10 — Services

**ID:** SHN-E10  
**Priority:** P0

## SHN-090 — Service Entity

```text
Id
TenantId
CategoryId
Name
Description
DurationMinutes
Price
IsActive
```

---

## SHN-091 — Create Service

### Acceptance Criteria

- Name required
- Duration > 0
- Price >= 0
- Category معتبر باشد
- Tenant isolation رعایت شود

---

## SHN-092 — Update Service

---

## SHN-093 — Activate / Deactivate Service

Service غیرفعال:

- برای Appointment جدید قابل انتخاب نباشد
- Appointmentهای قبلی حفظ شوند

---

## SHN-094 — Service List

Filters:

```text
Search
Category
Status
```

---

## SHN-095 — Service Details

نمایش:

- اطلاعات اصلی
- Staffهای ارائه‌دهنده
- تعداد Appointment
- درآمد پایه

Revenue statistics در P1.

---

# EPIC 11 — Staff

**ID:** SHN-E11  
**Priority:** P0

## SHN-100 — Staff Entity

```text
Id
TenantId
UserId?
FirstName
LastName
Phone
Email
IsActive
```

---

## SHN-101 — Create Staff

Staff الزاماً در ابتدا User Login نباشد.

---

## SHN-102 — Assign Staff to Branch

```text
StaffBranch
```

---

## SHN-103 — Assign Services to Staff

```text
StaffService
```

---

## SHN-104 — Staff List

Filters:

- Search
- Branch
- Service
- Status

---

## SHN-105 — Staff Profile

نمایش:

- اطلاعات
- شعب
- خدمات
- Schedule
- Appointments

---

# EPIC 12 — Staff Schedule

**ID:** SHN-E12  
**Priority:** P0

## SHN-110 — Weekly Schedule

مثال:

```text
Saturday
09:00
18:00
```

---

## SHN-111 — Schedule Break

```text
13:00 - 14:00
```

---

## SHN-112 — Day Off

امکان غیرفعال کردن روز کاری.

---

## SHN-113 — Special Schedule

**Priority:** P1

```text
2026-10-15
10:00 - 14:00
```

---

## SHN-114 — Time Off

**Priority:** P1

- Vacation
- Personal Leave
- Sick Leave

---

# EPIC 13 — Customer Management

**ID:** SHN-E13  
**Priority:** P0

## SHN-120 — Customer Entity

```text
Id
TenantId
FirstName
LastName
Mobile
Email
Birthday
Gender
Notes
IsVip
```

VIP در MVP فقط Field است و Business Logic بعداً اضافه می‌شود.

---

## SHN-121 — Create Customer

### Acceptance Criteria

- Mobile قابل جستجو باشد
- Duplicate detection انجام شود
- Tenant isolation رعایت شود

---

## SHN-122 — Customer Search

```text
Name
Mobile
Email
```

---

## SHN-123 — Customer Details

نمایش:

- Profile
- Appointment History
- Payment History
- Notes

---

## SHN-124 — Customer Notes

Reception / Staff مجاز بتواند Note ثبت کند.

---

## SHN-125 — Customer Activity Timeline

**Priority:** P1

---

# EPIC 14 — Appointment Core

**ID:** SHN-E14  
**Priority:** P0 — Highest Priority

## SHN-130 — Appointment Entity

```text
Id
TenantId
BranchId
CustomerId
StaffId
ServiceId
Date
StartTime
EndTime
Price
Status
Notes
```

---

## SHN-131 — Appointment Status

```text
Pending
Confirmed
Upcoming
InProgress
Completed
Cancelled
NoShow
```

State Transition Rules باید تعریف شوند.

---

## SHN-132 — Create Appointment

### Backend

```text
CreateAppointmentCommand
CreateAppointmentHandler
```

### Validation

سیستم باید بررسی کند:

- Customer معتبر
- Branch معتبر
- Staff معتبر
- Service معتبر
- Staff در Branch فعال
- Staff Service را ارائه می‌دهد
- Staff در آن ساعت Schedule دارد
- Time Off ندارد
- Appointment overlap وجود ندارد

---

## SHN-133 — Available Slots

```text
GET /appointments/available-slots
```

Input:

```text
BranchId
ServiceId
StaffId?
Date
```

Output:

```text
09:00
09:30
10:30
...
```

---

## SHN-134 — Appointment Conflict Detection

Logic باید در Backend enforce شود.

### Tests

```text
Same Staff + overlapping time → reject
Different Staff + same time → allow
Same Staff + adjacent time → allow
Cancelled appointment overlap → allow
```

---

## SHN-135 — Appointment Details

نمایش:

- Customer
- Staff
- Service
- Branch
- Date
- Time
- Price
- Status
- Notes

---

## SHN-136 — Appointment List

Filters:

```text
Date
Date Range
Status
Staff
Branch
Service
Customer
```

---

## SHN-137 — Calendar View

Frontend:

- Day
- Week

Month View در P1.

---

## SHN-138 — Reschedule Appointment

### Acceptance Criteria

تمام Availability Validation دوباره اجرا شود.

---

## SHN-139 — Cancel Appointment

```text
CancellationReason
CancelledAt
CancelledBy
```

---

## SHN-140 — Start Appointment

```text
Confirmed / Upcoming
        ↓
InProgress
```

---

## SHN-141 — Complete Appointment

```text
InProgress
   ↓
Completed
```

---

## SHN-142 — Mark No Show

Reception / Admin بتواند Appointment را NoShow کند.

---

# EPIC 15 — Appointment State Machine

**ID:** SHN-E15  
**Priority:** P0

Transitions:

```text
Pending
 ↓
Confirmed
 ↓
InProgress
 ↓
Completed
```

Alternative:

```text
Pending → Cancelled
Confirmed → Cancelled
Confirmed → NoShow
```

Transition غیرمجاز:

```text
Completed → InProgress
```

---

# EPIC 16 — Payments

**ID:** SHN-E16  
**Priority:** P0

## SHN-150 — Payment Entity

```text
Id
TenantId
AppointmentId
Amount
Method
Status
PaidAt
Reference
```

---

## SHN-151 — Payment Method

```text
Cash
Card
Online
Other
```

---

## SHN-152 — Payment Status

```text
Pending
Paid
PartiallyPaid
Refunded
Cancelled
```

---

## SHN-153 — Register Payment

Reception / Owner بتواند پرداخت ثبت کند.

---

## SHN-154 — Partial Payment

**Priority:** P1

---

## SHN-155 — Payment History

در Appointment و Customer قابل مشاهده باشد.

---

# EPIC 17 — Dashboard

**ID:** SHN-E17  
**Priority:** P0

## SHN-160 — Today Appointments KPI

- Total
- Upcoming
- Completed
- Cancelled

---

## SHN-161 — Revenue KPI

```text
Today
This Week
This Month
```

---

## SHN-162 — New Customers KPI

---

## SHN-163 — Recent Appointments

فیلتر بر اساس تاریخ.

---

## SHN-164 — Revenue by Service

**Priority:** P1

---

## SHN-165 — Featured Services

**Priority:** P1

Metrics:

- Appointment Count
- Revenue
- Growth

---

## SHN-166 — Dashboard Filters

```text
Branch
Date Range
```

---

# EPIC 18 — Onboarding

**ID:** SHN-E18  
**Priority:** P1

## SHN-170 — Onboarding Checklist

```text
Complete Business Profile
Create Service
Add Staff
Configure Schedule
Create Customer
Create Appointment
```

---

## SHN-171 — Onboarding Progress

```text
3 / 6 completed
```

---

## SHN-172 — Empty State Navigation

مثال:

```text
هنوز خدمتی ایجاد نکرده‌اید
[ایجاد اولین خدمت]
```

---

# EPIC 19 — Public Booking

**ID:** SHN-E19  
**Priority:** P1

## SHN-180 — Public Business Page

```text
/book/{tenantSlug}
```

---

## SHN-181 — Select Branch

اگر فقط یک Branch وجود دارد، مرحله Skip شود.

---

## SHN-182 — Select Service

فقط Serviceهای فعال.

---

## SHN-183 — Select Staff

گزینه:

```text
Any Available Staff
```

نیز وجود داشته باشد.

---

## SHN-184 — Public Available Slots

از همان Availability Engine اصلی استفاده شود.

---

## SHN-185 — Identify Customer

با Mobile.

اگر موجود بود Customer فعلی استفاده شود، در غیر این صورت Customer ساخته شود.

---

## SHN-186 — Public Appointment Creation

Rate Limiting الزامی.

---

## SHN-187 — Booking Confirmation

نمایش:

- Service
- Staff
- Date
- Time
- Branch

---

# EPIC 20 — Notifications

**ID:** SHN-E20  
**Priority:** P1

## SHN-190 — Notification Abstraction

```text
INotificationSender
```

---

## SHN-191 — Appointment Created Notification

---

## SHN-192 — Appointment Reminder

مثال:

24 ساعت قبل.

---

## SHN-193 — Appointment Cancelled Notification

---

# EPIC 21 — Audit Trail

**ID:** SHN-E21  
**Priority:** P1

## SHN-200 — Audit Log

```text
Tenant
Branch
User
Action
Entity
EntityId
Timestamp
```

---

## SHN-201 — Appointment Audit

ثبت:

- Create
- Reschedule
- Cancel
- Complete
- NoShow

---

## SHN-202 — Audit Viewer

**Priority:** P2

---

# EPIC 22 — Localization

**ID:** SHN-E22  
**Priority:** P0

## SHN-210 — RTL

تمام UI فارسی RTL.

---

## SHN-211 — Persian Date Display

Storage:

```text
Gregorian
UTC
```

UI:

```text
fa-IR
Persian Calendar
```

---

## SHN-212 — Money Formatting

Abstraction برای IRR / Toman.

---

# EPIC 23 — Search & Pagination

**ID:** SHN-E23  
**Priority:** P0

تمام List Endpointها:

```text
Page
PageSize
Search
Sort
Filters
```

Response:

```text
Items
TotalCount
Page
PageSize
```

---

# EPIC 24 — Observability

**ID:** SHN-E24  
**Priority:** P1

## SHN-220 — Structured Logging

```text
TraceId
TenantId
UserId
RequestPath
```

---

## SHN-221 — Request Tracing

---

## SHN-222 — Exception Monitoring

---

## SHN-223 — Health Check

```text
/health/live
/health/ready
```

---

# EPIC 25 — Security

**ID:** SHN-E25  
**Priority:** P0

## SHN-230 — Rate Limiting

برای:

- Login
- Registration
- Public Booking

---

## SHN-231 — Tenant Injection Protection

TenantId از Request Body قابل اعتماد نباشد.

از CurrentTenant گرفته شود.

---

## SHN-232 — Branch Access Validation

داشتن Tenant Access به تنهایی برای Branch کافی نیست.

---

## SHN-233 — Input Sanitization

---

## SHN-234 — Security Headers

---

# EPIC 26 — Testing

**ID:** SHN-E26  
**Priority:** P0

## SHN-240 — Domain Tests

برای:

- Appointment Transitions
- Availability
- Schedule
- Subscription

---

## SHN-241 — Application Tests

برای Command / Query Handlerها.

---

## SHN-242 — Integration Tests

حداقل برای:

```text
Authentication
Tenant Isolation
Create Service
Create Customer
Create Appointment
Appointment Conflict
Cancel Appointment
Complete Appointment
Register Payment
```

---

## SHN-243 — E2E Playwright

Critical Journey:

```text
Login
 ↓
Create Service
 ↓
Create Staff
 ↓
Create Customer
 ↓
Create Appointment
 ↓
Complete Appointment
```

---

## SHN-244 — Registration E2E

```text
Landing
 ↓
Select Plan
 ↓
Register
 ↓
Workspace
 ↓
Dashboard
```

---

# EPIC 27 — CI/CD

**ID:** SHN-E27  
**Priority:** P1

Pipeline:

```text
Restore
Build
Unit Tests
Integration Tests
Frontend Build
E2E
Publish
```

---

# 4. MVP Exit Criteria

MVP آماده Release نیست مگر اینکه:

```text
[ ] Tenant isolation tests pass
[ ] User can register
[ ] Workspace is created
[ ] Owner can login
[ ] Service can be created
[ ] Staff can be created
[ ] Staff can be assigned to service
[ ] Schedule can be configured
[ ] Customer can be created
[ ] Appointment can be created
[ ] Appointment conflict is prevented
[ ] Appointment can be rescheduled
[ ] Appointment can be cancelled
[ ] Appointment can be completed
[ ] NoShow can be registered
[ ] Payment can be registered
[ ] Dashboard shows real data
[ ] Permissions are enforced by backend
[ ] Subscription features are enforced
[ ] Persian RTL UI works
[ ] Critical Integration Tests pass
[ ] Critical Playwright Tests pass
```

---

# 5. Post-MVP — Release 1.1

# EPIC 28 — VIP Customers

## SHN-300 — VIP Status

## SHN-301 — VIP Rules

## SHN-302 — VIP Badge

---

# EPIC 29 — Force Appointment

## SHN-310 — Force Appointment Request

فقط VIP Customer.

```text
No Slot
 ↓
Request Force Appointment
 ↓
Preferred Date / Time
 ↓
Salon Review
```

---

## SHN-311 — Review Force Request

Salon:

```text
Accept
Reject
Suggest Alternative
```

---

## SHN-312 — Force Appointment Pricing

**Priority:** P2

```text
Normal Price
+
Force Fee
```

---

# EPIC 30 — Loyalty

**Priority:** P2

- Points
- Level
- Reward
- Customer Tier

---

# EPIC 31 — Advanced Reports

**Priority:** P2

```text
Revenue by Staff
Revenue by Service
Revenue by Branch
Cancellation Rate
NoShow Rate
Customer Retention
Repeat Customer Rate
Staff Utilization
```

---

# EPIC 32 — Packages & Discounts

**Priority:** P2

- Service Package
- Coupon
- Discount
- Promotional Pricing

---

# 6. Suggested Development Order

```text
01 Foundation
02 Authentication
03 Tenant
04 Branch
05 Permission
06 Plan / Feature / Subscription
07 Registration
08 Business Profile
09 Service Category
10 Service
11 Staff
12 Staff Service
13 Schedule
14 Customer
15 Appointment Model
16 Availability Engine
17 Appointment Creation
18 Appointment Management
19 Payment
20 Dashboard
21 Onboarding
22 Public Booking
23 Notifications
24 Hardening
25 MVP Release
```

---

# 7. Suggested Sprint Structure

## Sprint 1

```text
Authentication
Tenant
Branch
Current Tenant
Tenant Isolation
```

## Sprint 2

```text
Permissions
Plans
Features
Subscription
Registration
```

## Sprint 3

```text
Business Profile
Service Categories
Services
```

## Sprint 4

```text
Staff
Staff Branch
Staff Service
Schedule
```

## Sprint 5

```text
Customers
Customer Details
Customer History
```

## Sprint 6

```text
Appointment Domain
Availability Engine
Conflict Detection
```

## Sprint 7

```text
Appointment UI
Calendar
Reschedule
Cancel
Status Workflow
```

## Sprint 8

```text
Payments
Dashboard
KPIs
```

## Sprint 9

```text
Public Booking
Onboarding
Notifications
```

## Sprint 10

```text
Bug Fix
Integration Tests
Playwright
Performance
Security
Production Hardening
```

---

# 8. Definition of Done

هیچ Story نباید Done شود مگر اینکه:

```text
✓ Business Rule implemented
✓ Backend implemented
✓ Frontend implemented if required
✓ Authorization implemented
✓ Validation implemented
✓ Tenant isolation checked
✓ Database migration added if required
✓ Application tests added
✓ Integration tests added for critical paths
✓ Error states handled in UI
✓ Empty states handled
✓ Loading states handled
✓ RTL checked
✓ Dark mode checked
✓ API documentation updated
✓ No known P0/P1 bug remains
```

---

# 9. Feature Implementation Checklist

```text
FEATURE
│
├── Domain
│   ├── Entity
│   ├── Value Objects
│   └── Business Rules
│
├── Database
│   ├── Configuration
│   ├── Index
│   └── Migration
│
├── Backend
│   ├── Command
│   ├── Query
│   ├── Handler
│   ├── Validator
│   └── Endpoint
│
├── Authorization
│   ├── Permission
│   └── Feature Gate
│
├── Frontend
│   ├── Route
│   ├── Page
│   ├── Components
│   ├── Form
│   ├── Loading
│   ├── Empty State
│   └── Error State
│
└── Testing
    ├── Unit
    ├── Application
    ├── Integration
    └── E2E
```

---

# 10. Highest-Risk Areas

### 1. Tenant Isolation

خطا در این قسمت Security Incident محسوب می‌شود.

### 2. Appointment Availability

هسته اصلی Business شاینراست.

### 3. Appointment Concurrency

دو درخواست همزمان نباید بتوانند یک Slot را رزرو کنند.

### 4. Permission Scope

Staff نباید داده خارج از Scope خود را ببیند.

### 5. Date / Time

ذخیره‌سازی و نمایش تاریخ نباید با تقویم شمسی مخلوط شود.

### 6. Subscription Feature Gating

Feature فقط در Frontend مخفی نشود؛ Backend نیز باید آن را enforce کند.

---

# 11. MVP Golden Path

```text
Owner opens Shinera

→ Selects Salon Plan

→ Registers

→ Tenant is created

→ Main Branch is created

→ Owner enters Dashboard

→ Creates "Haircut" service

→ Creates Staff "Sara"

→ Assigns Haircut to Sara

→ Configures Sara:
   Saturday 09:00 - 18:00

→ Creates Customer "Neda"

→ Selects Haircut

→ Selects Sara

→ Selects Saturday

→ Shinera calculates available slots

→ Owner selects 10:00

→ Appointment is created

→ 10:00 becomes unavailable

→ Appointment starts

→ Appointment completes

→ Payment is registered

→ Dashboard revenue updates
```

اگر این Scenario با UX خوب، بدون خطای Business و با تست کامل اجرا شود، هسته MVP Shinera آماده است.
