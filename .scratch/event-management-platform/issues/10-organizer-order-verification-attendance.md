# 10 — Organizer Order Verification & Attendance Marking

**What to build:** Organizer transaction dashboard to view orders across events, filter by status, inspect payment proof images, and accept or reject orders after a confirmation dialog. Accepting an order generates a unique alphanumeric booking code (`EVT-YYYY-XXXXXX`) and sends ticket confirmation email. Rejecting triggers full inventory and reward rollback with explanation email. Also delivers lazy 3-day auto-cancellation for unattended orders and attendance toggles for completed events whose end date has passed.

**Blocked by:** 09 — Payment Proof Upload & Lazy-Evaluated Serverless Expiry Timers

**Status:** ready-for-agent

- [ ] Implement query-time lazy evaluation check for `status === WAITING_FOR_ADMIN_CONFIRMATION` and `now() > confirmationDeadline` (3 days), auto-updating to `CANCELED` and executing atomic inventory/reward rollback
- [ ] Implement `PATCH /api/v1/transactions/:id/accept` generating booking code (`EVT-YYYY-XXXXXX`), setting status to `DONE`, and sending asynchronous ticket confirmation email
- [ ] Implement `PATCH /api/v1/transactions/:id/reject` accepting optional rejection reason, setting status to `REJECTED`, rolling back seats/points/vouchers/coupons in `prisma.$transaction()`, and sending rejection email
- [ ] Implement `GET /api/v1/events/:id/attendees` and `PATCH /api/v1/transactions/:id/attend` allowing Organizers to view completed ticket holders and toggle `isAttended = true` only after `event.endDate < now()`
- [ ] Build Organizer transaction dashboard (`app/dashboard/transactions`) with sortable table, status/event filters, payment proof image inspection modal, and accept/reject confirmation dialogs
- [ ] Build attendee management page (`app/dashboard/events/[id]/attendees`) displaying attendee table with attendance checkbox (disabled with explanatory message if event is still ongoing/upcoming)
- [ ] TDD: Write integration test suites (`apps/api/__tests__/integration/transactions/manage-transactions.test.ts`, `mark-attendance.test.ts`) via supertest verifying booking code generation on accept, atomic rollback on reject, lazy 3-day SLA auto-cancellation, and attendance toggling after event end date
- [ ] TDD: Write React Testing Library unit test (`apps/web/__tests__/components/AttendeeTable.test.tsx`) verifying disabled attendance checkbox state when event is still ongoing/upcoming
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

