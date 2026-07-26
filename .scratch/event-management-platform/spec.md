# Event Management Platform Specification

Status: ready-for-agent

## Problem Statement

Event Organizers often struggle to find an all-in-one platform to publish events, manage ticket sales across multiple pricing tiers, create dynamic promotion campaigns (such as vouchers and referral rewards), and track real-time analytics. At the same time, Customers face fragmented event discovery experiences, lack transparent discount pricing mechanisms, and have no reliable way to verify legitimate event reviews from actual verified attendees.

## Solution

The Event Management Platform is a full-stack web application that seamlessly bridges Organizers and Customers. It provides Organizers with comprehensive event publishing tools, customizable ticket tiers, promotion management (vouchers and referral coupon systems), attendance verification, and visual analytics dashboards. For Customers, it offers intuitive event discovery with server-side search, filtering, and sorting, secure ticket purchasing with automated discount and point ledger calculations, payment proof uploading with lazy-evaluated countdown deadlines, and verified post-event reviews.

## User Stories

1. As a Visitor, I want to create a Customer or Organizer account using my email and password, so that I can access platform features tailored to my role.
2. As a Visitor submitting registration with a valid referral code, I want to receive an automated 10% discount coupon, so that I am rewarded for joining through an existing user.
3. As a Customer whose referral code is used by a new user, I want to receive 10,000 reward points in my ledger, so that I can apply point discounts on future ticket purchases.
4. As a registered user, I want to log in with my email and password, so that I can access my account dashboard or ticket history without session disruption.
5. As a registered user, I want my authentication tokens managed via secure httpOnly cookies with automated silent refreshing, so that my session remains secure against cross-site scripting without forcing frequent manual re-logins.
6. As a registered user who forgot my password, I want to request a secure, time-limited reset link via email, so that I can regain account access securely without exposing my email status to enumeration attacks.
7. As a user with a valid reset link, I want to set a new password and automatically revoke all existing sessions, so that my account remains secure across all devices.
8. As a logged-in user, I want to view and update my profile details (first name, last name, profile picture), so that my account identity and avatar remain accurate.
9. As a logged-in user, I want to change my account password after confirming with a dialog prompt, so that I can maintain account security.
10. As a Visitor, I want to browse a landing page showcasing featured upcoming events and pre-seeded category cards, so that I can quickly discover interesting events.
11. As a Visitor, I want to search for events by name or description with debounced input, so that I can find relevant events without overloading the system.
12. As a Visitor, I want to filter events by category, location, free/paid status, price range, and date range, so that I can narrow down discovery to my exact preferences.
13. As a Visitor, I want to sort event listings by start date, price, or name in ascending or descending order, so that I can view listings in my preferred sequence.
14. As a Visitor, I want to view paginated event results processed entirely on the backend, so that page load performance remains fast even with large event catalogs.
15. As a Visitor, I want to view an event detail page displaying ticket tiers, available seat counts, organizer details, and verified customer reviews, so that I can make an informed decision before purchasing.
16. As an Organizer, I want to create a new event with multiple named ticket tiers or a single flat price, so that I can monetize general admission or VIP seating arrangements.
17. As an Organizer creating an event, I want to upload a banner image that is automatically processed and hosted in cloud storage, so that my event page is visually appealing.
18. As an Organizer, I want to edit my existing event details and ticket quotas, so that I can adjust event parameters as long as I do not reduce seat counts below what has already been sold.
19. As an Organizer, I want to soft-delete an event that has no active pending transactions, so that I can remove cancelled or obsolete events while preserving historical financial records.
20. As a Visitor, I want to view an organizer's public profile page showing their bio, aggregate review rating, and published event catalog, so that I can evaluate the organizer's reputation.
21. As a Customer, I want to select ticket quantities and view a real-time price breakdown applying either a voucher or a coupon alongside my point balance, so that I can see my exact final price before confirming purchase.
22. As a Customer submitting a ticket order, I want seats immediately reserved and my discount entitlements applied in an atomic transaction, so that my reservation is guaranteed while awaiting payment proof upload.
23. As a Customer with a pending ticket order, I want to view a 2-hour countdown timer and upload payment proof image evidence, so that my purchase can proceed to organizer verification.
24. As a Customer whose payment deadline expires before proof upload, I want the system to automatically mark the order expired on next read and roll back reserved seats, points, and promotional discounts, so that my balances are restored accurately.
25. As a Customer, I want to browse my ticket history and view order status badges, so that I can track which tickets are awaiting confirmation, completed, or rejected.
26. As an Organizer, I want to view all customer transactions across my events, filter by status, and inspect uploaded payment proof images, so that I can verify customer payments.
27. As an Organizer, I want to accept a customer's payment proof after confirmation prompt, automatically generating a unique booking code and emailing the ticket confirmation, so that the customer receives their entry verification.
28. As an Organizer, I want to reject an invalid payment proof with an optional explanation, triggering an atomic rollback of reserved seats and customer rewards, so that inventory and customer balances are restored.
29. As an Organizer, I want orders awaiting confirmation for over 3 days to auto-cancel on read with full inventory and reward rollback, so that abandoned orders do not permanently lock seats.
30. As an Organizer whose event end date has passed, I want to toggle an attendance checkmark on completed customer transactions, so that actual attendees are verified in the system.
31. As a Customer who has attended a completed event, I want to submit a 1-to-5 star rating and text review, so that I can share my verified experience with future visitors.
32. As a Customer who did not attend or already reviewed an event, I want the system to block duplicate or unverified review submissions, so that overall event ratings remain authentic.
33. As an Organizer, I want to create promotional vouchers with percentage or fixed discounts, usage caps, and validity dates linked to specific events I manage, so that I can run marketing campaigns.
34. As an Organizer, I want to view and soft-delete my promotional vouchers, so that I can retire expired or exhausted discount campaigns.
35. As an Organizer, I want to access an analytics dashboard displaying summary cards for total revenue, total attendees, total events, and average rating, so that I can evaluate my overall business performance.
36. As an Organizer, I want to interact with dashboard visual charts depicting monthly revenue, ticket sales trends, and category distribution, so that I can analyze historical sales patterns.
37. As an Organizer, I want to filter my dashboard statistics and charts by a custom date range, so that I can generate performance reports for specific operational periods.

## Implementation Decisions

- **Monorepo Separation:** The system is structured as an npm workspace separating a rendering-focused frontend app from a domain-focused API backend app.
- **Layered Backend Architecture:** The backend strictly enforces separation of concerns across routing, middleware chains, thin controllers, domain logic services, and ORM repository layers.
- **Authentication & Security:** Access and refresh tokens are stored in secure httpOnly cookies. The API layer returns uniform success responses on sensitive flows (such as password reset requests) to prevent email enumeration attacks.
- **Transaction State Machine:** Purchases follow a strict six-state lifecycle (`WAITING_FOR_PAYMENT`, `WAITING_FOR_ADMIN_CONFIRMATION`, `DONE`, `REJECTED`, `EXPIRED`, `CANCELED`). All state transitions that release inventory or cancel orders execute within atomic database transactions.
- **Lazy Evaluation for Timers:** Due to serverless execution constraints, time-dependent state changes (2-hour payment window expiration and 3-day admin confirmation expiration) are evaluated dynamically during query read operations rather than relying on background background daemons or scheduled timers.
- **Immutable Point Ledger:** Customer point balances are calculated dynamically from an immutable ledger of credit and debit entries. Point spending follows FIFO ordering where oldest-expiring credit entries are consumed first.
- **Discount Stacking Rules:** Ticket purchases permit either an organizer voucher or a referral coupon, but never both simultaneously. Point balances may be applied on top of voucher or coupon discounts, with the final price floored at zero.
- **Soft Deletion Scope:** Core domain entities (users, events, transactions, reviews, ticket tiers, and vouchers) implement soft deletion timestamps. Database middleware automatically excludes soft-deleted records from standard read queries.
- **UI Design System & Aesthetics:** The user interface is built from scratch using styling utility classes following a dark-mode-first aesthetic with curated color palettes, typography scales, responsive grid layouts, and interactive feedback components (modals, confirmation dialogs, toast notifications, and loading skeletons).
- **Asynchronous Communications:** Email notifications (welcome messages, ticket confirmations with booking codes, rejection notices, and password resets) fire asynchronously without blocking primary HTTP request-response cycles.

## Testing Decisions

- **Test Runner & Architecture:** Adopt **Jest** (`ts-jest` for backend, official `next/jest` preset for frontend) as the unified monorepo test runner. Test files must live in dedicated `__tests__/` directories per package.
- **Testing External Behavior:** Tests must assert public contracts and observable system side-effects (HTTP response status codes, payload structures, database state mutations) rather than internal implementation details or private methods.
- **Primary Seam (Backend HTTP API Boundary):** The primary integration testing seam is established at the HTTP route handler level using `supertest` assertions against a real PostgreSQL test database. This verifies the complete integration of middleware, validation schemas, controllers, domain services, and database queries in a single test execution.
- **Secondary Seam (Frontend Component Testing):** Use **React Testing Library (RTL)** for targeted frontend component tests focusing strictly on form validation behavior, conditional rendering logic (RBAC visibility, disabled states), and UI state transitions (loading skeletons, empty states). Do not test API data fetching or visual styling.
- **External Integration Mocking:** Third-party dependencies (Cloudinary image uploads and SMTP email transmission) are mocked at their respective service wrappers (`jest.mock()`) during test execution to ensure fast, deterministic, and offline-capable test suites. `nock` is available as a secondary interceptor when verifying outgoing HTTP network shapes.
- **Database Isolation Lifecycle:** Use a truncate-and-reseed strategy between test files via a shared `truncateAll()` helper (raw SQL `TRUNCATE ... CASCADE`) and factory functions for minimal fixtures. `globalSetup.ts` executes `prisma migrate deploy` once before the test suite runs.
- **CI/CD Integration (GitHub Actions):** Merge to `main` is blocked by 5 automated verification gates: TypeScript type-check (`tsc --noEmit`), ESLint (`eslint .`), Jest backend tests, Jest frontend tests, and production bundle build (`npm run build`). Runs in a PostgreSQL 16 service container without an enforced percentage coverage threshold (relying on spec-driven assertions).


## Out of Scope

- Third-party online payment gateway integrations (credit card processing, virtual accounts, e-wallets); payment verification relies on manual proof upload and organizer review.
- Multi-event shopping cart functionality; purchases are executed per event.
- Social media login integrations (OAuth via Google, Facebook, Apple, or GitHub).
- Real-time WebSocket push notifications or live chat between customers and organizers.
- Automated email address verification during initial user registration.
- Ticket transfer, resale, or gifting mechanisms between customer accounts.
- Exporting analytics reports or attendee lists to external file formats (CSV, Excel, PDF).

## Further Notes

- A complete suite of pre-seeded reference data (covering 8 fixed categories, multiple organizer and customer accounts, free/paid events, ticket tiers, transaction states, promotional vouchers, point ledgers, and reviews) must be available via database seeding scripts to facilitate immediate local development, testing, and UI evaluation.
- All destructive or data-modifying user actions across the frontend interface must require explicit user confirmation via an interactive dialog component prior to API submission.

## Git Workflow & Commit Strategy (Minimum 20 Commits Requirement)

To comply with project git workflow requirements, development execution across all issue tickets must follow an atomic, incremental commit strategy guaranteeing at least **20 meaningful Conventional Commits**:
- **Atomic Execution**: Developers and agents must split feature tickets into distinct backend, frontend, and infrastructure commits (e.g., separating Prisma schema migrations from API controllers and UI forms).
- **25-Commit Execution Plan**:
  1. `chore(infra): initialize monorepo with next.js 15, express, and tailwind css v4` (Ticket #01)
  2. `feat(db): implement prisma schema with 11 core models and junction tables` (Ticket #01)
  3. `chore(db): add database seeding script with reference categories and dummy accounts` (Ticket #01)
  4. `feat(be/auth): implement register, login, and jwt authentication middleware` (Ticket #02)
  5. `feat(fe/auth): build login, registration, and password reset ui pages` (Ticket #02)
  6. `feat(profile): implement profile editing and avatar upload via cloudinary` (Ticket #03)
  7. `feat(security): implement change password and account deletion workflows` (Ticket #03)
  8. `feat(fe/home): build responsive landing page with hero, search, and category cards` (Ticket #04)
  9. `feat(catalog): implement event catalog with search debouncing, filters, and pagination` (Ticket #04)
  10. `feat(event-detail): build dynamic event detail page with ticket tier selector` (Ticket #05)
  11. `feat(organizer): implement public organizer profile page and event listings` (Ticket #05)
  12. `feat(be/events): implement event creation and multipart ticket tier ingestion` (Ticket #06)
  13. `feat(fe/events): build organizer event creation and edit form with zod validation` (Ticket #06)
  14. `feat(events): implement soft deletion and active transaction validation guards` (Ticket #06)
  15. `feat(vouchers): implement promotional voucher creation with multi-event binding` (Ticket #07)
  16. `feat(vouchers): add voucher management table and soft-deletion workflow` (Ticket #07)
  17. `feat(be/checkout): implement 14-step acid checkout transaction and seat reservation` (Ticket #08)
  18. `feat(fe/checkout): build checkout summary modal with voucher, coupon, and points selectors` (Ticket #08)
  19. `feat(payment): implement payment proof image upload and 2-hour countdown timer` (Ticket #09)
  20. `feat(tickets): build customer transaction dashboard with lazy expiry status badges` (Ticket #09)
  21. `feat(verification): implement organizer accept/reject workflow and atomic inventory rollback` (Ticket #10)
  22. `feat(attendance): add post-event attendance toggle and review unlocking guard` (Ticket #10)
  23. `feat(reviews): implement attendee star rating and comment submission` (Ticket #11)
  24. `feat(be/analytics): implement sql time-series aggregation queries for revenue and attendance` (Ticket #12)
  25. `feat(fe/dashboard): build recharts analytical dashboard with date range filtering` (Ticket #12)

## Comments

- Initial specification created from relentless grilling session and domain modeling analysis on 2026-07-25.
