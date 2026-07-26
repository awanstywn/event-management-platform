# Architecture & Design System

## 1. System Architecture

### 1.1 Monorepo Layout

```
event-management-platform/
├── apps/
│   ├── api/                              ← Express.js Backend
│   │   ├── src/
│   │   │   ├── index.ts                  ← Entry point (start server)
│   │   │   ├── app.ts                    ← Express app setup (middleware, routes, error handler)
│   │   │   ├── config/
│   │   │   │   ├── database.ts           ← Prisma client singleton
│   │   │   │   ├── cloudinary.ts         ← Cloudinary SDK config
│   │   │   │   └── mailer.ts             ← Nodemailer transporter factory
│   │   │   ├── routes/
│   │   │   │   ├── index.ts              ← Route aggregator
│   │   │   │   ├── auth.routes.ts
│   │   │   │   ├── event.routes.ts
│   │   │   │   ├── transaction.routes.ts
│   │   │   │   ├── review.routes.ts
│   │   │   │   ├── voucher.routes.ts
│   │   │   │   ├── user.routes.ts
│   │   │   │   └── dashboard.routes.ts
│   │   │   ├── controllers/              ← Thin — calls service, sends response
│   │   │   │   ├── auth.controller.ts
│   │   │   │   ├── event.controller.ts
│   │   │   │   ├── transaction.controller.ts
│   │   │   │   ├── review.controller.ts
│   │   │   │   ├── voucher.controller.ts
│   │   │   │   ├── user.controller.ts
│   │   │   │   └── dashboard.controller.ts
│   │   │   ├── services/                 ← Business logic lives here
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── event.service.ts
│   │   │   │   ├── transaction.service.ts
│   │   │   │   ├── review.service.ts
│   │   │   │   ├── voucher.service.ts
│   │   │   │   ├── user.service.ts
│   │   │   │   ├── dashboard.service.ts
│   │   │   │   └── mailer.service.ts
│   │   │   ├── repositories/             ← Prisma queries only
│   │   │   │   ├── user.repository.ts
│   │   │   │   ├── event.repository.ts
│   │   │   │   ├── transaction.repository.ts
│   │   │   │   ├── review.repository.ts
│   │   │   │   ├── voucher.repository.ts
│   │   │   │   ├── point.repository.ts
│   │   │   │   ├── coupon.repository.ts
│   │   │   │   └── category.repository.ts
│   │   │   ├── middlewares/
│   │   │   │   ├── authenticate.ts       ← JWT verification → req.user
│   │   │   │   ├── authorize.ts          ← Role check: authorize(["ORGANIZER"])
│   │   │   │   ├── validate.ts           ← Zod schema validation
│   │   │   │   ├── upload.ts             ← Multer config (memory, 2MB, image types)
│   │   │   │   └── errorHandler.ts       ← Global error handler (MUST be last middleware)
│   │   │   ├── validators/               ← Zod schemas
│   │   │   │   ├── auth.validator.ts
│   │   │   │   ├── event.validator.ts
│   │   │   │   ├── transaction.validator.ts
│   │   │   │   ├── review.validator.ts
│   │   │   │   ├── voucher.validator.ts
│   │   │   │   └── user.validator.ts
│   │   │   ├── types/                    ← DTOs, interfaces, enums
│   │   │   │   ├── express.d.ts          ← Express Request augmentation (req.user)
│   │   │   │   ├── auth.types.ts
│   │   │   │   ├── event.types.ts
│   │   │   │   ├── transaction.types.ts
│   │   │   │   └── common.types.ts       ← ApiResponse<T>, PaginatedResponse<T>
│   │   │   ├── utils/
│   │   │   │   ├── slug.ts               ← generateSlug(name)
│   │   │   │   ├── code-generator.ts     ← generateReferralCode(), generateBookingCode()
│   │   │   │   ├── token.ts              ← generateToken(), verifyToken(), hashToken()
│   │   │   │   └── password.ts           ← hashPassword(), comparePassword()
│   │   │   └── templates/                ← HTML email templates
│   │   │       ├── welcome.ts
│   │   │       ├── ticket-confirmation.ts
│   │   │       ├── transaction-rejected.ts
│   │   │       ├── transaction-expired.ts
│   │   │       └── password-reset.ts
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── seed.ts
│   │   ├── __tests__/                    ← Backend integration & unit test suite
│   │   │   ├── integration/              ← supertest HTTP route handler boundary tests
│   │   │   │   ├── auth/
│   │   │   │   ├── events/
│   │   │   │   ├── transactions/
│   │   │   │   ├── reviews/
│   │   │   │   ├── vouchers/
│   │   │   │   └── users/
│   │   │   ├── unit/                     ← Service & utility unit tests
│   │   │   └── setup/                    ← Test database isolation & factory helpers
│   │   │       ├── globalSetup.ts        ← Executes prisma migrate deploy once
│   │   │       └── helpers.ts            ← truncateAll() & fixture factories
│   │   ├── jest.config.ts                ← ts-jest configuration
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── .env.example
│   │
│   └── web/                              ← Next.js Frontend
│       ├── app/
│       │   ├── layout.tsx                ← Root layout (providers, fonts, navbar/footer)
│       │   ├── page.tsx                  ← Landing page (/)
│       │   ├── login/page.tsx
│       │   ├── register/page.tsx
│       │   ├── forgot-password/page.tsx
│       │   ├── reset-password/page.tsx
│       │   ├── events/
│       │   │   ├── page.tsx              ← Event browsing (/events)
│       │   │   └── [id]/page.tsx         ← Event detail (/events/[id])
│       │   ├── organizers/
│       │   │   └── [id]/page.tsx         ← Public organizer profile
│       │   ├── profile/page.tsx
│       │   ├── my-tickets/
│       │   │   ├── page.tsx              ← Customer ticket list
│       │   │   └── [id]/page.tsx         ← Transaction detail + upload proof
│       │   └── dashboard/
│       │       ├── layout.tsx            ← Sidebar layout (organizer only)
│       │       ├── page.tsx              ← Statistics dashboard
│       │       ├── events/
│       │       │   ├── page.tsx
│       │       │   ├── new/page.tsx
│       │       │   └── [id]/
│       │       │       ├── page.tsx      ← Edit event
│       │       │       └── attendees/page.tsx
│       │       ├── transactions/page.tsx
│       │       └── vouchers/
│       │           ├── page.tsx
│       │           └── new/page.tsx
│       ├── components/
│       │   ├── ui/                       ← Shared primitives
│       │   │   ├── Button.tsx
│       │   │   ├── Input.tsx
│       │   │   ├── Textarea.tsx
│       │   │   ├── Select.tsx
│       │   │   ├── Modal.tsx
│       │   │   ├── ConfirmDialog.tsx
│       │   │   ├── Card.tsx
│       │   │   ├── Badge.tsx
│       │   │   ├── Table.tsx
│       │   │   ├── Pagination.tsx
│       │   │   ├── Spinner.tsx
│       │   │   ├── EmptyState.tsx
│       │   │   ├── ErrorState.tsx
│       │   │   ├── Skeleton.tsx
│       │   │   ├── DatePicker.tsx
│       │   │   ├── FileUpload.tsx
│       │   │   ├── StarRating.tsx
│       │   │   └── ThemeToggle.tsx
│       │   ├── layout/
│       │   │   ├── Navbar.tsx
│       │   │   ├── Footer.tsx
│       │   │   ├── Sidebar.tsx
│       │   │   └── DashboardHeader.tsx
│       │   ├── auth/
│       │   │   ├── LoginForm.tsx
│       │   │   ├── RegisterForm.tsx
│       │   │   ├── ForgotPasswordForm.tsx
│       │   │   └── ResetPasswordForm.tsx
│       │   ├── events/
│       │   │   ├── EventCard.tsx
│       │   │   ├── EventGrid.tsx
│       │   │   ├── EventForm.tsx
│       │   │   ├── EventFilter.tsx
│       │   │   ├── SearchBar.tsx
│       │   │   ├── TicketSelector.tsx
│       │   │   ├── DateRangePicker.tsx
│       │   │   └── CategoryFilter.tsx
│       │   ├── transactions/
│       │   │   ├── TransactionCard.tsx
│       │   │   ├── TransactionTable.tsx
│       │   │   ├── PaymentUpload.tsx
│       │   │   ├── Countdown.tsx
│       │   │   ├── CheckoutSummary.tsx
│       │   │   └── TransactionStatusBadge.tsx
│       │   ├── dashboard/
│       │   │   ├── StatCard.tsx
│       │   │   ├── RevenueChart.tsx
│       │   │   ├── TicketSalesChart.tsx
│       │   │   ├── CategoryPieChart.tsx
│       │   │   ├── AttendeeTable.tsx
│       │   │   └── DashboardDateFilter.tsx
│       │   ├── reviews/
│       │   │   ├── ReviewForm.tsx
│       │   │   ├── ReviewList.tsx
│       │   │   └── ReviewCard.tsx
│       │   └── vouchers/
│       │       ├── VoucherForm.tsx
│       │       └── VoucherTable.tsx
│       ├── hooks/
│       │   ├── useAuth.ts
│       │   ├── useDebounce.ts
│       │   ├── useCountdown.ts
│       │   └── usePagination.ts
│       ├── lib/
│       │   ├── api/
│       │   │   ├── axios.ts              ← Axios instance with interceptors
│       │   │   ├── auth.api.ts
│       │   │   ├── event.api.ts
│       │   │   ├── transaction.api.ts
│       │   │   ├── review.api.ts
│       │   │   ├── voucher.api.ts
│       │   │   ├── user.api.ts
│       │   │   └── dashboard.api.ts
│       │   ├── validators/
│       │   │   ├── auth.schema.ts
│       │   │   ├── event.schema.ts
│       │   │   ├── transaction.schema.ts
│       │   │   ├── review.schema.ts
│       │   │   └── voucher.schema.ts
│       │   └── utils.ts                  ← formatCurrency, formatDate, etc.
│       ├── types/
│       │   ├── auth.types.ts
│       │   ├── event.types.ts
│       │   ├── transaction.types.ts
│       │   ├── review.types.ts
│       │   ├── voucher.types.ts
│       │   └── common.types.ts
│       ├── providers/
│       │   ├── AuthProvider.tsx
│       │   ├── QueryProvider.tsx
│       │   └── ThemeProvider.tsx
│       ├── __tests__/                    ← Frontend component & utility test suite
│       │   ├── components/               ← RTL UI component validation & rendering tests
│       │   ├── hooks/                    ← Custom hook RTL tests
│       │   └── lib/                      ← API client & validator tests
│       ├── jest.config.ts                ← next/jest configuration
│       ├── package.json
│       ├── tsconfig.json
│       ├── next.config.ts
│       └── .env.example
│
├── .github/
│   └── workflows/
│       └── ci.yml                        ← 5-gate CI/CD verification pipeline
├── package.json                          ← Root: { "workspaces": ["apps/*"] }
├── .gitignore
├── README.md
├── CONTEXT.md
└── docs/
    └── prd/
```

### 1.2 Backend Architecture (Layered)

```
Request → Router → Middleware Chain → Controller → Service → Repository → Database
                                                       ↓
                                                  Mailer Service (async, fire-and-forget)
                                                  Cloudinary (file upload)
```

**Layer Responsibilities:**

| Layer | Responsibility | Rules |
|---|---|---|
| **Router** | Maps HTTP method + path to middleware chain + controller method | No logic. Just wiring. |
| **Middleware** | Cross-cutting concerns: auth, role check, file parsing, validation | Runs in declared order. Each calls `next()` or `next(err)`. |
| **Controller** | Extracts data from `req`, calls service, sends `res`. | **THIN** — no `if` statements about business rules. No direct Prisma calls. |
| **Service** | All business logic, orchestration, transactions. | Can call multiple repositories. Throws typed errors. |
| **Repository** | Single-table Prisma queries. | No business logic. No HTTP concepts. Returns raw Prisma types. |

### 1.3 Authentication Flow

```
Registration:
  Client POST /api/v1/auth/register → hash password → create User → generate referralCode
  → generate accessToken + refreshToken → set httpOnly cookies → 201

Login:
  Client POST /api/v1/auth/login → verify credentials → generate tokens → set cookies → 200

Token Refresh:
  Client POST /api/v1/auth/refresh → read refreshToken cookie → verify → generate new pair
  → set new cookies → 200

Logout:
  Client POST /api/v1/auth/logout → clear cookies → 200

Every authenticated request:
  authenticate middleware → read accessToken cookie → verify JWT → attach req.user
  → if expired, return 401 → frontend interceptor calls /refresh → retries original request
```

**Token payload:**
```ts
interface JwtPayload {
  userId: string;
  role: "CUSTOMER" | "ORGANIZER";
  iat: number;
  exp: number;
}
```

### 1.4 Timer Mechanism (Query-Time Lazy Evaluation)

Since the backend runs on Vercel serverless (no persistent process), timers use lazy evaluation:

```
Transaction created → store paymentDeadline = createdAt + 2 hours
                    → store confirmationDeadline = null (set when proof uploaded)

On any Transaction read (GET /transactions/:id, GET /transactions list):
  if (status === WAITING_FOR_PAYMENT && now > paymentDeadline):
    → update status to EXPIRED
    → rollback: restore seats, return points/voucher/coupon
    → return updated transaction

  if (status === WAITING_FOR_ADMIN_CONFIRMATION && now > confirmationDeadline):
    → update status to CANCELED
    → rollback: restore seats, return points/voucher/coupon
    → return updated transaction
```

### 1.5 File Upload Flow

```
Client (multipart/form-data) → Multer (memory storage, 2MB max, jpg/png/webp)
  → Controller extracts req.file
  → Service uploads buffer to Cloudinary
  → Cloudinary returns secure_url
  → Service saves URL to database
  → On update: delete old file from Cloudinary by public_id
```

### 1.6 Email Flow

```
Service completes business logic
  → calls mailerService.sendEmail(template, recipient, data)
  → mailerService fires Nodemailer asynchronously (no await)
  → .catch(err => logger.error("Email failed", err))
  → API response returns immediately, not blocked by email delivery
```

### 1.7 TDD Architecture & Integration Seams

```
Backend Seam:
  HTTP Client / supertest → Express Router → Middleware → Controller → Service → Real Test DB (PostgreSQL)
                                                                       ↓
                                                      Mocked External SDKs (Cloudinary, SMTP)

Frontend Seam:
  React Testing Library (RTL) → Component / Page → Mocked API Hooks / nock Interception
```

**Key Testing Rules (ADR 0002):**
1. **Unified Runner:** Use Jest (`ts-jest` for `apps/api`, `next/jest` for `apps/web`). All tests reside in dedicated `__tests__/` directories.
2. **Primary Seam (Backend):** Test at the HTTP route handler boundary using `supertest` against a real PostgreSQL 16 test database. Verifies middleware, validation, controller, and DB schema atomically.
3. **Secondary Seam (Frontend):** Use React Testing Library (RTL) for component validation behavior, conditional RBAC rendering, and loading/empty states.
4. **External Mocking:** Cloudinary and Nodemailer are mocked at the service wrapper level (`jest.mock()`) for deterministic, offline-capable test suites.
5. **Database Lifecycle:** Truncate-and-reseed between test files using `truncateAll()` helper and factory functions. `globalSetup.ts` runs migrations once before the suite.
6. **CI/CD Verification:** Merge-blocking GitHub Actions pipeline (`.github/workflows/ci.yml`) enforcing 5 gates: type-check, lint, Jest backend, Jest frontend, and production bundle build.

---

## 2. Design System

### 2.1 Color Palette

| Token | Light Mode | Dark Mode | Usage |
|---|---|---|---|
| `--bg-primary` | `#FFFFFF` | `#0F172A` | Page background |
| `--bg-secondary` | `#F8FAFC` | `#1E293B` | Card, sidebar background |
| `--bg-tertiary` | `#F1F5F9` | `#334155` | Input background, hover states |
| `--text-primary` | `#0F172A` | `#F8FAFC` | Main body text |
| `--text-secondary` | `#64748B` | `#94A3B8` | Muted text, labels |
| `--text-tertiary` | `#94A3B8` | `#64748B` | Placeholder text |
| `--accent-primary` | `#3B82F6` | `#3B82F6` | Primary buttons, links |
| `--accent-primary-hover` | `#2563EB` | `#60A5FA` | Primary button hover |
| `--accent-success` | `#10B981` | `#10B981` | Success states, DONE badge |
| `--accent-warning` | `#F59E0B` | `#F59E0B` | Warning states, WAITING badge |
| `--accent-danger` | `#EF4444` | `#EF4444` | Error states, REJECTED badge |
| `--border` | `#E2E8F0` | `#334155` | Card borders, dividers |
| `--border-focus` | `#3B82F6` | `#3B82F6` | Input focus ring |

### 2.2 Typography

```css
/* Google Font: Inter */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');

--font-family: 'Inter', sans-serif;

/* Scale */
--text-xs:   0.75rem;   /* 12px — badges, captions */
--text-sm:   0.875rem;  /* 14px — secondary text, table cells */
--text-base: 1rem;      /* 16px — body text */
--text-lg:   1.125rem;  /* 18px — card titles */
--text-xl:   1.25rem;   /* 20px — section headings */
--text-2xl:  1.5rem;    /* 24px — page titles */
--text-3xl:  1.875rem;  /* 30px — hero heading */
--text-4xl:  2.25rem;   /* 36px — landing hero */
```

### 2.3 Spacing & Sizing

```
--radius-sm: 6px;     /* Buttons, inputs, badges */
--radius-md: 8px;     /* Cards, dropdowns */
--radius-lg: 12px;    /* Modals, large containers */
--radius-full: 9999px; /* Avatar, pill badges */

--shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
--shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
--shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);
--shadow-xl: 0 20px 25px -5px rgb(0 0 0 / 0.1);
```

### 2.4 Responsive Breakpoints

| Breakpoint | Min Width | Target |
|---|---|---|
| `sm` | `640px` | Large phones |
| `md` | `768px` | Tablets |
| `lg` | `1024px` | Small desktops |
| `xl` | `1280px` | Large desktops |

**Mobile-first approach**: Default styles target mobile. Use `sm:`, `md:`, `lg:`, `xl:` prefixes for larger screens.

### 2.5 Component Inventory

All components are built from scratch with Tailwind CSS. No component library.

| Component | File | Description |
|---|---|---|
| `Button` | `components/ui/Button.tsx` | Variants: primary, secondary, outline, danger, ghost. Sizes: sm, md, lg. Loading state with spinner. |
| `Input` | `components/ui/Input.tsx` | Text input with label, error message, icon support. Auto-uppercase variant for referral code. |
| `Textarea` | `components/ui/Textarea.tsx` | Multi-line input with character counter. |
| `Select` | `components/ui/Select.tsx` | Dropdown select with label and error. |
| `Modal` | `components/ui/Modal.tsx` | Centered overlay with backdrop blur. Close on ESC and backdrop click. |
| `ConfirmDialog` | `components/ui/ConfirmDialog.tsx` | Confirmation modal with "Are you sure?" pattern. Required for all data modifications. |
| `Card` | `components/ui/Card.tsx` | Container with padding, border, shadow. |
| `Badge` | `components/ui/Badge.tsx` | Status badges (success/warning/danger/info). Used for TransactionStatus. |
| `Table` | `components/ui/Table.tsx` | Responsive table with sortable headers. |
| `Pagination` | `components/ui/Pagination.tsx` | Page-based navigation with page numbers. |
| `Spinner` | `components/ui/Spinner.tsx` | Loading spinner (animated SVG). |
| `EmptyState` | `components/ui/EmptyState.tsx` | "No results" illustration with message and optional CTA. |
| `ErrorState` | `components/ui/ErrorState.tsx` | Error display with retry button. |
| `Skeleton` | `components/ui/Skeleton.tsx` | Skeleton loading placeholder (animated pulse). |
| `FileUpload` | `components/ui/FileUpload.tsx` | Drag-and-drop + click upload with preview. Accepts: jpg, png, webp. Max: 2MB. |
| `StarRating` | `components/ui/StarRating.tsx` | 1-5 star interactive rating (for reviews) and display-only mode. |
| `ThemeToggle` | `components/ui/ThemeToggle.tsx` | Dark/light mode toggle button. |
| `DatePicker` | `components/ui/DatePicker.tsx` | Date/datetime picker input. |

### 2.6 Page Layout Patterns

**Public pages** (Landing, Events, Event Detail, Login, Register):
```
┌──────────────────────────────────────┐
│ Navbar (logo, search, login/avatar)  │
├──────────────────────────────────────┤
│                                      │
│           Page Content               │
│                                      │
├──────────────────────────────────────┤
│ Footer (links, copyright)            │
└──────────────────────────────────────┘
```

**Customer pages** (Profile, My Tickets):
```
┌──────────────────────────────────────┐
│ Navbar (logo, nav links, avatar)     │
├──────────────────────────────────────┤
│                                      │
│           Page Content               │
│                                      │
├──────────────────────────────────────┤
│ Footer                               │
└──────────────────────────────────────┘
```

**Dashboard pages** (Organizer):
```
┌──────────────────────────────────────┐
│ Dashboard Header (title, avatar)     │
├────────┬─────────────────────────────┤
│        │                             │
│ Sidebar│      Page Content           │
│ (nav)  │                             │
│        │                             │
├────────┴─────────────────────────────┤
```
Sidebar collapses to a hamburger menu on mobile (`< md` breakpoint).

### 2.7 TransactionStatus Badge Colors

| Status | Badge Color | Text |
|---|---|---|
| `WAITING_FOR_PAYMENT` | `bg-amber-100 text-amber-800` / dark: `bg-amber-900/30 text-amber-400` | "Waiting for Payment" |
| `WAITING_FOR_ADMIN_CONFIRMATION` | `bg-blue-100 text-blue-800` / dark: `bg-blue-900/30 text-blue-400` | "Awaiting Confirmation" |
| `DONE` | `bg-emerald-100 text-emerald-800` / dark: `bg-emerald-900/30 text-emerald-400` | "Completed" |
| `REJECTED` | `bg-red-100 text-red-800` / dark: `bg-red-900/30 text-red-400` | "Rejected" |
| `EXPIRED` | `bg-gray-100 text-gray-800` / dark: `bg-gray-900/30 text-gray-400` | "Expired" |
| `CANCELED` | `bg-orange-100 text-orange-800` / dark: `bg-orange-900/30 text-orange-400` | "Canceled" |
