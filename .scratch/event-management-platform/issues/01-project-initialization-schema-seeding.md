# 01 — Project Initialization, Database Schema & Reference Data Seeding

**What to build:** Initializes the npm workspaces (`apps/web` Next.js 15 and `apps/api` Express 4), configures Prisma with PostgreSQL and applies our complete database schema (11 models + junction tables), implements automated seeding of 30+ reference records across all 8 fixed categories, and configures cross-cutting backend middleware (error handler, CORS, cookie parser, Zod validator).

**Blocked by:** None — can start immediately

**Status:** ready-for-agent

- [ ] Initialize monorepo root with `package.json` specifying npm workspaces `["apps/*"]`
- [ ] Scaffold Next.js 15 App Router frontend in `apps/web` with Tailwind CSS v4 and TypeScript
- [ ] Scaffold Express 4 backend in `apps/api` with TypeScript and CORS/cookie-parser middleware
- [ ] Configure Prisma ORM with PostgreSQL database schema covering all 11 domain models (`User`, `Category`, `Event`, `TicketType`, `Transaction`, `TransactionItem`, `Voucher`, `VoucherEvent`, `Coupon`, `PointLedger`, `Review`, `PasswordResetToken`) and enums
- [ ] Implement database soft-deletion middleware filtering `deletedAt IS NULL` by default on queries
- [ ] Build automated seed script (`prisma/seed.ts`) generating 30+ records each for events, transactions, and point ledgers across the 8 pre-seeded categories (`Music`, `Technology`, `Sports`, `Arts & Culture`, `Food & Drink`, `Business`, `Education`, `Entertainment`)
- [ ] Implement API standard response helpers (`sendSuccess`, `sendPaginated`, `sendError`) and global error handler middleware (`errorHandler.ts`)
- [ ] Configure unified Jest test runner (`ts-jest` for `apps/api`, `next/jest` for `apps/web`) with dedicated `__tests__/` directory structure
- [ ] Implement test database isolation utilities (`truncateAll()` helper in `__tests__/setup/helpers.ts`, `globalSetup.ts` running `prisma migrate deploy`)
- [ ] Configure GitHub Actions CI/CD workflow (`.github/workflows/ci.yml`) with PostgreSQL 16 service container enforcing 5 merge-blocking gates (type-check, lint, Jest backend, Jest frontend, build)
- [ ] TDD: Write integration test suite (`apps/api/__tests__/integration/core/error-handler.test.ts`) via supertest verifying standard response helpers, global error handler, and database soft-deletion middleware

