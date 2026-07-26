# 03 — Profile Management & Password Security

**What to build:** End-to-end profile management where users can view their read-only referral code, edit firstName/lastName, upload avatar images to cloud storage (with automatic deletion of old avatars), change their password after a confirmation dialog, and use the OWASP-compliant forgot/reset password flow without exposing email registration status to enumeration attacks.

**Blocked by:** 02 — Authentication & Session Management

**Status:** ready-for-agent

- [ ] Implement `GET /api/v1/users/me` returning full user profile including read-only referralCode, calculated point balance, and active coupon list
- [ ] Implement `PATCH /api/v1/users/me` accepting multipart/form-data to update firstName/lastName and upload profilePicture to Cloudinary (deleting previous avatar if present)
- [ ] Implement `PATCH /api/v1/users/me/password` verifying current password and hashing new password
- [ ] Implement OWASP-compliant `POST /api/v1/auth/forgot-password` always returning 200 success (to prevent email enumeration) while generating SHA-256 hashed token stored in `PasswordResetToken` table and sending reset link email
- [ ] Implement `POST /api/v1/auth/reset-password` validating token hash and expiry, updating password, and revoking all existing user sessions
- [ ] Build UI profile page (`app/profile`) with form editing, image file preview/upload, referral code copy button, and interactive confirmation dialog before saving modifications
- [ ] Build UI forgot password (`app/forgot-password`) and reset password (`app/reset-password`) pages with validation
- [ ] TDD: Write integration test suites (`apps/api/__tests__/integration/users/profile.test.ts`, `change-password.test.ts`, `apps/api/__tests__/integration/auth/forgot-password.test.ts`, `reset-password.test.ts`) via supertest with `jest.mock()` on Cloudinary and email service wrappers
- [ ] TDD: Write React Testing Library unit test (`apps/web/__tests__/components/ProfileForm.test.tsx`) verifying read-only referralCode rendering and confirmation dialog on save
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

