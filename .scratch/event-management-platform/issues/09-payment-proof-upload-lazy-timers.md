# 09 — Payment Proof Upload & Lazy-Evaluated Serverless Expiry Timers

**What to build:** Customer ticket detail page displaying a live 2-hour countdown timer and payment proof image upload interface. If the countdown expires before upload, the next read query lazily transitions the order to `EXPIRED` and atomically rolls back reserved seats, points, and promotional discounts. Uploading proof sets a 3-day admin confirmation deadline and transitions status to `WAITING_FOR_ADMIN_CONFIRMATION`.

**Blocked by:** 08 — Ticket Purchasing & Immediate Entitlement Reservation

**Status:** ready-for-agent

- [ ] Implement query-time lazy evaluation helper executed on all Transaction read operations (`GET /transactions` and `GET /transactions/:id`), checking if `status === WAITING_FOR_PAYMENT` and `now() > paymentDeadline`
- [ ] Build atomic rollback transaction logic triggered upon lazy expiration: updating status to `EXPIRED`, restoring seat quotas, reversing PointLedger debits with matching expiry, decrementing voucher usage, unmarking coupons, and sending expired notification email
- [ ] Implement `PATCH /api/v1/transactions/:id/upload-proof` accepting image upload (jpg/png/webp, max 2MB) to Cloudinary, updating status to `WAITING_FOR_ADMIN_CONFIRMATION`, and setting `confirmationDeadline = now() + 3 days`
- [ ] Implement `GET /api/v1/transactions` (Customer view) and `GET /api/v1/transactions/:id` endpoints returning transaction status, items, and deadline timestamps
- [ ] Build Customer ticket list page (`app/my-tickets/page.tsx`) with status filtering dropdown and paginated cards
- [ ] Build ticket detail page (`app/my-tickets/[id]/page.tsx`) with live countdown timer (`useCountdown` hook), status badges, and image dropzone with confirmation dialog before submitting proof
- [ ] TDD: Write integration test suites (`apps/api/__tests__/integration/transactions/list-transactions.test.ts`, `upload-proof.test.ts`) via supertest verifying query-time lazy expiry transition, inventory rollback, and Cloudinary image upload within deadline
- [ ] TDD: Write React Testing Library unit test (`apps/web/__tests__/components/Countdown.test.tsx`) verifying live timer format and upload button disabling upon countdown reaching 00:00:00
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

