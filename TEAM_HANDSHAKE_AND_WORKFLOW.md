# 🤝 Team Handshake & Workflow Guide
**Event Management Platform — Purwadhika Bootcamp Mini Project**

Welcome to the team workflow blueprint! Before writing a single line of feature code, we ran our product requirements through a rigorous **AI-Powered Architecture & TDD Alignment Workflow** (based on the Matt Pocock `/ask-matt` methodology). 

Instead of guessing how to structure our monorepo, arguing over file naming, or fighting massive git merge conflicts later, we have generated a **complete technical specification, database architecture, and 12 AI-Ready Issue Tickets**.

This document explains what we built, where everything lives, how we divide the work without stepping on each other's toes, and the exact steps to build features using AI and Test-Driven Development (TDD).

---

## 📂 1. What We Built & Where Everything Lives

All project documentation is already formatted and checked into our repository. When you share or push this project to GitHub, **share the entire repository root**, especially these critical folders:

### 📕 A0. Root Project README (`README.md`)
* Located at the project root, this fulfills **Section 4.1 README requirements**. It contains our project feature summary, tech stack, Mermaid Entity Relationship Diagram (ERD), local setup guide, staging deployment URLs, and demo accounts!

### 📙 A1. Requirements Traceability Audit Report (`TRACEABILITY_AUDIT_REPORT.md`)
* Located at the project root, this document proves **100% requirement coverage** across all 16 pages of `miniproject-requirement.pdf`. It includes our complete master traceability matrix mapping features, edge cases, TDD seams, and CI/CD rules to our PRDs and 12 issue tickets!

### 📗 A. Product Requirements & Architecture (`docs/prd/`)
This is our **Single Source of Truth**. Whenever you or an AI agent need to know how an endpoint behaves, what a UI looks like, or how prices are calculated, refer to these files:
* **`00-global-context.md`**: Our tech stack (Next.js 15, Express 4, Prisma, Tailwind v4, Recharts), naming conventions, and standard API JSON response envelope. *(Pro-tip: Paste this content at the top of your AI prompts!)*
* **`01-architecture-and-design.md`**: Our monorepo tree (`apps/web` and `apps/api`), folder responsibilities, and our 5-gate CI/CD verification pipeline.
* **`02-database-schema.md`**: Our 11 Prisma database models, price/discount calculation formula, seeder specs, and test database reset utilities (`truncateAll`).
* **`03-api-endpoints.md`**: Complete REST API dictionary for all `/api/v1` routes with request/response payloads.
* **`04-shared-infrastructure.md`**: Shared cross-cutting code (Axios interceptor for silent token refresh, global error handler, custom hooks).
* **`05` through `08` User Stories**: Deep-dive user stories for Authentication, Events, Transactions, and Social/Dashboard features with automated testing rules.

### 📘 B. Architecture Decision Records (`docs/adr/`)
* **`0001-database-schema-decisions.md`**: Explains why we chose UUIDs, Decimal pricing, and soft deletion (`deletedAt`).
* **`0002-tdd-and-cicd-strategy.md`**: Explains our locked TDD decisions (Jest runner, supertest API route seams, React Testing Library UI seams, offline SDK mocking, and GitHub Actions CI/CD).

### 📙 C. The Issue Tracker (`.scratch/event-management-platform/issues/`)
We translated our specifications into **12 vertical-slice issue tickets (`01` through `12`)**. 
* Each ticket represents a self-contained feature that can be built and tested in isolation.
* Each ticket lists its **Blockers** (which ticket must be merged before starting it) and explicit **TDD Red-Green-Refactor tasks**.

### 📓 D. Core Domain Glossary (`CONTEXT.md`)
* Located at the project root, this defines our Ubiquitous Language (e.g., what a `Customer` vs `Organizer` is, what `DONE` status means, how point ledgers work).

---

## 🧠 2. Built With Matt Pocock's Agentic Skills & How to Continue Using Them

This entire project foundation was created using **Matt Pocock's Agentic Skills (`mattpocock/skills`)** — a structured AI engineering methodology designed to prevent LLM hallucinations, enforce clean architecture, and maintain high code quality across multi-session builds.

### What We Already Executed:
1. **`/grill-with-docs`**: An interactive interview with the AI that challenged our initial assumptions, locked down edge cases, and generated our Architecture Decision Records (`ADR 0001`, `ADR 0002`) and domain glossary (`CONTEXT.md`).
2. **`/to-spec`**: Collapsed those interview decisions into our comprehensive 9-part Product Requirements Document suite (`docs/prd/`).
3. **`/to-tickets`**: Broke the master specification down into our 12 tracer-bullet issue tickets (`.scratch/event-management-platform/issues/`), each declaring explicit blocking dependencies and TDD checklists.

### How Your Team Can Continue Using These Skills Daily:
You and your teammate can invoke these specialized commands in your AI chat window at any point during development:

| Command / Skill | When to Use | What It Does |
|---|---|---|
| **`/implement`** | Starting a new ticket | Drives development from an issue ticket by executing test-driven development (TDD) internally — one red-green slice at a time — and running a final code review before committing. |
| **`/tdd`** | Building a standalone feature or bugfix | Writes a failing test first (Red), implements minimum passing code (Green), and cleans up (Refactor) without needing a full specification file. |
| **`/code-review`** | Before opening a Pull Request | Spawns two parallel sub-agents to review your branch against two axes: **Standards** (clean code, repo rules) and **Spec** (did you fulfill 100% of the ticket without breaking anything?). |
| **`/diagnosing-bugs`** | When something breaks or flakes | For tough bugs or regressions. It refuses to guess; it first creates a single command that fails reliably on the bug, then fixes it with a regression test. |
| **`/improve-codebase-architecture`** | During refactoring or technical debt cleanup | Scans the codebase for "deepening opportunities" and architectural seams to keep the project clean and AI-navigable. |
| **`/grill-with-docs`** | Adding a brand new feature in the future | If your team decides to add a new feature later, run this first! It will interview you about edge cases and update `CONTEXT.md` and ADRs before you write code. |

---

## 🤖 3. How Our AI-Powered TDD Workflow Works

We follow 5 golden rules to keep our development fast, bug-free, and clean:

### Rule 1: One Ticket, One Fresh AI Session 🧼
When starting a new ticket, **never reuse an old AI chat session**. As context windows fill up, AI models degrade and start hallucinating or forgetting architectural rules. 
* Always open a **brand new AI chat session** for each ticket.
* Give the AI our global context and point it directly to the specific ticket file in `.scratch/event-management-platform/issues/`.

### Rule 2: Test-Driven Development (Red-Green-Refactor) 🚦
We do not write production code first and test later. For every feature:
1. **Red:** Let the AI write the failing test suite first (using `supertest` for backend API routes or `React Testing Library` for frontend UI).
2. **Green:** Write the minimum production code needed to make the tests pass.
3. **Refactor:** Clean up the code while keeping tests green.

### Rule 3: Automated Code Review Before PR Submission 🔍
Before opening a Pull Request to merge your branch into `main`, ask your AI assistant to run `/code-review`. This reviews your branch against two axes:
1. **Standards:** Does the code follow our monorepo conventions and clean code practices?
2. **Spec:** Does the diff fulfill 100% of what the issue ticket and PRD asked for without missing edge cases?

### Rule 4: The 5-Gate CI/CD Verification Pipeline 🛡️
We have configured GitHub Actions (`.github/workflows/ci.yml`) to enforce 5 merge-blocking gates on every pull request against a live PostgreSQL 16 service container:
1. **Gate 1:** TypeScript Type Check (`npm run type-check`)
2. **Gate 2:** ESLint Check (`npm run lint`)
3. **Gate 3:** Backend Integration & Unit Tests (`npm test --workspace=apps/api`)
4. **Gate 4:** Frontend Component Tests (`npm test --workspace=apps/web`)
5. **Gate 5:** Production Bundle Build (`npm run build`)

### Rule 5: Strict Git Workflow & Branching Standards 🌿
To comply with our project requirements (Section 4.2 Git Workflow), our team enforces the following Git standards:
* **Branching Strategy:** We use `main` strictly for production releases and `develop` as our active development integration branch. Feature branches must be created from `develop` (e.g., `git checkout -b feature/auth develop`).
* **No Direct Pushes to Main:** Direct pushing to the `main` branch is strictly prohibited. All feature changes must be submitted via Pull Requests targeting `develop`. Production promotions go via Pull Request from `develop` to `main`.
* **Commit Standards:** We guarantee a **minimum of 20 meaningful commits** across our 12 issue tickets. Every commit message must use a descriptive format (e.g., `feat: add pagination to product list` or `feat(be/auth): implement login endpoint`).
* **Gitignore Hygiene:** Our root `.gitignore` explicitly excludes `node_modules`, `.env`, `dist`, and `build` from source control.

---

## 🗺️ 4. How to Divide & Conquer the 12 Tickets

Our 12 tickets are organized into a dependency tree. Here is how we divide them between two team members so we never work on overlapping files or block each other:

```
                      [ Ticket 01: Project Init & Seeding ]
                                       │
                                       ▼
                   [ Ticket 02: Auth & Session Management ]
                                       │
                ┌──────────────────────┴──────────────────────┐
                ▼                                             ▼
  Teammate A (Customer & Marketplace Focus)     Teammate B (Organizer & Management Focus)
  ─────────────────────────────────────────     ─────────────────────────────────────────
  [04] Event Discovery & Catalog Browsing       [03] Profile Mgmt & Password Security
                │                                             │
                ▼                                             ▼
  [05] Event Detail View & Organizer Profile    [06] Event Management Lifecycle
                │                                             │
                ▼                                             ▼
  [08] Ticket Purchasing & Entitlement          [07] Promotional Campaigns & Vouchers
                │                                             │
                ▼                                             ▼
  [09] Payment Proof Upload & Lazy Timers       [10] Organizer Order Verification & Attendance
                │                                             │
                └───────────────┬─────────────────────────────┘
                                ▼
         [11] Post-Event Reviews & Verified Rating System
                                │
                                ▼
         [12] Organizer Visual Analytics Dashboard (Recharts)
```

### Phase 1: Foundation (Done Together or by 1 Person)
* **Ticket 01 — Project Initialization, Schema & Seeding**: This MUST be done first. It sets up our monorepo workspaces, Express backend, Next.js frontend, Prisma schema with PostgreSQL, reference data seeder (30+ records), Jest runner, and GitHub Actions CI/CD.

### Phase 2: Parallel Tracks (Divide and Conquer!)
Once Ticket 01 is merged into `main`, both teammates can work in parallel:

#### 🧑‍💻 Teammate A: Customer Marketplace & Checkout Track
1. **Ticket 02 (Auth & Sessions):** Login, registration, JWT httpOnly cookies, and AuthProvider context.
2. **Ticket 04 (Event Catalog):** Browsing page `/events`, search bar with 300ms debounce, category/price filtering, and pagination.
3. **Ticket 05 (Event Detail):** Event detail page `/events/[id]`, ticket selection tier cards, and public organizer profile.
4. **Ticket 08 (Ticket Purchasing):** The 14-step atomic checkout transaction, voucher/coupon mutual exclusivity, FIFO point usage, and seat reservations.
5. **Ticket 09 (Payment Proof & Timers):** Customer `/my-tickets/[id]` page, live countdown timer (`useCountdown`), payment proof image upload to Cloudinary, and query-time lazy expiration.

#### 🧑‍💻 Teammate B: Organizer Management & Dashboard Track
1. **Ticket 03 (Profile & Security):** Profile editing, avatar upload, referral code copy, forgot/reset password email flows via Nodemailer.
2. **Ticket 06 (Event Lifecycle):** Organizer `/dashboard/events` creation and editing forms with dynamic ticket tier rows, image dropzone, and soft deletion protection.
3. **Ticket 07 (Vouchers & Points):** Voucher management `/dashboard/vouchers`, referral code reward calculation (10k points + 10% coupon), and point ledger balance aggregation.
4. **Ticket 10 (Order Verification):** Organizer `/dashboard/transactions` table, inspecting payment proofs, accept/reject modal, booking code generation, and attendee check-in (`isAttended` toggle).

### Phase 3: Final Integration (Done Together)
1. **Ticket 11 (Reviews & Ratings):** Post-event star rating submission (only for attended completed transactions) and dynamic average rating calculation.
2. **Ticket 12 (Analytics Dashboard):** Organizer visual dashboard `/dashboard` using raw SQL aggregation and Recharts (Revenue time series, ticket velocity, category pie chart).

---

## 🚀 5. Step-by-Step: How to Start Coding Your First Ticket

When you sit down to start a ticket (e.g., Ticket 02), follow these exact 4 steps:

1. **Pull the latest `develop` branch:**
   ```bash
   git checkout develop
   git pull origin develop
   git checkout -b feature/issue-02-auth-and-session
   ```

2. **Open a fresh AI Chat Window** in your IDE and paste this exact prompt:
   ```markdown
   We are implementing Ticket 02 from our issue tracker. 
   Please read `docs/prd/00-global-context.md` for our stack rules, and read `.scratch/event-management-platform/issues/02-auth-and-session-management.md` for our task checklist.
   
   Let's execute this ticket using our locked TDD strategy (ADR 0002):
   1. First, write the failing integration tests via supertest for our auth endpoints (Red).
   2. Then implement the backend routes, controllers, and services to make tests pass (Green).
   3. Build the Next.js UI login/register components with React Testing Library assertions.
   
   Let's begin with Step 1!
   ```

3. **Verify locally before committing:**
   ```bash
   # Run all verification gates locally
   npm run type-check --workspaces
   npm run lint --workspaces
   npm test --workspaces
   npm run build --workspaces
   ```

4. **Ask AI for a final Code Review:**
   In your chat window, prompt: *"Please run `/code-review` on my git diff against our coding standards and Ticket 02 requirements."* Once approved, push your branch (`git push -u origin feature/issue-02-auth-and-session`) and open a Pull Request targeting the **`develop`** branch!

---
*Happy Coding! Let's build an extraordinary, premium Event Management Platform! 🎯🚀*
