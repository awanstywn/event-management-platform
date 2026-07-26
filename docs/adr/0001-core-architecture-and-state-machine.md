# 0001. Core Architecture and Transaction State Machine

Date: 2026-07-25

## Status

Accepted

## Context

We are building the **Event Management Platform** as a monorepo (`apps/web` for Next.js and `apps/api` for Express.js + Prisma + PostgreSQL on Supabase).
To satisfy both business requirements (such as 2-hour payment expiry, discount promotions, point ledger systems, and review verification) and deployment constraints (Vercel serverless execution), several foundational architectural decisions needed to be locked down before code generation.

## Decisions

### 1. Monorepo Separation (Next.js + Express)
We use npm workspaces to split the frontend (`apps/web`) and backend (`apps/api`). The frontend is strictly for rendering and UI interactions, while the Express backend owns all domain logic, database queries, and third-party integrations (Cloudinary, Nodemailer).

### 2. Authentication via httpOnly Cookies
To prevent XSS token theft, access tokens (15-minute expiry) and refresh tokens (7-day expiry) are stored exclusively in secure, httpOnly, SameSite cookies. The frontend Axios instance uses an interceptor to automatically hit `/api/v1/auth/refresh` when a `401 Unauthorized` response occurs.

### 3. Lazy Evaluation for Timers (No Background Workers)
Because Vercel serverless functions do not support persistent processes or `setTimeout` timers, expiry events are evaluated lazily at query time:
- When a `Transaction` is read, if `status === WAITING_FOR_PAYMENT` and `now > paymentDeadline` (2 hours), the backend automatically updates the status to `EXPIRED` and triggers the rollback mechanism.
- Similarly, if `status === WAITING_FOR_ADMIN_CONFIRMATION` and `now > confirmationDeadline` (3 days), it auto-updates to `CANCELED` and rolls back.

### 4. Point Ledger with FIFO Debit
Point balances are not stored as a mutable integer on the user. Instead, we use a `PointLedger` table recording immutable `CREDIT` and `DEBIT` entries. Each credit expires 3 months from creation. When spending points, debits consume available credits in FIFO order (oldest expiring first).

### 5. Transaction State Machine & Rollback
Transactions follow exactly six states: `WAITING_FOR_PAYMENT`, `WAITING_FOR_ADMIN_CONFIRMATION`, `DONE`, `REJECTED`, `EXPIRED`, `CANCELED`.
- Seats/quotas are reserved immediately upon creation.
- If a transaction moves to a terminal failure state (`REJECTED`, `EXPIRED`, `CANCELED`), a database transaction (`prisma.$transaction`) atomically restores seat quotas, reverses point debits, decrements voucher usage counts, and unmarks coupons.

### 6. Review Eligibility via Attendance Flag
Attendance is independent of the transaction payment flow. An organizer toggles `isAttended = true` on a `DONE` transaction after the event's `endDate` has passed. A customer can only write a review if they have a transaction where `status === DONE` AND `isAttended === true`.

## Consequences

- All backend database mutations involving transactions, vouchers, or points must use `prisma.$transaction()` to guarantee atomic rollbacks.
- Every read query on `Transaction` in repositories must invoke the lazy expiry check helper before returning data to controllers.
- The UI must build all components from scratch using Tailwind CSS v4, adhering to our dark-mode-first design tokens without relying on external UI libraries like shadcn/ui.
