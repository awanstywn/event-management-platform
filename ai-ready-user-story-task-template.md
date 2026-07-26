# User Story & Technical Task Template — Hardened for AI Code Generation

> Same structure as before, but every element is now specified at a level of detail that leaves an AI coding tool nothing to guess. Includes a fully worked dummy example at the bottom.

## Why Vague Specs Cause AI Hallucination

An AI code generator will always produce *something* — if a detail is missing, it fills the gap with a plausible-sounding guess instead of stopping to ask. Each row below is a real failure mode, not a hypothetical:

| If you leave this vague... | The AI typically does this |
|---|---|
| Field types not stated | Guesses `string` when you needed `Decimal`, or vice versa |
| No exact endpoint path | Invents `/api/events/create` instead of your actual `POST /api/v1/events` |
| No enum values listed | Invents status strings like `"active"` when your DB expects `"PUBLISHED"` |
| No file path / folder convention | Scatters files inconsistently across the codebase |
| No "out of scope" section | Silently builds extra features you didn't ask for |
| No explicit DTO/interface | Field names drift between frontend and backend (`eventName` vs `name`) |
| No instruction to ask when unsure | Never tells you it guessed — you find out at runtime |

**The fix:** give exact types, exact paths, exact enum values, and an explicit instruction to stop and ask rather than assume. That's what's added below.

---

## STEP 0 — Global Project Context Block (fill ONCE, reuse for every story)

Paste this at the top of every AI prompt, alongside the specific story. Without it, the AI has no idea what conventions already exist and will invent its own each time — which is why the same app ends up with three different naming styles.

```markdown
## Global Context

**Stack & Versions:**
- Frontend: Next.js 14 (App Router), TypeScript 5.x, TailwindCSS, React Hook Form + Zod, TanStack Query
- Backend: Node.js 20, Express 4.x, TypeScript 5.x, Prisma ORM 5.x
- Database: PostgreSQL 15
- Auth: JWT — access token (15 min) + refresh token (7 days), stored in httpOnly cookies named `accessToken` / `refreshToken`

**Folder Structure:**
backend/src/{routes,controllers,services,middlewares,validators,repositories,utils}
backend/prisma/schema.prisma
frontend/app/{route folders}
frontend/components/, frontend/hooks/, frontend/lib/, frontend/types/

**Naming Conventions:**
- Variables/functions: camelCase
- Components/Types/Interfaces: PascalCase
- Non-component files: kebab-case (e.g. `event.service.ts`)
- React component files: PascalCase.tsx
- DB columns: camelCase in Prisma (mapped automatically), no manual @map unless noted

**API Response Envelope (all endpoints, no exceptions):**
Success: { success: true, message: string, data: T }
Error:   { success: false, message: string, errors?: Record<string, string> }

**AI Directive:** If any detail needed to implement this story is not explicitly stated below,
do NOT invent a plausible default. Insert `// TODO: clarify — [what's missing]` and continue
with the rest, or stop and ask.
```

---

## PART 1 — USER STORY COMPONENTS (hardened)

All 8 original elements, plus two new ones that close the biggest hallucination gaps:

### 9. Assumptions
**What it is:** Things you're taking as given so the AI doesn't have to guess them.
💡 **AI Directive:** State them as flat facts: "Categories are pre-seeded, not created here." An AI without this will happily generate a category-creation form you never asked for.

### 10. Out of Scope / Non-Goals
**What it is:** Explicitly what this story does *not* cover.
💡 **AI Directive:** This is the single highest-leverage anti-hallucination addition. AI models are trained to be "helpful," which means unconstrained they will over-build. Naming the boundary stops that.

---

## PART 2 — TECHNICAL TASK BREAKDOWN (hardened)

The difference from the previous version: every row must contain the **actual artifact** (a type, a path, a schema), not a description of one.

### A. Frontend

| Element | Hardened requirement |
|---|---|
| Page / View | Exact file path, e.g. `app/dashboard/events/new/page.tsx`, plus route protection rule |
| UI Components | Exact component names + file paths, not just "a form" |
| Types | A real TypeScript interface for the form payload, written out in full |
| State Management | Name the exact tool per layer (e.g. `react-hook-form` for local, `TanStack Query useMutation` for server state) — never just "state" |
| Form Validation | The actual Zod schema code, matching the backend validator field-for-field |
| UI States | Explicit behavior per state (loading/error/empty), not just "handle errors" |

### B. Backend

| Element | Hardened requirement |
|---|---|
| Endpoint | Exact method + full path |
| Router file | Exact path, e.g. `src/routes/event.routes.ts` |
| Middleware order | Numbered list, in execution order — order bugs are a common AI mistake |
| Request DTO | Full TypeScript interface |
| Response DTO | Full TypeScript interface, success AND error shape |
| Validator | Actual Zod/Joi schema code, not a text description of the rules |
| Service | Exact function signature + what it orchestrates |
| Controller | Confirm it stays thin — name what it's NOT allowed to contain (business logic) |
| Repository | Exact Prisma call pattern |
| Error handling | Confirm all errors go through `next(err)` to the named global handler file |

### C. Database

| Element | Hardened requirement |
|---|---|
| Schema | Actual `schema.prisma` model block, not a field list in prose |
| Relationships | State cardinality explicitly (one-to-many, many-to-many + junction table name) |
| Transaction | State which exact operations must be wrapped in `prisma.$transaction()` |
| Soft delete | State the exact field name (`deletedAt`) and whether queries must filter it out by default |
| Seed | State what data and how much |

---

## Anti-Hallucination Checklist (run before sending to the AI)

- [ ] Global Context Block is included in the same prompt as the story
- [ ] Every field has an explicit type — no bare "etc."
- [ ] Every enum has its exact string values listed
- [ ] Every endpoint has full method + path
- [ ] Every file has an explicit path, not "somewhere in components"
- [ ] "Out of Scope" section is filled in, not left blank
- [ ] Request/Response DTOs are written as real interfaces, not descriptions
- [ ] The AI Directive ("don't assume, ask") is present
- [ ] Prisma schema is pasted as real code, not summarized

---

## Recommended Fill-In Order

1. Global Context Block — once per project
2. Part 1, items 1 → 10, in order
3. Part 2, section A → B → C
4. Cross-check: every row in the Error Handling Matrix maps to something concrete in Part 2 (a status code, a UI state)

---

# WORKED DUMMY EXAMPLE

*Feature: Event Organizer creates a new event. This demonstrates the level of detail expected — copy this pattern for your real stories.*

## 🎯 User Story: Organizer Creates a New Event

**Story:** As an Event Organizer, I want to create a new event with full details (name, category, schedule, pricing, seats, optional ticket types, banner image), so that it becomes visible to customers on the discovery page.

**Context:** Entry point of the Event Discovery feature. Once created, the event appears on the public Landing Page. This story covers creation only — editing and deleting are separate stories.

**Roles:** Organizer only. Customers cannot access this page or endpoint.

**Assumptions:** Organizer is already authenticated. Categories are pre-seeded, not created here. Ticket types are optional — an event can have a single flat price instead.

**Out of Scope:** Editing/deleting events, draft/publish toggle (events are published immediately on creation for this MVP), voucher creation, ticket type editing after creation.

**Happy Path:**
1. Organizer opens `/dashboard/events/new`
2. Fills form: name, category, location, description, start/end datetime, total seats, isFree toggle, price (if paid), optional ticket types, optional banner image
3. Client-side Zod validation passes
4. Frontend sends `POST /api/v1/events` as `multipart/form-data`
5. Backend runs auth → role check → file parsing → validation
6. Banner image (if present) uploaded to cloud storage
7. Event + TicketType rows created together in one DB transaction
8. `201` returned with the created event
9. Frontend shows success toast, redirects to `/dashboard/events/[id]`

**Unhappy Paths:**
- Client validation fails → inline field errors, no request sent
- `endDate <= startDate` → `400`, no event created
- `isFree = false` and `price` missing → `400`
- Invalid image type/size → `400`, event not created
- No/expired token → `401`
- Role ≠ ORGANIZER → `403`
- DB or upload failure mid-operation → transaction rolled back, `500`, no partial event created

**Business Constraints:**
- `endDate` strictly after `startDate`
- `totalSeats`: integer, > 0
- `price` required and > 0 if `isFree = false`
- Banner image: max 2MB, `jpg`/`png`/`webp` only
- `name`: 3–100 characters
- `description`: max 2000 characters

**Error Handling Matrix:**
| Trigger | Code | Message |
|---|---|---|
| Missing required field | 400 | `"{field} is required"` |
| `endDate <= startDate` | 400 | `"End date must be after start date"` |
| `price` missing while `isFree=false` | 400 | `"Price is required for paid events"` |
| Invalid image type | 400 | `"Only JPG, PNG, or WEBP images are allowed"` |
| Image > 2MB | 400 | `"Image must be smaller than 2MB"` |
| No/expired token | 401 | `"Please log in to continue"` |
| Role ≠ ORGANIZER | 403 | `"Only event organizers can create events"` |
| DB/upload failure | 500 | `"Something went wrong. Please try again."` |

**Acceptance Criteria:**
- Given a logged-in organizer with valid data, When they submit the form, Then a new event is created with status `PUBLISHED` and they're redirected to the event detail page.
- Given `endDate` before `startDate`, When submitted, Then the API returns `400` with message `"End date must be after start date"` and no event row is created.
- Given a logged-in customer, When they attempt to access `/dashboard/events/new`, Then they are redirected away.
- Given `isFree = false` and empty `price`, When submitted, Then a `400` validation error is returned for the `price` field.

**Definition of Done:**
- [ ] Unit tests for the validator (valid + each invalid case)
- [ ] Unit test for the date-range rule
- [ ] RBAC verified on both the Next.js route guard and the Express middleware
- [ ] No `console.log` left in the code
- [ ] Responsive on mobile/tablet/desktop
- [ ] All responses follow the standard envelope
- [ ] Seed script includes at least 3 sample events across categories

---

## 🛠️ Technical Task Breakdown

### Frontend

- **Page:** `app/dashboard/events/new/page.tsx` — server component wrapper that checks role server-side before rendering the client form
- **Components:** `components/events/EventForm.tsx`, `components/events/ImageUploadField.tsx`, `components/events/TicketTypeRepeater.tsx`, `components/events/DateRangePicker.tsx`

- **Type (`types/event.ts`):**
```ts
export interface CreateEventFormValues {
  name: string;
  categoryId: string;
  location: string;
  description: string;
  startDate: string;   // ISO 8601
  endDate: string;     // ISO 8601
  totalSeats: number;
  isFree: boolean;
  price?: number;
  ticketTypes?: { name: string; price: number; quota: number }[];
  bannerImage?: File;
}
```

- **State:** Form state via `react-hook-form` (local). Submission via TanStack Query `useMutation` calling `createEvent()` in `lib/api/events.ts` (server state). No global state needed for this story.

- **Validation (`lib/validators/event.schema.ts`):**
```ts
import { z } from "zod";

export const createEventClientSchema = z.object({
  name: z.string().min(3).max(100),
  categoryId: z.string().uuid(),
  location: z.string().min(3),
  description: z.string().max(2000),
  startDate: z.string().datetime(),
  endDate: z.string().datetime(),
  totalSeats: z.coerce.number().int().positive(),
  isFree: z.boolean(),
  price: z.coerce.number().positive().optional(),
}).refine(d => new Date(d.endDate) > new Date(d.startDate), {
  message: "End date must be after start date", path: ["endDate"],
}).refine(d => d.isFree || d.price !== undefined, {
  message: "Price is required for paid events", path: ["price"],
});
```

- **UI States:** Submit button shows spinner + disabled while mutation is pending. Field-level errors rendered inline from Zod. Toast on success/failure. Image field shows its own upload-progress state independent of form submit state.

### Backend

- **Endpoint:** `POST /api/v1/events`
- **Router:** `src/routes/event.routes.ts`
- **Content type:** `multipart/form-data` (required because of the file field). `ticketTypes` must be sent as a JSON-stringified field and parsed server-side before validation — this is exactly the kind of detail that gets silently dropped if not stated.

- **Middleware order (exact):**
  1. `authenticate` (verifies JWT, attaches `req.user`)
  2. `authorize(["ORGANIZER"])`
  3. `upload.single("bannerImage")` (Multer, memory storage, 2MB limit, mimetype filter)
  4. `parseTicketTypesField` (JSON.parse on `req.body.ticketTypes` if present)
  5. `validate(createEventSchema)`
  6. `eventController.create`

- **Request DTO (`src/types/event.dto.ts`):**
```ts
export interface CreateEventDTO {
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
}
```

- **Response DTO:**
```ts
export interface EventResponse {
  id: string;
  name: string;
  slug: string;
  organizerId: string;
  status: "PUBLISHED";
  bannerUrl: string | null;
  createdAt: string;
}
```
Success: `201 { success: true, message: "Event created successfully", data: EventResponse }`
Error: `400 { success: false, message: "Validation failed", errors: { field: string } }`

- **Validator (`src/validators/event.validator.ts`):** same rules as the client schema above, server-side Zod — this is the source of truth; client validation is UX only.

- **Service (`src/services/event.service.ts`):**
  `createEvent(organizerId: string, dto: CreateEventDTO, bannerFile?: Express.Multer.File): Promise<EventResponse>`
  Responsibilities: generate slug from `name`, upload `bannerFile` to cloud storage if present, call repository inside `prisma.$transaction()` to create the `Event` row and its `TicketType` rows together.

- **Controller (`src/controllers/event.controller.ts`):** calls the service, wraps in try/catch, forwards errors via `next(err)`. Contains no business logic — if you find yourself writing an `if` statement about business rules here, it belongs in the service instead.

- **Repository (`src/repositories/event.repository.ts`):**
```ts
create(tx: Prisma.TransactionClient, data: Prisma.EventCreateInput) {
  return tx.event.create({ data, include: { ticketTypes: true } });
}
```

- **Error handling:** all errors passed to `src/middlewares/errorHandler.ts` (the single global handler — do not create per-route error handling).

### Database

```prisma
model Event {
  id             String       @id @default(uuid())
  name           String
  slug           String       @unique
  organizerId    String
  organizer      User         @relation(fields: [organizerId], references: [id])
  categoryId     String
  category       Category     @relation(fields: [categoryId], references: [id])
  location       String
  description    String
  startDate      DateTime
  endDate        DateTime
  totalSeats     Int
  availableSeats Int
  isFree         Boolean      @default(false)
  price          Decimal?     @db.Decimal(12, 2)
  bannerUrl      String?
  status         EventStatus  @default(PUBLISHED)
  ticketTypes    TicketType[]
  createdAt      DateTime     @default(now())
  updatedAt      DateTime     @updatedAt
  deletedAt      DateTime?

  @@index([organizerId])
  @@index([categoryId])
}

enum EventStatus {
  DRAFT
  PUBLISHED
  CANCELLED
}

model TicketType {
  id      String  @id @default(uuid())
  eventId String
  event   Event   @relation(fields: [eventId], references: [id])
  name    String
  price   Decimal @db.Decimal(12, 2)
  quota   Int
}
```

- **Relationships:** `Event → User` (many-to-one, organizer), `Event → Category` (many-to-one), `Event → TicketType` (one-to-many)
- **Transaction:** `Event` create + `TicketType` creates must be wrapped in a single `prisma.$transaction()` — two related tables, one logical operation
- **Soft delete:** `deletedAt` field present now for future use; all `findMany`/`findFirst` queries on `Event` must filter `deletedAt: null` by default (not exercised by this story, but the field must exist from day one to avoid a migration later)
- **Seed:** 3 categories, 5 sample events distributed across them, at least 2 with `ticketTypes`, at least 1 free event
