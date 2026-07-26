# 🎟️ Event Management & Ticket Booking Platform
**Purwadhika Bootcamp Mini Project — Full-Stack Monorepo Application**

A state-of-the-art, dark-mode-first Event Management Platform built with **Next.js 15**, **Express 4**, **Prisma ORM**, and **PostgreSQL (Supabase)**. Developed using **Test-Driven Development (TDD)** and Matt Pocock's AI Agentic Skills (`/ask-matt`), this platform enables organizers to publish and manage events while giving customers a seamless, atomic checkout experience with seat reservations, promotional vouchers, FIFO point ledgers, and live countdown timers.

---

## 🌟 1. Main Application Features

* **🔍 Event Discovery & Catalog Browsing**: Debounced search (300ms), category filtering (8 reference categories), location filtering, date filtering, and server-side pagination.
* **🏢 Organizer Event Management**: Full CRUD lifecycle for events, dynamic ticket tier configuration (Free vs. Paid, multiple pricing tiers), image upload via Cloudinary, and soft deletion protection (`deletedAt`).
* **🛒 14-Step ACID Checkout & Ticket Purchasing**: Atomic database transactions ensuring zero overselling, real-time seat inventory reservation, promotional voucher validation, 10% discount coupons, and automated FIFO point usage (expiring oldest points first).
* **⏳ Payment Proof Upload & Lazy Timers**: Cloudinary payment receipt upload, interactive customer dashboard `/my-tickets`, and automated 2-hour countdown timers with query-time lazy expiration for Vercel serverless compatibility.
* **✅ Order Verification & Attendance Tracking**: Organizer transaction management table, modal proof inspection, accept/reject workflows, booking code generation (`TIX-XXXXXX`), and post-event check-in toggles (`isAttended`).
* **⭐ Verified Review & Rating System**: Post-event star ratings and written reviews gated strictly to attendees of completed transactions, with dynamic average rating calculations.
* **📊 Visual Analytics Dashboard**: Raw SQL database aggregation powered by **Recharts**, displaying revenue time series, ticket sales velocity, and category distribution pie charts.
* **🔐 Role-Based Security & Auth**: Distinct `CUSTOMER` and `ORGANIZER` roles, JWT authentication via httpOnly cookies, bcrypt password hashing, and Nodemailer email recovery flows.

---

## 🛠️ 2. Technology Stack

* **Monorepo Architecture**: npm workspaces (`apps/web` for frontend, `apps/api` for backend).
* **Frontend**: Next.js 15 (App Router), React 19, Vanilla CSS + Tailwind CSS v4, Lucide React icons, Recharts, Axios with silent token refresh interceptors.
* **Backend**: Node.js, Express 4, TypeScript, Prisma ORM, PostgreSQL (Supabase), Cloudinary SDK, Nodemailer, Multer, Zod validation.
* **Testing & Quality Assurance**: Jest (`ts-jest`, `next/jest`), Supertest (HTTP route handler verification), React Testing Library (RTL), database isolation helper (`truncateAll`).
* **CI/CD & DevOps**: GitHub Actions 5-gate automated verification pipeline (Type check, ESLint, Backend Jest, Frontend Jest, Production build).

---

## 📐 3. Entity Relationship Diagram (ERD)

Below is the database architecture governing our 11 core models and relational mappings:

```mermaid
erDiagram
    User ||--o{ Event : "organizes"
    User ||--o{ Transaction : "purchases"
    User ||--o{ Review : "writes"
    User ||--o{ PointLedger : "earns/spends"
    User ||--o{ Coupon : "receives"
    User ||--o{ PasswordResetToken : "requests"

    Event }o--|| Category : "belongs to"
    Event ||--o{ TicketType : "has"
    Event ||--o{ Transaction : "sold via"
    Event ||--o{ Review : "reviewed by"
    Event }o--o{ Voucher : "discounted by"

    Voucher }o--|| User : "created by"
    Voucher }o--o{ Event : "applies to"

    Transaction ||--o{ TransactionItem : "contains"
    Transaction }o--o| Voucher : "uses"
    Transaction }o--o| Coupon : "uses"

    TransactionItem }o--o| TicketType : "for"
```

*(For full Prisma schema definitions, indexes, and decimal precision formulas, reference [docs/prd/02-database-schema.md](./docs/prd/02-database-schema.md)).*

---

## 🚀 4. Local Setup Guide (Steps to Run Locally)

Follow these steps to initialize and run the full-stack platform on your local machine:

### Prerequisites
* Node.js v20+ and npm v10+
* A running PostgreSQL database (e.g., Supabase or local PostgreSQL 16)
* Cloudinary and Mailtrap/Nodemailer credentials (for image upload & emails)

### Step 1: Clone the Repository
```bash
git clone https://github.com/awanstywn/event-management-platform.git
cd event-management-platform
git checkout develop
```

### Step 2: Install Dependencies
From the root directory, install all workspace packages:
```bash
npm install
```

### Step 3: Configure Environment Variables
Create `.env` files in both the API and Web workspaces by copying the example files:
```bash
cp apps/api/.env.example apps/api/.env
cp apps/web/.env.example apps/web/.env
```
Update `apps/api/.env` with your PostgreSQL database URL and API keys:
```env
PORT=8000
DATABASE_URL="postgresql://postgres:password@localhost:5432/event_platform_db?schema=public"
JWT_SECRET="super-secret-jwt-key-minimum-32-characters"
JWT_REFRESH_SECRET="super-secret-refresh-key-minimum-32-characters"
CLOUDINARY_CLOUD_NAME="your-cloud-name"
CLOUDINARY_API_KEY="your-api-key"
CLOUDINARY_API_SECRET="your-api-secret"
SMTP_HOST="smtp.mailtrap.io"
SMTP_PORT=2525
SMTP_USER="your-mailtrap-user"
SMTP_PASS="your-mailtrap-password"
CLIENT_URL="http://localhost:3000"
```

### Step 4: Run Database Migrations & Seeding
Apply the Prisma schema to your database and seed reference categories and demo accounts:
```bash
npm run db:migrate --workspace=apps/api
npm run db:seed --workspace=apps/api
```

### Step 5: Start the Development Server
Run frontend (`http://localhost:3000`) and backend (`http://localhost:8000`) concurrently from root:
```bash
npm run dev
```

---

## 🌐 5. Deployment URLs

* **Frontend Web Application (Vercel)**: `https://event-platform-web.vercel.app` *(Staging/Demo URL)*
* **Backend REST API (Vercel Serverless)**: `https://event-platform-api.vercel.app/api/v1` *(Staging/Demo URL)*

---

## 🔑 6. Demo Accounts

The database seeder automatically creates the following demo accounts for immediate testing:

| Role | Email Address | Password | Account State & Wallet |
|---|---|---|---|
| **Customer** | `customer1@example.com` | `Password123!` | 10,000 Reward Points (expires in 3 months) + 10% Discount Coupon |
| **Customer** | `customer2@example.com` | `Password123!` | Fresh account with 0 points |
| **Organizer** | `organizer1@example.com` | `Password123!` | Managing 5 active Music & Tech events with active vouchers |
| **Organizer** | `organizer2@example.com` | `Password123!` | Managing 3 Sports & Art events |

---

## 🌿 7. Git Workflow & Team Handshake

We adhere strictly to **Section 4.2 Git Workflow Standards** and Matt Pocock AI Agentic Skills:
* **Branching Strategy**: `main` is reserved exclusively for production releases. `develop` is our active development branch. All feature branches (`feature/*`) must branch off of `develop`.
* **Branch Protection**: Direct pushes to `main` are strictly prohibited. All changes must pass our 5-gate CI/CD pipeline and merge via Pull Requests.
* **Commit Requirement**: Minimum **20 meaningful Conventional Commits** guaranteed across our 12 issue tickets.
* **Team Handshake**: For instructions on how teammates divide tickets, use AI `/implement` sessions, and execute TDD, read our root [TEAM_HANDSHAKE_AND_WORKFLOW.md](./TEAM_HANDSHAKE_AND_WORKFLOW.md).
* **Technical Specs**: Read our full 9-part PRD suite in [docs/prd/README.md](./docs/prd/README.md).
