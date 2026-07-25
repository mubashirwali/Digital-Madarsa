# Digital Madrasa Platform — Codebase Scaffold

This is the initial code scaffold for the platform described in the Product
Blueprint (Vision, Roles, User Flow). It's a real, buildable Next.js project
with the folder structure, database schema, auth, and RBAC wired up — most
business logic inside routes is stubbed with `TODO`s for you (or me, in a
follow-up) to fill in.

## Stack

- **Next.js 16** (App Router, TypeScript, Turbopack)
- **Tailwind CSS v4**
- **Prisma 6** + **SQLite for local dev / PostgreSQL for production** —
  pinned to Prisma v6 deliberately; Prisma 7 (released Nov 2025) moved the
  database URL out of `schema.prisma` into a separate `prisma.config.ts` and
  now requires instantiating `PrismaClient` with an explicit driver adapter
  package. That's more setup than this scaffold needs, so `package.json`
  pins `"prisma"` and `"@prisma/client"` to `^6.19.0`. The schema defaults to
  `provider = "sqlite"` so you can run everything locally with zero signup —
  see "Switching to Postgres for production" below for the one-line change
  needed to point at Neon.
- **Auth.js (NextAuth v5)** with a Credentials provider (email/password),
  ready to add Google OAuth
- Route-level **RBAC** via `src/lib/rbac.ts` and `src/middleware.ts`

## Architecture note: one app, one login, role-based views

This is a single Next.js application with a single login page and a single
database — not four separate apps. When someone signs in, the same
`/login` form authenticates everyone; `middleware.ts` checks their role from
the session and routes them into `/dashboard/student`, `/dashboard/teacher`,
`/dashboard/parent`, or `/dashboard/admin`. A teacher's live class and a
student's enrollment in it are the same underlying database rows, just
rendered differently depending on who's looking — that's what keeps
attendance, payments, and messaging consistent across roles instead of
being disconnected silos.

## What's implemented

- Full **Prisma schema** (`prisma/schema.prisma`) covering users, roles,
  institutes, courses, enrollments, live classes, attendance, homework,
  submissions, exams, certificates, Hifz tracking, payments, payouts,
  messaging, notifications, support tickets, and audit logs.
- **Auth**: credentials login (redirects to the correct role-specific
  dashboard after sign-in), JWT session, role + institute on the session
  token. Split into `src/lib/auth.config.ts` (edge-safe, used by
  middleware) and `src/lib/auth.ts` (full config with the Prisma adapter,
  used everywhere else) — see the note in `auth.config.ts` for why.
- **RBAC**: a single permission table mapping each of the 8 roles to what
  they can do (`src/lib/rbac.ts`), matching the Product Blueprint's
  permission matrix exactly, plus per-role data-loading guards in
  `src/lib/current-user.ts`.
- **Middleware** that protects `/dashboard/*` routes by role
  (`src/middleware.ts`).
- A real, designed **landing page** and full **marketing site** (About,
  Pricing, Courses, FAQ, Contact, Blog) — not placeholders.
- **All four dashboards fully data-wired**, each querying Prisma directly
  against the signed-in user's own scoped data:
  - **Student** — overview, courses, homework, attendance, certificates,
    payments, messages.
  - **Teacher** — overview, schedule, live classes (host queue), students,
    materials, payments/payouts.
  - **Parent** — overview (all linked children), a per-child detail page
    (`/dashboard/parent/children/[studentId]`, authorization-checked so a
    parent can only see their own children), fees, messages.
  - **Admin** (institute admin / super admin / finance / content /
    support) — overview, students, teachers (with verification status),
    courses, finance, reports, tickets, and a super-admin-only institutes
    page for approving new madrasas onto the platform.
- A **seed script** (`prisma/seed.ts`) with realistic demo data spanning
  every role — see the table under "Getting started" below.
- **API route stubs** for courses, enrollments, live classes, attendance,
  homework, exams, Hifz records, certificates, and payments (Stripe,
  JazzCash, EasyPaisa) under `src/app/api/*` — each checks auth and returns
  a `501` with a `TODO` marking where the real logic goes. Note the
  dashboards above read data directly via server-component Prisma queries
  rather than through these routes; the API routes matter once client-side
  mutations (submitting homework, approving a teacher, sending a message)
  are wired up.

## What's intentionally stubbed (not yet built)

- Actual query/mutation logic inside the API routes above (marked `TODO`).
- Interactive actions inside the dashboards: approving a pending teacher or
  institute, grading a submission, starting a live class, uploading course
  material — all render real data but their buttons aren't wired to
  mutations yet.
- Live class integration (LiveKit/Jitsi room creation).
- AI features (Tajweed checker, homework assistant, exam generator).
- Payment provider integrations (Stripe Checkout, JazzCash/EasyPaisa signed
  requests) — the request/response shape is scaffolded, the provider SDK
  calls are not.
- Email/push notification sending (Resend, Firebase Cloud Messaging) — OTP
  verification currently accepts any 6-digit code as a placeholder.
- Message composers (all Messages pages are currently read-only).

## Getting started

```bash
npm install
copy .env.example .env       # Windows — use `cp` instead on macOS/Linux
```
Default `DATABASE_URL` points at a local SQLite file — no signup needed.
You still need to set `AUTH_SECRET` in `.env` for login to work:
```bash
npx auth secret
```

```bash
npx prisma generate
npx prisma migrate dev --name init
npm run db:seed
npm run dev
```

`npm run db:seed` populates the database with a demo institute (plus a
second, pending institute), two students, two teachers (one pending
verification), a parent linked to both students, an institute admin, a
super admin, two courses (Hifz and Tajweed) with live classes, attendance,
homework, Hifz records, an exam attempt, a certificate, payments, payouts,
a support ticket, and cross-role messages — so every dashboard has real
data to show immediately. Demo logins (password for all: `Demo1234!`):

| Role | Email | Notes |
|---|---|---|
| Student | `student@demo.com` | Amina — enrolled in both courses |
| Student | `student2@demo.com` | Yusuf — enrolled in Tajweed only |
| Teacher | `teacher@demo.com` | Verified, teaches both courses |
| Teacher | `teacher2@demo.com` | Pending verification, no courses yet |
| Parent | `parent@demo.com` | Linked to both students |
| Institute admin | `admin@demo.com` | Scoped to Al-Falah Academy |
| Super admin | `superadmin@demo.com` | Platform-wide, sees both institutes |

Open http://localhost:3000 and log in as any of the above — each role
lands on its own dashboard automatically.

### Switching to Postgres for production

1. In `prisma/schema.prisma`, change `provider = "sqlite"` to `provider = "postgresql"` in the `datasource` block.
2. In `.env`, comment out the `file:./dev.db` line and uncomment/set the Postgres connection string (a free one from [neon.tech](https://neon.tech) works well).
3. Run `npx prisma migrate dev --name init` again against the new database.

> **Note on this scaffold's build environment:** this project was built in a
> sandboxed container without access to `binaries.prisma.sh` or
> `fonts.googleapis.com`, so `prisma generate` and the Google Fonts fetch
> couldn't run here — meaning the Prisma queries in the student dashboard
> pages and seed script were checked by hand against `schema.prisma` field
> and relation names, not run against a live database. Both `npx tsc --noEmit`
> and `npx eslint` are clean. If you hit a Prisma error when you actually run
> it, it's almost certainly a typo I couldn't catch this way — send me the
> exact error and I'll fix it fast.

## Troubleshooting

**`Module not found: Can't resolve '.prisma/client/default'`** — the Prisma
client hasn't been generated yet. Run `npx prisma generate` before
`npm run dev`.

**`Error: The datasource property 'url' is no longer supported in schema
files'` (P1012)** — this means npm installed Prisma 7 instead of the pinned
v6. Run `npm install` again to pick up the version pin in `package.json`,
then re-run `npx prisma generate`.

## Folder structure

```
src/
  app/
    (marketing)/        landing, about, pricing, courses, faq, contact, blog
    (auth)/              login, signup, forgot-password, verify
    dashboard/
      layout.tsx          role-aware sidebar nav
      student/            overview, courses, homework, attendance, certificates, payments, messages
      teacher/             overview, schedule, students, live-classes, materials, payments
      parent/               overview, children/[studentId], fees, messages
      admin/                 overview, students, teachers, courses, finance, reports, tickets, institutes
    api/
      auth/[...nextauth]/  auth/register/
      courses/ enrollments/ live-classes/ attendance/
      homework/ exams/ hifz/ certificates/
      payments/{stripe,jazzcash,easypaisa}/
    unauthorized/         role-mismatch landing page
    providers.tsx          SessionProvider wrapper
  components/
    dashboard/shared.tsx   PageHeader, EmptyState
    marketing/              SiteHeader, SiteFooter, GeometricDivider
  lib/
    db.ts                  Prisma client singleton
    auth.ts                 full Auth.js config (Node runtime — Prisma adapter, Credentials)
    auth.config.ts           edge-safe Auth.js config, shared with middleware
    current-user.ts          per-role data-loading guards (requireStudentProfile, etc.)
    rbac.ts                  permission table + can()/assertCan()
  types/next-auth.d.ts       session/JWT type augmentation (role, instituteId)
  middleware.ts             role-based route protection (edge runtime)
prisma/
  schema.prisma
  seed.ts                  demo data across every role
```

## Next steps

This scaffold covers the foundation. Natural next slices to build out:

1. Wire real Prisma queries into the API route stubs (start with
   courses/enrollments — everything else depends on them).
2. Build out the student/teacher dashboard pages with real data.
3. Implement Stripe Checkout end-to-end (simplest payment provider to start
   with before JazzCash/EasyPaisa).
4. Add the LiveKit/Jitsi room creation + join flow.
