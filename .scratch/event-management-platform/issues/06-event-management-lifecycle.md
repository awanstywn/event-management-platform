# 06 — Event Management Lifecycle (Create, Edit, Soft-Delete)

**What to build:** Organizer dashboard event management allowing organizers to create new events with banner image uploads and dynamic ticket tiers (where totalSeats equals sum of tier quotas), edit existing event details without reducing seat quotas below already sold counts, and soft-delete events that have no active pending transactions after an interactive confirmation dialog.

**Blocked by:** 02 — Authentication & Session Management, 05 — Event Detail View & Public Organizer Profile

**Status:** ready-for-agent

- [ ] Implement `POST /api/v1/events` endpoint accepting multipart/form-data with bannerImage upload to Cloudinary and JSON-stringified `ticketTypes` array, creating Event and TicketType records within an atomic `prisma.$transaction()`
- [ ] Implement `PUT /api/v1/events/:id` allowing organizers to update event details and replace banner images (automatically deleting previous image from Cloudinary), enforcing validation that quotas cannot drop below sold ticket counts
- [ ] Implement `DELETE /api/v1/events/:id` executing soft deletion (`deletedAt = now()`) and rejecting deletion if active pending transactions exist (`WAITING_FOR_PAYMENT` or `WAITING_FOR_ADMIN_CONFIRMATION`)
- [ ] Build organizer event list page (`app/dashboard/events`) displaying table of organizer's events with status badges and edit/delete action buttons
- [ ] Build event creation (`app/dashboard/events/new`) and editing (`app/dashboard/events/[id]`) form pages with dynamic ticket tier input rows, datetime pickers, image dropzone, and interactive confirmation dialogs
- [ ] TDD: Write integration test suites (`apps/api/__tests__/integration/events/create-event.test.ts`, `update-event.test.ts`, `delete-event.test.ts`) via supertest with Cloudinary mocks verifying atomic transaction creation, quota reduction protection, and active transaction soft-delete blocking
- [ ] TDD: Write React Testing Library unit test (`apps/web/__tests__/components/EventForm.test.tsx`) verifying dynamic ticket tier row addition/removal and date validation errors
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

