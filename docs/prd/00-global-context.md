# Global Context Block

> **Usage:** Paste this block at the top of every AI prompt alongside a specific user story. Without it, the AI has no idea what conventions exist and will invent its own.

```markdown
## Global Context

**Project:** Event Management Platform (Mini Project — Purwadhika Bootcamp)

**Stack & Versions:**
- Frontend: Next.js 15 (App Router), TypeScript 5.x, Tailwind CSS v4, React Hook Form + Zod, TanStack Query v5, Axios, date-fns, React Hot Toast, Recharts
- Backend: Node.js 20, Express 4.x, TypeScript 5.x, Prisma ORM 5.x
- Database: PostgreSQL 15 (Supabase)
- Auth: JWT — access token (15 min) + refresh token (7 days), stored in httpOnly cookies named `accessToken` / `refreshToken`
- File Upload: Multer (memory storage) → Cloudinary (cloud)
- Email: Nodemailer — Mailtrap (dev) / Gmail SMTP (prod)
- Deployment: Frontend → Vercel, Backend → Vercel (serverless), Database → Supabase
- Chart Library: Recharts
- Testing: Jest (`ts-jest`, `next/jest`), `supertest`, React Testing Library (RTL), `nock`, `jest.mock()`, GitHub Actions CI/CD

**Monorepo Structure:**
Root/
├── .github/workflows/ci.yml (5-gate verification pipeline)
├── apps/
│   ├── api/     (Express backend)
│   └── web/     (Next.js frontend)
├── package.json (npm workspaces: ["apps/*"])

**Backend Folder Structure:**
apps/api/src/{config,routes,controllers,services,repositories,middlewares,validators,types,utils,templates}
apps/api/__tests__/{integration,unit,setup}/
apps/api/prisma/{schema.prisma,seed.ts}
apps/api/jest.config.ts

**Frontend Folder Structure:**
apps/web/app/{route folders with page.tsx, layout.tsx}
apps/web/components/{ui,layout,auth,events,transactions,dashboard,reviews,vouchers}/
apps/web/__tests__/{components,hooks,lib}/
apps/web/hooks/
apps/web/lib/{api,validators,utils.ts}
apps/web/types/
apps/web/providers/
apps/web/jest.config.ts


**Naming Conventions:**
- Variables/functions: camelCase
- Components/Types/Interfaces: PascalCase
- Non-component files: kebab-case (e.g. `event.service.ts`, `event.routes.ts`)
- React component files: PascalCase.tsx (e.g. `EventCard.tsx`)
- DB columns: camelCase in Prisma (auto-mapped)
- Route files: kebab-case (e.g. `auth.routes.ts`)
- Enum values: UPPER_SNAKE_CASE (e.g. `WAITING_FOR_PAYMENT`)

**API Response Envelope (all endpoints, no exceptions):**
Success: { success: true, message: string, data: T }
Paginated: { success: true, message: string, data: T[], meta: { total: number, page: number, totalPages: number, limit: number } }
Error: { success: false, message: string, errors?: Record<string, string> }

**HTTP Status Codes Used:**
200 — OK (successful GET, PUT, PATCH)
201 — Created (successful POST that creates a resource)
400 — Bad Request (validation errors)
401 — Unauthorized (missing/expired token)
403 — Forbidden (wrong role)
404 — Not Found
409 — Conflict (duplicate resource)
500 — Internal Server Error

**Currency:** IDR (Indonesian Rupiah) — all prices stored as Decimal(12,2)

**Git Workflow & Branching Standards (Section 4.2):**
- Branching: `main` for production, `develop` for active development. Feature branches (`feature/*`) must branch off of `develop`.
- Direct push to `main` is strictly prohibited. All feature changes must merge into `develop` via PRs.
- Minimum 20 meaningful commits required across all issue tickets using descriptive Conventional Commits (e.g. `feat: add pagination to product list` or `feat(be/auth): implement login endpoint`).
- Root `.gitignore` must exclude: `node_modules`, `.env`, `dist`, and `build`.

**AI Directive:** If any detail needed to implement this story is not explicitly stated below,
do NOT invent a plausible default. Insert `// TODO: clarify — [what's missing]` and continue
with the rest, or stop and ask.
```
