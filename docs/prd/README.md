# Event Management Platform — PRD & Technical Documentation

> Comprehensive Product Requirements Document and Technical Task specification designed to prevent AI hallucination during code generation.

## How to Use This Documentation

1. **Before every AI prompt:** Copy the [Global Context Block](./00-global-context.md) and paste it at the top of your prompt.
2. **When building a feature:** Copy the relevant user story from the user stories files alongside the global context.
3. **When designing the database:** Reference [Database Schema](./02-database-schema.md) for the full Prisma schema and ERD.
4. **When building an API endpoint:** Reference [API Endpoints](./03-api-endpoints.md) for exact routes, DTOs, and error matrices.
5. **When building shared infrastructure:** Reference [Shared Infrastructure](./04-shared-infrastructure.md) for middleware, utilities, and configs — build these FIRST.

## Document Index

| # | Document | Contents |
|---|---|---|
| 00 | [Global Context](./00-global-context.md) | Tech stack, folder structure, naming conventions, API envelope — paste with every AI prompt |
| 01 | [Architecture & Design](./01-architecture-and-design.md) | System architecture, layered backend, auth flow, timer mechanism, design system (colors, typography, components), layout patterns |
| 02 | [Database Schema](./02-database-schema.md) | ERD (Mermaid), full Prisma schema (all models + enums), relationships, indexes, soft delete middleware, seed data spec, price formula |
| 03 | [API Endpoints](./03-api-endpoints.md) | Every endpoint with method, path, auth, middleware chain, request/response DTOs, error matrices |
| 04 | [Shared Infrastructure](./04-shared-infrastructure.md) | Error handler, auth/authorize middleware, validation, file upload, Cloudinary, mailer, token/password/code utils, cookie helpers, Express setup, env vars, Axios instance |
| 05 | [User Stories: Auth & Profile](./05-user-stories-auth.md) | US-01 to US-06: Registration, Login, Forgot/Reset Password, Edit Profile, Change Password |
| 06 | [User Stories: Events](./06-user-stories-events.md) | US-07 to US-13: Landing Page, Event Browsing, Event Detail, Create/Edit/Delete Event, Organizer Profile |
| 07 | [User Stories: Transactions](./07-user-stories-transactions.md) | US-14 to US-18: Purchase Tickets, Upload Proof, My Tickets, Organizer Transaction Management, Mark Attendance |
| 08 | [User Stories: Vouchers/Reviews/Dashboard](./08-user-stories-social-dashboard.md) | US-19 to US-22: Create/Manage Vouchers, Write Reviews, Organizer Dashboard |

## Traceability & Issue Ticket Mapping

To ensure complete verification during implementation, every PRD Feature Epic maps 1-to-1 with the atomic user stories in `.scratch/event-management-platform/spec.md` and the executable issue tickets in `.scratch/event-management-platform/issues/`.

> [!NOTE]
> **Why Ticket #01 is not in User Stories:** Ticket [01-project-initialization-schema-seeding.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/01-project-initialization-schema-seeding.md) is a structural foundation ticket tasked with setting up npm workspaces, scaffolding Next.js 15 and Express 4, configuring Prisma schema (11 models + junction tables), soft deletion middleware, and seeding 30+ records across 8 categories. Because it builds core infrastructure rather than end-user features, it is governed by PRD documents [00-global-context.md](./00-global-context.md), [01-architecture-and-design.md](./01-architecture-and-design.md), [02-database-schema.md](./02-database-schema.md), and [04-shared-infrastructure.md](./04-shared-infrastructure.md) rather than a specific user story epic.

| PRD Epic ID | PRD Feature Title | Mapped Atomic Spec Stories (`spec.md`) | Mapped Issue Ticket (`.scratch/issues/`) |
|---|---|---|---|
| **INFRA-01** | Project Init & Schema Seeding | Foundational Setup (PRD 00, 01, 02, 04) | [01-project-initialization-schema-seeding.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/01-project-initialization-schema-seeding.md) |
| **US-01** | User Registration | Stories #1, #2, #3 | [02-auth-and-session-management.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/02-auth-and-session-management.md) & [07-promotional-campaigns-vouchers-points.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/07-promotional-campaigns-vouchers-points.md) |
| **US-02** | User Login | Stories #4, #5 | [02-auth-and-session-management.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/02-auth-and-session-management.md) |
| **US-03** | Forgot Password | Story #6 | [03-profile-management-password-security.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/03-profile-management-password-security.md) |
| **US-04** | Reset Password | Story #7 | [03-profile-management-password-security.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/03-profile-management-password-security.md) |
| **US-05** | Edit Profile | Story #8 | [03-profile-management-password-security.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/03-profile-management-password-security.md) |
| **US-06** | Change Password | Story #9 | [03-profile-management-password-security.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/03-profile-management-password-security.md) |
| **US-07** | Landing Page | Story #10 | [04-event-discovery-catalog-browsing.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/04-event-discovery-catalog-browsing.md) |
| **US-08** | Event Browsing | Stories #11, #12, #13, #14 | [04-event-discovery-catalog-browsing.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/04-event-discovery-catalog-browsing.md) |
| **US-09** | Event Detail Page | Story #15 | [05-event-detail-view-organizer-profile.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/05-event-detail-view-organizer-profile.md) |
| **US-10** | Create Event (Org) | Stories #16, #17 | [06-event-management-lifecycle.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/06-event-management-lifecycle.md) |
| **US-11** | Edit Event (Org) | Story #18 | [06-event-management-lifecycle.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/06-event-management-lifecycle.md) |
| **US-12** | Delete Event (Org) | Story #19 | [06-event-management-lifecycle.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/06-event-management-lifecycle.md) |
| **US-13** | Organizer Profile | Story #20 | [05-event-detail-view-organizer-profile.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/05-event-detail-view-organizer-profile.md) |
| **US-14** | Purchase Tickets | Stories #21, #22 | [08-ticket-purchasing-entitlement-reservation.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/08-ticket-purchasing-entitlement-reservation.md) |
| **US-15** | Upload Proof | Stories #23, #24 | [09-payment-proof-upload-lazy-timers.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/09-payment-proof-upload-lazy-timers.md) |
| **US-16** | View My Tickets | Story #25 | [09-payment-proof-upload-lazy-timers.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/09-payment-proof-upload-lazy-timers.md) |
| **US-17** | Org Transaction Mgmt | Stories #26, #27, #28, #29 | [10-organizer-order-verification-attendance.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/10-organizer-order-verification-attendance.md) |
| **US-18** | Mark Attendance | Story #30 | [10-organizer-order-verification-attendance.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/10-organizer-order-verification-attendance.md) |
| **US-19** | Create Voucher | Story #33 | [07-promotional-campaigns-vouchers-points.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/07-promotional-campaigns-vouchers-points.md) |
| **US-20** | Manage Vouchers | Story #34 | [07-promotional-campaigns-vouchers-points.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/07-promotional-campaigns-vouchers-points.md) |
| **US-21** | Write Review | Stories #31, #32 | [11-post-event-reviews-rating-system.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/11-post-event-reviews-rating-system.md) |
| **US-22** | Organizer Dashboard | Stories #35, #36, #37 | [12-organizer-analytics-dashboard.md](file:///Users/tuanstrange/Documents/Fullstack%20Engineer/Purwadhika%20Bootcamp/Mini%20Project/.scratch/event-management-platform/issues/12-organizer-analytics-dashboard.md) |

## Recommended Build Order

### Phase 0 — Project Setup
1. Initialize monorepo with npm workspaces
2. Set up Next.js app (`apps/web`)
3. Set up Express app (`apps/api`)
4. Configure Prisma with Supabase
5. Run `prisma migrate dev` with the full schema
6. Run `prisma db seed` with seed data

### Phase 1 — Shared Infrastructure (build FIRST)
1. API response helpers + types
2. Global error handler
3. Auth middleware (authenticate + authorize)
4. Validation middleware
5. File upload middleware (Multer + Cloudinary)
6. Mailer service
7. Token, password, code, slug, cookie utilities
8. Frontend: Axios instance with interceptors
9. Frontend: AuthProvider + QueryProvider
10. Frontend: UI primitives (Button, Input, Modal, Card, Table, Pagination, etc.)

### Phase 2 — Authentication
1. US-01: Registration (with referral logic)
2. US-02: Login
3. US-03: Forgot Password
4. US-04: Reset Password

### Phase 3 — Events Core
1. US-10: Create Event
2. US-11: Edit Event
3. US-12: Delete Event
4. US-07: Landing Page
5. US-08: Event Browsing (search/filter/sort/pagination)
6. US-09: Event Detail Page

### Phase 4 — Transaction Flow
1. US-14: Purchase Tickets
2. US-15: Upload Payment Proof
3. US-16: My Tickets
4. US-17: Organizer Transaction Management
5. US-18: Mark Attendance

### Phase 5 — Promotions & Social
1. US-19: Create Voucher
2. US-20: Manage Vouchers
3. US-21: Write Review
4. US-13: Public Organizer Profile

### Phase 6 — Dashboard & Polish
1. US-22: Organizer Dashboard
2. US-05: Edit Profile
3. US-06: Change Password
4. Dark/light mode toggle
5. Responsive design pass
6. Empty states, loading states, error states audit

### Phase 7 — Deployment & Documentation
1. Deploy frontend to Vercel
2. Deploy backend to Vercel (serverless)
3. Migrate + seed production database (Supabase)
4. Write README.md (features, ERD, setup guide, demo accounts, URLs)
5. Final testing in production

---

## Git Workflow & Commit Strategy (Minimum 20 Commits Requirement)

To comply with git workflow best practices and ensure project traceability, development must follow an incremental commit strategy guaranteeing a **minimum of 20 meaningful commits**.

### Commit Standards
1. **Conventional Commits**: Every commit message must follow the `<type>(<scope>): <subject>` format (e.g., `feat(be/auth): implement login endpoint`, `chore(db): run prisma migrations`).
2. **Atomic Increments**: Commits must represent self-contained, functional units of work (e.g., separating backend API implementation from frontend UI assembly). Monolithic, all-in-one commits per ticket are strictly prohibited.

### 25-Commit Execution Plan across Issue Tickets
By splitting our 12 vertical-slice issue tickets into logical backend, frontend, and infrastructure milestones, we execute exactly **25 structured commits**:

- **Ticket 01: Project Initialization & Database Seeding** (3 commits)
  1. `chore(infra): initialize monorepo with next.js 15, express, and tailwind css v4`
  2. `feat(db): implement prisma schema with 11 core models and junction tables`
  3. `chore(db): add database seeding script with reference categories and dummy accounts`
- **Ticket 02: Authentication & Role-Based Access Control** (2 commits)
  4. `feat(be/auth): implement register, login, and jwt authentication middleware`
  5. `feat(fe/auth): build login, registration, and password reset ui pages`
- **Ticket 03: Profile Management & Password Security** (2 commits)
  6. `feat(profile): implement profile editing and avatar upload via cloudinary`
  7. `feat(security): implement change password and account deletion workflows`
- **Ticket 04: Event Discovery, Catalog & Browsing** (2 commits)
  8. `feat(fe/home): build responsive landing page with hero, search, and category cards`
  9. `feat(catalog): implement event catalog with search debouncing, filters, and pagination`
- **Ticket 05: Event Detail View & Public Organizer Profile** (2 commits)
  10. `feat(event-detail): build dynamic event detail page with ticket tier selector`
  11. `feat(organizer): implement public organizer profile page and event listings`
- **Ticket 06: Event Management & Lifecycle - Organizer** (3 commits)
  12. `feat(be/events): implement event creation and multipart ticket tier ingestion`
  13. `feat(fe/events): build organizer event creation and edit form with zod validation`
  14. `feat(events): implement soft deletion and active transaction validation guards`
- **Ticket 07: Promotional Campaigns (Vouchers & Points)** (2 commits)
  15. `feat(vouchers): implement promotional voucher creation with multi-event binding`
  16. `feat(vouchers): add voucher management table and soft-deletion workflow`
- **Ticket 08: Ticket Purchasing & Entitlement Reservation** (2 commits)
  17. `feat(be/checkout): implement 14-step acid checkout transaction and seat reservation`
  18. `feat(fe/checkout): build checkout summary modal with voucher, coupon, and points selectors`
- **Ticket 09: Payment Proof Upload & Lazy Timers** (2 commits)
  19. `feat(payment): implement payment proof image upload and 2-hour countdown timer`
  20. `feat(tickets): build customer transaction dashboard with lazy expiry status badges`
- **Ticket 10: Organizer Order Verification & Attendance** (2 commits)
  21. `feat(verification): implement organizer accept/reject workflow and atomic inventory rollback`
  22. `feat(attendance): add post-event attendance toggle and review unlocking guard`
- **Ticket 11: Post-Event Reviews & Rating System** (1 commit)
  23. `feat(reviews): implement attendee star rating and comment submission`
- **Ticket 12: Organizer Analytics Dashboard** (2 commits)
  24. `feat(be/analytics): implement sql time-series aggregation queries for revenue and attendance`
  25. `feat(fe/dashboard): build recharts analytical dashboard with date range filtering`

## Anti-Hallucination Checklist

Before sending any user story to an AI, verify:

- [ ] Global Context Block is included in the same prompt
- [ ] Every field has an explicit TypeScript type — no bare "etc."
- [ ] Every enum has its exact string values listed
- [ ] Every endpoint has full method + path
- [ ] Every file has an explicit path, not "somewhere in components"
- [ ] "Out of Scope" section is filled in
- [ ] Request/Response DTOs are written as real interfaces
- [ ] The AI Directive ("don't assume, ask") is present
- [ ] Prisma schema is pasted as real code, not summarized
- [ ] Middleware order is numbered and exact
- [ ] Error handling matrix has specific status codes and messages
