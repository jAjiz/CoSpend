# CoSpend

A full-stack expense-sharing application built with Angular 21 and Node.js.

Groups of users log shared expenses. CoSpend tracks balances and computes the minimum number of payments required to settle all debts — using a greedy debt-minimisation algorithm.

---

## Stack

| | Technology | Notes |
|-|-----------|-------|
| **Frontend** | Angular 21 | Standalone components, Signals, lazy-loaded routes |
| **Styling** | TailwindCSS v4 | CSS-first config, class-based dark mode |
| **Backend** | Node.js + Express 5 | TypeScript (ESM), no ORM |
| **Database** | PostgreSQL 16 | `NUMERIC(12,2)` monetary amounts |
| **Auth** | JWT + bcrypt | Stateless, long-lived token |
| **Validation** | Zod | Schema-first, backend only |
| **Testing** | Jest + Playwright | Unit + E2E |
| **Hosting** | Vercel · Render · Neon | Frontend · Backend · DB |

---

## Features

- Register and log in with email/password
- Create groups and invite members by email
- Log expenses with category, payer, and equal split among any subset of members
- Automatic balance calculation per group and across all groups
- Optimal settlement suggestions — minimum transactions to clear all debts
- Record payments and track settlement history
- Dark mode with system-preference detection and manual override

---

## Architecture

```
CoSpend/
├── frontend/          Angular 21 SPA
│   └── src/app/
│       ├── core/      Auth, guards, interceptors, models, services
│       ├── features/  Lazy-loaded route components (one per feature)
│       └── shared/    Reusable components and pipes
├── backend/           Express REST API
│   └── src/
│       ├── controllers/
│       ├── services/  Business logic (settlement algorithm)
│       ├── routes/
│       ├── middleware/ Auth, validation, error handling
│       ├── validators/ Zod schemas
│       └── db/        pg Pool, migrations
└── docker-compose.yml
```

---

## Getting Started

**Prerequisites:** Node.js ≥ 20, pnpm ≥ 9, Docker

```bash
# Database
docker-compose up -d

# Backend
cd backend
cp .env.example .env    # set DATABASE_URL and JWT_SECRET
pnpm install
pnpm migrate
pnpm dev                # http://localhost:3000

# Frontend
cd frontend
pnpm install
pnpm exec ng serve      # http://localhost:4200
```

---

## Running Tests

```bash
# Backend unit tests
cd backend && pnpm exec jest

# Frontend build verification
cd frontend && pnpm exec ng build --configuration production
```

---

## Status

See [ROADMAP.md](./ROADMAP.md) for the full delivery plan and current progress.
