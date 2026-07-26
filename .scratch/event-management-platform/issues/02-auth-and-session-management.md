# 02 — Authentication & Session Management

**What to build:** End-to-end user registration and login workflows with role selection (`CUSTOMER` or `ORGANIZER`), JWT generation stored in secure httpOnly cookies, silent token refresh interceptor on the frontend, and logout functionality. Users can create accounts, log in, and be redirected to role-specific landing pages.

**Blocked by:** 01 — Project Initialization, Database Schema & Reference Data Seeding

**Status:** ready-for-agent

- [ ] Implement `POST /api/v1/auth/register` validating fields with Zod, hashing passwords with bcrypt, generating unique referral code (`REF-XXXXXX`), generating JWT access (15m) and refresh (7d) tokens, setting httpOnly cookies, and firing asynchronous welcome email
- [ ] Implement `POST /api/v1/auth/login` verifying email and password, setting auth cookies, and returning user profile data
- [ ] Implement `POST /api/v1/auth/refresh` and `POST /api/v1/auth/logout` endpoints managing cookie lifecycle
- [ ] Implement backend auth middleware (`authenticate.ts` and `authorize.ts`) attaching user identity to Express Request
- [ ] Build frontend auth client (`lib/api/axios.ts`) with automatic silent token refresh interceptor on 401 Unauthorized responses
- [ ] Create UI registration (`app/register`) and login (`app/login`) pages with form validation using React Hook Form and Zod
- [ ] Build AuthProvider context maintaining user session and automatically redirecting Customers to `/` and Organizers to `/dashboard` upon login
- [ ] TDD: Write failing integration test suites (`apps/api/__tests__/integration/auth/register.test.ts`, `login.test.ts`) via supertest verifying cookie generation, enumeration defense, and referral rewards before implementing endpoints (Red-Green-Refactor)
- [ ] TDD: Write React Testing Library unit tests (`apps/web/__tests__/components/LoginForm.test.tsx`, `RegisterForm.test.tsx`) verifying Zod inline error rendering and role redirection
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

