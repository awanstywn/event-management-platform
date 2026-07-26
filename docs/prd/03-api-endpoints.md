# API Endpoints

> All endpoints are prefixed with `/api/v1`. All responses follow the standard envelope.
> Auth-required endpoints expect `accessToken` httpOnly cookie.
> **TDD Verification Seam:** In accordance with **ADR 0002**, every endpoint defined in this specification is verified at the HTTP route handler boundary using `supertest` integration test suites located in `apps/api/__tests__/integration/`.

## 1. Authentication

### POST /api/v1/auth/register
**Auth:** None
**Content-Type:** `application/json`

**Request Body:**
```ts
interface RegisterDTO {
  email: string;          // valid email format
  password: string;       // min 8 chars, 1 uppercase, 1 lowercase, 1 number, 1 special char
  firstName: string;      // min 2, max 50
  lastName: string;       // min 2, max 50
  role: "CUSTOMER" | "ORGANIZER";
  referralCode?: string;  // optional, format REF-XXXXXX
}
```

**Success Response (201):**
```ts
{
  success: true,
  message: "Registration successful",
  data: {
    id: string;
    email: string;
    firstName: string;
    lastName: string;
    role: "CUSTOMER" | "ORGANIZER";
    referralCode: string;
  }
}
// Sets httpOnly cookies: accessToken, refreshToken
```

**Error Responses:**
| Code | Trigger | Message |
|---|---|---|
| 400 | Validation fails | Field-specific errors |
| 409 | Email already exists | "Email is already registered" |
| 404 | Referral code invalid | "Referral code not found" |

**Side Effects:**
- If `referralCode` provided and valid: create Coupon (10%, 3-month expiry) for new user, credit 10,000 points to referrer's PointLedger
- Send welcome email asynchronously

---

### POST /api/v1/auth/login
**Auth:** None
**Content-Type:** `application/json`

**Request Body:**
```ts
interface LoginDTO {
  email: string;
  password: string;
}
```

**Success Response (200):**
```ts
{
  success: true,
  message: "Login successful",
  data: {
    id: string;
    email: string;
    firstName: string;
    lastName: string;
    role: "CUSTOMER" | "ORGANIZER";
    profilePicture: string | null;
  }
}
// Sets httpOnly cookies: accessToken, refreshToken
```

**Error Responses:**
| Code | Trigger | Message |
|---|---|---|
| 400 | Validation fails | Field-specific errors |
| 401 | Wrong email or password | "Invalid email or password" |

---

### POST /api/v1/auth/logout
**Auth:** Required
**Success Response (200):** `{ success: true, message: "Logged out successfully" }`
Clears `accessToken` and `refreshToken` cookies.

---

### POST /api/v1/auth/refresh
**Auth:** refreshToken cookie
**Success Response (200):** `{ success: true, message: "Token refreshed" }`
Sets new `accessToken` and `refreshToken` cookies.

**Error:** 401 if refresh token is invalid/expired.

---

### POST /api/v1/auth/forgot-password
**Auth:** None
**Content-Type:** `application/json`

**Request Body:**
```ts
interface ForgotPasswordDTO {
  email: string;
}
```

**Success Response (200):**
```ts
{
  success: true,
  message: "If an account with that email exists, a reset link has been sent"
}
// ALWAYS returns 200 regardless of whether email exists (prevent enumeration)
```

**Side Effect:** If email exists, generate hashed token, save to PasswordResetToken, send email with reset link.

---

### POST /api/v1/auth/reset-password
**Auth:** None
**Content-Type:** `application/json`

**Request Body:**
```ts
interface ResetPasswordDTO {
  token: string;     // from email link query param
  password: string;  // new password, same rules as register
}
```

**Success Response (200):** `{ success: true, message: "Password reset successfully" }`
Deletes the token, invalidates all refresh tokens for the user.

**Error Responses:**
| Code | Trigger | Message |
|---|---|---|
| 400 | Token invalid or expired | "Reset link is invalid or has expired" |

---

## 2. Events

### GET /api/v1/events
**Auth:** None (public)
**Query Parameters:**
```ts
interface EventQueryParams {
  search?: string;       // searches name and description
  category?: string;     // category slug (comma-separated for multiple)
  location?: string;     // location text match
  isFree?: "true" | "false";
  minPrice?: number;
  maxPrice?: number;
  startDate?: string;    // ISO 8601 — filter events starting after this date
  endDate?: string;      // ISO 8601 — filter events starting before this date
  sortBy?: "startDate" | "price" | "name" | "createdAt";  // default: "startDate"
  order?: "asc" | "desc";   // default: "asc"
  page?: number;         // default: 1
  limit?: number;        // default: 10
}
```

**Success Response (200):**
```ts
{
  success: true,
  message: "Events retrieved successfully",
  data: EventListItem[],
  meta: { total: number, page: number, totalPages: number, limit: number }
}

interface EventListItem {
  id: string;
  name: string;
  slug: string;
  location: string;
  startDate: string;
  endDate: string;
  isFree: boolean;
  price: number | null;
  bannerUrl: string | null;
  availableSeats: number;
  totalSeats: number;
  category: { id: string; name: string; slug: string };
  organizer: { id: string; firstName: string; lastName: string; profilePicture: string | null };
}
```

**Constraints:**
- Only returns events where `status = PUBLISHED` and `deletedAt IS NULL`
- Backend must handle all search, filter, sort, pagination — never frontend-only
- Empty results return `data: []` with `meta.total: 0`
- Debounce is frontend-only (300ms); backend processes whatever arrives

---

### GET /api/v1/events/:id
**Auth:** None (public)

**Success Response (200):**
```ts
{
  success: true,
  message: "Event retrieved successfully",
  data: {
    id: string;
    name: string;
    slug: string;
    description: string;
    location: string;
    startDate: string;
    endDate: string;
    totalSeats: number;
    availableSeats: number;
    isFree: boolean;
    price: number | null;
    bannerUrl: string | null;
    status: "PUBLISHED";
    category: { id: string; name: string; slug: string };
    organizer: { id: string; firstName: string; lastName: string; profilePicture: string | null };
    ticketTypes: { id: string; name: string; price: number; quota: number; availableQuota: number }[];
    averageRating: number | null;
    totalReviews: number;
  }
}
```

**Error:** 404 if event not found or soft-deleted.

---

### POST /api/v1/events
**Auth:** Required (ORGANIZER only)
**Content-Type:** `multipart/form-data`

**Middleware Order:**
1. `authenticate`
2. `authorize(["ORGANIZER"])`
3. `upload.single("bannerImage")` — Multer, memory, 2MB, jpg/png/webp
4. `parseJsonField("ticketTypes")` — JSON.parse `req.body.ticketTypes` if present
5. `validate(createEventSchema)`
6. `eventController.create`

**Form Fields:**
```ts
interface CreateEventDTO {
  name: string;            // 3-100 chars
  categoryId: string;      // UUID
  location: string;        // min 3 chars
  description: string;     // max 2000 chars
  startDate: string;       // ISO 8601
  endDate: string;         // ISO 8601, must be after startDate
  totalSeats: number;      // positive integer
  isFree: boolean;
  price?: number;          // required if isFree=false and no ticketTypes, Decimal
  ticketTypes?: string;    // JSON string: [{ name: string, price: number, quota: number }]
  bannerImage?: File;      // optional image file
}
```

**Success Response (201):**
```ts
{
  success: true,
  message: "Event created successfully",
  data: {
    id: string;
    name: string;
    slug: string;
    status: "PUBLISHED";
    bannerUrl: string | null;
    createdAt: string;
  }
}
```

**Error Responses:**
| Code | Trigger | Message |
|---|---|---|
| 400 | Validation fails | Field-specific errors |
| 400 | endDate <= startDate | "End date must be after start date" |
| 400 | isFree=false, no price, no ticketTypes | "Price is required for paid events" |
| 400 | Invalid image type/size | "Only JPG, PNG, or WEBP images are allowed" / "Image must be smaller than 2MB" |
| 401 | Not authenticated | "Please log in to continue" |
| 403 | Not ORGANIZER | "Only event organizers can access this resource" |

**Transaction:** Event + TicketType rows created in a single `prisma.$transaction()`.

---

### PUT /api/v1/events/:id
**Auth:** Required (ORGANIZER, own event only)
**Content-Type:** `multipart/form-data`

Same fields as POST. Additionally:
- If `bannerImage` is provided and event already has a banner, delete old image from Cloudinary.
- Cannot change `organizerId`.
- If event has existing transactions with status !== EXPIRED/CANCELED/REJECTED, certain fields may be restricted (e.g., cannot reduce totalSeats below already-sold count).

---

### DELETE /api/v1/events/:id
**Auth:** Required (ORGANIZER, own event only)

**Soft delete:** Sets `deletedAt = now()`. Does NOT hard delete.
**Constraint:** Cannot delete if active transactions exist (status WAITING_FOR_PAYMENT or WAITING_FOR_ADMIN_CONFIRMATION).

**Success Response (200):** `{ success: true, message: "Event deleted successfully" }`

---

## 3. Transactions

### POST /api/v1/transactions
**Auth:** Required (CUSTOMER only)
**Content-Type:** `application/json`

**Request Body:**
```ts
interface CreateTransactionDTO {
  eventId: string;
  items: { ticketTypeId?: string; quantity: number }[];
  voucherCode?: string;    // optional
  couponCode?: string;     // optional, mutually exclusive with voucherCode
  pointsToUse?: number;    // optional, from available balance
}
```

**Success Response (201):**
```ts
{
  success: true,
  message: "Transaction created successfully",
  data: {
    id: string;
    totalPrice: number;
    discountAmount: number;
    pointsUsed: number;
    finalPrice: number;
    status: "WAITING_FOR_PAYMENT";
    paymentDeadline: string;    // ISO 8601, createdAt + 2 hours
    createdAt: string;
    items: { ticketTypeName: string | null; quantity: number; pricePerUnit: number; subtotal: number }[];
  }
}
```

**Error Responses:**
| Code | Trigger | Message |
|---|---|---|
| 400 | Both voucher and coupon | "Cannot use both voucher and coupon" |
| 400 | Insufficient seats | "Not enough available seats" |
| 400 | Insufficient points | "Not enough points available" |
| 400 | Voucher not valid for event | "Voucher is not applicable to this event" |
| 400 | Voucher/coupon expired | "Voucher has expired" / "Coupon has expired" |
| 400 | Voucher max usage reached | "Voucher usage limit reached" |
| 400 | Coupon already used | "Coupon has already been used" |
| 400 | Event not available | "Event is not available for purchase" |
| 401 | Not authenticated | "Please log in to continue" |
| 403 | Not CUSTOMER | "Only customers can purchase tickets" |

**Transaction (SQL):** All in `prisma.$transaction()`:
1. Check seat availability (with row-level locking if possible)
2. Decrement `availableSeats` on Event (or `availableQuota` on TicketType)
3. Validate and apply voucher/coupon
4. Debit points from PointLedger (FIFO — oldest expiring first)
5. Increment voucher `usageCount`
6. Mark coupon `isUsed = true`
7. Create Transaction + TransactionItem rows
8. Set `paymentDeadline = now + 2 hours`

---

### PATCH /api/v1/transactions/:id/upload-proof
**Auth:** Required (CUSTOMER, own transaction only)
**Content-Type:** `multipart/form-data`

**Middleware Order:**
1. `authenticate`
2. `authorize(["CUSTOMER"])`
3. `upload.single("paymentProof")` — Multer, memory, 2MB, jpg/png/webp
4. `transactionController.uploadProof`

**Constraint:** Only allowed when `status === WAITING_FOR_PAYMENT` and `now < paymentDeadline`.

**Success Response (200):**
```ts
{
  success: true,
  message: "Payment proof uploaded successfully",
  data: {
    id: string;
    status: "WAITING_FOR_ADMIN_CONFIRMATION";
    paymentProofUrl: string;
    confirmationDeadline: string;  // uploadedAt + 3 days
  }
}
```

---

### PATCH /api/v1/transactions/:id/accept
**Auth:** Required (ORGANIZER, own event's transaction only)
**Constraint:** Only when `status === WAITING_FOR_ADMIN_CONFIRMATION`.

**Side Effects:**
- Generate `bookingCode` (format: `EVT-YYYY-XXXXXX`)
- Send ticket confirmation email with booking code
- Status → `DONE`

---

### PATCH /api/v1/transactions/:id/reject
**Auth:** Required (ORGANIZER, own event's transaction only)
**Content-Type:** `application/json`

**Request Body:**
```ts
interface RejectTransactionDTO {
  reason?: string;  // optional rejection reason
}
```

**Side Effects (in prisma.$transaction):**
- Status → `REJECTED`
- Restore `availableSeats` on Event (or `availableQuota` on TicketType)
- Return points to PointLedger (CREDIT entry with same expiry as original)
- Reset voucher `usageCount -= 1`
- Reset coupon `isUsed = false`
- Send rejection email

---

### GET /api/v1/transactions
**Auth:** Required
**Behavior varies by role:**
- **CUSTOMER**: returns own transactions
- **ORGANIZER**: returns transactions for their events

**Query Parameters:**
```ts
interface TransactionQueryParams {
  status?: TransactionStatus;
  eventId?: string;
  page?: number;    // default: 1
  limit?: number;   // default: 10
}
```

**Note:** On every read, run the lazy expiry check:
- If `WAITING_FOR_PAYMENT` and `now > paymentDeadline` → update to `EXPIRED` + rollback
- If `WAITING_FOR_ADMIN_CONFIRMATION` and `now > confirmationDeadline` → update to `CANCELED` + rollback

---

### GET /api/v1/transactions/:id
**Auth:** Required (own transaction or own event's transaction)
Same lazy expiry check applies.

---

### PATCH /api/v1/transactions/:id/attend
**Auth:** Required (ORGANIZER, own event's transaction only)
**Constraint:** Only when `status === DONE` and `event.endDate < now`.
Sets `isAttended = true`.

---

## 4. Reviews

### POST /api/v1/reviews
**Auth:** Required (CUSTOMER only)
**Content-Type:** `application/json`

**Request Body:**
```ts
interface CreateReviewDTO {
  eventId: string;
  rating: number;     // 1-5 integer
  comment?: string;   // max 500 chars
}
```

**Constraints:**
- Customer must have a Transaction with `status = DONE` and `isAttended = true` for this event
- One review per customer per event (unique constraint)

**Error Responses:**
| Code | Trigger | Message |
|---|---|---|
| 400 | Rating not 1-5 | "Rating must be between 1 and 5" |
| 400 | Not attended | "You can only review events you have attended" |
| 409 | Already reviewed | "You have already reviewed this event" |

---

### GET /api/v1/reviews?eventId=xxx
**Auth:** None (public)
**Query Parameters:** `eventId` (required), `page`, `limit`

Returns reviews with user info (firstName, lastName, profilePicture).

---

## 5. Vouchers

### POST /api/v1/vouchers
**Auth:** Required (ORGANIZER only)
**Content-Type:** `application/json`

**Request Body:**
```ts
interface CreateVoucherDTO {
  code: string;           // unique, uppercase, 3-20 chars
  discountType: "PERCENTAGE" | "FIXED";
  discountAmount: number; // > 0; if PERCENTAGE, max 100
  maxUsage: number;       // positive integer
  validFrom: string;      // ISO 8601
  validUntil: string;     // ISO 8601, must be after validFrom
  eventIds: string[];     // array of event IDs (must be organizer's own events)
}
```

---

### GET /api/v1/vouchers
**Auth:** Required (ORGANIZER only)
Returns organizer's vouchers with linked events.

### DELETE /api/v1/vouchers/:id
**Auth:** Required (ORGANIZER, own voucher only)
Soft delete.

---

## 6. Users & Profile

### GET /api/v1/users/me
**Auth:** Required
Returns full profile including referralCode, point balance (computed from PointLedger), active coupons.

### PATCH /api/v1/users/me
**Auth:** Required
**Content-Type:** `multipart/form-data`

**Form Fields:**
```ts
interface UpdateProfileDTO {
  firstName?: string;
  lastName?: string;
  profilePicture?: File;  // image, 2MB max
}
```
If new profilePicture provided and old one exists, delete old from Cloudinary.

### PATCH /api/v1/users/me/password
**Auth:** Required
**Content-Type:** `application/json`

```ts
interface ChangePasswordDTO {
  currentPassword: string;
  newPassword: string;    // same rules as register
}
```

### GET /api/v1/users/:id/profile
**Auth:** None (public organizer profile)
Returns organizer's public info: name, profilePicture, events, average rating, reviews.

---

## 7. Dashboard (Organizer)

### GET /api/v1/dashboard/statistics
**Auth:** Required (ORGANIZER only)

**Query Parameters:**
```ts
interface DashboardQueryParams {
  startDate?: string;  // ISO 8601
  endDate?: string;    // ISO 8601
}
```

**Success Response (200):**
```ts
{
  success: true,
  message: "Dashboard statistics retrieved",
  data: {
    summary: {
      totalRevenue: number;        // sum of finalPrice where status=DONE
      totalAttendees: number;      // count of DONE transactions
      totalEvents: number;         // count of organizer's events
      averageRating: number | null; // avg rating across all events
    };
    revenueByMonth: { month: string; revenue: number }[];
    ticketSalesByMonth: { month: string; count: number }[];
    eventsByCategory: { category: string; count: number }[];
  }
}
```

**Constraints:**
- All aggregation done on the backend using SQL COUNT, SUM, AVG, GROUP BY
- Date filter applies to all metrics
- Only the organizer's own events

---

## 8. Categories

### GET /api/v1/categories
**Auth:** None (public)
Returns all categories. No pagination needed (small, fixed list).

```ts
{
  success: true,
  message: "Categories retrieved",
  data: { id: string; name: string; slug: string }[]
}
```

---

## 9. Attendees

### GET /api/v1/events/:id/attendees
**Auth:** Required (ORGANIZER, own event only)
**Query Parameters:** `page`, `limit`

Returns list of attendees (transactions with status DONE):
```ts
{
  success: true,
  message: "Attendees retrieved",
  data: {
    id: string;              // transaction ID
    user: { firstName: string; lastName: string; email: string };
    ticketType: string | null;
    quantity: number;
    totalPaid: number;
    isAttended: boolean;
    bookingCode: string;
  }[],
  meta: { total, page, totalPages, limit }
}
```
