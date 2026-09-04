# Gym Operating System — v0 (Single Gym)

Monorepo with two independent apps. See [gym-operating-system-scaffold.md](gym-operating-system-scaffold.md) Section 0 for the current build scope (single gym, no multi-tenancy) and the rest of the doc for the future multi-tenant target architecture.

```text
gop/
├── frontend/   Next.js (App Router, TypeScript, Tailwind)
└── backend/    NestJS + Prisma + PostgreSQL API
```

## Roles (v0)

- **Super Admin** — full access (the gym owner)
- **Admin** — day-to-day operations (members, attendance, payments, memberships)
- **Member** — own profile/membership/attendance/payments

## Conventions

- All backend DB access goes through a repository/service layer per module — never inline Prisma calls in controllers. This is what lets multi-tenant scoping get added later in one place.
- Primary keys are UUIDs.
- Gym configuration lives in a `gym_settings` table row, never hardcoded constants.
- No `tenant_id`/`branch_id` columns yet — do not add them preemptively.
- Backend modules mirror the module boundaries in the scaffold doc's Section 6 (`members/`, `attendance/`, `payments/`, etc.), even though there's no tenant boundary today.

## Running locally

```bash
cd backend && npm run start:dev
cd frontend && npm run dev
```
