# Multi-Tenant Gym Operating System — Product & Technical Scaffold

## 0. Current Build Scope — Single-Gym v0 (Multi-Tenancy Deferred)

Everything below this section describes the eventual multi-tenant SaaS target architecture. **We are not building that yet.** We are building for one gym first, and only moving to multi-tenant once the single-gym product proves it works and there's a reason (money/demand) to generalize it.

### v0 roles (replaces Section 3 for now)

- **Super Admin (owner/you)** — full access: settings, staff accounts, financial data, everything. Equivalent to "Gym Owner" in Section 3, just literally you.
- **Admin** — day-to-day operations: members, attendance, payments, memberships, reports. Covers what Section 3 splits into Manager + Receptionist + Accountant. Split into narrower roles later only if real staffing needs it.
- **Member** — own profile, membership, attendance, workout/diet (once built), payments. Same as Section 3.

Trainer role and its permissions stay deferred until the workout/diet module is actually built (Phase 2), per Section 41.

### v0 architecture (replaces Sections 4 and 6 for now)

No `tenant_id`, no `branch_id`, no Row-Level Security. One gym, one database, no tenant scoping layer.

Future-proofing principles — cheap to follow now, expensive to retrofit later:

- **Route every query through a repository/service layer** — never inline DB queries in route handlers/controllers. When multi-tenancy is added later, scoping gets added in one layer instead of being hunted through the whole codebase.
- **Use UUIDs for all primary keys**, not auto-increment integers — makes merging/migrating data across future tenants painless.
- **Store gym configuration as a row**, not hardcoded constants (`gym_settings` table: name, GST, hours, branding, timezone). "One gym" becomes "one row" later instead of a rewrite.
- **Keep the modular folder structure** from Section 6 (`members/`, `attendance/`, `payments/`, etc.) even without a tenant boundary — the module seams are what let `tenant_id` scoping get bolted on later without restructuring.

Do NOT add empty `tenant_id`/`branch_id` columns "just in case" — that's premature complexity in the other direction. Multi-tenancy gets added when there's a second gym to onboard, not before.

### v0 tech stack (replaces Section 5 for now)

Simplified from the target stack — no separate NestJS backend, no Redis/BullMQ, until real async load justifies them:

- **Next.js** (App Router) — frontend + API routes/server actions, single deployable
- **PostgreSQL + Prisma**
- **Razorpay** for payments (still behind a `PaymentProvider` interface per Section 12 — cheap to keep, expensive to retrofit)
- **WhatsApp/Email** notifications sent synchronously or via simple cron, no queue yet
- Deploy as a single app (e.g., Vercel or a small container) — no ECS/managed Redis needed yet

Reintroduce NestJS + Redis/BullMQ + full multi-tenancy (Sections 3–6) only when there's a second paying gym or the async workload actually demands a queue.

### v0 MVP scope (replaces Section 40 for now)

- Auth (single gym, no tenant switching)
- 3 roles above, basic permission checks (no granular RBAC matrix yet)
- Members, membership plans, memberships, attendance (manual/QR to start), payments, invoices/receipts
- Basic dashboard (Section 30 principles: what happened / what needs attention / what to do)
- Light operational alerts (expiring memberships, attendance drop) — no full workflow engine yet

Everything else in this document (Sections 1–3, 5–6, 40+ as originally written) is the **future direction**, not the current build target. Keep it as the roadmap so early decisions don't box out the multi-tenant path later — just don't build it yet.

---

## 1. Product Vision

Build a multi-tenant, India-first gym operating system (SaaS) that includes the standard gym-management capabilities users already expect from products such as GymAdminX, GymForce, Kasratbook, etc., but goes beyond CRUD management by adding an operational layer.

### Core product thesis

The product should not merely tell a gym owner what happened.

It should help the gym understand:

- What needs attention?
- Why does it need attention?
- Who should act?
- What action should be taken?
- Can the action be automated?
- What happened after the action?

### Product layers

1. **Platform**
   - Multi-tenancy
   - Authentication
   - RBAC
   - Branches
   - Security
   - Audit logs
   - Billing/subscriptions
   - Integrations

2. **Gym Management**
   - Members
   - Memberships
   - Attendance
   - Payments
   - Trainers
   - Workouts
   - Diet
   - PT
   - Classes
   - CRM/leads
   - Inventory
   - Expenses
   - Reports

3. **Operational Engine**
   - Events
   - Rules
   - Workflows
   - Tasks
   - Alerts
   - Approvals
   - Automated actions
   - Exception handling

4. **Intelligence**
   - Analytics
   - Member engagement
   - Retention/churn signals
   - Revenue insights
   - Operational recommendations
   - AI assistant (later)

---

# 2. Target Customers

Initial target:

- Independent gyms
- Small/medium gym chains
- Multi-branch gyms
- Fitness studios
- Personal-training businesses

Do not try to optimize for every fitness business from day one.

A strong initial target should be gyms with enough operational complexity that automation has clear value.

---

# 3. User Roles

## SaaS-level roles

### Super Admin
- Manage all tenants
- Create/suspend tenants
- Manage subscriptions
- View platform metrics
- Manage feature flags
- Support access
- Audit platform activity

## Tenant-level roles

### Gym Owner
- Full access to tenant
- Financial data
- Branch management
- Staff
- Members
- Memberships
- Reports
- Settings
- Integrations

### Gym Manager
- Operational access
- Members
- Attendance
- Memberships
- Staff workflows
- Reports according to permission

### Receptionist
- Member registration
- Check-in
- Membership lookup
- Payments
- Renewals
- Basic member information

### Trainer
- Assigned members
- Workout plans
- Diet plans
- PT sessions
- Progress
- Member notes

### Accountant
- Payments
- Invoices
- Expenses
- Revenue
- Financial reports

### Member
- Own profile
- Membership
- Attendance
- Workout
- Diet
- Progress
- PT/classes
- Payments

Permissions must be granular and branch-aware.

---

# 4. Multi-Tenant Architecture

Use a shared database/shared schema initially.

Every tenant-owned table should have:

- `tenant_id`
- `branch_id` where applicable
- `created_at`
- `updated_at`

Example:

```text
tenants
  └── branches
       ├── users
       ├── members
       ├── memberships
       ├── attendance
       ├── payments
       ├── trainers
       └── classes
```

Tenant isolation is critical.

Never allow a request to access another tenant's data through IDs alone.

Every repository/service query must be tenant-scoped.

Consider PostgreSQL Row-Level Security for defense in depth.

---

# 5. Recommended Technology Stack

## Frontend

- Next.js
- React
- TypeScript
- Tailwind CSS
- shadcn/ui
- React Hook Form
- Zod
- TanStack Query
- Recharts/ECharts

## Backend

- NestJS
- TypeScript
- REST API initially
- OpenAPI/Swagger

## Database

- PostgreSQL
- Prisma ORM

## Async processing

- Redis
- BullMQ
- Background workers

## Storage

- Amazon S3 or Cloudflare R2

## Authentication

Use a robust authentication provider or secure custom authentication.

Authorization should remain in the application domain:

- tenant
- branch
- role
- permission

## Payments

Start with Razorpay abstraction.

Design provider interfaces so other providers can be added later.

## Messaging

Create a provider abstraction:

```text
NotificationService
├── WhatsApp
├── Email
├── SMS
└── Push
```

Do not couple business logic directly to a WhatsApp provider.

## Deployment

Start simple:

- Next.js deployment
- Containerized NestJS backend
- Managed PostgreSQL
- Managed Redis
- S3/R2

AWS ECS/RDS is a good long-term option.

Do not start with Kubernetes or microservices.

---

# 6. Architecture Style

Use a **modular monolith** initially.

Suggested backend modules:

```text
src/
├── auth/
├── tenants/
├── branches/
├── users/
├── roles/
├── permissions/
├── members/
├── memberships/
├── attendance/
├── payments/
├── billing/
├── trainers/
├── workouts/
├── exercises/
├── nutrition/
├── personal-training/
├── classes/
├── crm/
├── inventory/
├── expenses/
├── notifications/
├── whatsapp/
├── automation/
├── workflows/
├── tasks/
├── reports/
├── analytics/
├── integrations/
├── audit/
└── common/
```

Do not split these into microservices at the beginning.

Extract services later only when there is a real scaling/ownership reason.

---

# 7. Core Modules

## 7.1 Tenant Management

Super Admin:

- Create tenant
- Activate/deactivate tenant
- Trial
- Subscription
- Plan
- Feature flags
- Tenant usage
- Branch count
- Member count

Tenant:

- Gym name
- Logo
- Branding
- Contact details
- Address
- GST details
- Timezone
- Currency
- Business settings

---

# 8. Branch Management

Support multiple branches from the beginning.

Features:

- Create branch
- Branch details
- Operating hours
- Holidays
- Staff assignment
- Branch-level permissions
- Branch-specific attendance
- Branch-specific network configuration
- Branch-level reports

Example:

```text
Tenant
├── Hyderabad
├── Vijayawada
└── Visakhapatnam
```

---

# 9. Member Management

Member profile:

- Member ID
- Name
- Photo
- Phone
- Email
- DOB
- Gender
- Address
- Emergency contact
- Join date
- Notes
- Documents
- Status

Statuses:

- Active
- Expiring Soon
- Expired
- Frozen
- Cancelled
- Pending

Member history should show:

- Memberships
- Payments
- Attendance
- Workouts
- PT
- Classes
- Communications
- Notes
- Progress
- Actions/workflows

---

# 10. Membership Management

Gym admins can create configurable plans.

Plan properties:

- Name
- Duration
- Price
- Tax
- Joining fee
- Registration fee
- Discount
- PT inclusion
- Class access
- Branch access
- Freeze rules
- Visit limits
- Benefits

Membership lifecycle:

```text
Lead
→ Trial
→ Membership purchased
→ Active
→ Expiring
→ Renewed / Expired
→ Frozen / Cancelled
```

Support:

- Upgrade
- Downgrade
- Extend
- Freeze
- Resume
- Cancel
- Transfer
- Renewal
- Membership history

---

# 11. Attendance

Support multiple attendance methods:

- Wi-Fi/network-based
- QR
- Manual
- Reception
- Kiosk
- Future hardware integrations

## Wi-Fi attendance

Important limitation:

A normal browser cannot reliably read the Wi-Fi SSID.

Instead, when internet is available, identify the gym based on the request's public IP.

Flow:

```text
Member connects to gym Wi-Fi
        ↓
Opens website
        ↓
Backend sees public IP
        ↓
IP matches registered gym/branch
        ↓
Authenticated member identified
        ↓
Membership validated
        ↓
Attendance recorded
```

Do not rely on SSID alone.

Do not use phone MAC address as the identity.

Allow the gym to configure:

- Registered public IP(s)
- Branch
- Attendance enabled
- Duplicate attendance rules

Potentially support multiple IP ranges.

Attendance should record:

- tenant_id
- branch_id
- member_id
- timestamp
- method
- source IP
- device/session metadata where appropriate
- created_by/system

## Attendance analytics

- Today's attendance
- Weekly/monthly trend
- Member visit history
- Visits per member
- Peak hours
- Attendance frequency
- Inactive members
- Unusual attendance changes

---

# 12. Payment & Billing

Support:

- Cash
- UPI
- Card
- Net banking
- Payment gateway
- Partial payments
- Discounts
- Refunds
- Outstanding balance
- Recurring payments

India-focused:

- Razorpay
- UPI
- GST
- GST invoices
- Payment links
- Webhooks
- Recurring/AutoPay where supported

Never couple domain logic directly to Razorpay.

Use:

```text
PaymentProvider
├── createPayment()
├── refundPayment()
├── createSubscription()
└── verifyWebhook()
```

Store provider references and transaction states.

---

# 13. Invoices & Receipts

Support:

- Invoice generation
- Receipt generation
- GST
- Tax configuration
- Invoice numbering
- Payment history
- Refund documents

PDF generation should happen asynchronously when needed.

---

# 14. Trainers

Trainer profile:

- Name
- Photo
- Contact
- Specialization
- Working hours
- Salary/commission
- Branch
- Assigned members

Trainer dashboard:

- Today's sessions
- Upcoming sessions
- Members
- Workout plans
- Diet plans
- Assessments
- Progress
- PT sessions

---

# 15. Exercise Library

Exercise entity:

```text
Exercise
├── Name
├── Muscle group
├── Target muscle
├── Equipment
├── Difficulty
├── Video
├── Instructions
├── Common mistakes
└── Alternatives
```

Allow gym-specific exercises in addition to global exercises.

---

# 16. Workout Management

Trainer can create:

- Workout templates
- Workout plans
- Day-based splits
- Exercise prescriptions
- Sets
- Reps
- Weight
- Rest time
- Tempo
- Notes

Example:

```text
Day 1 — Chest + Triceps

Bench Press
4 × 8–10

Incline DB Press
3 × 10

Cable Fly
3 × 12

Triceps Pushdown
3 × 12
```

Members can view and log workouts.

Track:

- Completed workouts
- Weight
- Reps
- Personal records
- Workout adherence

---

# 17. Nutrition

Diet plans:

- Meals
- Meal times
- Foods
- Portions
- Calories
- Protein
- Carbs
- Fat
- Notes

Allow trainers/nutrition staff to assign plans to members.

---

# 18. Progress & Assessments

Track:

- Weight
- Height
- Body fat %
- Chest
- Waist
- Arms
- Thighs
- Other configurable measurements

Progress:

- Charts
- Before/after photos
- PRs
- Assessment history

---

# 19. Personal Training

Features:

- PT packages
- Session allocation
- Trainer assignment
- Scheduling
- Session completion
- Remaining sessions
- PT payments
- Trainer commission
- PT history

Example:

```text
20-session package
Used: 13
Remaining: 7
```

---

# 20. Classes

Support:

- Class types
- Schedule
- Capacity
- Trainer
- Booking
- Cancellation
- Waitlist
- Attendance

Examples:

- Yoga
- Zumba
- HIIT
- Boxing
- CrossFit
- Spinning

---

# 21. CRM / Leads

Lead entity:

- Name
- Phone
- Email
- Source
- Interested plan
- Assigned staff
- Follow-up date
- Status
- Notes

Pipeline:

```text
New
→ Contacted
→ Trial
→ Interested
→ Joined
→ Lost
```

Metrics:

- Lead count
- Conversion rate
- Trial conversion
- Source performance
- Staff performance
- Lost leads

---

# 22. Inventory

Support:

- Products
- Categories
- Suppliers
- Purchases
- Sales
- Stock
- Low-stock alerts
- Inventory valuation

Examples:

- Supplements
- Merchandise
- Accessories
- Drinks

---

# 23. Expenses & Finance

Expenses:

- Rent
- Electricity
- Salaries
- Equipment
- Maintenance
- Cleaning
- Marketing
- Internet
- Other

Reports:

```text
Revenue
- Expenses
= Net result
```

Keep accounting logic modular so it can be expanded later.

---

# 24. Notifications

Notification channels:

- WhatsApp
- Email
- SMS
- Push

Events:

- Welcome
- Membership expiry
- Payment received
- Payment failed
- Birthday
- Workout assigned
- Diet assigned
- PT reminder
- Class reminder
- Attendance-related alerts

Use templates and variables.

Example:

```text
Hi {{member_name}},
your membership expires on {{expiry_date}}.
Renew here: {{payment_link}}
```

---

# 25. WhatsApp

Treat WhatsApp as an integration, not as the business logic itself.

Support:

- Template messages
- Renewal reminders
- Payment reminders
- Welcome messages
- Birthday messages
- Missed-visit nudges
- Payment links
- Staff alerts
- Workflow-triggered messages

Important:

- Respect WhatsApp Business/API policies
- Store message status
- Handle delivery failures
- Handle retries
- Handle opt-outs/preferences
- Never create uncontrolled message loops

---

# 26. Operational Engine

This is the main differentiation layer.

Use an event-driven internal architecture.

Core events:

```text
TenantCreated
MemberCreated
MembershipPurchased
MembershipExpiring
MembershipExpired
PaymentReceived
PaymentFailed
AttendanceRecorded
WorkoutAssigned
WorkoutCompleted
PTSessionCompleted
LeadCreated
LeadConverted
```

Events should be persisted/reliable where necessary.

Then build rules/workflows.

Example:

```text
MembershipExpiring
        ↓
Check member engagement
        ↓
Check renewal history
        ↓
Determine priority
        ↓
Create action
        ↓
Send notification / request approval
        ↓
Track outcome
```

---

# 27. Workflow Engine

Initial workflow model:

```text
Trigger
  ↓
Conditions
  ↓
Actions
```

Example:

```text
Trigger:
Membership expires in 7 days

Conditions:
Membership active
Payment not pending
Member has opted into WhatsApp

Actions:
Send WhatsApp
Create staff task
```

Later support:

- Delays
- Branching
- Approvals
- Retries
- Escalation
- Multiple actions
- Outcome tracking

Do not build a fully generic workflow builder in MVP.

Start with predefined workflows.

---

# 28. Operational Tasks

System can create tasks for staff.

Example:

```text
TASK

Member: Rahul
Reason: Attendance dropped significantly

Assigned to: Trainer
Priority: High
Due: Today
```

Task statuses:

- Open
- In Progress
- Completed
- Dismissed
- Snoozed

Track outcomes.

---

# 29. Operational Examples

## Attendance drop

```text
Member normally visits 4x/week
Current visits dropped to 1x/week

↓
Detect anomaly

↓
Create attention item

↓
Notify trainer

↓
Optional WhatsApp message

↓
Track whether member returns
```

## Membership renewal

```text
Membership expires in 5 days

↓
Analyze:
- attendance
- previous renewals
- payment history
- engagement

↓
Assign renewal priority

↓
Send appropriate message

↓
Payment

↓
Automatically extend membership
```

## Failed payment

```text
Payment failed

↓
Record failure

↓
Retry according to provider rules

↓
Notify member

↓
Create staff task if unresolved

↓
Escalate if still unpaid
```

The system should progressively reduce manual work.

---

# 30. Dashboard

## Owner dashboard

Show:

- Active members
- New members
- Today's attendance
- Revenue
- Outstanding payments
- Renewals due
- Expired members
- Leads
- PT revenue
- Branch performance
- Operational actions requiring attention

Do not overwhelm users with 50 charts.

Prioritize:

1. What happened?
2. What needs attention?
3. What should I do?

---

# 31. Analytics

Metrics:

### Members
- Growth
- Churn
- Retention
- Active/expired
- Plan distribution

### Attendance
- Visits
- Frequency
- Peak hours
- Branch trends
- Member engagement

### Revenue
- Revenue
- MRR
- ARPU
- Collection rate
- Outstanding
- Plan revenue
- PT revenue

### CRM
- Leads
- Conversion
- Trial conversion
- Source performance

### Trainers
- Member count
- PT sessions
- Revenue/commission
- Member engagement

---

# 32. Member Portal

Responsive web application/PWA initially.

Pages:

```text
Dashboard
Membership
Attendance
Workout
Diet
Progress
PT
Classes
Payments
Profile
```

Member should be able to:

- View membership
- View attendance
- View workout
- Log workouts
- View diet
- View progress
- Book classes
- View PT sessions
- Pay/renew
- Receive notifications

Do not build native mobile apps initially unless there is a validated need.

---

# 33. AI — Later, Not MVP

AI should not be the initial differentiator unless a validated use case requires it.

Potential future capabilities:

### Owner AI assistant

Questions:

- Which members are at risk?
- Why did revenue drop?
- Which branch is underperforming?
- Which memberships expire this week?
- What requires attention today?

### Member assistant

- Workout questions
- Exercise explanations
- General fitness guidance
- Plan navigation

### Operational intelligence

- Churn prediction
- Attendance anomaly detection
- Renewal likelihood
- Lead conversion prediction
- Revenue forecasting

AI should use tenant data securely and respect tenant boundaries.

---

# 34. Security Requirements

Treat security as a first-class requirement.

Must have:

- Strong authentication
- RBAC
- Tenant isolation
- Branch-level permissions
- Input validation
- Rate limiting
- Secure cookies/tokens
- CSRF protection where applicable
- Secure file uploads
- Audit logs
- Encryption in transit
- Encryption at rest through managed infrastructure
- Secrets management
- Webhook signature verification
- Payment idempotency
- Database backups
- Restore testing
- Data export/deletion controls

Never trust:

- tenant_id from the frontend
- branch_id from the frontend
- member_id from the frontend
- payment status from the client
- attendance claims from the client

Derive/validate authorization server-side.

---

# 35. Audit Logging

Audit important actions:

```text
User
Tenant
Action
Entity
Entity ID
Timestamp
IP
Metadata
```

Examples:

- Member deleted
- Membership extended
- Payment refunded
- User permissions changed
- Branch created
- Attendance manually modified
- Workflow changed

Audit logs should be append-only from normal application flows.

---

# 36. Background Jobs

Use Redis + BullMQ for:

- WhatsApp sending
- Email
- SMS
- Renewal reminders
- Report generation
- PDF generation
- Analytics processing
- Workflow actions
- Retryable integrations

API requests should not wait for slow external operations.

---

# 37. Idempotency

Required for:

- Payment webhooks
- Attendance creation
- Workflow actions
- Notifications
- Membership renewal
- External integrations

Never create duplicate payments/membership extensions/messages because an API request or webhook was retried.

---

# 38. API Design

REST initially.

Example:

```text
POST   /auth/login
GET    /me

GET    /tenants/:tenantId
GET    /branches
POST   /branches

GET    /members
POST   /members
GET    /members/:id
PATCH  /members/:id

GET    /memberships
POST   /memberships

GET    /attendance
POST   /attendance/check-in

GET    /payments
POST   /payments

GET    /workouts
POST   /workouts

GET    /workflows
POST   /workflows

GET    /tasks
PATCH  /tasks/:id

GET    /reports/...
```

Every endpoint must enforce authorization and tenant isolation.

---

# 39. Database Principles

Use normalized relational models where appropriate.

Core entities:

```text
Tenant
Branch
User
Role
Permission
Member
MembershipPlan
Membership
Attendance
Payment
Invoice
Trainer
Exercise
WorkoutPlan
Workout
WorkoutLog
NutritionPlan
NutritionMeal
ProgressAssessment
PTPackage
PTSession
Class
ClassBooking
Lead
Task
Workflow
WorkflowExecution
Notification
NotificationTemplate
Product
InventoryTransaction
Expense
AuditLog
GymNetwork
```

Use UUIDs or another non-sequential public identifier strategy.

Use database constraints for important invariants.

---

# 40. MVP Scope

Do NOT build everything initially.

## MVP Phase 1

### Platform
- Multi-tenancy
- Authentication
- RBAC
- Branches
- Audit logging

### Gym operations
- Members
- Membership plans
- Memberships
- Attendance
- Payments
- Invoices/receipts
- Basic dashboard

### Differentiation
- Wi-Fi/public-IP attendance
- Event foundation
- Basic operational alerts
- Basic tasks

This should be a usable real-gym product.

---

# 41. Phase 2

- Trainers
- Exercise library
- Workout plans
- Workout logging
- Diet plans
- Progress tracking
- PT
- Classes
- Member portal
- Basic CRM

---

# 42. Phase 3

- WhatsApp Business integration
- Payment automation
- Automated renewal workflows
- Attendance workflows
- Lead follow-up workflows
- Staff tasks
- Advanced notifications
- Expenses
- Inventory
- Multi-branch analytics

---

# 43. Phase 4

- Advanced analytics
- Churn prediction
- Revenue forecasting
- AI owner assistant
- AI member assistant
- Advanced workflow engine
- More payment providers
- Hardware integrations
- White-label
- Public API

---

# 44. What NOT to Build Initially

Avoid:

- Native mobile apps
- Microservices
- Kubernetes
- Generic workflow builder
- Full accounting ERP
- Complex AI
- Marketplace
- Social network
- Custom biometric hardware
- Too many payment providers
- Too many notification providers

Prove the core product first.

---

# 45. Product Differentiation

The product must NOT be marketed merely as:

> "Gym management software with more features."

Competitors already have many features.

Position the product around:

> **Gym operations that run themselves.**

Potential differentiation:

- Operational alerts
- Action recommendations
- Staff task generation
- Automated workflows
- Cross-module intelligence
- Member engagement tracking
- Revenue recovery
- Branch operational visibility
- Frictionless attendance
- Eventually AI-driven operations

The important distinction:

```text
Traditional software:
Data → Dashboard → Human decides → Human acts

Operational software:
Data → Detect → Decide → Action → Measure outcome
```

---

# 46. UX Principles

The application should be:

- Fast
- Simple
- Desktop-first for admin
- Mobile-friendly
- Clear
- Action-oriented

Avoid:

- Huge forms
- Too many dashboards
- Deep navigation
- Excessive configuration
- Technical terminology

For the owner dashboard, prioritize actionable information.

---

# 47. Development Principles for Claude

Before writing significant code:

1. Understand the architecture.
2. Produce a technical implementation plan.
3. Identify dependencies.
4. Define database schema.
5. Define API contracts.
6. Define authorization rules.
7. Define module boundaries.
8. Define tests.
9. Implement incrementally.
10. Run tests after each meaningful change.

Do not rewrite large parts of the project without a reason.

Do not introduce dependencies unnecessarily.

Do not use microservices.

Do not put business logic into React components.

Do not bypass authorization for convenience.

Do not use mock data in production paths.

---

# 48. Required Development Order

Recommended order:

```text
1. Repository/project setup
        ↓
2. Authentication
        ↓
3. Tenant + branch model
        ↓
4. RBAC + permissions
        ↓
5. Database conventions
        ↓
6. Member management
        ↓
7. Membership plans/memberships
        ↓
8. Payments
        ↓
9. Attendance
        ↓
10. Wi-Fi/IP attendance
        ↓
11. Dashboard
        ↓
12. Event system
        ↓
13. Operational alerts/tasks
        ↓
14. Real-gym validation
        ↓
15. Trainers/workouts/diet
        ↓
16. WhatsApp
        ↓
17. Advanced workflows
        ↓
18. Analytics
        ↓
19. AI
```

---

# 49. Testing Strategy

## Unit tests

Test:

- Membership lifecycle
- Payment calculations
- Authorization
- Tenant isolation
- Attendance rules
- Renewal rules
- Workflow conditions

## Integration tests

Test:

- Database
- Payment webhooks
- WhatsApp provider
- Attendance
- Authentication
- Background jobs

## E2E tests

Critical flows:

```text
Create gym
→ Create branch
→ Create user
→ Create member
→ Create membership
→ Receive payment
→ Check in
→ View dashboard
→ Renew membership
```

Also test cross-tenant access attempts.

---

# 50. Observability

Use:

- Structured logging
- Sentry
- OpenTelemetry where appropriate
- Cloud provider metrics
- Job/queue monitoring

Monitor:

- API latency
- Error rate
- Database performance
- Queue failures
- Notification failures
- Payment failures
- Authentication failures

---

# 51. SaaS Pricing Architecture

Don't hard-code pricing into application logic.

Create:

```text
SubscriptionPlan
PlanFeature
TenantSubscription
UsageMetric
```

Possible plans later:

```text
Starter
Growth
Pro
Enterprise
```

Features can be enabled per plan/tenant.

---

# 52. Future Integrations

Design integration interfaces for:

- Razorpay
- WhatsApp Business
- Email provider
- SMS provider
- UPI/payment providers
- Biometric attendance hardware
- Accounting software
- CRM
- S3-compatible storage

Do not implement all integrations initially.

---

# 53. Key Product Metrics

Track product-level metrics:

- Active tenants
- Paying tenants
- Tenant retention
- Members per tenant
- Attendance events
- Payments processed
- Renewal rate
- Churn
- Workflow execution
- Automation success rate
- Staff task completion
- Member engagement

The product should eventually prove operational ROI.

Example:

```text
Before:
Gym staff manually follows up with 80 renewals.

After:
System automatically handles 60.
Staff manually handles 20.

Outcome:
Less admin work + better renewal rate.
```

---

# 54. Final Product Principle

The system should evolve from:

```text
Gym Management Software
```

into:

```text
Gym Operating System
```

The management layer provides the data.

The operational layer turns the data into actions.

The intelligence layer improves those actions over time.

Build the platform so these layers can evolve independently without requiring a rewrite.
