# Shinera — Product Requirements Document (PRD)

**Product:** Shinera  
**Category:** Beauty Business Management SaaS  
**Target:** سالن‌های زیبایی، آرایشگاه‌ها، متخصصان مستقل و کسب‌وکارهای خدمات زیبایی  
**Platform:** Web-first SaaS  
**Business Model:** Subscription SaaS  
**Document Role:** Product Source of Truth

---

## 1. Product Vision

Shinera یک پلتفرم SaaS برای مدیریت کامل کسب‌وکارهای حوزه زیبایی است.

هدف Shinera این است که صاحب کسب‌وکار بتواند از یک سیستم واحد بخش‌های اصلی کسب‌وکار خود را مدیریت کند:

- نوبت‌دهی
- مشتریان
- خدمات
- پرسنل
- شیفت و زمان کاری
- پرداخت‌ها
- شعب
- گزارش‌ها و KPIها
- اشتراک
- کاربران و دسترسی‌ها
- ارتباط با مشتری

Shinera باید برای دو مدل اصلی کسب‌وکار مناسب باشد:

### Solo

متخصص مستقلی که به تنهایی فعالیت می‌کند، مانند:

- Nail Artist
- Hair Stylist
- Makeup Artist
- Barber
- Lash Artist

### Salon

کسب‌وکاری دارای چند نیروی متخصص و در صورت نیاز چند شعبه، مانند:

- سالن زیبایی
- مجموعه آرایشگاهی
- کلینیک خدمات زیبایی سبک
- مجموعه Nail / Lash / Skin

---

## 2. Problem Statement

مدیریت بسیاری از کسب‌وکارهای زیبایی هنوز با ترکیبی از ابزارهای پراکنده انجام می‌شود:

- دفتر نوبت
- پیام‌رسان
- Excel
- تماس تلفنی
- کارت بانکی
- نرم‌افزار حسابداری جدا
- یادداشت‌های شخصی
- شبکه‌های اجتماعی

این وضعیت باعث مشکلاتی مثل موارد زیر می‌شود:

- تداخل نوبت‌ها
- فراموش شدن نوبت‌ها
- نبود تاریخچه منسجم مشتری
- مشخص نبودن ظرفیت پرسنل
- دشواری محاسبه درآمد
- دشواری مدیریت چند شعبه
- نبود گزارش مدیریتی
- از دست رفتن مشتریان
- سخت بودن پیگیری پرداخت
- نبود سیستم وفاداری و VIP

Shinera این اطلاعات و عملیات را در یک سیستم متمرکز می‌کند.

---

## 3. Product Goals

اهداف اصلی محصول:

1. کاهش زمان مدیریت روزانه سالن
2. کاهش خطای نوبت‌دهی
3. افزایش استفاده بهینه از ظرفیت پرسنل
4. ایجاد بانک اطلاعات مشتری
5. ایجاد تجربه حرفه‌ای برای مشتری
6. ایجاد دید مدیریتی نسبت به عملکرد کسب‌وکار
7. فراهم کردن مسیر رشد از Solo به Salon
8. ایجاد درآمد SaaS تکرارشونده
9. فراهم کردن زیرساخت Feature-based Subscription
10. فراهم کردن پایه مناسب برای اتوماسیون‌های آینده

---

## 4. Non-Goals

در نسخه‌های اولیه Shinera قرار نیست تبدیل شود به:

- سیستم حسابداری کامل
- ERP
- سیستم حقوق و دستمزد
- Marketplace بزرگ خدمات زیبایی
- CRM سازمانی پیچیده
- نرم‌افزار پزشکی
- پلتفرم تبلیغات
- سیستم انبارداری پیچیده

این قابلیت‌ها در صورت نیاز می‌توانند در آینده یا از طریق Integration اضافه شوند.

---

## 5. Target Personas

### 5.1 Solo Professional

نیازها:

- مدیریت نوبت
- مدیریت مشتری
- مدیریت خدمات
- مدیریت درآمد
- برنامه کاری
- مشاهده برنامه روزانه

### 5.2 Salon Owner

نیازها:

- مشاهده کل کسب‌وکار
- مدیریت کارکنان
- مدیریت شعب
- مدیریت خدمات
- گزارش درآمد
- مدیریت دسترسی‌ها
- مشاهده نوبت‌ها
- مشاهده عملکرد پرسنل

### 5.3 Staff / Specialist

نیازها:

- مشاهده نوبت‌های خود
- مشاهده برنامه کاری
- مشاهده مشتری
- شروع و تکمیل نوبت
- ثبت وضعیت خدمت

### 5.4 Receptionist

نیازها:

- ثبت نوبت
- جابه‌جایی نوبت
- ثبت مشتری
- مشاهده برنامه کارکنان
- لغو نوبت
- مدیریت پرداخت اولیه

### 5.5 Customer

نیازها:

- مشاهده خدمات
- انتخاب متخصص
- مشاهده زمان‌های آزاد
- رزرو نوبت
- مشاهده نوبت
- لغو یا تغییر نوبت
- دریافت اطلاع‌رسانی

---

## 6. Business Model

Shinera بر اساس Subscription فعالیت می‌کند.

پلن‌های پایه:

- Solo
- Solo Pro
- Salon
- Salon Pro

Featureها بر اساس Plan فعال یا غیرفعال می‌شوند.

---

## 7. Multi-Tenancy Model

هر کسب‌وکار در Shinera یک Tenant است.

```text
Tenant
 ├── Business Profile
 ├── Branches
 ├── Users
 ├── Staff
 ├── Services
 ├── Customers
 ├── Appointments
 └── Subscription
```

در زمان ثبت‌نام:

```text
Register
   ↓
Create Tenant
   ↓
Create Business Profile
   ↓
Create Main Branch
   ↓
Create Owner User
   ↓
Create Subscription
```

حتی Solo نیز Tenant دارد تا در آینده بدون بازطراحی داده به Salon تبدیل شود.

---

## 8. Authentication

قابلیت‌های پایه:

- Login
- Logout
- Refresh Token
- Token Revocation
- Password Reset
- Session Management

Authentication بر پایه OpenIddict انجام می‌شود.

---

## 9. Registration

ثبت‌نام به صورت Wizard انجام می‌شود.

```text
Choose Plan
   ↓
Business Information
   ↓
Owner Information
   ↓
Branch Information
   ↓
Review
   ↓
Payment / Subscription
   ↓
Workspace Creation
```

اطلاعات اصلی:

### Business
- Business Name
- Business Type
- Phone
- City
- Address
- Business Mode

### Owner
- First Name
- Last Name
- Mobile
- Email
- Password

### Branch
- Branch Name
- Address
- Phone

---

## 10. Workspace

پس از ورود:

```text
Login
 ↓
Tenant Selection
 ↓
Branch Selection
 ↓
Dashboard
```

اگر کاربر فقط یک Tenant داشته باشد، Tenant Selection حذف می‌شود.

اگر Tenant فقط یک Branch داشته باشد، Branch Selection نیز حذف می‌شود.

---

## 11. Dashboard

Dashboard باید نمای مدیریتی سریع و داده‌محور ارائه دهد.

KPIهای اصلی:

- Today's Appointments
- Today's Revenue
- New Customers
- Completed Appointments
- Cancelled Appointments
- Upcoming Appointments
- Customer Satisfaction
- Popular Services

Widgets اصلی:

- Recent Appointments
- Revenue by Services
- Featured Services
- Customer Acquisition Channels
- Revenue Overview

---

## 12. Services Management

ساختار:

```text
Service Category
      ↓
Service
```

Service شامل:

- Name
- Category
- Duration
- Price
- Description
- Active Status

قابلیت‌های آینده:

- Branch-specific Price
- Staff-specific Price
- Add-ons
- Packages
- Discounts

---

## 13. Staff Management

Staff شامل:

- Name
- Contact
- Role
- Skills
- Branches
- Services
- Work Schedule
- Status

هر Staff می‌تواند:

- در چند Branch فعالیت کند
- چند Service ارائه دهد
- برنامه کاری مستقل داشته باشد

---

## 14. Staff Schedule

Schedule شامل:

- Day
- Start Time
- End Time
- Breaks
- Days Off
- Vacation
- Special Schedule

Appointment فقط در بازه معتبر قابل ثبت است.

---

## 15. Customer Management

Customer شامل:

- Name
- Mobile
- Email
- Birthday
- Gender
- Notes
- Tags
- VIP Status

Customer Profile:

```text
Customer
 ├── Appointments
 ├── Payments
 ├── Services
 ├── Notes
 └── Activity History
```

---

## 16. Appointment Management

Appointment مهم‌ترین بخش MVP است.

Appointment شامل:

- Customer
- Service
- Staff
- Branch
- Date
- StartTime
- EndTime
- Status
- Price
- Notes

Statusها:

```text
Pending
Confirmed
Upcoming
InProgress
Completed
Cancelled
NoShow
```

---

## 17. Appointment Creation Flow

### Staff/Admin Flow

```text
Create Appointment
       ↓
Select Customer
       ↓
Select Service
       ↓
Select Staff
       ↓
Select Date
       ↓
Find Available Slots
       ↓
Select Slot
       ↓
Confirm
       ↓
Appointment Created
```

سیستم باید قبل از ثبت بررسی کند:

- Staff فعال است
- Service فعال است
- Staff آن Service را ارائه می‌دهد
- Staff در Branch فعالیت دارد
- Staff Schedule معتبر است
- Time Slot آزاد است
- Appointment Overlap وجود ندارد

---

## 18. Customer Booking Flow

```text
Customer
   ↓
Select Business
   ↓
Select Branch
   ↓
Select Service
   ↓
Select Specialist
   ↓
Select Date
   ↓
Select Available Time
   ↓
Enter / Confirm Information
   ↓
Confirm Booking
   ↓
Appointment Created
```

Payment یا Deposit می‌تواند در مراحل بعدی اضافه شود.

---

## 19. Appointment Reschedule Flow

```text
Appointment
    ↓
Reschedule
    ↓
Select New Date
    ↓
Select Available Slot
    ↓
Validate Availability
    ↓
Update Appointment
```

تمام تغییرات مهم باید Audit شوند.

---

## 20. Appointment Cancellation Flow

```text
Appointment
    ↓
Cancel
    ↓
Select Reason
    ↓
Confirm Cancellation
    ↓
Appointment Cancelled
```

Cancellation Policy می‌تواند در آینده Tenant-configurable باشد.

---

## 21. Force Appointment

Force Appointment یکی از قابلیت‌های متمایز Shinera است.

Rule پیشنهادی:

فقط مشتری VIP بتواند Force Appointment Request ارسال کند.

```text
No Available Slot
       ↓
VIP Customer
       ↓
Request Force Appointment
       ↓
Select Preferred Time
       ↓
Send Request
       ↓
Staff / Salon Review
       ↓
Accept / Reject / Suggest Alternative
```

این قابلیت در MVP اصلی قرار نمی‌گیرد و برای Release 1.1 در نظر گرفته می‌شود.

---

## 22. Payments

در MVP پرداخت ساده باقی می‌ماند.

Payment شامل:

- Appointment
- Amount
- Payment Method
- Payment Status
- Payment Date

Payment Method:

- Cash
- Card
- Online
- Other

سیستم حسابداری کامل بخشی از MVP نیست.

---

## 23. Subscription

هر Tenant باید Subscription داشته باشد.

```text
Tenant
  ↓
Subscription
  ↓
Plan
  ↓
Features
```

نمونه Featureها:

```text
Feature.MultipleBranches
Feature.AdvancedReports
Feature.ForceAppointment
Feature.CustomerLoyalty
```

Feature Gating باید هم در Backend و هم در Frontend رعایت شود.

---

## 24. Authorization

مدل دسترسی فقط Role-based نیست.

```text
User
 ↓
Role
 ↓
Permission
```

Permission دارای Scope است:

```text
Tenant
Branch
Own
Child
```

نمونه Permission:

```text
appointments.view
appointments.create
appointments.update
appointments.cancel

customers.view
customers.create

staff.manage
services.manage
```

---

## 25. Branch Management

هر Tenant می‌تواند چند Branch داشته باشد.

Main Branch همیشه وجود دارد.

```text
Tenant Created
     ↓
Main Branch Created
```

اگر Multi Branch فعال نباشد، فقط Main Branch استفاده می‌شود.

---

## 26. Notifications

قابلیت‌های آینده:

- Appointment Reminder
- Appointment Confirmation
- Appointment Cancellation
- Payment Confirmation

کانال‌ها:

- SMS
- Email
- Push
- Messenger Integration

در MVP فقط زیرساخت لازم در صورت نیاز آماده می‌شود.

---

## 27. Audit Trail

عملیات مهم باید Audit شوند:

- Appointment Created
- Appointment Cancelled
- Appointment Rescheduled
- Customer Updated
- Payment Updated
- Staff Schedule Changed

ساختار پایه:

```text
User
Tenant
Branch
Action
Entity
EntityId
Timestamp
```

---

## 28. Primary User Flows

### Owner Registration

```text
Landing Page
   ↓
Pricing
   ↓
Select Plan
   ↓
Registration Wizard
   ↓
Business Information
   ↓
Owner Information
   ↓
Branch Information
   ↓
Payment / Subscription
   ↓
Workspace Provisioning
   ↓
Onboarding
   ↓
Dashboard
```

### Owner Onboarding

```text
Dashboard
   ↓
Create Services
   ↓
Create Staff
   ↓
Configure Schedule
   ↓
Add Customers
   ↓
Create First Appointment
```

Checklist:

```text
□ Business Profile
□ Add First Service
□ Add Staff
□ Configure Working Hours
□ Add First Customer
□ Create First Appointment
```

### Daily Salon Flow

```text
Login
 ↓
Dashboard
 ↓
Today's Appointments
 ↓
Select Appointment
 ↓
Start Service
 ↓
Complete Appointment
 ↓
Register Payment
 ↓
Appointment Completed
```

### Reception Flow

```text
Customer Calls
   ↓
Search Customer
   ↓
Customer Found?
   ↓
Yes → Select Customer
No  → Create Customer
   ↓
Select Service
   ↓
Select Staff
   ↓
Select Available Slot
   ↓
Create Appointment
```

### Staff Flow

```text
Login
 ↓
My Schedule
 ↓
Today's Appointments
 ↓
Select Appointment
 ↓
View Customer
 ↓
Start
 ↓
Complete
```

### Customer Booking Flow

```text
Booking Page
 ↓
Select Service
 ↓
Select Staff
 ↓
Select Date
 ↓
Available Slots
 ↓
Select Time
 ↓
Customer Information
 ↓
Confirm
 ↓
Booking Success
```

---

## 29. Core Entities

Minimum Domain Entities:

```text
Tenant
Branch
BusinessProfile

User
Role
Permission

Staff
StaffService
StaffSchedule

ServiceCategory
Service

Customer

Appointment
Payment

Plan
Feature
PlanFeature
Subscription

AuditLog
```

---

## 30. MVP Definition

هدف MVP:

اثبات اینکه یک سالن یا متخصص مستقل می‌تواند عملیات اصلی روزانه خود را با Shinera انجام دهد.

```text
Register
 ↓
Create Business
 ↓
Create Services
 ↓
Create Staff
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

## 31. MVP Scope

### Authentication

IN:

- Login
- Logout
- Refresh Token
- User Session
- Password Hashing

OUT:

- Social Login
- MFA
- Passwordless

### Tenant

IN:

- Create Tenant
- Update Tenant
- Activate / Deactivate Tenant

### Branch

IN:

- Main Branch
- Branch CRUD
- Branch Selection

### Business Profile

IN:

- Business Name
- Phone
- Address
- Logo
- Business Type
- Working Hours

### Services

IN:

- Service Categories
- Service CRUD
- Duration
- Price
- Active / Inactive

### Staff

IN:

- Staff CRUD
- Assign Services
- Assign Branch
- Working Schedule

### Customers

IN:

- Customer CRUD
- Search Customer
- Customer Notes
- Appointment History

### Appointments

IN:

- Create Appointment
- Update Appointment
- Cancel Appointment
- Reschedule Appointment
- Complete Appointment
- No-show
- Conflict Detection
- Available Slots

### Payments

IN:

- Register Payment
- Payment Method
- Payment Status
- Appointment Payment

OUT:

- Accounting
- Full Invoice Engine
- Online Payment Gateway

### Dashboard

IN:

- Today's Appointments
- Upcoming Appointments
- Today's Revenue
- Monthly Revenue
- New Customers
- Top Services

### Subscription

IN:

- Plan
- Plan Feature
- Subscription
- Feature Gating

---

## 32. MVP Permission Model

Roles:

```text
Owner
Admin
Staff
Receptionist
```

Permissions:

```text
appointments.*
customers.*
services.*
staff.*
payments.*
reports.*
settings.*
```

---

## 33. Features Excluded From MVP

- Force Appointment — Release 1.1
- VIP Business Rules — Release 1.1
- Loyalty — Release 1.2
- Referral System — Later
- Marketing Automation — Later
- Advanced Reports — Later
- Customer Mobile App — Later
- Inventory — Later
- Payroll — Later
- Marketplace — Later

---

## 34. Technical Architecture

Backend:

```text
.NET 10

Domain
Application
Infrastructure
API
```

Architecture:

```text
Modular Monolith
Vertical Slice
CQRS
Minimal API
```

Infrastructure:

```text
EF Core
OpenIddict
Scalar
IDispatcher / CQRS
Result Pattern
Global Exception Handling
Health Checks
```

Frontend:

```text
Angular 20
Angular Material
Trezo
RTL
Persian UI
```

Modules:

```text
auth
workspace
dashboard
appointments
customers
staff
services
payments
settings
subscription
```

---

## 35. Localization

Storage:

```text
Gregorian Date
UTC Timestamp
```

Display:

```text
Persian Calendar
fa-IR
RTL
```

---

## 36. Non-Functional Requirements

### Performance

هدف پایه برای APIهای CRUD معمول:

```text
P95 < 500ms
```

### Security

- Secure Token Storage
- Refresh Token Rotation
- Tenant Isolation
- Permission Validation
- Input Validation
- Rate Limiting
- Audit Logging

### Multi-Tenant Isolation

هیچ Query نباید اطلاعات Tenant دیگری را برگرداند.

TenantId باید در تمام Entityهای Tenant-specific وجود داشته باشد.

### Reliability

Appointment creation باید Transactional باشد.

Conflict Detection باید در Backend انجام شود.

Frontend validation به تنهایی قابل اعتماد نیست.

### Observability

سیستم باید قابلیت مشاهده موارد زیر را داشته باشد:

- Requests
- Exceptions
- Logs
- Traces
- Health
- Slow Requests

---

## 37. Analytics Events

```text
signup_started
signup_completed
onboarding_started
service_created
staff_created
customer_created
appointment_created
appointment_completed
appointment_cancelled
payment_registered
```

---

## 38. Product KPIs

### Activation Rate

درصد Tenantهایی که در 24 ساعت اول حداقل یک Appointment ایجاد می‌کنند.

### Appointment Completion

درصد نوبت‌های Completed.

### Weekly Active Businesses

تعداد Tenantهای فعال هفتگی.

### Customer Retention

Retention کسب‌وکارهای عضو Shinera.

### Feature Adoption

میزان استفاده از Featureهای اصلی.

---

## 39. MVP Success Criteria

MVP زمانی موفق محسوب می‌شود که یک سالن واقعی بتواند بدون ابزار جانبی چرخه اصلی را انجام دهد:

```text
Register
 ↓
Configure Business
 ↓
Add Service
 ↓
Add Staff
 ↓
Add Customer
 ↓
Book Appointment
 ↓
Complete Appointment
 ↓
Register Payment
```

---

## 40. Development Priority

### Phase 1 — Foundation

```text
Architecture
Database
Authentication
Tenant
Branch
Permission
```

### Phase 2 — Subscription

```text
Plan
Feature
Subscription
Feature Gating
```

### Phase 3 — Business Setup

```text
Business Profile
Services
Staff
Schedules
```

### Phase 4 — Customers

```text
Customer Management
Customer Search
Customer History
```

### Phase 5 — Appointments

```text
Appointment CRUD
Availability
Conflict Detection
Calendar
Status Workflow
```

### Phase 6 — Payments

```text
Payments
Payment Status
Revenue Calculation
```

### Phase 7 — Dashboard

```text
KPIs
Recent Appointments
Revenue
Popular Services
```

### Phase 8 — Customer Booking

```text
Public Booking
Available Slots
Booking Confirmation
```

---

## 41. MVP Final Boundary

نسخه MVP شامل 9 بخش اصلی است:

1. Authentication
2. Tenant / Workspace
3. Branch
4. Services
5. Staff / Schedule
6. Customers
7. Appointments
8. Payments
9. Dashboard

Subscription و Permission جزو Infrastructure ضروری هستند و باید از ابتدا وجود داشته باشند.

---

## 42. Post-MVP Roadmap

```text
MVP
 ↓
Force Appointment
 ↓
VIP Customers
 ↓
Notification System
 ↓
Online Payments
 ↓
Advanced Reporting
 ↓
Customer Loyalty
 ↓
Packages
 ↓
Discounts
 ↓
Marketing
 ↓
Marketplace
```

---

## 43. Core Product Principle

هر Feature در Shinera باید حداقل یکی از این اهداف را برآورده کند:

```text
Reduce Work
Reduce Error
Increase Revenue
Increase Retention
Improve Customer Experience
```

Featureهایی که هیچ‌کدام از این اهداف را برآورده نمی‌کنند نباید وارد Core Product شوند.

---

## 44. Shinera MVP Product Statement

Shinera MVP یک SaaS چندمستاجری برای مدیریت کسب‌وکارهای زیبایی است که به متخصص مستقل یا سالن اجازه می‌دهد:

- کسب‌وکار خود را ایجاد کند
- خدمات خود را تعریف کند
- کارکنان خود را مدیریت کند
- برنامه کاری تعریف کند
- مشتریان را مدیریت کند
- نوبت ثبت کند
- تداخل زمانی را مدیریت کند
- نوبت را تکمیل کند
- پرداخت ثبت کند
- عملکرد کسب‌وکار را مشاهده کند

اگر این Flow با کیفیت، سرعت و UX مناسب کار کند، Shinera آماده اولین استفاده واقعی و Validation بازار است.
