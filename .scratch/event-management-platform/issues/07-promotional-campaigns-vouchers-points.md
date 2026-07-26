# 07 — Promotional Campaigns (Vouchers, Referral Coupons & Point Ledger)

**What to build:** Promotion engine enabling Organizers to create percentage or fixed discount vouchers with usage caps and date restrictions linked to their events, and soft-delete vouchers. For Customers, it delivers the referral reward workflow where registering with a referral code automatically generates a 10% coupon for the new user and credits 10,000 points (3-month expiry) to the referrer's immutable point ledger.

**Blocked by:** 02 — Authentication & Session Management, 06 — Event Management Lifecycle (Create, Edit, Soft-Delete)

**Status:** ready-for-agent

- [ ] Implement `POST /api/v1/vouchers` allowing Organizers to create percentage (max 100%) or fixed discount vouchers with usage limits, date ranges, and associated event IDs linked via `VoucherEvent` junction table in an atomic transaction
- [ ] Implement `GET /api/v1/vouchers` and `DELETE /api/v1/vouchers/:id` (soft delete) for organizer voucher management
- [ ] Implement point ledger calculation logic computing available customer point balance by summing non-expired `CREDIT` entries minus `DEBIT` entries
- [ ] Build organizer voucher management UI (`app/dashboard/vouchers` and `app/dashboard/vouchers/new`) with auto-uppercasing code input, multi-select event dropdown, and confirmation dialog on deletion
- [ ] Verify referral code reward integration during user registration ensuring 10,000 points are credited to referrer and a 10% single-use coupon is assigned to the new user
- [ ] TDD: Write integration test suites (`apps/api/__tests__/integration/vouchers/create-voucher.test.ts`, `manage-vouchers.test.ts`) via supertest verifying organizer ownership validation, voucher date limits, soft deletion, and point ledger calculation logic
- [ ] TDD: Write React Testing Library unit test (`apps/web/__tests__/components/VoucherForm.test.tsx`) verifying auto-uppercasing code input and multi-select event dropdown
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

