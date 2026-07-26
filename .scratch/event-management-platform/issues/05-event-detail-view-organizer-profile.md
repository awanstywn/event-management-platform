# 05 — Event Detail View & Public Organizer Profile

**What to build:** Detailed event view page displaying full description, banner image, date/location, ticket tiers (or flat price/free badge), available seat counts, and a link to the Organizer's public profile page showing their bio, aggregate rating, and complete catalog of published events.

**Blocked by:** 04 — Event Discovery & Catalog Browsing

**Status:** ready-for-agent

- [ ] Implement `GET /api/v1/events/:id` returning complete event information including ticketTypes array, organizer summary, average rating, and total review count
- [ ] Implement `GET /api/v1/users/:id/profile` returning organizer public profile data, aggregate star rating across all events, and paginated list of their published events
- [ ] Build UI event detail page (`app/events/[id]/page.tsx`) displaying banner header, location/time details, descriptive text, and ticket tier breakdown
- [ ] Build `TicketSelector` component showing ticket types with prices/quotas, flat price, or free badge, with login redirection for unauthenticated visitors
- [ ] Build public organizer profile page (`app/organizers/[id]/page.tsx`) displaying organizer avatar, join date, rating badge, and grid of published events
- [ ] Integrate display-only `StarRating` component and review preview section on event detail and organizer profile pages
- [ ] TDD: Write integration test suites (`apps/api/__tests__/integration/events/get-event.test.ts`, `apps/api/__tests__/integration/users/organizer-profile.test.ts`) via supertest verifying relational data fetching and 404 fallbacks for invalid or soft-deleted IDs
- [ ] TDD: Write React Testing Library unit test (`apps/web/__tests__/components/TicketSelector.test.tsx`) verifying price display and login redirection for unauthenticated visitors
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

