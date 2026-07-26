# 11 — Post-Event Reviews & Verified Rating System

**What to build:** Customer review submission interface on past event pages allowing customers with `DONE` transactions marked as `isAttended = true` to submit a 1-5 star rating and comment. The system enforces strict unique constraints (one review per customer per event) and dynamically updates event and organizer average ratings displayed across public profile pages.

**Blocked by:** 10 — Organizer Order Verification & Attendance Marking

**Status:** ready-for-agent

- [ ] Implement `POST /api/v1/reviews` validating rating (1-5 integer) and optional comment (max 500 chars) with Zod
- [ ] Enforce review eligibility in service layer ensuring Customer has a transaction for the target event where `status === DONE` and `isAttended === true`
- [ ] Handle database unique constraint preventing duplicate reviews (`[userId, eventId]`), returning clean 409 Conflict message
- [ ] Implement `GET /api/v1/reviews?eventId=xxx` paginated endpoint returning review cards with author first/last name, avatar, rating, and timestamp
- [ ] Build interactive `ReviewForm` component on event detail page (`app/events/[id]`) displayed only for eligible attendees, with interactive star rating selector and submit confirmation dialog
- [ ] Verify dynamic average rating calculation across event catalog cards, event detail page, and public organizer profile page
- [ ] TDD: Write integration test suite (`apps/api/__tests__/integration/reviews/create-review.test.ts`) via supertest verifying attendance eligibility (`status === DONE` and `isAttended === true`), 1-5 integer rating validation, and 409 Conflict on duplicate review attempts
- [ ] TDD: Write React Testing Library unit test (`apps/web/__tests__/components/ReviewForm.test.tsx`) verifying interactive star rating selector and form visibility rules
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

