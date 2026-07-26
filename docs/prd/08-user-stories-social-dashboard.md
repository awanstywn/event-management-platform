# User Stories — Vouchers, Reviews & Dashboard

---

## US-19: Create Voucher (Organizer)

**Story:** As an Organizer, I want to create discount vouchers for my events, so that I can promote them.
**Traceability:** Mapped to Spec Story #33 | Issue Ticket #07

**Roles:** ORGANIZER only.

**Assumptions:** Organizer has at least one published event.

**Out of Scope:** Editing vouchers after creation, auto-generating voucher codes.

**Happy Path:**
1. Organizer navigates to `/dashboard/vouchers/new`
2. Fills form: code (manual entry, auto-uppercased), discount type (percentage or fixed), discount amount, max usage count, valid from, valid until, select events (multi-select from own events)
3. Client Zod validation passes
4. **Confirm dialog** → confirms
5. `POST /api/v1/vouchers`
6. Backend creates Voucher + VoucherEvent rows in transaction
7. Toast: "Voucher created successfully"
8. Redirect to `/dashboard/vouchers`

**Business Constraints:**
- Code: unique, 3-20 chars, uppercase
- If `discountType = PERCENTAGE`: `discountAmount` must be 1-100
- If `discountType = FIXED`: `discountAmount` must be > 0 (in IDR)
- `validUntil` must be after `validFrom`
- `maxUsage`: positive integer
- `eventIds`: at least 1 event, must all belong to the organizer

**Business Constraints:**
- Code: unique, 3-20 chars, uppercase
- If `discountType = PERCENTAGE`: `discountAmount` must be 1-100
- If `discountType = FIXED`: `discountAmount` must be > 0 (in IDR)
- `validUntil` must be after `validFrom`
- `maxUsage`: positive integer
- `eventIds`: at least 1 event, must all belong to the organizer

**Unhappy Paths:**
- Duplicate voucher code → Backend rejects creation with 409 Conflict `"Voucher code already exists"`.
- Attempting to link voucher to an event owned by another organizer → Backend returns 403 Forbidden.
- Percentage discount entered > 100% → Client-side Zod validation prevents submission and shows inline field error.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Duplicate code | 409 | `"Voucher code already exists"` |
| Event not owned | 403 | `"You can only create vouchers for your own events"` |
| Percentage > 100 | 400 | `"Discount percentage cannot exceed 100"` |
| `validUntil <= validFrom` | 400 | `"End date must be after start date"` |

**Acceptance Criteria:**
- Given an authenticated Organizer, When submitting a valid voucher form with confirmation, Then the voucher is created and linked to selected events with 201 Created returned.
- Given a duplicate code, When submitted, Then the API returns 409 Conflict.
- Given `validUntil <= validFrom`, When submitted, Then client validation blocks submission.

**Definition of Done:**
- [ ] Voucher code auto-uppercases on input
- [ ] Zod schema enforces percentage (1-100) vs fixed discount rules
- [ ] Confirmation dialog prompts Organizer before save
- [ ] Prisma transaction creates Voucher and VoucherEvent junction rows atomically

### Technical Tasks

**Frontend:**
- **Page:** `app/dashboard/vouchers/new/page.tsx`
- **Components:**
  - `components/vouchers/VoucherForm.tsx` — form with code input (auto-uppercase), type toggle (percentage/fixed), amount, dates, event multi-select
  - `components/ui/ConfirmDialog.tsx`
- **Validation (`lib/validators/voucher.schema.ts`):**
  ```ts
  export const createVoucherSchema = z.object({
    code: z.string().min(3).max(20).transform(v => v.toUpperCase()),
    discountType: z.enum(["PERCENTAGE", "FIXED"]),
    discountAmount: z.coerce.number().positive(),
    maxUsage: z.coerce.number().int().positive(),
    validFrom: z.string().datetime(),
    validUntil: z.string().datetime(),
    eventIds: z.array(z.string().uuid()).min(1, "Select at least one event"),
  }).refine(d => d.discountType !== "PERCENTAGE" || d.discountAmount <= 100, {
    message: "Percentage cannot exceed 100", path: ["discountAmount"],
  }).refine(d => new Date(d.validUntil) > new Date(d.validFrom), {
    message: "End date must be after start date", path: ["validUntil"],
  });
  ```

**Backend:**
- **Endpoint:** `POST /api/v1/vouchers`
- **Middleware:** `authenticate` → `authorize(["ORGANIZER"])` → `validate(createVoucherSchema)` → `voucherController.create`
- **Service:** Verify all eventIds belong to organizer → create Voucher + VoucherEvent rows in `prisma.$transaction()`

**Database:**
- **Transaction:** Execute `Voucher.create()` and `VoucherEvent.createMany()` inside an atomic `prisma.$transaction()` to ensure relational consistency.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/vouchers/create-voucher.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createUser({ role: 'ORGANIZER' })` and `createEvent({ organizerId })`
- **Required Assertions:**
  - Assert 201 Created with valid payload → Voucher + VoucherEvent rows exist in DB
  - Assert 400 Bad Request when event IDs don't belong to the authenticated organizer
  - Assert 400 Bad Request when `validUntil <= now()`
  - Assert 400 Bad Request when `discountPercentage > 100` or `discountAmount < 0`
  - Assert 403 Forbidden when authenticated as CUSTOMER

---

## US-20: Manage Vouchers (Organizer)

**Story:** As an Organizer, I want to view and delete my vouchers.
**Traceability:** Mapped to Spec Story #34 | Issue Ticket #07

**Context:** Promotional management dashboard table for active and expired vouchers.

**Roles:** ORGANIZER only.

**Assumptions:**
- Authenticated Organizer owns the vouchers being viewed or deleted.

**Out of Scope:** Editing voucher discount values after creation (immutable).

**Happy Path:**
1. Organizer navigates to `/dashboard/vouchers`
2. Sees table: code, discount type/amount, usage (current/max), valid dates, status (active/expired), linked events
3. Can delete (soft) a voucher → **Confirm dialog**

**Unhappy Paths:**
- Attempting to delete someone else's voucher → Backend returns 403 Forbidden.
- Organizer has 0 vouchers → Table renders `EmptyState` component cleanly.

**Business Constraints:**
- Deleting a voucher sets `deletedAt = now()` (soft delete) to preserve historical discount ledger calculations on past orders while preventing new checkouts from using the code.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Not voucher owner | 403 | `"You can only modify your own vouchers"` |
| Voucher not found | 404 | `"Voucher not found"` |

**Acceptance Criteria:**
- Given an Organizer on `/dashboard/vouchers`, When loaded, Then all owned vouchers and their usage counts are displayed.
- Given a voucher owner clicking delete with confirmation, When submitted, Then `deletedAt` is set and the code is disabled for future purchases.

**Definition of Done:**
- [ ] Table displays dynamic badges (Active, Expired, Fully Used)
- [ ] Delete CTA prompts confirmation modal
- [ ] Soft deletion verified on backend repository

### Technical Tasks

**Frontend:**
- **Page:** `app/dashboard/vouchers/page.tsx`
- **Components:**
  - `components/vouchers/VoucherTable.tsx` — table with delete action
  - `components/ui/ConfirmDialog.tsx`
  - `components/ui/Badge.tsx` — "Active" / "Expired" / "Fully Used"

**Backend:**
- **Endpoint:** `DELETE /api/v1/vouchers/:id`
- **Service:** Verify ownership → execute soft deletion (`deletedAt = now()`).

**Database:**
- **Repository:** Executes `Voucher.update({ where: { id }, data: { deletedAt: new Date() } })` to perform soft deletion.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/vouchers/manage-vouchers.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createVoucher({ organizerId })` to seed vouchers
- **Required Assertions:**
  - Assert 200 OK on list → returns only authenticated organizer's vouchers
  - Assert 200 OK on delete → `deletedAt` set on Voucher row, excluded from subsequent list queries
  - Assert 403 Forbidden when deleting another organizer's voucher
  - Assert soft-deleted voucher no longer appears in customer checkout voucher validation

---

## US-21: Write Review (Customer)

**Story:** As a Customer who attended an event, I want to leave a rating and optional comment, so that other users can assess the event.
**Traceability:** Mapped to Spec Stories #31, #32 | Issue Ticket #11

**Roles:** CUSTOMER only.

**Assumptions:** Customer has a transaction with `status = DONE` and `isAttended = true` for this event.

**Out of Scope:** Editing/deleting reviews, organizer responses to reviews, reporting inappropriate reviews.

**Happy Path:**
1. Customer navigates to `/events/[id]` for a past attended event
2. Sees "Write a Review" section (only if eligible: DONE + isAttended)
3. Selects star rating (1-5), optionally writes comment (max 500 chars)
4. **Confirm dialog** → confirms
5. `POST /api/v1/reviews`
6. Toast: "Review submitted successfully"
7. Review appears in the review list below

**Business Constraints:**
- Rating: 1-5 integer (required)
- Comment: optional, max 500 characters
- One review per customer per event
- Cannot review if not attended (no DONE + isAttended transaction)
- Reviews cannot be edited or deleted once submitted

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Not attended | 400 | `"You can only review events you have attended"` |
| Already reviewed | 409 | `"You have already reviewed this event"` |
| Rating not 1-5 | 400 | `"Rating must be between 1 and 5"` |
| Comment > 500 chars | 400 | `"Comment must be 500 characters or less"` |

**Unhappy Paths:**
- Customer did not attend (`isAttended === false` or no `DONE` transaction) → API blocks submission with 400 Bad Request; review section is hidden in UI.
- Customer already submitted a review for this event → Database unique constraint catches duplicate; API returns 409 Conflict.

**Acceptance Criteria:**
- Given an attendee with a confirmed and attended transaction, When submitting a valid review with confirmation, Then the review is saved and aggregate event/organizer ratings recalculate.
- Given an unattended user or duplicate submission, When submitted, Then the API returns 400 or 409 and blocks creation.

**Definition of Done:**
- [ ] Review section visible only to verified attendees
- [ ] StarRating component supports interactive click selection
- [ ] Confirmation dialog prompts user before submission
- [ ] Database unique constraint `@@unique([userId, eventId])` enforced

### Technical Tasks

**Frontend:**
- **Components:**
  - `components/reviews/ReviewForm.tsx` — star rating (interactive), textarea, submit button
  - `components/ui/StarRating.tsx` — interactive mode (click to set) + display mode
  - `components/ui/ConfirmDialog.tsx`
- **Validation (`lib/validators/review.schema.ts`):**
  ```ts
  export const createReviewSchema = z.object({
    eventId: z.string().uuid(),
    rating: z.coerce.number().int().min(1).max(5),
    comment: z.string().max(500).optional().or(z.literal("")),
  });
  ```

**Backend:**
- **Endpoint:** `POST /api/v1/reviews`
- **Middleware:** `authenticate` → `authorize(["CUSTOMER"])` → `validate(createReviewSchema)` → `reviewController.create`
- **Service:**
  1. Check transaction exists with `userId + eventId`, `status = DONE`, `isAttended = true`
  2. Check unique constraint (no existing review for this user + event)
  3. Create Review row

**Database:**
- **Repository:** Executes `Review.create()` on `Review` table; database schema enforces `@@unique([userId, eventId])` index preventing duplicate reviews.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/reviews/create-review.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createUser()`, `createEvent()`, `createTransaction({ status: 'DONE', isAttended: true })`
- **Required Assertions:**
  - Assert 201 Created with valid rating (1-5) and comment → Review row exists in DB
  - Assert 400 Bad Request with rating outside 1-5 range
  - Assert 400 Bad Request when customer did not attend (isAttended = false)
  - Assert 409 Conflict when duplicate review for same event+user
  - Assert 403 Forbidden when authenticated as ORGANIZER

**Frontend Tests:**
- **File:** `apps/web/__tests__/components/ReviewForm.test.tsx`
- **Library:** React Testing Library
- **Required Assertions:**
  - Assert star rating component allows selection 1-5
  - Assert submit button disabled when no rating selected
  - Assert review form hidden when `isAttended === false`

---

## US-22: Organizer Dashboard

**Story:** As an Organizer, I want to see statistics about my events, revenue, and attendees in a visual dashboard, so that I can track my performance.
**Traceability:** Mapped to Spec Stories #35, #36, #37 | Issue Ticket #12

**Context:** Business intelligence analytics dashboard with date range filtering and Recharts visualization.

**Roles:** ORGANIZER only.

**Assumptions:**
- Organizer has at least one event with transactions.
- Database contains indexed timestamps on `Transaction.createdAt`.

**Out of Scope:** Export to CSV/PDF, comparative analytics, real-time updates.

**Happy Path:**
1. Organizer navigates to `/dashboard`
2. Sees date range filter at the top (default: current year)
3. Sees 4 summary cards:
   - Total Revenue (IDR formatted)
   - Total Attendees (count)
   - Total Events (count)
   - Average Rating (stars)
4. Sees 3 charts:
   - Revenue Over Time (bar chart, grouped by month)
   - Tickets Sold Over Time (line chart, grouped by month)
   - Events by Category (donut chart)
5. Changes date range → all cards and charts update simultaneously

**Unhappy Paths:**
- Organizer has 0 events or 0 revenue in the selected date range → Charts and cards render clean zero values (`Rp 0`, `0 attendees`) and empty state placeholders without throwing query errors.

**Business Constraints:**
- All data filtered by organizer's own events only
- Revenue = sum of `finalPrice` where `status = DONE`
- Attendees = count of transactions with `status = DONE`
- All aggregation done on backend (SQL COUNT, SUM, AVG, GROUP BY)
- Charts show empty state if no data for the period
- Loading state (skeleton) on initial load
- Error state with retry button on failure

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Invalid date format | 400 | `"Invalid date format"` |
| Not ORGANIZER | 403 | `"Only event organizers can access dashboard analytics"` |

**Acceptance Criteria:**
- Given an authenticated Organizer, When accessing `/dashboard`, Then summary statistics and 3 visual Recharts render accurate aggregations for their events.
- Given date range filters applied, When queried, Then all charts and cards re-aggregate within the specified timeframe.
- Given an organizer with 0 transactions, When loaded, Then charts display empty states without errors.

**Definition of Done:**
- [ ] Recharts bar, line, and donut components responsive and styled
- [ ] Date range filter triggers TanStack Query re-fetch
- [ ] Raw SQL queries sanitized and parameterized against SQL injection
- [ ] IDR currency formatting utility tested

### Technical Tasks

**Frontend:**
- **Page:** `app/dashboard/page.tsx`
- **Components:**
  - `components/dashboard/StatCard.tsx` — card with icon, label, value, optional trend indicator
  - `components/dashboard/RevenueChart.tsx` — Recharts `BarChart` with monthly data
  - `components/dashboard/TicketSalesChart.tsx` — Recharts `LineChart` with monthly data
  - `components/dashboard/CategoryPieChart.tsx` — Recharts `PieChart` / `DonutChart`
  - `components/dashboard/DashboardDateFilter.tsx` — date range picker (start/end)
  - `components/ui/Skeleton.tsx` — for loading states on cards and charts
- **Data fetching:** TanStack Query `useQuery` with date range as params. Re-fetches on date change.
- **Chart library:** Recharts — import `BarChart`, `LineChart`, `PieChart`, `ResponsiveContainer`, etc.
- **Currency formatting:**
  ```ts
  // lib/utils.ts
  export function formatCurrency(amount: number): string {
    return new Intl.NumberFormat("id-ID", {
      style: "currency",
      currency: "IDR",
      minimumFractionDigits: 0,
    }).format(amount);
  }
  ```

**Backend:**
- **Endpoint:** `GET /api/v1/dashboard/statistics`
- **Service (`src/services/dashboard.service.ts`):**
  ```ts
  async getStatistics(organizerId: string, startDate?: Date, endDate?: Date) {
    // Summary cards
    const summary = await prisma.$queryRaw`
      SELECT
        COALESCE(SUM(t."finalPrice"), 0) as "totalRevenue",
        COUNT(t.id) as "totalAttendees",
        COUNT(DISTINCT e.id) as "totalEvents"
      FROM "Transaction" t
      JOIN "Event" e ON t."eventId" = e.id
      WHERE e."organizerId" = ${organizerId}
        AND t.status = 'DONE'
        AND t."deletedAt" IS NULL
        ${startDate ? Prisma.sql`AND t."createdAt" >= ${startDate}` : Prisma.empty}
        ${endDate ? Prisma.sql`AND t."createdAt" <= ${endDate}` : Prisma.empty}
    `;

    // Average rating
    const rating = await prisma.review.aggregate({
      where: { event: { organizerId } },
      _avg: { rating: true },
    });

    // Revenue by month
    const revenueByMonth = await prisma.$queryRaw`
      SELECT
        TO_CHAR(t."createdAt", 'YYYY-MM') as month,
        SUM(t."finalPrice") as revenue
      FROM "Transaction" t
      JOIN "Event" e ON t."eventId" = e.id
      WHERE e."organizerId" = ${organizerId}
        AND t.status = 'DONE'
        AND t."deletedAt" IS NULL
      GROUP BY TO_CHAR(t."createdAt", 'YYYY-MM')
      ORDER BY month ASC
    `;

    // Tickets sold by month
    const ticketSalesByMonth = await prisma.$queryRaw`
      SELECT
        TO_CHAR(t."createdAt", 'YYYY-MM') as month,
        COUNT(t.id) as count
      FROM "Transaction" t
      JOIN "Event" e ON t."eventId" = e.id
      WHERE e."organizerId" = ${organizerId}
        AND t.status = 'DONE'
        AND t."deletedAt" IS NULL
      GROUP BY TO_CHAR(t."createdAt", 'YYYY-MM')
      ORDER BY month ASC
    `;

    // Events by category
    const eventsByCategory = await prisma.$queryRaw`
      SELECT c.name as category, COUNT(e.id) as count
      FROM "Event" e
      JOIN "Category" c ON e."categoryId" = c.id
      WHERE e."organizerId" = ${organizerId}
        AND e."deletedAt" IS NULL
      GROUP BY c.name
      ORDER BY count DESC
    `;

    return { summary, averageRating: rating._avg.rating, revenueByMonth, ticketSalesByMonth, eventsByCategory };
  }
  ```

**Database:**
- **Technical Task:** Execute raw SQL analytical aggregation queries (`prisma.$queryRaw` and `Prisma.sql` template literals) computing time-series monthly revenue, ticket volume, category distribution, and average review ratings across organizer's events.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/analytics/dashboard.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createEvent()` with `createTransaction({ status: 'DONE' })` across multiple months
- **Required Assertions:**
  - Assert 200 OK → returns summary stats (total events, total revenue, total attendees)
  - Assert time-series data: monthly revenue and ticket aggregations are accurate across date range filter
  - Assert date range filter works correctly (`?from=2026-01-01&to=2026-06-30`)
  - Assert 200 OK with 0 events → returns zeroed summary stats and empty time-series
  - Assert 403 Forbidden when authenticated as CUSTOMER

**Frontend Tests:**
- **File:** `apps/web/__tests__/components/DashboardCharts.test.tsx`
- **Library:** React Testing Library
- **Required Assertions:**
  - Assert summary stat cards render with correct labels
  - Assert date range filter defaults to current year
  - Assert empty state component renders when no data available
