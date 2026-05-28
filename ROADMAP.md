# CoSpend — Roadmap

## Philosophy

This project is built **phase by phase, with full understanding at each step**.

No phase begins until the previous one is complete and understood. Each phase is small enough to reason about entirely. Implementation details are defined during execution, not upfront — this document stays intentionally high-level so it remains stable as the project evolves.

---

## Phases

---

### Phase 0 — Project Foundation

**Objective**
Understand the full-stack project structure and make explicit, conscious decisions about every tooling choice before writing any application code.

**Expected knowledge gained**
- Why Angular and Express live in separate directories (separation of concerns, independent deploy targets)
- What each configuration file does (`angular.json`, `tsconfig.json`, `package.json`, `tailwind.config.js`)
- How environment variables work in both Angular (build-time replacement) and Node.js (runtime)
- Why Docker Compose is used for local development and what it replaces
- How PostCSS processes Tailwind at build time

**Main deliverables**
- Scaffolded Angular project with Tailwind configured and building cleanly
- Scaffolded Express project with a working `/health` endpoint
- Docker Compose file running PostgreSQL locally
- Environment variable handling in both apps (dev vs. production)
- `vercel.json` with HTML5 routing rewrite rule explained
- `.gitignore` covering generated files, secrets, and build output

**Prerequisites**
- Node.js ≥ 20, npm, Docker installed
- Basic familiarity with TypeScript and JavaScript

---

### Phase 1 — Database Design

**Objective**
Design the full PostgreSQL schema before writing any application code. Understand the data model deeply, because every feature built later depends on it.

**Expected knowledge gained**
- How to translate real-world domain concepts into relational tables
- Primary keys: UUID vs. serial — when to use each and why
- Foreign keys, `ON DELETE` strategies, and their implications
- Junction tables for many-to-many relationships (group members)
- Why money should be stored as `NUMERIC(12,2)`, not `FLOAT`
- How to write SQL migrations and why order matters
- Index design: when indexes help, when they hurt

**Main deliverables**
- Entity–relationship diagram (documented in comments or a diagram file)
- `001_initial.sql` migration script covering all tables
- A migration runner (`npm run migrate`) that applies files in order
- A clear explanation in code comments of why each constraint exists

**Prerequisites**
- Phase 0 complete

---

### Phase 2 — Authentication

**Objective**
Build secure user registration and login, end to end, with a thorough understanding of every decision made.

**Expected knowledge gained**
- How JWT works: structure (header, payload, signature), signing, verification, expiry
- Why we hash passwords with bcrypt and what the cost factor means
- Stateless vs. stateful authentication — trade-offs for a SPA
- Angular `HttpInterceptor`: what it is and when to use one
- Angular route guards: `CanActivate` — how they integrate with the router
- How Angular Signals replace `BehaviorSubject` for local state
- Input validation with Zod: why schema-first validation at the boundary
- Rate limiting on auth endpoints: why and how

**Main deliverables**
- `POST /api/auth/register` and `POST /api/auth/login` endpoints
- `AuthService` holding `token` and `currentUser` as signals
- `JwtInterceptor` attaching the token to every outgoing request
- `authGuard` and `guestGuard` protecting routes
- Login and Register components with reactive forms and validation feedback

**Prerequisites**
- Phase 1 complete (users table exists)

---

### Phase 3 — Groups & Members

**Objective**
Build the group management feature, establishing the patterns that all future features will follow.

**Expected knowledge gained**
- REST resource design: URLs as nouns, HTTP verbs as actions
- Nested routes (`/groups/:id/members`) and when they make sense
- Angular lazy-loaded routes: why they improve initial load time
- The Angular service layer: why components should not call `HttpClient` directly
- Reactive state with `signal()` and `computed()` — when each is appropriate
- Optimistic UI updates: how to apply changes before the server confirms
- PostgreSQL transactions: when and why to wrap multiple queries

**Main deliverables**
- Full CRUD for groups (`GET`, `POST`, `PATCH`, `DELETE`)
- Member management: add by email, list, remove
- Favourite toggle with optimistic update
- Dashboard component: total balance, favourite groups, all groups list
- `LayoutComponent` with header, dark-mode toggle, and sign-out

**Prerequisites**
- Phase 2 complete (authenticated user exists)

---

### Phase 4 — Expenses

**Objective**
Implement expense creation and listing with equal splitting, going deep on form design and monetary arithmetic correctness.

**Expected knowledge gained**
- Angular Reactive Forms: `FormBuilder`, `Validators`, typed controls
- Why floating-point arithmetic is dangerous for money — and how to handle rounding
- PostgreSQL `NUMERIC` vs `FLOAT`: the practical difference
- Database transactions for multi-table writes (expense + splits in one atomic operation)
- Computed signals: deriving UI state (share per person) from form values in real time
- API query parameters for filtering: how to design flexible, safe filter endpoints
- Zod validation for complex payloads (arrays, conditional fields)

**Main deliverables**
- `POST /api/groups/:id/expenses` creating expense and splits atomically
- `GET /api/groups/:id/expenses` with category/month/payer filters
- `DELETE /api/groups/:id/expenses/:id` (creator only)
- Expense form: amount, description, category picker, paid-by, participant checkboxes, date
- Group detail page: tabbed view of expenses and members

**Prerequisites**
- Phase 3 complete

---

### Phase 5 — Settlement Algorithm

**Objective**
Build the debt-minimisation feature, learning how to design and test a non-trivial algorithm as a pure function.

**Expected knowledge gained**
- What "net balance" means and how to compute it in a single SQL query
- Greedy algorithms: when they are optimal, when they are not
- Pure functions: why isolating logic from I/O makes it far easier to test
- Unit testing with Jest: what makes a good unit test
- The difference between a suggested settlement (calculated) and a recorded settlement (persisted)
- Why separating `settlement.algorithm.js` from `settlement.service.js` matters

**Main deliverables**
- `settlement.algorithm.js`: pure function, fully unit-tested
- `GET /api/groups/:id/balances` — per-member net balance
- `GET /api/groups/:id/settlements/suggested` — optimal payment plan
- `POST /api/groups/:id/settlements` — record a payment
- Settlements UI: balance summary, suggested payments, mark-as-paid, history

**Prerequisites**
- Phase 4 complete

---

### Phase 6 — UI Polish & Dark Mode

**Objective**
Improve the visual quality and user experience, learning how to layer progressive enhancement onto a working application.

**Expected knowledge gained**
- TailwindCSS class-based dark mode vs. media-query dark mode — trade-offs
- `localStorage` for persisting UI preferences without a database round-trip
- Skeleton loading states: why they feel better than spinners
- Angular animations: `@angular/animations` basics for route and element transitions
- Toast notifications: an event-driven approach without global state
- Mobile-first CSS: designing for the smallest screen first

**Main deliverables**
- Dark mode toggle, preference persisted across sessions
- Loading skeletons on all data-fetching views
- Toast notification system for success and error feedback
- Responsive layout validated on mobile viewport
- Page titles set via Angular `Title` service on each route

**Prerequisites**
- Phase 5 complete (all features working, even if visually rough)

---

### Phase 7 — Testing

**Objective**
Add automated tests at multiple levels and understand the distinct purpose of each kind.

**Expected knowledge gained**
- The testing pyramid: unit vs. integration vs. E2E — what each protects against
- Angular `TestBed`: how to test components and services in isolation
- Mocking HTTP with `HttpClientTestingModule`
- Playwright: browser automation, selectors, and writing stable tests
- GitHub Actions: defining a CI workflow that runs on every push
- What to test and what not to — avoiding tests that only duplicate the implementation

**Main deliverables**
- Jest unit tests: settlement algorithm, auth controller, key validators
- Angular `TestBed` tests: `AuthService`, `ThemeService`
- Playwright E2E suite: register → create group → add expense → settle up
- GitHub Actions workflow: build + test on every PR push

**Prerequisites**
- Phase 6 complete

---

### Phase 8 — Deployment

**Objective**
Ship the application to production cloud services and understand every environment difference between dev and prod.

**Expected knowledge gained**
- How Vercel serves an Angular SPA (build output, rewrite rules, CDN caching)
- How Render/Railway runs a Node.js process (start command, env vars, health checks)
- Neon's serverless PostgreSQL: connection pooling differences vs. a traditional DB
- CORS: why it exists, how to configure it correctly for a split frontend/backend deploy
- SSL in production: why `sslmode=require` matters for database connections
- Environment parity: keeping dev and prod as similar as possible

**Main deliverables**
- Frontend live on Vercel with correct SPA routing
- Backend live on Render/Railway with health check passing
- Neon PostgreSQL provisioned with migrations applied
- README updated with live URLs
- Production environment variables documented (without secrets)

**Prerequisites**
- Phase 7 complete

---

## Post-MVP Ideas

Tracked here for future learning opportunities, not scheduled:

- **OAuth (Google login)** — how OAuth 2.0 flows work, passport.js
- **Exact and percentage splits** — form mode switching, Zod discriminated unions
- **File upload for group images** — multipart forms, Cloudflare R2 or Supabase Storage
- **Multi-currency** — `currency` column, open exchange-rate APIs, rounding across currencies
- **Recurring expenses** — cron jobs in Node.js, idempotency
- **PWA** — service workers, offline support, "Add to Home Screen"
