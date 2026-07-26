# 08 — Ticket Purchasing & Immediate Entitlement Reservation

**What to build:** Core ticket checkout workflow for Customers where selecting ticket quantities and applying either a voucher OR a coupon alongside available ledger points dynamically calculates the final discounted price. Submitting the order executes an atomic database transaction that reserves seat quotas immediately, consumes point credits in FIFO order, increments promotion usage counts, sets a 2-hour payment deadline, and redirects to the ticket status view.

**Blocked by:** 02 — Authentication & Session Management, 05 — Event Detail View & Public Organizer Profile, 07 — Promotional Campaigns (Vouchers, Referral Coupons & Point Ledger)

**Status:** ready-for-agent

- [ ] Implement `POST /api/v1/transactions` validating event seat availability, ticket tier quotas, voucher/coupon eligibility (mutual exclusivity), and available point balance
- [ ] Build atomic transaction logic in `prisma.$transaction()` decrementing `Event.availableSeats` (or `TicketType.availableQuota`), incrementing voucher `usageCount`, marking coupon `isUsed = true`, creating FIFO `PointLedger` debit entries, creating `Transaction` / `TransactionItem` records, and setting `paymentDeadline = now() + 2 hours`
- [ ] Enforce free event purchase logic where `isFree = true` bypasses discount inputs, sets `finalPrice = 0`, immediately transitions status to `DONE`, generates booking code, and sends confirmation email
- [ ] Build checkout summary UI component (`CheckoutSummary`) on event detail page with real-time price calculation: `finalPrice = max(0, totalPrice - discount - pointsUsed)`
- [ ] Implement interactive confirmation dialog ("Confirm purchase?") before submitting ticket order and redirecting Customer to `/my-tickets/[id]`
- [ ] TDD: Write integration test suite (`apps/api/__tests__/integration/transactions/create-transaction.test.ts`) via supertest verifying 14-step atomic transaction, seat decrement, voucher/coupon mutual exclusivity, FIFO point consumption, and price formula accuracy
- [ ] TDD: Write React Testing Library unit test (`apps/web/__tests__/components/CheckoutSummary.test.tsx`) verifying mutual exclusivity of voucher/coupon inputs and points slider max limit
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

