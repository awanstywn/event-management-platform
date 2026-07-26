# 12 — Organizer Visual Analytics Dashboard (Recharts & SQL Aggregation)

**What to build:** Organizer analytics dashboard featuring interactive date-range filtering, summary statistics cards (total revenue from completed orders, total attendees, total events, average rating), and Recharts visual graphics depicting monthly revenue trends, ticket sales velocity, and event distribution by category.

**Blocked by:** 10 — Organizer Order Verification & Attendance Marking, 11 — Post-Event Reviews & Verified Rating System

**Status:** ready-for-agent

- [ ] Implement `GET /api/v1/dashboard/statistics` accepting optional `startDate` and `endDate` query parameters
- [ ] Write SQL aggregation queries (`prisma.$queryRaw`) computing total revenue (`SUM(finalPrice)` where `status = DONE`), total attendees (`COUNT(id)` where `status = DONE`), and total published events for the authenticated Organizer
- [ ] Write SQL aggregation queries generating monthly revenue time series, monthly ticket sales time series, and event count distribution grouped by category name
- [ ] Build Organizer dashboard page (`app/dashboard/page.tsx`) with date range picker component re-fetching query metrics upon date change
- [ ] Build summary statistics UI cards (`StatCard`) formatting IDR currency (`formatCurrency`) and numerical counts with loading skeleton placeholders
- [ ] Integrate Recharts library components building responsive visual charts: `RevenueChart` (`BarChart`), `TicketSalesChart` (`LineChart`), and `CategoryPieChart` (`PieChart` / donut chart) with empty state fallbacks when data is absent
- [ ] TDD: Write integration test suite (`apps/api/__tests__/integration/analytics/dashboard.test.ts`) via supertest verifying raw SQL analytical aggregation accuracy across date range filters and zeroed fallbacks when 0 events exist
- [ ] TDD: Write React Testing Library unit test (`apps/web/__tests__/components/DashboardCharts.test.tsx`) verifying summary card rendering and empty state fallback when data is absent
- [ ] Verify CI/CD pipeline passes all 5 verification gates on feature commit submission

