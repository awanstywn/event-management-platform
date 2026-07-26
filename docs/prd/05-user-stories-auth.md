# User Stories — Authentication & Profile

> Each story follows the AI-ready template. Paste the Global Context Block (00-global-context.md) alongside any story when prompting an AI.

---

## US-01: User Registration

**Story:** As a visitor, I want to create an account with my email and password, so that I can access the platform as a Customer or Organizer.
**Traceability:** Mapped to Spec Stories #1, #2, #3 | Issue Tickets #02 & #07

**Context:** Entry point to the platform. After registration, the user is automatically logged in and redirected to the appropriate landing page based on their role. If a referral code is provided, the referrer gets 10,000 points and the new user gets a 10% coupon.

**Roles:** None (public page).

**Assumptions:**
- Categories are already seeded.
- Referral code validation happens server-side.
- The registration form includes a role selector (Customer or Organizer).

**Out of Scope:** Social login (Google, GitHub), email verification, admin approval.

**Happy Path:**
1. Visitor navigates to `/register`
2. Fills form: firstName, lastName, email, password, confirmPassword, role selection, optional referral code (auto-uppercased)
3. Client-side Zod validation passes
4. Frontend sends `POST /api/v1/auth/register`
5. Backend validates, hashes password, generates unique referralCode
6. If referral code provided: validates code → credits 10,000 points to referrer (PointLedger CREDIT, expires 3 months) → creates Coupon (10%, expires 3 months) for new user
7. Creates User row
8. Generates accessToken + refreshToken, sets httpOnly cookies
9. Sends welcome email asynchronously
10. Returns user data, frontend redirects to `/` (Customer) or `/dashboard` (Organizer)

**Unhappy Paths:**
- Client validation fails → inline field errors, no request sent
- Email already registered → `409`, "Email is already registered"
- Referral code not found → `404`, "Referral code not found"
- Weak password → `400`, field-level error on `password`

**Business Constraints:**
- Email: valid email format, unique
- Password: min 8 chars, at least 1 uppercase, 1 lowercase, 1 number, 1 special char
- firstName/lastName: 2-50 chars each
- Role: exactly `CUSTOMER` or `ORGANIZER`
- ReferralCode: auto-generated `REF-XXXXXX`, unique, immutable
- Referral reward: 10,000 points, expires 3 months from credit date
- Referral coupon: 10% discount, single-use, expires 3 months from creation

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Missing required field | 400 | `"{field} is required"` |
| Invalid email format | 400 | `"Invalid email address"` |
| Weak password | 400 | `"Password must be at least 8 characters with uppercase, lowercase, number, and special character"` |
| Email already exists | 409 | `"Email is already registered"` |
| Invalid referral code | 404 | `"Referral code not found"` |
| Server error | 500 | `"Something went wrong. Please try again."` |

**Acceptance Criteria:**
- Given valid registration data without referral, When submitted, Then a new User is created with a unique referralCode and the user is logged in.
- Given valid registration data with a valid referral code, When submitted, Then the referrer receives 10,000 points (expiring in 3 months) and the new user receives a 10% coupon (expiring in 3 months).
- Given an email that already exists, When submitted, Then a 409 error is returned and no duplicate user is created.
- Given an invalid referral code, When submitted, Then a 404 error is returned and no user is created.

**Definition of Done:**
- [ ] Registration form with all fields and validation
- [ ] Referral code input auto-uppercases
- [ ] Backend creates User + handles referral rewards in a single transaction
- [ ] httpOnly cookies set correctly
- [ ] Welcome email sent asynchronously
- [ ] Responsive on mobile/tablet/desktop
- [ ] Confirm dialog not needed (registration is creation, not modification)
- [ ] No `console.log` left

### Technical Tasks

**Frontend:**
- **Page:** `app/register/page.tsx`
- **Components:** `components/auth/RegisterForm.tsx`
- **Type (`types/auth.types.ts`):**
  ```ts
  export interface RegisterFormValues {
    firstName: string;
    lastName: string;
    email: string;
    password: string;
    confirmPassword: string;
    role: "CUSTOMER" | "ORGANIZER";
    referralCode?: string;
  }
  ```
- **State:** Form via `react-hook-form`. Submission via TanStack Query `useMutation` calling `registerUser()` in `lib/api/auth.api.ts`.
- **Validation (`lib/validators/auth.schema.ts`):**
  ```ts
  import { z } from "zod";

  export const registerSchema = z.object({
    firstName: z.string().min(2, "Minimum 2 characters").max(50),
    lastName: z.string().min(2, "Minimum 2 characters").max(50),
    email: z.string().email("Invalid email address"),
    password: z.string()
      .min(8, "Minimum 8 characters")
      .regex(/[A-Z]/, "Must contain an uppercase letter")
      .regex(/[a-z]/, "Must contain a lowercase letter")
      .regex(/[0-9]/, "Must contain a number")
      .regex(/[^A-Za-z0-9]/, "Must contain a special character"),
    confirmPassword: z.string(),
    role: z.enum(["CUSTOMER", "ORGANIZER"]),
    referralCode: z.string().regex(/^REF-[A-Z0-9]{6}$/).optional().or(z.literal("")),
  }).refine(d => d.password === d.confirmPassword, {
    message: "Passwords do not match", path: ["confirmPassword"],
  });
  ```
- **UI States:** Submit button spinner while pending. Inline field errors from Zod. Toast on success/error. Referral code input has `onChange` handler that transforms to uppercase.

**Backend:**
- **Endpoint:** `POST /api/v1/auth/register`
- **Router:** `src/routes/auth.routes.ts`
- **Middleware order:** `validate(registerSchema)` → `authController.register`
- **Service (`src/services/auth.service.ts`):**
  ```ts
  async register(dto: RegisterDTO): Promise<{ user: UserResponse; accessToken: string; refreshToken: string }>
  ```
  Responsibilities: check email uniqueness, hash password, generate referralCode, handle referral rewards in `prisma.$transaction()`, generate tokens, send welcome email.
- **Controller:** Calls service, sets cookies via `setAuthCookies()`, returns 201.
- **Repository:** `userRepository.create()`, `pointRepository.credit()`, `couponRepository.create()`

**Database:**
- Creates: User row
- Conditionally creates (if referral): PointLedger CREDIT row (referrer), Coupon row (new user)
- Transaction: All in `prisma.$transaction()` when referral code is present

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/auth/register.test.ts`
- **Mocking:** `jest.mock()` on `emailService.send` (welcome email); `nock` not needed (no external HTTP calls)
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createUser()` to seed referrer when testing referral flow
- **Required Assertions:**
  - Assert 201 Created with valid payload → User row exists in DB with hashed password and unique `REF-XXXXXX` referralCode
  - Assert httpOnly cookies (`accessToken`, `refreshToken`) are set in response headers
  - Assert 409 Conflict when email already exists → no duplicate User row created
  - Assert 404 Not Found when referral code is invalid → no User, PointLedger, or Coupon rows created
  - Assert 400 Bad Request when password fails complexity rules (missing uppercase/number/special char)
  - Assert referral reward: referrer gets PointLedger CREDIT row (10,000 points, 3-month expiry) and new user gets Coupon row (10%, single-use, 3-month expiry)
  - Assert `emailService.send` was called once with welcome email template

---

## US-02: User Login

**Story:** As a registered user, I want to log in with my email and password, so that I can access my account.
**Traceability:** Mapped to Spec Stories #4, #5 | Issue Ticket #02

**Context:** Standard login. After success, Customer → `/`, Organizer → `/dashboard`.

**Roles:** None (public page).

**Assumptions:**
- User has a pre-existing registered account.
- Database service is operational and reachable.

**Out of Scope:** Social login (Google, GitHub), "Remember me" toggle, 2FA, biometric authentication.

**Happy Path:**
1. User navigates to `/login`
2. Fills email + password
3. Client Zod validation passes
4. `POST /api/v1/auth/login`
5. Backend verifies credentials against database
6. Generates access and refresh tokens, sets httpOnly cookies
7. Returns user data, frontend redirects by role

**Unhappy Paths:**
- Missing or invalid email/password format → Client-side Zod validation stops submission and displays inline field errors.
- Unregistered email or incorrect password → Backend returns 401 Unauthorized with a generic message `"Invalid email or password"` without disclosing which field failed.

**Business Constraints:**
- Cryptographic security: Access tokens expire in 15 minutes; refresh tokens expire in 7 days and are rotated on renewal.
- Enumeration defense: Failed login attempts return identical error strings for invalid email versus invalid password.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Missing field / invalid email format | 400 | Field-level Zod validation error |
| Wrong credentials (email or password) | 401 | `"Invalid email or password"` |
| Server / database error | 500 | `"Something went wrong. Please try again."` |

**Acceptance Criteria:**
- Given valid user credentials, When submitted to `/api/v1/auth/login`, Then the system sets secure httpOnly cookies and returns 200 OK with the user profile.
- Given an unregistered email or wrong password, When submitted, Then the system returns 401 Unauthorized with `"Invalid email or password"`.
- Given an authenticated Customer, When login completes, Then they are redirected to `/`.
- Given an authenticated Organizer, When login completes, Then they are redirected to `/dashboard`.

**Definition of Done:**
- [ ] Login form implemented with Zod client validation
- [ ] Backend login endpoint verifies bcrypt hashes securely
- [ ] httpOnly cookies set for both accessToken and refreshToken
- [ ] Silent refresh Axios interceptor configured on client
- [ ] Redirects correctly based on user role

### Technical Tasks

**Frontend:**
- **Page:** `app/login/page.tsx`
- **Components:** `components/auth/LoginForm.tsx`
- **Link to:** `/register` ("Don't have an account?") and `/forgot-password` ("Forgot password?")

**Backend:**
- **Endpoint:** `POST /api/v1/auth/login`
- **Router:** `src/routes/auth.routes.ts`
- **Middleware:** `validate(loginSchema)` → `authController.login`
- **Service:** Verify email exists → compare password hash → generate tokens
- **Security:** Same error message for wrong email OR wrong password (prevent enumeration)

**Database:**
- **N/A (Read-only query):** Login performs a single indexed read lookup on the `User` table by `email` and does not mutate schema or run transactional writes.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/auth/login.test.ts`
- **Mocking:** None required (no external services called during login)
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createUser({ password: 'hashed' })` to seed a test user
- **Required Assertions:**
  - Assert 200 OK with valid credentials → response body contains user profile, httpOnly cookies set
  - Assert 401 Unauthorized with wrong password → generic `"Invalid email or password"` message (no field disclosure)
  - Assert 401 Unauthorized with unregistered email → same generic message (enumeration defense)
  - Assert 400 Bad Request with missing/invalid email format → Zod validation errors

---

## US-03: Forgot Password

**Story:** As a user who forgot my password, I want to request a reset link via email, so that I can regain access.
**Traceability:** Mapped to Spec Story #6 | Issue Ticket #03

**Context:** Public recovery request flow. Sends a time-limited password reset link to the provided email address if registered.

**Roles:** None (public page).

**Assumptions:**
- SMTP / Nodemailer email service is configured and operational.
- User has access to their email inbox.

**Out of Scope:** SMS-based recovery, security verification questions, admin password resets.

**Happy Path:**
1. User navigates to `/forgot-password`
2. Enters email → `POST /api/v1/auth/forgot-password`
3. Backend checks email; if found, generates random token, hashes it, stores in PasswordResetToken (expires 1 hour)
4. Sends email with link: `{FRONTEND_URL}/reset-password?token={rawToken}`
5. Frontend shows: "If an account with that email exists, a reset link has been sent"

**Unhappy Paths:**
- **N/A (Enumeration Defense):** To adhere to OWASP security guidelines, this endpoint never exposes whether an email is registered or not. Unregistered emails receive the exact same 200 OK success response and UI message as registered emails, simply omitting the email sending step internally.

**Business Constraints:**
- Reset tokens must expire exactly 1 hour from creation.
- Raw reset tokens are never stored in plain text; only SHA-256 token hashes are persisted in the database.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Invalid email format | 400 | `"Please enter a valid email address"` |
| Email not registered | 200 | `"If an account with that email exists, a reset link has been sent"` (silent drop) |
| SMTP transmission error | 500 | `"Failed to send recovery email. Please try again later."` |

**Acceptance Criteria:**
- Given a registered email address, When submitted to `/api/v1/auth/forgot-password`, Then a SHA-256 token hash is stored in `PasswordResetToken` with a 1-hour expiry and an email is sent.
- Given an unregistered email address, When submitted, Then the endpoint returns 200 OK without sending an email or creating a database token.
- Given any submission, When response returns, Then the UI displays the confirmation message without confirming account existence.

**Definition of Done:**
- [ ] Forgot password form implemented with Zod validation
- [ ] SHA-256 token hashing utility implemented in backend
- [ ] Nodemailer password reset email template created and verified
- [ ] OWASP enumeration defense verified via tests

### Technical Tasks

**Frontend:**
- **Page:** `app/forgot-password/page.tsx`
- **Components:** `components/auth/ForgotPasswordForm.tsx`

**Backend:**
- **Endpoint:** `POST /api/v1/auth/forgot-password`
- **Service:** `generateRandomToken()` → `hashToken()` → store hash in DB → `sendEmail(passwordResetTemplate)`

**Database:**
- **Repository:** `passwordResetRepository.create({ tokenHash, userId, expiresAt: now + 1h })` in `PasswordResetToken` table.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/auth/forgot-password.test.ts`
- **Mocking:** `jest.mock()` on `emailService.send` (password reset email)
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createUser()` to seed a registered user
- **Required Assertions:**
  - Assert 200 OK with registered email → `PasswordResetToken` row created with SHA-256 hash and 1-hour expiry; `emailService.send` called once
  - Assert 200 OK with unregistered email → no `PasswordResetToken` row created; `emailService.send` NOT called (OWASP enumeration defense)
  - Assert 400 Bad Request with invalid email format → Zod validation error

---

## US-04: Reset Password

**Story:** As a user with a valid reset link, I want to set a new password, so that I can log in again.
**Traceability:** Mapped to Spec Story #7 | Issue Ticket #03

**Context:** Token-gated password reset page accessed via email link.

**Roles:** None (public page, token-gated).

**Assumptions:**
- User clicked the recovery link within 1 hour of token creation.
- URL contains the raw unhashed recovery token in query parameters.

**Out of Scope:** Re-sending recovery link from this page (redirects to `/forgot-password` on expiry).

**Happy Path:**
1. User clicks reset link → `/reset-password?token=xxx`
2. Enters new password + confirm
3. `POST /api/v1/auth/reset-password` with token + new password
4. Backend hashes token, finds matching PasswordResetToken, verifies expiry
5. Updates user password, deletes the token, deletes all user's refresh tokens
6. Redirects to `/login` with success toast

**Unhappy Paths:**
- Expired or tampered token → Backend returns 400 Bad Request; UI displays an error callout with a link back to `/forgot-password`.
- Passwords do not match or fail complexity rules → Client-side Zod validation prevents submission and shows inline field errors.

**Business Constraints:**
- Setting a new password must immediately invalidate and delete all active refresh tokens for that user across all devices.
- Once consumed, the password reset token must be permanently deleted from the database.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Token invalid | 400 | `"Reset link is invalid or has expired"` |
| Token expired | 400 | `"Reset link is invalid or has expired"` |
| Weak password | 400 | Field-level Zod validation errors |

**Acceptance Criteria:**
- Given a valid, unexpired reset token, When submitted with a valid new password, Then the user's password hash is updated and all active refresh tokens are deleted.
- Given an expired or invalid token, When submitted, Then the API returns 400 Bad Request and the password remains unchanged.

**Definition of Done:**
- [ ] Reset password page reads token from URL query string
- [ ] Zod validation enforces password complexity and confirmation matching
- [ ] Backend verifies token hash against database and checks timestamp
- [ ] Atomic database transaction updates password and purges tokens

### Technical Tasks

**Frontend:**
- **Page:** `app/reset-password/page.tsx`
- **Components:** `components/auth/ResetPasswordForm.tsx`
- Reads `token` from URL search params

**Backend:**
- **Endpoint:** `POST /api/v1/auth/reset-password`
- **Service:** Hash incoming token → find by hash → check expiry → update password → delete token

**Database:**
- **Transaction:** Execute `User.update({ passwordHash })`, `PasswordResetToken.delete({ id })`, and `RefreshToken.deleteMany({ userId })` in an atomic `prisma.$transaction()`.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/auth/reset-password.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createUser()` + `createPasswordResetToken({ userId, expiresAt })` to seed valid/expired tokens
- **Required Assertions:**
  - Assert 200 OK with valid unexpired token → User `passwordHash` updated in DB; `PasswordResetToken` row deleted; all `RefreshToken` rows for user deleted
  - Assert 400 Bad Request with expired token → password unchanged, token still in DB
  - Assert 400 Bad Request with tampered/non-existent token → no DB mutations
  - Assert 400 Bad Request with weak new password → Zod validation errors

---

## US-05: Edit Profile

**Story:** As a logged-in user, I want to edit my profile (name, profile picture), so that my information stays up to date.
**Traceability:** Mapped to Spec Story #8 | Issue Ticket #03

**Context:** Authenticated user profile page. Allows updating identity attributes and uploading cloud-hosted avatars.

**Roles:** Both CUSTOMER and ORGANIZER.

**Assumptions:**
- User is authenticated with a valid access token.
- Cloudinary storage service is configured and accessible.

**Out of Scope:** Changing email address, changing user role (`CUSTOMER` ↔ `ORGANIZER`).

**Happy Path:**
1. User navigates to `/profile`
2. Sees current info: name, email (read-only), role (read-only), referralCode (read-only + copy button), profile picture, point balance (Customer), active coupons (Customer)
3. Edits firstName/lastName or uploads new profile picture
4. Clicks save → **Confirm dialog** appears → confirms
5. `PATCH /api/v1/users/me` as multipart/form-data
6. Backend updates user, uploads new picture to Cloudinary (deletes old if exists)
7. Toast: "Profile updated successfully"

**Unhappy Paths:**
- Uploading non-image format (e.g. PDF/EXE) or file > 2MB → Multer middleware rejects file before controller execution, returning 400 Bad Request.
- Canceling confirmation dialog → Form changes remain in UI state without firing API request.

**Business Constraints:**
- Profile picture: jpg/png/webp only, max 2MB filesize limit.
- Old profile picture asset must be deleted from Cloudinary storage when replaced to prevent orphan cloud assets.
- ReferralCode and email are immutable display-only attributes on this form.

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Unsupported file type | 400 | `"Only .jpg, .png, and .webp formats are allowed"` |
| File size > 2MB | 400 | `"File size must not exceed 2MB"` |
| Missing authorization cookie | 401 | `"Authentication required"` |

**Acceptance Criteria:**
- Given an authenticated user, When updating first/last name with confirmation, Then the profile is updated and 200 OK returned.
- Given a valid image upload (<2MB), When saved, Then the image is uploaded to Cloudinary, old avatar deleted, and new URL saved.
- Given an invalid file format or size > 2MB, When submitted, Then a 400 error is returned and profile remains unchanged.

**Definition of Done:**
- [ ] Profile form displays read-only attributes (email, role, referralCode) correctly
- [ ] Multer file upload middleware limits filesize to 2MB and validates mime type
- [ ] Confirmation modal prompts user before executing save
- [ ] Cloudinary deletion helper purges old avatar on replacement

### Technical Tasks

**Frontend:**
- **Page:** `app/profile/page.tsx`
- **Components:** `components/auth/ProfileForm.tsx`
- **Sections:** Profile form, Referral code display + copy button, Point balance (if Customer), Active coupons list (if Customer)
- **Confirm dialog** on save (required by rubric for data modifications)

**Backend:**
- **Endpoint:** `PATCH /api/v1/users/me`
- **Router:** `src/routes/user.routes.ts`
- **Middleware:** `authenticate` → `upload.single("profilePicture")` → `validate(updateProfileSchema)` → `userController.update`

**Database:**
- **Repository:** `userRepository.update({ where: { id: userId }, data: { firstName, lastName, profilePicture } })` on `User` table.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/users/profile.test.ts`
- **Mocking:** `jest.mock()` on `cloudinaryService.upload` (returns mock URL) and `cloudinaryService.delete` (returns void)
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createUser()` to seed authenticated user
- **Required Assertions:**
  - Assert 200 OK with valid name update → User row updated in DB
  - Assert 200 OK with valid image upload → `cloudinaryService.upload` called with buffer; User `profilePicture` URL updated
  - Assert 400 Bad Request with unsupported file type (e.g., `.pdf`) → profile unchanged
  - Assert 400 Bad Request with file > 2MB → profile unchanged
  - Assert 401 Unauthorized without auth cookie → rejected

**Frontend Tests:**
- **File:** `apps/web/__tests__/components/ProfileForm.test.tsx`
- **Library:** React Testing Library
- **Required Assertions:**
  - Assert referralCode and email fields are rendered as read-only
  - Assert confirmation dialog appears on save button click

---

## US-06: Change Password

**Story:** As a logged-in user, I want to change my password from within my profile.
**Traceability:** Mapped to Spec Story #9 | Issue Ticket #03

**Context:** Authenticated security update workflow accessible from account settings.

**Roles:** Both CUSTOMER and ORGANIZER.

**Assumptions:**
- User is authenticated and knows their existing account password.

**Out of Scope:** 2FA setup, hardware token security keys.

**Happy Path:**
1. On `/profile` page, user clicks "Change Password" section
2. Enters current password + new password + confirm
3. **Confirm dialog** → confirms
4. `PATCH /api/v1/users/me/password`
5. Backend verifies current password, updates to new hash
6. Toast: "Password changed successfully"

**Unhappy Paths:**
- Current password incorrect → Backend returns 400 Bad Request with `"Current password is incorrect"`.
- New password matches current password → Validation blocks update.
- New password fails complexity criteria → Client Zod validation shows inline field errors.

**Business Constraints:**
- Must require confirmation dialog before submitting password modification.
- New password must meet OWASP complexity rules (8+ chars, uppercase, lowercase, number, special char).

**Error Handling Matrix:**

| Trigger | Code | Message |
|---|---|---|
| Wrong current password | 400 | `"Current password is incorrect"` |
| Weak new password | 400 | Field-level Zod validation errors |
| Passwords do not match | 400 | `"Confirm password must match new password"` |

**Acceptance Criteria:**
- Given an authenticated user with correct current password, When submitting a valid new password with confirmation, Then the password hash is updated and 200 OK returned.
- Given an incorrect current password, When submitted, Then a 400 error is returned and the password remains unchanged.

**Definition of Done:**
- [ ] Password change form validated with Zod complexity rules
- [ ] Confirmation modal prompts before API submission
- [ ] Backend compares current password hash before overwriting
- [ ] Returns clear error messages on mismatch

### Technical Tasks

**Frontend:**
- **Page:** `app/profile/page.tsx` (Password tab/section)
- **Components:** `components/auth/ChangePasswordForm.tsx`
- **Confirm dialog** on save

**Backend:**
- **Endpoint:** `PATCH /api/v1/users/me/password`
- **Router:** `src/routes/user.routes.ts`
- **Middleware:** `authenticate` → `validate(changePasswordSchema)` → `userController.changePassword`

**Database:**
- **Repository:** `userRepository.updatePassword({ userId, newPasswordHash })` on `User` table.

**Automated Tests:**
- **Seam:** HTTP route boundary via `supertest`
- **File:** `apps/api/__tests__/integration/users/change-password.test.ts`
- **Mocking:** None required
- **DB Lifecycle:** `truncateAll()` in `beforeEach`; factory `createUser({ password: 'known' })` to seed user with known password
- **Required Assertions:**
  - Assert 200 OK with correct current password and valid new password → `passwordHash` changed in DB
  - Assert 400 Bad Request with wrong current password → `"Current password is incorrect"`; hash unchanged
  - Assert 400 Bad Request with weak new password → Zod validation errors
  - Assert 400 Bad Request when new password matches current password → blocked
