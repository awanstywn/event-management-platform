# 04 — Event Discovery & Catalog Browsing

**What to build:** Public event browsing experience featuring the landing page (hero section, upcoming events, category cards) and the event catalog page with server-side full-text search (300ms debounce), multi-param filtering (category, location, price range, free/paid), sorting (date, price, name), and pagination (10 items per page).

**Blocked by:** 01 — Project Initialization, Database Schema & Reference Data Seeding

**Status:** ready-for-agent

- [ ] Implement `GET /api/v1/events` endpoint with query parameters for search, category slug, location, isFree, minPrice, maxPrice, date range, sortBy, order, page, and limit (default 10)
- [ ] Build Prisma repository query dynamically combining filter conditions, sorting rules, and pagination metadata, filtering only `status = PUBLISHED` and `deletedAt IS NULL`
- [ ] Implement `GET /api/v1/categories` returning all 8 pre-seeded category items
- [ ] Build public landing page (`app/page.tsx`) displaying hero banner, search bar, horizontal category chips, and grid of 6 upcoming published events
- [ ] Build event catalog page (`app/events/page.tsx`) with interactive sidebar filter panel, sort dropdown, and paginated event cards
- [ ] Implement custom `useDebounce` hook (300ms delay) on search input ensuring backend filtering without excessive API requests
- [ ] Build reusable UI components: `EventCard`, `EventGrid`, `Pagination`, `EmptyState` ("No events found" with clear filters button), and `Skeleton` loading states
- [ ] TDD: Write integration test suites (`apps/api/__tests__/integration/events/list-events.test.ts`, `browse-events.test.ts`) via supertest verifying search query matching, category/price/date filtering, sorting, and pagination metadata
- [ ] TDD: Write React Testing Library unit test (`apps/web/__tests__/components/EventCard.test.tsx`) verifying event card badge display and `EmptyState` rendering when 0 events match
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

