# CoSpend — Roadmap

## Approach

Incremental delivery, one phase per domain layer. Each phase ships a independently reviewable slice of the application. No phase introduces speculative infrastructure — only what the current feature set requires.

---

## Phase 0 — Foundation & Tooling

**Objective**
Establish the full-stack project structure with all tooling configured and a clean baseline build.

**Deliverables**
- Angular 21 project: standalone components, strict TypeScript, TailwindCSS v3 + PostCSS
- Express project: folder structure, nodemon, environment validation via Zod on startup
- Docker Compose: PostgreSQL 16 for local development
- `vercel.json` with SPA rewrite for HTML5 routing
- Angular `environment.ts` / `environment.prod.ts` with build-time file replacement wired in `angular.json`
- `.gitignore` covering `node_modules`, `dist`, `.angular`, `.env`

**Notes**
- Angular uses the `@angular/build:application` builder (esbuild-based, default in v17+)
- Backend stays plain JavaScript — TypeScript adds friction without meaningful gain for an Express API at this scale
- Tailwind v3 over v4: better Angular CLI integration, more ecosystem stability at time of writing

---

## Phase 1 — Data Model & Migrations

**Objective**
Define the complete PostgreSQL schema upfront. Every subsequent phase depends on this foundation being correct.

**Deliverables**
- Full schema: `users`, `groups`, `group_members`, `expenses`, `expense_splits`, `settlements`
- `001_initial.sql` migration with constraints, enums, and indexes
- `migrate.js` runner applying files in alphabetical order
- `npm run migrate` script

**Notes**
- UUIDs (`uuid-ossp`) as primary keys — avoids sequential ID leakage, plays well with distributed inserts
- `NUMERIC(12,2)` for all monetary amounts — `FLOAT` is unsuitable for financial data
- `expense_splits` stores the resolved monetary amount per participant, not a ratio — simplifies every balance query downstream
- `ON DELETE CASCADE` on group-owned resources; `ON DELETE RESTRICT` on `users` to prevent silent data loss

---

## Phase 2 — Authentication

**Objective**
Stateless JWT authentication, end to end.

**Deliverables**
- `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/auth/me`
- bcrypt password hashing (cost factor 12)
- Zod validation on all auth payloads
- Rate limiting on auth endpoints (20 req / 15 min)
- Angular `AuthService` with `token` and `currentUser` signals
- `JwtInterceptor` (functional interceptor, `withInterceptors`)
- `authGuard` and `guestGuard` as functional guards

**Notes**
- Long-lived access token (30d), no refresh token — appropriate simplification for an MVP without user-facing session management requirements
- Token stored in `localStorage` — acknowledged trade-off vs. `httpOnly` cookie; acceptable for a portfolio project without a dedicated BFF
- `AuthService` state is signal-based: `isAuthenticated = computed(() => !!this.token())`

---

## Phase 3 — Groups & Members

**Objective**
Core group management with per-group balance computation.

**Deliverables**
- Groups CRUD (`GET`, `POST`, `PATCH`, `DELETE`)
- `PATCH /api/groups/:id/favorite` — per-user favourite flag
- Member management: add by email, list, remove
- Balance included in `GET /api/groups` and `GET /api/groups/:id` — single SQL query, not computed in application code
- Dashboard: total balance, favourite groups, all groups list
- Group creation form
- `LayoutComponent` with header, dark-mode toggle, sign-out

**Notes**
- Balance SQL joins `expenses`, `expense_splits`, and `settlements` in one query per group — avoids N+1 and keeps balance logic in one place
- Favourite toggle uses an optimistic update pattern: update signal immediately, revert on API error
- Lazy-loaded routes from this phase forward — each feature directory maps to one route chunk

---

## Phase 4 — Expenses

**Objective**
Expense creation with equal splitting and per-group expense history.

**Deliverables**
- `POST /api/groups/:id/expenses` — atomic: inserts `expenses` + `expense_splits` in a transaction
- `GET /api/groups/:id/expenses` — filterable by `category`, `month`, `paidBy`
- `DELETE /api/groups/:id/expenses/:id` — creator only
- Expense form: amount, description, category (8 types), paid-by, participant selection, date
- Live per-person share preview via `computed()` signal
- Group detail: tabbed expenses / members view, category filter chips

**Notes**
- Rounding: equal split distributes remainder (≤ 1 cent) to the first participant — keeps `SUM(splits) == expense.amount` exact
- Only equal split in MVP — schema has a `split_type` enum ready for exact/percentage modes post-MVP
- Filters are passed as query params to the API, not applied client-side — keeps the frontend stateless and the dataset bounded

---

## Phase 5 — Settlement Engine

**Objective**
Debt-minimisation algorithm and settlement recording.

**Deliverables**
- `settlement.algorithm.js` — pure function, no I/O dependencies
- `GET /api/groups/:id/balances` — per-member net balance
- `GET /api/groups/:id/settlements/suggested` — optimal transaction list
- `POST /api/groups/:id/settlements` — record a payment
- `GET /api/groups/:id/settlements` — payment history
- Settlements view: balance summary, suggested payments, mark-as-paid, history tab
- Jest unit tests for the algorithm

**Notes**
- Algorithm: greedy O(n log n) — separate creditors and debtors, match largest against largest. Optimal for the general case with small group sizes (n < 50)
- Separating the pure function from `settlement.service.js` (which touches the DB) is what makes it trivially testable without mocking
- Suggested settlements are ephemeral (computed on request); only recorded settlements are persisted

---

## Phase 6 — UI Polish & Dark Mode

**Objective**
Production-quality UX: loading states, dark mode, notifications, responsive layout.

**Deliverables**
- Dark mode: class-based (`dark:` prefix), `ThemeService` persists preference in `localStorage`, respects `prefers-color-scheme` on first visit
- Skeleton loading states on all data-fetching views
- Toast notification system for user feedback (success / error)
- Expense filter UI wired to API (category, month, payer)
- "Add member" UI in the Members tab
- Page titles via Angular `Title` service
- Responsive layout validated at mobile viewport

**Notes**
- Toast system uses a signal-based queue in a singleton service — no global state management library needed at this scale
- Skeletons rather than spinners: avoids layout shift and feels more native on mobile

---

## Phase 7 — Testing & CI

**Objective**
Automated quality gates at unit and E2E level.

**Deliverables**
- Jest: settlement algorithm, auth controller, Zod validators
- Angular `TestBed`: `AuthService`, `ThemeService`
- Playwright E2E: register → create group → add expense → settle up → verify balance is zero
- GitHub Actions workflow: lint + build + test on every PR push

**Notes**
- E2E tests cover the golden path and the one critical invariant: balances net to zero after settling
- No snapshot tests — they couple tests to markup, not behaviour
- CI runs `ng build --configuration production` to catch build-time type errors that dev builds skip

---

## Phase 8 — Production Deployment

**Objective**
Live deployment on free-tier cloud with production configuration correct.

**Deliverables**
- Neon PostgreSQL provisioned, migrations applied, connection string with `sslmode=require`
- Backend deployed to Render/Railway, health check passing, env vars set
- Frontend deployed to Vercel, `vercel.json` SPA rewrite active, `apiUrl` pointing to backend
- CORS origin updated to Vercel domain
- README updated with live URLs

**Notes**
- Neon uses connection pooling differently from a traditional PostgreSQL server — `pg` pool settings need to account for serverless connection limits
- `NODE_ENV=production` triggers SSL in the pool config (`rejectUnauthorized: false` for Neon's self-signed cert is acceptable)

---

## Post-MVP

Not scheduled. Tracked for future consideration:

- **OAuth (Google)** — passport.js or Auth0
- **Exact / percentage splits** — schema is ready; needs form mode switching and updated split logic
- **Group image upload** — replace external URL with object storage (Cloudflare R2 / Supabase Storage)
- **Multi-currency** — `currency` column on expenses, open exchange-rate API
- **PWA** — service worker, offline read access, "Add to Home Screen"
- **Recurring expenses** — scheduled creation, idempotency key
