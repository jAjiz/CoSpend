# CoSpend

A full-stack expense-sharing web app built with Angular 21, Express, and PostgreSQL.

---

## What It Does

CoSpend lets users create groups, log shared expenses, and automatically calculate the minimum number of payments needed to settle all debts within a group.

> **Status:** Under active development — see [ROADMAP.md](./ROADMAP.md) for current progress.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Angular 21 (standalone components, Signals) |
| Styling | TailwindCSS v3 |
| Backend | Node.js + Express 4 |
| Database | PostgreSQL 16 |
| Auth | JWT + bcrypt |
| Validation | Zod |
| Local DB | Docker Compose |
| Hosting | Vercel (frontend) · Render (backend) · Neon (database) |

---

## Getting Started

> Prerequisites: Node.js ≥ 20, Docker

```bash
# 1. Start local PostgreSQL
docker-compose up -d

# 2. Backend
cd backend
cp .env.example .env   # fill in JWT_SECRET
npm install
npm run migrate
npm run dev            # http://localhost:3000

# 3. Frontend
cd frontend
npm install
npx ng serve           # http://localhost:4200
```

---

## Project Structure

```
CoSpend/
├── frontend/    Angular SPA
├── backend/     Express REST API
└── docker-compose.yml
```

---

## Author

Built by Juan. See [ROADMAP.md](./ROADMAP.md) for the full development plan.
