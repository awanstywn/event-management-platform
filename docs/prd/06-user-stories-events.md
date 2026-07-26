# User Stories — Event Discovery & Management

---

## US-07: Landing Page

**Story:** As a visitor, I want to see a landing page with featured upcoming events, so that I can quickly discover interesting events.
**Traceability:** Mapped to Spec Story #10 | Issue Ticket #04

**Context:** The first page users see. Should be visually striking (hero section) and immediately show events. No auth required.

**Roles:** All (public).

**Assumptions:** Events are pre-seeded. Only `PUBLISHED` events with `endDate > now` are shown.

**Out of Scope:** Personalized recommendations, geolocation-based filtering.

**Happy Path:**
1. Visitor opens `/`
2. Sees hero section with search bar and CTA
3. Sees "Upcoming Events" section (6 most recent upcoming events, sorted by startDate ASC)
4. Sees "Browse by Category" section (8 category cards)
5. Clicks an event card → redirected to `/events/[id]`
6. Clicks "View All Events" → redirected to `/events`
7. Types in search bar (debounced 300ms) → redirected to `/events?search=query`

**Unhappy Paths:**
- **N/A (Graceful Degradation):** As a public read-only catalog view, absence of events or categories is not treated as an application error. If zero upcoming events exist in the database, the UI renders an `EmptyState` component cleanly without throwing HTTP errors or blocking rendering.

**Business Constraints:**
- Must only display events with `status === 'PUBLISHED'`, `deletedAt === null`, and `endDate > now()`.
- Limited strictly to the 6 soonest upcoming events sorted by `startDate ASC`.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Database / Server unavailable | 500 | `"Failed to load upcoming events. Please try again later."` |

**Acceptance Criteria:**
- Given a visitor accessing `/`, When the page loads, Then the top 6 upcoming published events and 8 category cards are displayed.
- Given an event card is clicked, When selected, Then the browser navigates to `/events/[id]`.
- Given 0 upcoming events in the database, When loaded, Then a friendly empty state message is rendered.

**Definition of Done:**
- [ ] Responsive grid layout (1 col mobile, 2 cols tablet, 3 cols desktop)
- [ ] TanStack Query fetches data with skeleton loaders
- [ ] Empty state component rendered when event list is empty

### Technical Tasks

**Frontend:**
- **Page:** `app/page.tsx`
- **Components:**
  - `components/events/EventCard.tsx` — reusable card: banner, name, date, location, price/free badge, category badge
  - `components/events/EventGrid.tsx` — responsive grid layout (1 col mobile, 2 cols tablet, 3 cols desktop)
  - `components/events/SearchBar.tsx` — search input with debounce hook, navigates to `/events?search=`
  - `components/events/CategoryFilter.tsx` — horizontal scrollable category chips
- **Data fetching:** TanStack Query `useQuery` calling `getEvents({ limit: 6, sortBy: "startDate", order: "asc" })`
- **UI States:** Skeleton loading for event cards. Empty state if no events.

**Backend:**
- Uses `GET /api/v1/events` with `limit=6` and `GET /api/v1/categories` (both public, no auth)

**Database:**
- **N/A (Read-only query):** Performs a read-only indexed SELECT query on `Event` and `Category` tables; no transactional writes or schema modifications occur.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/events/list-events.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createEvent()` and `createCategory()` to seed test data
- **Required Assertions:**
  - Assert 200 OK with `limit=6` → returns max 6 events sorted by `startDate ASC`
  - Assert only `PUBLISHED` events with `deletedAt === null` and `endDate > now()` are returned
  - Assert 200 OK with empty DB → returns `data: [], meta.total: 0`

**Frontend Tests:**
- **File:** `apps/web/__tests__/components/EventCard.test.tsx`
- **Library:** React Testing Library
- **Required Assertions:**
  - Assert EventCard renders event name, date, location, and price/free badge
  - Assert EmptyState component renders when event list is empty

---

## US-08: Event Browsing with Search, Filter, Sort, Pagination

**Story:** As a visitor, I want to browse events with search, filters, sorting, and pagination, so that I can find events that interest me.
**Traceability:** Mapped to Spec Stories #11, #12, #13, #14 | Issue Ticket #04

**Context:** Main discovery page. All search/filter/sort/pagination processed on the backend.

**Roles:** All (public).

**Assumptions:**
- 10 items per page fixed pagination limit.
- Database has composite indexes on `status` and `deletedAt`.

**Out of Scope:** Map view, saved searches, notifications for new events.

**Happy Path:**
1. Visitor navigates to `/events`
2. Sees search bar, filter panel, sort dropdown, event grid
3. Types in search bar → 300ms debounce → API call with `search` param
4. Selects category filter → API call with `category` param
5. Toggles "Free events" → API call with `isFree=true`
6. Sets price range → API call with `minPrice` and `maxPrice`
7. Changes sort → API call with `sortBy` and `order`
8. Clicks page 2 → API call with `page=2`
9. All filters reflected in URL query params (shareable/bookmarkable)

**Unhappy Paths:**
- Search query or filter combination yields 0 results → The UI renders an `EmptyState` component with a "Clear all filters" CTA button without throwing an API error.
- Invalid pagination parameter (e.g. `page=-5` or `page=abc`) → Backend Zod query validator resets or rejects query with 400 Bad Request.

**Business Constraints:**
- Search: matches `event.name` and `event.description` (case-insensitive)
- Debounce: 300ms minimum before triggering API call
- Pagination: 10 events per page
- Default sort: `startDate` ascending (soonest first)
- Only `PUBLISHED` events with `deletedAt IS NULL`
- Empty state: "No events found" with "Clear filters" button

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Invalid query params (non-numeric page/price) | 400 | Validation errors |
| No results | 200 | `data: [], meta.total: 0` (not an error) |

**Acceptance Criteria:**
- Given a visitor on `/events`, When typing a search string, Then results filter after a 300ms debounce and URL search params update.
- Given category or price filters applied, When queried, Then only matching published events are returned.
- Given a search yielding 0 matches, When rendered, Then an Empty State component with a "Clear filters" button is displayed.

**Definition of Done:**
- [ ] URL query string sync implemented with `useSearchParams()`
- [ ] 300ms debounce hook integrated into search input
- [ ] Pagination controls render total pages from API metadata correctly
- [ ] Prisma dynamic WHERE query builder tested against all filter combinations

### Technical Tasks

**Frontend:**
- **Page:** `app/events/page.tsx`
- **Components:**
  - `components/events/EventFilter.tsx` — filter panel: category multi-select, location input, free toggle, price range
  - `components/events/SearchBar.tsx` — with `useDebounce(300)` hook
  - `components/events/EventGrid.tsx` — responsive grid
  - `components/events/EventCard.tsx`
  - `components/ui/Pagination.tsx`
  - `components/ui/EmptyState.tsx`
  - `components/ui/Skeleton.tsx` — skeleton loading for cards
- **State:** URL search params as single source of truth. `useSearchParams()` to read/write. TanStack Query with params as query key.
- **Hook:** `hooks/useDebounce.ts`
  ```ts
  export function useDebounce<T>(value: T, delay: number = 300): T {
    const [debouncedValue, setDebouncedValue] = useState(value);
    useEffect(() => {
      const timer = setTimeout(() => setDebouncedValue(value), delay);
      return () => clearTimeout(timer);
    }, [value, delay]);
    return debouncedValue;
  }
  ```

**Backend:**
- **Endpoint:** `GET /api/v1/events` (see 03-api-endpoints.md for full query params)
- **Repository:** Uses Prisma `where` with dynamic conditions, `orderBy`, `skip`, `take`
  ```ts
  // Example query building
  const where: Prisma.EventWhereInput = {
    deletedAt: null,
    status: "PUBLISHED",
    ...(search && {
      OR: [
        { name: { contains: search, mode: "insensitive" } },
        { description: { contains: search, mode: "insensitive" } },
      ],
    }),
    ...(category && { category: { slug: { in: category.split(",") } } }),
    ...(isFree !== undefined && { isFree: isFree === "true" }),
    ...(minPrice && { price: { gte: Number(minPrice) } }),
    ...(maxPrice && { price: { lte: Number(maxPrice) } }),
    ...(startDate && { startDate: { gte: new Date(startDate) } }),
    ...(endDate && { startDate: { lte: new Date(endDate) } }),
  };
  ```

**Database:**
- **N/A (Read-only query):** Dynamic read-only query using Prisma `findMany` and `count`; no transactional writes or schema modifications occur.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/events/browse-events.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createEvent()` with various categories, prices, and dates
- **Required Assertions:**
  - Assert 200 OK with `search=keyword` → only events matching `name` or `description` returned (case-insensitive)
  - Assert 200 OK with `category=music` → only events in that category returned
  - Assert 200 OK with `isFree=true` → only free events returned
  - Assert 200 OK with `page=2&limit=10` → correct pagination metadata (`meta.total`, `meta.page`, `meta.totalPages`)
  - Assert 200 OK with no matching results → `data: [], meta.total: 0`
  - Assert 400 Bad Request with invalid query params (e.g., `page=-1`)

---

## US-09: Event Detail Page

**Story:** As a visitor, I want to view an event's full details, so that I can decide whether to attend.
**Traceability:** Mapped to Spec Story #15 | Issue Ticket #05

**Context:** Detailed view of a single event. Shows all info, ticket types (if any), organizer info, and reviews.

**Roles:** All (public for viewing). Buy button only for logged-in CUSTOMER.

**Assumptions:**
- Event ID corresponds to an existing record in the database.
- Event status is `PUBLISHED` and `deletedAt` is null.

**Out of Scope:** Social sharing buttons, event following/favoriting, waitlists.

**Happy Path:**
1. Visitor navigates to `/events/[id]`
2. Sees: banner image, name, category badge, date/time, location, description
3. Sees: ticket section — if ticket types exist, shows each type with name/price/available; if flat price, shows single price; if free, shows "Free" badge
4. Sees: organizer card (name, picture, link to `/organizers/[id]`)
5. Sees: reviews section (average rating, review list with pagination)
6. If logged in as CUSTOMER: "Get Tickets" button opens ticket selection
7. If not logged in: "Get Tickets" button redirects to `/login`

**Unhappy Paths:**
- Event ID does not exist or is soft-deleted (`deletedAt !== null`) → Backend returns 404 Not Found; frontend renders a custom 404 Not Found page.
- Logged in as ORGANIZER attempting to purchase tickets → "Get Tickets" CTA button is disabled or hidden with tooltip *"Organizers cannot purchase tickets"*.

**Business Constraints:**
- Buy button disabled if `availableSeats === 0` across all ticket tiers (displays "Sold Out" badge).
- Reviews list must show paginated reviews sorted by newest first.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Invalid UUID / ID format | 400 | `"Invalid event ID format"` |
| Event not found or soft-deleted | 404 | `"Event not found"` |

**Acceptance Criteria:**
- Given a valid event ID, When navigating to `/events/[id]`, Then the full event details, ticket tiers, organizer summary, and reviews are displayed.
- Given a sold-out event, When viewed, Then a "Sold Out" badge is shown and ticket purchase controls are disabled.
- Given an invalid or deleted event ID, When requested, Then a 404 error page is rendered.

**Definition of Done:**
- [ ] Dynamic routing page `app/events/[id]/page.tsx` created
- [ ] TicketSelector component handles free, flat-rate, and tiered pricing modes
- [ ] Review list displays pagination and star rating components
- [ ] 404 Not Found fallback verified for non-existent UUIDs

### Technical Tasks

**Frontend:**
- **Page:** `app/events/[id]/page.tsx`
- **Components:**
  - `components/events/TicketSelector.tsx` — shows ticket types or flat price, quantity selector, "Buy" button
  - `components/reviews/ReviewList.tsx` — paginated review cards
  - `components/reviews/ReviewCard.tsx` — user name, rating stars, comment, date
  - `components/ui/StarRating.tsx` — display-only mode (filled stars)
- **Data fetching:** `useQuery` for event detail + `useQuery` for reviews (separate, paginated)

**Backend:**
- **Endpoint:** `GET /api/v1/events/:id` (includes ticketTypes, averageRating, totalReviews)
- **Endpoint:** `GET /api/v1/reviews?eventId=xxx` (paginated)

**Database:**
- **N/A (Read-only query):** Relational read query fetching `Event` with nested `Category`, `TicketType`, and `User` (organizer) relations; no schema mutation occurs.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/events/get-event.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createEvent()` with ticket types, organizer, and reviews
- **Required Assertions:**
  - Assert 200 OK with valid event ID → response includes event details, ticket types, organizer info, average rating
  - Assert 404 Not Found with non-existent UUID
  - Assert 404 Not Found with soft-deleted event (`deletedAt !== null`)
  - Assert 400 Bad Request with invalid ID format

---

## US-10: Create Event (Organizer)

**Story:** As an Organizer, I want to create a new event with full details, so that it becomes visible to customers on the discovery page.
**Traceability:** Mapped to Spec Stories #16, #17 | Issue Ticket #06

**Context:** Entry point for event creation. Once created, the event is `PUBLISHED` immediately.

**Roles:** ORGANIZER only.

**Assumptions:** Categories are pre-seeded. Organizer is authenticated.

**Out of Scope:** Draft/publish toggle, scheduling publication, duplicating events.

**Happy Path:**
1. Organizer navigates to `/dashboard/events/new`
2. Fills form: name, category (dropdown), location, description, start datetime, end datetime, isFree toggle
3. If `isFree = false`: shows price field OR "Add Ticket Types" option
4. If ticket types: adds rows (name, price, quota) — can add/remove dynamically
5. Optionally uploads banner image (drag-and-drop or click)
6. Client Zod validation passes
7. **Confirm dialog** → confirms
8. `POST /api/v1/events` as `multipart/form-data`
9. Backend creates Event + TicketType rows in transaction, uploads banner to Cloudinary
10. Toast: "Event created successfully"
11. Redirect to `/dashboard/events/[id]`

**Business Constraints:**
- `name`: 3-100 characters
- `description`: max 2000 characters
- `endDate` strictly after `startDate`
- `totalSeats`: positive integer
- `price`: required and > 0 if `isFree = false` and no ticket types
- Banner image: max 2MB, jpg/png/webp
- If ticket types provided: `totalSeats = sum of all ticket type quotas`
- `ticketTypes` sent as JSON-stringified field in multipart/form-data

**Unhappy Paths:**
- `endDate` is set before or equal to `startDate` → Zod refinement fails on client and displays inline date error.
- Paid event submitted without price or ticket tiers → Validation rejects form submission.
- Banner image upload fails or exceeds 2MB → Multer middleware rejects payload with 400 Bad Request.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Missing required field | 400 | `"{field} is required"` |
| `endDate <= startDate` | 400 | `"End date must be after start date"` |
| `price` missing, not free, no ticket types | 400 | `"Price is required for paid events"` |
| Invalid image type | 400 | `"Only JPG, PNG, or WEBP images are allowed"` |
| Image > 2MB | 400 | `"Image must be smaller than 2MB"` |
| Not authenticated | 401 | `"Please log in to continue"` |
| Not ORGANIZER | 403 | `"Only event organizers can access this resource"` |

**Acceptance Criteria:**
- Given an authenticated Organizer, When submitting a valid event form with confirmation, Then the event and ticket types are created and 201 Created returned.
- Given `endDate <= startDate`, When submitted, Then Zod validation blocks request with inline error.
- Given a non-Organizer user, When attempting to access creation endpoint, Then a 403 Forbidden error is returned.

**Definition of Done:**
- [ ] Event creation form validates dates and pricing rules via Zod
- [ ] Multipart/form-data JSON parser middleware decodes `ticketTypes` array
- [ ] Confirmation dialog prompts Organizer prior to creation
- [ ] Atomic Prisma transaction creates event and ticket tiers together

### Technical Tasks

**Frontend:**
- **Page:** `app/dashboard/events/new/page.tsx`
- **Components:**
  - `components/events/EventForm.tsx` — full form with all fields
  - `components/events/DateRangePicker.tsx` — start/end datetime pickers
  - `components/ui/FileUpload.tsx` — drag-and-drop image upload
  - `components/ui/ConfirmDialog.tsx` — "Create this event?" confirmation
- **Type (`types/event.types.ts`):**
  ```ts
  export interface CreateEventFormValues {
    name: string;
    categoryId: string;
    location: string;
    description: string;
    startDate: string;
    endDate: string;
    totalSeats: number;
    isFree: boolean;
    price?: number;
    ticketTypes?: { name: string; price: number; quota: number }[];
    bannerImage?: File;
  }
  ```
- **Validation (`lib/validators/event.schema.ts`):**
  ```ts
  export const createEventSchema = z.object({
    name: z.string().min(3).max(100),
    categoryId: z.string().uuid(),
    location: z.string().min(3),
    description: z.string().max(2000),
    startDate: z.string().datetime(),
    endDate: z.string().datetime(),
    totalSeats: z.coerce.number().int().positive(),
    isFree: z.boolean(),
    price: z.coerce.number().positive().optional(),
    ticketTypes: z.array(z.object({
      name: z.string().min(1),
      price: z.coerce.number().positive(),
      quota: z.coerce.number().int().positive(),
    })).optional(),
  }).refine(d => new Date(d.endDate) > new Date(d.startDate), {
    message: "End date must be after start date", path: ["endDate"],
  }).refine(d => d.isFree || d.price !== undefined || (d.ticketTypes && d.ticketTypes.length > 0), {
    message: "Price or ticket types required for paid events", path: ["price"],
  });
  ```

**Backend:**
- **Endpoint:** `POST /api/v1/events`
- **Middleware order:** `authenticate` → `authorize(["ORGANIZER"])` → `upload.single("bannerImage")` → `parseJsonField("ticketTypes")` → `validate(createEventSchema)` → `eventController.create`
- **Service:** Generate slug, upload banner to Cloudinary, create Event + TicketType rows in `prisma.$transaction()`
- **Note:** `ticketTypes` arrives as a JSON string in multipart/form-data — must be JSON.parsed in middleware before validation

**Database:**
- **Transaction:** Execute `Event.create()` and `TicketType.createMany()` inside an atomic `prisma.$transaction()` to ensure structural consistency.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/events/create-event.test.ts`
- **Mocking:** `jest.mock()` on `cloudinaryService.upload` (returns mock banner URL)
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createUser({ role: 'ORGANIZER' })` and `createCategory()` to seed prerequisites
- **Required Assertions:**
  - Assert 201 Created with valid payload → Event + TicketType rows exist in DB
  - Assert 400 Bad Request when `endDate <= startDate`
  - Assert 400 Bad Request when paid event has no price and no ticket types
  - Assert 400 Bad Request with invalid image type or image > 2MB
  - Assert 401 Unauthorized without auth cookie
  - Assert 403 Forbidden when authenticated as CUSTOMER
  - Assert `cloudinaryService.upload` called with banner buffer

**Frontend Tests:**
- **File:** `apps/web/__tests__/components/EventForm.test.tsx`
- **Library:** React Testing Library
- **Required Assertions:**
  - Assert Zod validation error displayed when `endDate <= startDate`
  - Assert ticket type rows can be dynamically added and removed
  - Assert confirmation dialog appears on form submit

---

## US-11: Edit Event (Organizer)

**Story:** As an Organizer, I want to edit my event's details, so that I can keep information accurate.
**Traceability:** Mapped to Spec Story #18 | Issue Ticket #06

**Context:** Edit existing event. All fields editable except `organizerId`.

**Roles:** ORGANIZER (own events only).

**Assumptions:**
- Authenticated Organizer is the owner (`organizerId == user.id`) of the event being edited.

**Out of Scope:** Bulk editing, event duplication, changing event ownership.

**Happy Path:**
1. Organizer navigates to `/dashboard/events/[id]`
2. Form pre-filled with current data
3. Edits fields, optionally uploads new banner
4. **Confirm dialog** → confirms
5. `PUT /api/v1/events/:id`
6. If new banner: upload to Cloudinary, delete old banner
7. Toast: "Event updated successfully"

**Unhappy Paths:**
- Attempting to reduce ticket tier `quota` below already sold ticket count → Backend returns 400 Bad Request with `"Cannot reduce quota below already sold tickets"`.
- Attempting to edit another organizer's event → Backend returns 403 Forbidden.

**Business Constraints:**
- Cannot reduce `totalSeats` or tier `quota` below already-sold count.
- Old banner deleted from Cloudinary when replaced.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Reduce quota below sold count | 400 | `"Cannot reduce quota below already sold tickets"` |
| Not event owner | 403 | `"You can only modify your own events"` |

**Acceptance Criteria:**
- Given the event owner, When submitting valid updates with confirmation, Then the event record is updated and 200 OK returned.
- Given an update reducing quota below sold tickets, When submitted, Then a 400 error is returned and changes aborted.

**Definition of Done:**
- [ ] Edit form pre-populates existing values cleanly
- [ ] Quota reduction check verified in backend service before save
- [ ] Confirmation dialog prompts before save

### Technical Tasks

**Frontend:**
- **Page:** `app/dashboard/events/[id]/edit/page.tsx`
- **Components:** Reuses `EventForm.tsx` in edit mode

**Backend:**
- **Endpoint:** `PUT /api/v1/events/:id`
- **Middleware:** `authenticate` → `authorize(["ORGANIZER"])` → `upload.single("bannerImage")` → `validate(updateEventSchema)` → `eventController.update`

**Database:**
- **N/A (Standard Update):** Executes standard `Event.update()` and `TicketType.upsert()` queries; quota verification check is performed via read prior to write.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/events/update-event.test.ts`
- **Mocking:** `jest.mock()` on `cloudinaryService.upload` and `cloudinaryService.delete`
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createEvent({ organizerId })` with ticket types and sold tickets
- **Required Assertions:**
  - Assert 200 OK with valid update payload → Event row updated in DB
  - Assert 400 Bad Request when reducing quota below already-sold ticket count
  - Assert 403 Forbidden when updating another organizer's event

---

## US-12: Delete Event (Organizer)

**Story:** As an Organizer, I want to soft-delete my event when it's no longer needed.
**Traceability:** Mapped to Spec Story #19 | Issue Ticket #06

**Context:** Soft-deletion workflow to remove events from discovery while preserving accounting history.

**Roles:** ORGANIZER (own events only).

**Assumptions:**
- Organizer is authenticated and owns the target event.

**Out of Scope:** Hard deletion / permanent database purge.

**Happy Path:**
1. Organizer clicks "Delete" on event list or detail
2. **Confirm dialog**: "Are you sure? This action cannot be undone."
3. `DELETE /api/v1/events/:id`
4. Backend sets `deletedAt = now()` (soft delete)
5. Toast: "Event deleted successfully"
6. Redirect to `/dashboard/events`

**Unhappy Paths:**
- Event has active pending or confirmed transactions (`WAITING_FOR_PAYMENT` or `DONE`) → Backend rejects deletion with 400 Bad Request.

**Business Constraints:**
- Cannot delete if active transactions exist (WAITING_FOR_PAYMENT or WAITING_FOR_ADMIN_CONFIRMATION).

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Active transactions exist | 400 | `"Cannot delete event with active transactions"` |
| Not own event | 403 | `"You can only manage your own events"` |

**Acceptance Criteria:**
- Given an event with 0 active transactions, When deleted with confirmation, Then `deletedAt` timestamp is set and 200 OK returned.
- Given an event with active transactions, When deletion attempted, Then API returns 400 error and event remains published.

**Definition of Done:**
- [ ] Delete CTA triggers confirmation modal
- [ ] Backend checks transaction count before soft-deleting
- [ ] Soft-deleted events immediately disappear from catalog queries

### Technical Tasks

**Frontend:**
- **Page:** `app/dashboard/events/page.tsx`
- **Components:** Delete button + `ConfirmDialog.tsx`

**Backend:**
- **Endpoint:** `DELETE /api/v1/events/:id`
- **Service:** Verify ownership → check active transaction count → set `deletedAt: new Date()`

**Database:**
- **Repository:** Executes `Event.update({ where: { id }, data: { deletedAt: new Date(), status: 'CANCELLED' } })` to perform soft deletion.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/events/delete-event.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createEvent()` and optionally `createTransaction({ status: 'WAITING_FOR_PAYMENT' })`
- **Required Assertions:**
  - Assert 200 OK with 0 active transactions → `deletedAt` is set on Event row
  - Assert 400 Bad Request when active transactions exist → event remains published
  - Assert 403 Forbidden when deleting another organizer's event
  - Assert soft-deleted event no longer appears in `GET /api/v1/events` catalog query

---

## US-13: Public Organizer Profile

**Story:** As a visitor, I want to view an organizer's public profile with their events and ratings, so that I can assess their reputation.
**Traceability:** Mapped to Spec Story #20 | Issue Ticket #05

**Context:** Public reputation and discovery page for event organizers.

**Roles:** All (public).

**Assumptions:**
- Target ID corresponds to a user with `role === 'ORGANIZER'`.

**Out of Scope:** Direct messaging organizer, following organizer, reporting organizer profile.

**Happy Path:**
1. Visitor navigates to `/organizers/[id]` (from event detail "Organizer" link)
2. Sees: organizer name, profile picture, join date
3. Sees: average rating (across all events), total reviews count
4. Sees: list of their published events (paginated, 10 per page)
5. Sees: recent reviews across their events

**Unhappy Paths:**
- Target UUID does not exist or user is a Customer → Backend returns 404 Not Found; UI renders 404 error page.

**Business Constraints:**
- Aggregate rating calculated across all reviews for all events hosted by this organizer.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Organizer not found / not ORGANIZER role | 404 | `"Organizer profile not found"` |

**Acceptance Criteria:**
- Given a valid organizer ID, When navigating to `/organizers/[id]`, Then their profile, aggregate rating, and published events are displayed.
- Given a Customer ID or non-existent UUID, When requested, Then a 404 error page is rendered.

**Definition of Done:**
- [ ] Public organizer page renders aggregate star ratings correctly
- [ ] Event grid filters strictly to events owned by this organizer
- [ ] 404 fallback verified for invalid IDs

### Technical Tasks

**Frontend:**
- **Page:** `app/organizers/[id]/page.tsx`

**Backend:**
- **Endpoint:** `GET /api/v1/users/:id/profile`
- Returns organizer info + their events + aggregate rating

**Database:**
- **N/A (Read-only query):** Relational aggregation read query fetching Organizer profile with nested event count and review rating averages; no schema mutation occurs.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/users/organizer-profile.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createUser({ role: 'ORGANIZER' })` with events and reviews
- **Required Assertions:**
  - Assert 200 OK with valid organizer ID → profile includes aggregate rating and event list
  - Assert 404 Not Found with Customer ID
  - Assert 404 Not Found with non-existent UUID
