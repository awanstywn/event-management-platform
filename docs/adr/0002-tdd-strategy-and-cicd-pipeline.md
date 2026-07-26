# TDD Strategy & CI/CD Pipeline

We adopt Jest as the unified test runner across the monorepo, with integration tests at the HTTP route boundary as the primary backend seam and React Testing Library for targeted frontend component tests. CI/CD is enforced via GitHub Actions with five merge-blocking gates. This records the full set of testing and pipeline decisions made during the TDD & CI/CD grilling session.

## Status

Accepted

## Considered Options

### Test Runner
- **Jest (chosen):** Battle-tested ecosystem, official `next/jest` preset for `apps/web`, `ts-jest` for TypeScript. Heavier config for ESM but well-documented workarounds.
- **Vitest (rejected):** Faster, native ESM, but less mature ecosystem and no official Next.js preset. Rejected because the user prefers Jest's maturity and community support.

### Primary Backend Test Seam
- **Integration tests at the HTTP route boundary via supertest (chosen):** Tests the full middleware chain (auth → authorize → validate → controller → service → Prisma → PostgreSQL). Catches routing, middleware ordering, and query bugs that unit tests miss.
- **Isolated unit tests with mocked Prisma (rejected):** Faster and simpler but provides false confidence — mocking Prisma's query builder accurately is error-prone, and middleware chain bugs slip through.

### External Service Mocking
- **Service wrapper mocking via `jest.mock()` (chosen):** Thin wrappers around Cloudinary and SMTP are mocked at the module level. `nock` is available as an additional network-level interceptor when verifying outgoing HTTP request shapes.
- **Network-level-only mocking via `msw`/`nock` (rejected as sole strategy):** Adds setup complexity without meaningful confidence gain over service-level mocks for our use case.

### CI/CD Pipeline
- **GitHub Actions (chosen):** Full-featured runners, PostgreSQL 16 service containers, matrix builds. Five merge-blocking gates: TypeScript type-check, ESLint, Jest backend, Jest frontend, and production build.
- **Vercel built-in CI (rejected):** No support for running test suites, spinning up test databases, or custom pipeline steps.

### Coverage Threshold
- **No enforced threshold (chosen):** Test quality is driven by spec-defined assertions in each PRD user story's `Automated Tests` section. Coverage percentages incentivize shallow tests; explicit assertion specs ensure business-critical paths are tested.

### Frontend Testing
- **React Testing Library for targeted tests (chosen):** Tests form validation behavior, conditional rendering logic, and component state transitions. Does NOT test API data fetching (TanStack Query mocking is brittle) or visual styling.
- **E2E with Playwright/Cypress (rejected):** High confidence but slow, flaky, and heavyweight for CI.

## Consequences

- Every PRD user story (US-01 through US-22) gains an `Automated Tests` subsection under `### Technical Tasks` specifying the test seam, mocking boundaries, and required assertions.
- Tests are bundled into existing feature commits (not separate commits), keeping the 25-commit execution plan unchanged.
- Test files live in dedicated `__tests__/` directories per package, not co-located with source files.
- Test database isolation uses truncate-and-reseed between test files via a shared `truncateAll()` helper and factory functions.
- `globalSetup.ts` runs `prisma migrate deploy` once before the entire suite; each test file truncates and seeds only its minimal fixtures.
- Dependencies added: `jest`, `ts-jest`, `@types/jest`, `supertest`, `@types/supertest`, `nock`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`.
