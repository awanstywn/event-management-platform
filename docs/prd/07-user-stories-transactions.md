# User Stories — Transactions & Payments

---

## US-14: Purchase Tickets (Customer)

**Story:** As a Customer, I want to purchase tickets for an event, so that I can attend it.
**Traceability:** Mapped to Spec Stories #21, #22 | Issue Ticket #08

**Context:** The core transactional flow. Customer selects tickets, optionally applies a voucher/coupon/points, creates a transaction, then must upload payment proof within 2 hours.

**Roles:** CUSTOMER only.

**Assumptions:** Customer is logged in. Event is PUBLISHED with available seats.

**Out of Scope:** Online payment gateway integration, multiple events in one cart, ticket gifting.

**Happy Path:**
1. Customer is on `/events/[id]` and clicks "Get Tickets"
2. If ticket types exist: selects type and quantity. If flat price: selects quantity only. If free: quantity = 1.
3. Sees checkout summary: items, subtotal
4. Optionally enters voucher code or coupon code (mutually exclusive)
5. Optionally enters points to use (shows available balance)
6. Sees calculated total: `finalPrice = totalPrice - discount - pointsUsed`
7. **Confirm dialog**: "Confirm purchase?" → confirms
8. `POST /api/v1/transactions`
9. Backend validates everything, reserves seats, applies discounts in `prisma.$transaction()`
10. Returns transaction with `WAITING_FOR_PAYMENT` status and `paymentDeadline`
11. Redirect to `/my-tickets/[id]` showing 2-hour countdown

**Unhappy Paths:**
- Not enough seats → `400`
- Voucher not valid for this event → `400`
- Voucher expired or max usage reached → `400`
- Coupon already used or expired → `400`
- Both voucher and coupon provided → `400`
- Not enough points → `400`
- Event not available (ended, canceled, deleted) → `400`

**Business Constraints:**
- Seats reserved immediately (decremented on transaction creation)
- Voucher: must belong to event's organizer, be within valid dates, not exceed maxUsage
- Coupon: must belong to the customer, not used, not expired
- Points: FIFO debit (oldest-expiring first), cannot exceed available balance
- Cannot use both voucher AND coupon
- Points can be used ON TOP of voucher/coupon
- `finalPrice = max(0, totalPrice - discount - pointsUsed)`
- Free events: no voucher/coupon/points applicable, `finalPrice = 0`, status goes directly to `DONE` with booking code
- `paymentDeadline = createdAt + 2 hours`

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Not enough seats | 400 | `"Not enough available seats"` |
| Both voucher and coupon | 400 | `"Cannot use both voucher and coupon"` |
| Voucher not for this event | 400 | `"Voucher is not applicable to this event"` |
| Voucher expired | 400 | `"Voucher has expired"` |
| Voucher max usage | 400 | `"Voucher usage limit reached"` |
| Coupon already used | 400 | `"Coupon has already been used"` |
| Coupon expired | 400 | `"Coupon has expired"` |
| Insufficient points | 400 | `"Not enough points available"` |
| Event unavailable | 400 | `"Event is not available for purchase"` |
| Not authenticated | 401 | `"Please log in to continue"` |
| Not CUSTOMER | 403 | `"Only customers can purchase tickets"` |

**Acceptance Criteria:**
- Given a logged-in Customer with sufficient points/coupons, When checking out valid tickets, Then seat quota decrements, discounts apply, and an order is created in `WAITING_FOR_PAYMENT` with a 2-hour deadline.
- Given a free event checkout, When confirmed, Then status goes directly to `DONE` and a booking code is issued without requiring proof upload.
- Given insufficient seats or mutually exclusive promo codes, When submitted, Then the API returns 400 Bad Request and no database mutation occurs.

**Definition of Done:**
- [ ] TicketSelector component calculates seat subtotals and validates quantity limits
- [ ] CheckoutSummary enforces mutual exclusion between vouchers and coupons
- [ ] Points slider restricts debit to available non-expired balance
- [ ] Backend executes 14-step transaction atomically without race conditions

### Technical Tasks

**Frontend:**
- **Flow:** Event detail page → TicketSelector → CheckoutSummary → ConfirmDialog → redirect to transaction detail
- **Components:**
  - `components/events/TicketSelector.tsx` — ticket type selection with quantity spinners
  - `components/transactions/CheckoutSummary.tsx` — shows items, voucher/coupon input, points slider, price calculation
  - `components/ui/ConfirmDialog.tsx` — "Confirm purchase?"
- **Type (`types/transaction.types.ts`):**
  ```ts
  export interface CreateTransactionPayload {
    eventId: string;
    items: { ticketTypeId?: string; quantity: number }[];
    voucherCode?: string;
    couponCode?: string;
    pointsToUse?: number;
  }

  export interface TransactionResponse {
    id: string;
    totalPrice: number;
    discountAmount: number;
    pointsUsed: number;
    finalPrice: number;
    status: TransactionStatus;
    paymentProofUrl: string | null;
    bookingCode: string | null;
    isAttended: boolean;
    paymentDeadline: string;
    confirmationDeadline: string | null;
    createdAt: string;
    event: { id: string; name: string; bannerUrl: string | null; startDate: string; location: string };
    items: { ticketTypeName: string | null; quantity: number; pricePerUnit: number; subtotal: number }[];
  }

  export type TransactionStatus =
    | "WAITING_FOR_PAYMENT"
    | "WAITING_FOR_ADMIN_CONFIRMATION"
    | "DONE"
    | "REJECTED"
    | "EXPIRED"
    | "CANCELED";
  ```

**Backend:**
- **Endpoint:** `POST /api/v1/transactions`
- **Middleware:** `authenticate` → `authorize(["CUSTOMER"])` → `validate(createTransactionSchema)` → `transactionController.create`
- **Service (`src/services/transaction.service.ts`):**
  All in `prisma.$transaction()`:
  1. Validate event exists, is PUBLISHED, has available seats
  2. Validate seat availability per ticket type (if applicable)
  3. Validate voucher (if provided): exists, belongs to event's organizer, linked to this event via VoucherEvent, within dates, usage < maxUsage
  4. Validate coupon (if provided): exists, belongs to customer, not used, not expired
  5. Validate NOT both voucher and coupon
  6. Calculate price using the formula from 02-database-schema.md
  7. Validate points (if provided): sum available (non-expired CREDIT - DEBIT) >= pointsToUse
  8. Decrement `Event.availableSeats` (or `TicketType.availableQuota`)
  9. Increment `Voucher.usageCount` (if voucher)
  10. Set `Coupon.isUsed = true` (if coupon)
  11. Create PointLedger DEBIT entries (FIFO from oldest-expiring)
  12. Create Transaction row with `paymentDeadline = now + 2 hours`
  13. Create TransactionItem rows
  14. For free events: set status to `DONE`, generate bookingCode, send ticket confirmation email

**Database:**
- **Transaction:** Massive 14-step ACID transaction executing seat decrement, promotional ledger mutations (points FIFO, coupon consumption, voucher increment), and transaction item insertion in a single atomic database lock.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/transactions/create-transaction.test.ts`
- **Mocking:** `jest.mock()` on `emailService.send` (ticket confirmation for free events)
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factories `createUser()`, `createEvent()`, `createTicketType()`, `createVoucher()`, `createCoupon()`, `creditPoints()`
- **Required Assertions:**
  - Assert 201 Created with valid paid checkout → Transaction row with `WAITING_FOR_PAYMENT` status, seat count decremented, TransactionItem rows created
  - Assert 201 Created with free event → Transaction with `DONE` status, bookingCode generated, `emailService.send` called
  - Assert 400 Bad Request when not enough seats available
  - Assert 400 Bad Request when both voucher AND coupon provided
  - Assert 400 Bad Request when voucher is expired / max usage reached / not linked to event
  - Assert 400 Bad Request when coupon already used or expired
  - Assert 400 Bad Request when points exceed available balance
  - Assert `finalPrice = max(0, totalPrice - discount - pointsUsed)` formula accuracy
  - Assert 403 Forbidden when authenticated as ORGANIZER
  - Assert PointLedger DEBIT entries follow FIFO ordering (oldest-expiring consumed first)

**Frontend Tests:**
- **File:** `apps/web/__tests__/components/CheckoutSummary.test.tsx`
- **Library:** React Testing Library
- **Required Assertions:**
  - Assert voucher and coupon inputs are mutually exclusive (entering one disables the other)
  - Assert points slider max value equals available balance
  - Assert confirmation dialog appears before submission

---

## US-15: Upload Payment Proof (Customer)

**Story:** As a Customer with a pending transaction, I want to upload payment proof within 2 hours, so that my purchase can be confirmed.
**Traceability:** Mapped to Spec Stories #23, #24 | Issue Ticket #09

**Context:** Manual bank transfer verification step. Must be completed before the 2-hour reservation timer expires.

**Roles:** CUSTOMER (own transaction only).

**Assumptions:**
- Transaction exists and is currently in `WAITING_FOR_PAYMENT` state.
- Current server time is strictly before `paymentDeadline`.

**Out of Scope:** Automated bank OCR verification, direct credit card gateway integration.

**Happy Path:**
1. Customer navigates to `/my-tickets/[id]`
2. Sees 2-hour countdown timer (calculated from `paymentDeadline - now`)
3. Uploads payment proof image (drag-and-drop or click)
4. **Confirm dialog** → confirms
5. `PATCH /api/v1/transactions/:id/upload-proof` as multipart/form-data
6. Backend uploads image to Cloudinary, sets `confirmationDeadline = now + 3 days`
7. Status changes to `WAITING_FOR_ADMIN_CONFIRMATION`
8. Toast: "Payment proof uploaded successfully"

**Unhappy Paths:**
- Countdown reaches 0 → status auto-changes to `EXPIRED` on next read (lazy evaluation) and upload button is disabled.
- Invalid image format or file > 2MB → Multer middleware rejects request with 400 Bad Request.
- Transaction is not in `WAITING_FOR_PAYMENT` state → API rejects update with 400 Bad Request.

**Business Constraints:**
- Only permitted when `status === WAITING_FOR_PAYMENT` and `now < paymentDeadline`.
- Image: jpg/png/webp, max 2MB.
- Sets `confirmationDeadline = uploadedAt + 3 days` to initiate organizer SLA timer.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Timer expired / wrong status | 400 | `"Transaction is no longer waiting for payment"` |
| Unsupported file format | 400 | `"Only JPG, PNG, and WEBP images allowed"` |
| File size > 2MB | 400 | `"Image must not exceed 2MB"` |
| Not transaction owner | 403 | `"You can only modify your own transactions"` |

**Acceptance Criteria:**
- Given a valid transaction in `WAITING_FOR_PAYMENT` within deadline, When submitting a valid proof image (<2MB), Then status transitions to `WAITING_FOR_ADMIN_CONFIRMATION` and confirmation deadline is set.
- Given an expired transaction, When upload attempted, Then API returns 400 error and status remains expired.

**Definition of Done:**
- [ ] Live countdown timer component disables file input when 00:00 is reached
- [ ] Multer validates image type and 2MB limit
- [ ] Backend checks ownership and status before writing proof URL
- [ ] 3-day organizer verification window timestamp calculated accurately

### Technical Tasks

**Frontend:**
- **Page:** `app/my-tickets/[id]/page.tsx`
- **Components:**
  - `components/transactions/Countdown.tsx` — live countdown using `hooks/useCountdown.ts`
  - `components/transactions/PaymentUpload.tsx` — file upload area
  - `components/transactions/TransactionStatusBadge.tsx` — colored badge
- **Hook (`hooks/useCountdown.ts`):**
  ```ts
  export function useCountdown(deadline: string) {
    const [timeLeft, setTimeLeft] = useState(calculateTimeLeft(deadline));

    useEffect(() => {
      const timer = setInterval(() => {
        const left = calculateTimeLeft(deadline);
        setTimeLeft(left);
        if (left.total <= 0) clearInterval(timer);
      }, 1000);
      return () => clearInterval(timer);
    }, [deadline]);

    return timeLeft;
  }

  function calculateTimeLeft(deadline: string) {
    const total = new Date(deadline).getTime() - Date.now();
    return {
      total,
      hours: Math.max(0, Math.floor((total / (1000 * 60 * 60)) % 24)),
      minutes: Math.max(0, Math.floor((total / (1000 * 60)) % 60)),
      seconds: Math.max(0, Math.floor((total / 1000) % 60)),
    };
  }
  ```

**Backend:**
- **Endpoint:** `PATCH /api/v1/transactions/:id/upload-proof`
- **Middleware:** `authenticate` → `authorize(["CUSTOMER"])` → `upload.single("paymentProof")` → `transactionController.uploadProof`
- **Service:** Validate ownership + status + deadline → upload to Cloudinary → update transaction

**Database:**
- **N/A (Standard Update):** Single record update on `Transaction` table setting `paymentProofUrl`, `status = 'WAITING_FOR_ADMIN_CONFIRMATION'`, and `confirmationDeadline`; no complex schema transaction required.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/transactions/upload-proof.test.ts`
- **Mocking:** `jest.mock()` on `cloudinaryService.upload` (returns mock proof URL)
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createTransaction({ status: 'WAITING_FOR_PAYMENT', paymentDeadline: future })` and `createTransaction({ status: 'WAITING_FOR_PAYMENT', paymentDeadline: past })` for expired case
- **Required Assertions:**
  - Assert 200 OK with valid proof image within deadline → status changed to `WAITING_FOR_ADMIN_CONFIRMATION`, `confirmationDeadline` set to `now + 3 days`
  - Assert 400 Bad Request when transaction is expired (deadline passed) → status unchanged
  - Assert 400 Bad Request when transaction is not in `WAITING_FOR_PAYMENT` state
  - Assert 400 Bad Request with invalid image type or file > 2MB
  - Assert 403 Forbidden when uploading proof for another customer's transaction

**Frontend Tests:**
- **File:** `apps/web/__tests__/components/Countdown.test.tsx`
- **Library:** React Testing Library
- **Required Assertions:**
  - Assert countdown renders hours:minutes:seconds format
  - Assert upload button is disabled when countdown reaches 00:00:00

---

## US-16: View My Tickets / Transactions (Customer)

**Story:** As a Customer, I want to see all my transactions and their statuses.
**Traceability:** Mapped to Spec Story #25 | Issue Ticket #09

**Context:** Customer transaction dashboard with status filtering and lazy expiry evaluation.

**Roles:** CUSTOMER only.

**Assumptions:**
- Customer is authenticated.

**Out of Scope:** Downloading PDF ticket attachments, exporting order history to Excel.

**Happy Path:**
1. Customer navigates to `/my-tickets`
2. Sees list of transactions (newest first) as cards
3. Each card shows: event name, event banner, date, status badge, total paid, booking code (if DONE)
4. Can filter by status (dropdown)
5. Pagination (10 per page)
6. Clicks a card → goes to `/my-tickets/[id]` for full detail

**Unhappy Paths:**
- Customer has 0 transactions or 0 matching selected filter → UI renders an `EmptyState` component cleanly without throwing an error.

**Business Constraints:**
- **Lazy Expiry:** On every list fetch, backend checks each `WAITING_FOR_PAYMENT` transaction against current time; if `now > paymentDeadline`, status is auto-expired before returning results.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Invalid filter parameter | 400 | Validation errors |
| No transactions found | 200 | `data: [], meta.total: 0` |

**Acceptance Criteria:**
- Given an authenticated Customer, When viewing `/my-tickets`, Then all owned transactions are listed sorted by newest first.
- Given an order where `paymentDeadline` passed without proof upload, When queried, Then the system lazily updates status to `EXPIRED` before rendering.
- Given no transactions, When loaded, Then an Empty State component is displayed.

**Definition of Done:**
- [ ] Transaction list renders status badges with distinct colors
- [ ] Status dropdown filter and pagination tested
- [ ] Lazy evaluation helper verified on backend query

### Technical Tasks

**Frontend:**
- **Page:** `app/my-tickets/page.tsx`
- **Components:**
  - `components/transactions/TransactionCard.tsx` — summary card
  - `components/transactions/TransactionStatusBadge.tsx`
  - `components/ui/Pagination.tsx`
  - `components/ui/EmptyState.tsx` — "No tickets yet"

**Backend:**
- **Endpoint:** `GET /api/v1/transactions` (filtered by `userId`)
- **Service:** Run lazy evaluation on pending orders → return paginated transaction history.

**Database:**
- **N/A (Read + Lazy Update):** Read query combined with conditional lazy `updateMany` for expired items; handled seamlessly in service layer.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/transactions/list-transactions.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createTransaction()` with various statuses and deadlines
- **Required Assertions:**
  - Assert 200 OK → returns only authenticated customer's transactions, sorted newest first
  - Assert lazy expiry: transaction with `paymentDeadline < now` returned as `EXPIRED` status
  - Assert status filter works correctly (e.g., `?status=DONE` returns only confirmed orders)
  - Assert pagination metadata is accurate
  - Assert 200 OK with 0 transactions → `data: [], meta.total: 0`

---

## US-17: Organizer Transaction Management

**Story:** As an Organizer, I want to view, accept, and reject transactions for my events, so that I can manage payments.
**Traceability:** Mapped to Spec Stories #26, #27, #28, #29 | Issue Ticket #10

**Context:** Organizer order verification dashboard with Accept/Reject actions and 3-day lazy auto-cancellation.

**Roles:** ORGANIZER only.

**Assumptions:**
- Authenticated Organizer owns the event linked to the transactions.

**Out of Scope:** Partial refunds, manual fee adjustments, direct bank payout initiation.

**Happy Path (Accept):**
1. Organizer navigates to `/dashboard/transactions`
2. Sees table of transactions across all their events
3. Filters by status, by event
4. Clicks a `WAITING_FOR_ADMIN_CONFIRMATION` transaction
5. Views payment proof image (modal or inline)
6. Clicks "Accept" → **Confirm dialog** → confirms
7. `PATCH /api/v1/transactions/:id/accept`
8. Backend generates bookingCode, status → `DONE`
9. Sends ticket confirmation email to customer
10. Toast: "Transaction accepted"

**Happy Path (Reject):**
1. Organizer clicks "Reject" → enters optional reason → **Confirm dialog** → confirms
2. `PATCH /api/v1/transactions/:id/reject`
3. Backend: status → `REJECTED`, rollback (restore seats, return points/voucher/coupon)
4. Sends rejection email to customer
5. Toast: "Transaction rejected"

**Unhappy Paths:**
- Attempting to accept or reject an order not in `WAITING_FOR_ADMIN_CONFIRMATION` state → API returns 400 Bad Request.
- Attempting to manage an order for another organizer's event → API returns 403 Forbidden.

**Business Constraints:**
- Can only accept/reject when `status === WAITING_FOR_ADMIN_CONFIRMATION`.
- On accept: generate bookingCode `EVT-YYYY-XXXXXX`.
- On reject: full rollback in `prisma.$transaction()` (restore seat inventory and refund promotional coupons/points).
- 3-day auto-cancel: if organizer doesn't act within 3 days of proof upload, status → `CANCELED` + rollback (handled by lazy evaluation on read).

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Wrong status for action | 400 | `"Transaction cannot be accepted/rejected in its current state"` |
| Not organizer's event | 403 | `"You can only manage transactions for your own events"` |

**Acceptance Criteria:**
- Given an order in `WAITING_FOR_ADMIN_CONFIRMATION`, When accepted, Then status becomes `DONE` and a booking code is generated.
- Given an order in `WAITING_FOR_ADMIN_CONFIRMATION`, When rejected, Then status becomes `REJECTED`, seats are restored, and points/coupons are refunded.
- Given an order waiting >3 days without confirmation, When queried, Then lazy evaluation auto-cancels it and rolls back inventory.

**Definition of Done:**
- [ ] Transaction table renders payment proof modal cleanly
- [ ] Accept and Reject actions prompt confirmation modals
- [ ] Atomic rollback transaction verified on rejection and 3-day auto-cancel

### Technical Tasks

**Frontend:**
- **Page:** `app/dashboard/transactions/page.tsx`
- **Components:**
  - `components/transactions/TransactionTable.tsx` — sortable table with columns: customer name, event name, amount, status, date, actions
  - `components/ui/Modal.tsx` — view payment proof image
  - `components/ui/ConfirmDialog.tsx` — accept/reject confirmations
- **Filters:** Status dropdown, event dropdown
- **Pagination:** 10 per page

**Backend:**
- **Endpoints:**
  - `GET /api/v1/transactions` (ORGANIZER sees their events' transactions)
  - `PATCH /api/v1/transactions/:id/accept`
  - `PATCH /api/v1/transactions/:id/reject`
- **Service (reject):** All in `prisma.$transaction()`:
  1. Status → `REJECTED`
  2. Restore `Event.availableSeats` / `TicketType.availableQuota`
  3. Reverse PointLedger debits → create CREDIT entries with same expiry
  4. `Voucher.usageCount -= 1`
  5. `Coupon.isUsed = false`
  6. Send rejection email (async)

**Database:**
- **Transaction:** Execute atomic rollback transaction on Rejection or 3-day SLA auto-cancellation restoring seat inventory and refunding promotional ledger balances.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/transactions/manage-transactions.test.ts`
- **Mocking:** `jest.mock()` on `emailService.send` (confirmation and rejection emails)
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createTransaction({ status: 'WAITING_FOR_ADMIN_CONFIRMATION' })` with event, voucher, coupon, and point ledger entries
- **Required Assertions:**
  - Assert 200 OK on accept → status becomes `DONE`, bookingCode generated (`EVT-YYYY-XXXXXX` format), `emailService.send` called with confirmation template
  - Assert 200 OK on reject → status becomes `REJECTED`, seats restored, voucher usageCount decremented, coupon `isUsed` reset to false, PointLedger CREDIT entries created (reversing debits)
  - Assert 400 Bad Request when accepting/rejecting transaction not in `WAITING_FOR_ADMIN_CONFIRMATION` state
  - Assert 403 Forbidden when managing another organizer's event transactions
  - Assert lazy 3-day auto-cancel: transaction with `confirmationDeadline < now` returned as `CANCELED` with inventory restored

---

## US-18: Mark Attendance (Organizer)

**Story:** As an Organizer, I want to mark customers as attended after my event ends, so they can leave reviews.
**Traceability:** Mapped to Spec Story #30 | Issue Ticket #10

**Context:** Post-event attendance verification workflow required to unlock customer review eligibility.

**Roles:** ORGANIZER (own events only).

**Assumptions:**
- Target transaction is in `DONE` status.
- Event `endDate` has passed in real time (`now > event.endDate`).

**Out of Scope:** QR barcode scanning, customer self-check-in, partial attendance tracking.

**Happy Path:**
1. Organizer navigates to `/dashboard/events/[id]/attendees`
2. Sees table: customer name, email, ticket type, quantity, amount paid, booking code, attended status (checkbox)
3. Event's `endDate` has passed → attendance toggle is enabled
4. Clicks checkbox for a customer → `PATCH /api/v1/transactions/:id/attend`
5. `isAttended` set to `true`

**Unhappy Paths:**
- Attempting to mark attendance before the event ends (`now <= event.endDate`) → UI disables checkbox; API rejects request with 400 Bad Request.
- Target transaction status is not `DONE` (e.g. Canceled or Pending) → Checkbox hidden; API rejects with 400 Bad Request.

**Business Constraints:**
- Only available after `event.endDate < now`.
- Only for transactions with `status === DONE`.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Event not ended yet | 400 | `"Attendance can only be marked after the event ends"` |
| Order status not DONE | 400 | `"Can only mark attendance for confirmed transactions"` |
| Not event owner | 403 | `"You can only modify attendance for your own events"` |

**Acceptance Criteria:**
- Given a confirmed order for an ended event, When organizer toggles attendance, Then `isAttended` updates to `true` and 200 OK returned.
- Given an event that is still ongoing, When attendance toggle attempted, Then API returns 400 Bad Request.

**Definition of Done:**
- [ ] Attendee table disables checkboxes if event has not ended yet
- [ ] Backend validates `event.endDate < now` before toggling flag
- [ ] Toggling attendance immediately unlocks customer review form on client

### Technical Tasks

**Frontend:**
- **Page:** `app/dashboard/events/[id]/attendees/page.tsx`
- **Components:**
  - `components/dashboard/AttendeeTable.tsx` — table with attendance toggle
- **Disabled state:** If `event.endDate > now`, show message "Attendance can be marked after the event ends"

**Backend:**
- **Endpoint:** `PATCH /api/v1/transactions/:id/attend`
- **Middleware:** `authenticate` → `authorize(["ORGANIZER"])` → `transactionController.markAttendance`
- **Validation:** Check event.endDate < now, status === DONE, event belongs to organizer

**Database:**
- **N/A (Scalar Update):** Single record boolean update `Transaction.update({ where: { id }, data: { isAttended: true } })`; no multi-table transaction required.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/transactions/mark-attendance.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createEvent({ endDate: past })` and `createTransaction({ status: 'DONE' })`
- **Required Assertions:**
  - Assert 200 OK with ended event and DONE transaction → `isAttended` set to `true`
  - Assert 400 Bad Request when event has not ended yet (`endDate > now`)
  - Assert 400 Bad Request when transaction status is not `DONE`
  - Assert 403 Forbidden when marking attendance for another organizer's event
