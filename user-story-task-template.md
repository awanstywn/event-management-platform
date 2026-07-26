# User Story & Technical Task — Component Template Guide

> A reusable checklist for turning any feature idea into a developer-ready spec. Fill **Part 1** first (product/business view), then **Part 2** (engineering view). Reuse this same template for every feature in your Event Management Platform.

## How the Two Parts Connect

- **Part 1 – User Story** answers *what* the feature does and *why* it matters. This is what you'd agree on with a PM or client before any code is written.
- **Part 2 – Technical Task Breakdown** answers *how* it gets built — the actual FE/BE/DB work items a developer picks up.

**Rule of thumb:** lock down Part 1 before starting Part 2. If the "why" and "what" keep shifting, the technical breakdown gets thrown away and redone.

---

## PART 1 — USER STORY COMPONENTS

### 1. User Story Statement
**What it is:** The one-sentence summary of who benefits and why.

```
As a [role],
I want to [action],
so that [value/benefit].
```

💡 **Guidance:** Keep "so that" focused on human value, not implementation. *"So that I can pay for my ticket online"* is good. *"So that a POST request hits /api/transactions"* is not — that belongs in Part 2.

### 2. Context & Description
**What it is:** 2–4 sentences placing this story inside the bigger app — where the user comes from, what happens after.

💡 **Guidance:** Beginners often skip this and jump straight to screens. Without it, developers build the feature in isolation and miss how it connects (e.g. "this is step 2 of checkout, right after ticket selection").

### 3. User Persona & Roles
**What it is:** Exactly which role(s) can trigger this story, and any permission nuances.

💡 **Guidance:** Always state it explicitly — "Customer only," "Organizer only," or "Both, but the view differs." Ambiguity here is one of the most common causes of RBAC bugs later.

### 4. Request Flow / Workflow
**What it is:** Step-by-step: user action → system reaction → output, split into:
- **Happy Path** — everything works
- **Unhappy Path(s)** — validation fails, user cancels, session expires, etc.

💡 **Guidance:** Write it as a numbered list, not prose. If you can't number the steps cleanly, the flow isn't defined well enough for a developer to build against yet.

### 5. Business Constraints & Rules
**What it is:** Limits, thresholds, permissions, timing rules — e.g. "payment proof must be uploaded within 2 hours."

💡 **Guidance:** These are the rules that never show up in a UI mockup but absolutely must be coded. Any number, deadline, or "only if X" condition anywhere in the requirements belongs here.

### 6. Error Handling Matrix
**What it is:** A table mapping every failure trigger to what the user sees.

| Trigger / Condition | Error Code / Type | User-Facing Message |
|---|---|---|
| *e.g. Payment proof not uploaded within 2 hrs* | *410 / Expired* | *"Your transaction has expired."* |

💡 **Guidance:** For every happy-path step, ask "what if this fails?" — that's your matrix. Skipping this is the #1 reason frontend and backend disagree on error behavior later.

### 7. Acceptance Criteria (Gherkin format)
```
Given [initial context]
When [action taken]
Then [expected outcome]
```

💡 **Guidance:** Write one Given/When/Then block **per scenario** — happy path AND each edge case. If your AC list has only one scenario, you've probably only covered the happy path.

### 8. Definition of Done (DoD)
**What it is:** A checklist of "truly finished," not just "works on my machine" — tests written, docs updated, security checked, deployed.

💡 **Guidance:** AC = "does it work." DoD = "is it actually shippable." Beginners often confuse the two. DoD is usually the *same checklist* across every story (e.g. "unit tested," "no console.logs," "responsive," "no hardcoded secrets"), while AC is unique to each feature.

---

## PART 2 — TECHNICAL TASK BREAKDOWN

### A. Frontend (UI/UX Tasks)

| Element | What to specify | 💡 Guidance |
|---|---|---|
| Page / View | Route, e.g. `/events/[id]` | Ties the story to a physical screen in the app. |
| UI Components | Buttons, forms, modals, tables, toasts needed | List every component so devs know what to build vs. reuse. |
| State Management | Local (form fields, modal open/close) vs Global (auth user, cart) vs Server state (React Query/SWR cache) | The #1 thing beginners mix up — causes stale-UI bugs. |
| Form Validation | Client-side schema (Zod/Yup) | Should mirror backend validation, never replace it. |
| UI States | Loading / Error / Empty behavior | Your project rubric explicitly grades this — don't skip it. |
| Responsive Behavior | Breakpoints that matter for this page | Design mobile-first if users browse on phones. |

### B. Backend (BE Tasks)

| Element | What to specify | 💡 Guidance |
|---|---|---|
| API Endpoint | HTTP method + route, e.g. `POST /api/v1/transactions` | One row per endpoint this story touches. |
| Request Payload | JSON body/query/param schema | Be exact about required vs. optional fields and types. |
| Response Payload | Success + error shape | Use one consistent envelope app-wide: `{ success, message, data/errors }`. |
| Router | Which route file this lives in | Keeps routing organized as the app grows. |
| Middleware | Auth check, role guard, rate limit | List which middleware runs before the controller, in order. |
| Validator | Schema validation used (Zod/Joi) | Validation lives here — not inside the controller. |
| Service / Business Logic | Where the real rules live (points calc, seat check, voucher logic) | Keep separate from the controller; controllers should stay thin. |
| Controller | Orchestrates: validator → service → response | Should contain almost no logic of its own. |
| Repository / Data Access | Prisma queries used | Isolating this makes swapping ORMs or adding caching easier later. |
| Error Handling | try/catch → passed to global error handler | Never let a raw error object leak to the client. |

### C. Database & Data Modeling

| Element | What to specify | 💡 Guidance |
|---|---|---|
| Schema / Entity Changes | New/changed tables, fields, types | List the exact Prisma model changes. |
| Relationships & Constraints | Foreign keys, indexes, unique constraints | Get this wrong and you'll be rewriting migrations later. |
| Transactions | Which multi-table writes need `prisma.$transaction()` | Anything touching 2+ tables together (e.g. create transaction + decrement seats) needs this. |
| Soft Delete | Does this entity use `deletedAt` instead of a hard delete? | Your rubric requires this on main entities — decide per entity. |
| Seed Data | What demo data this story needs | Makes local dev and demo day painless. |

---

## Recommended Fill-In Order

1. Part 1, items 1 → 8, in order (don't skip to AC before Workflow is settled)
2. Part 2, section A → B → C
3. Cross-check: every row in the Error Handling Matrix (Part 1) should map to something in the Backend error handling / Frontend UI states (Part 2) — if it doesn't, one of the two is incomplete.

## One-Time Tip for Your Project Specifically

A few requirements from your rubric aren't really "per-feature" — they're shared infrastructure you'll build once and reuse:
- Search/filter/pagination pattern (backend query logic)
- Global error handler + response envelope
- Auth middleware + RBAC guard
- Mailer sending helper (Nodemailer, async)
- Cloud file upload helper (Multer + storage provider)

Worth writing a short **"Shared Infrastructure" spec** once, separate from individual feature stories, so you're not redefining the error envelope five times.

---

## Blank Copy-Paste Skeleton

```markdown
## 🎯 User Story: [Feature Name]

**Story:** As a ___, I want to ___, so that ___.
**Context:** 
**Roles:** 

**Happy Path:**
1. 
**Unhappy Path:**
1. 

**Constraints:**
- 

**Error Handling:**
| Trigger | Code | Message |
|---|---|---|
|  |  |  |

**Acceptance Criteria:**
- Given ___, When ___, Then ___

**DoD:**
- [ ] 

---

### Frontend
- Page:
- Components:
- State:
- Validation:

### Backend
- Endpoint:
- Request/Response:
- Middleware:
- Service logic:

### Database
- Schema changes:
- Relations:
- Transaction needed?:
- Soft delete?:
```
