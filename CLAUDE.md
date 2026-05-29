# CLAUDE.md — CoSpend Project Context

This file is the persistent context for Claude Code sessions working on this project.

---

## Project

CoSpend is a full-stack expense-sharing application. Angular 21 frontend, Express + Node.js backend, PostgreSQL database. Built as a serious single-developer portfolio project with production-quality standards.

**Stack:**
- Frontend: Angular 21, TailwindCSS v4, TypeScript (strict)
- Backend: Node.js, Express 5, TypeScript (strict, ESM)
- Database: PostgreSQL 16
- Auth: JWT (stateless, 30-day token)
- Validation: Zod (backend; also the source of inferred request types via `z.infer`)
- Testing: Jest (unit), Playwright (E2E)

---

## Repository Layout

```
CoSpend/
├── frontend/
│   └── src/app/
│       ├── core/            Auth service, guards, interceptors, models, API services
│       ├── features/        Lazy-loaded route components (one folder per feature)
│       └── shared/          Reusable components and pipes
├── backend/
│   └── src/
│       ├── config/          Environment validation (Zod, exits on startup if invalid)
│       ├── controllers/     Thin route handlers
│       ├── db/              pg Pool, migrations
│       ├── middleware/      Auth, validation, error handling
│       ├── routes/          Express routers
│       ├── services/        Business logic — pure functions where possible
│       └── validators/      Zod schemas, one file per resource
├── docker-compose.yml
├── README.md
├── ROADMAP.md
└── CLAUDE.md
```

---

## Branch Strategy

- **Development branch:** `claude/cospend-architecture-planning-NWXGO`
- **PR:** https://github.com/jAjiz/CoSpend/pull/1
- All work goes on the development branch.
- Never push directly to `main` without explicit instruction.

---

## Working Style

Work phase by phase as defined in ROADMAP.md. Each phase is a discrete, reviewable unit of work.

- Propose the implementation approach before writing code when the decision involves meaningful trade-offs.
- Ask before assuming when requirements are ambiguous.
- Generate minimal code — no speculative abstractions, no patterns the current feature set doesn't require.
- Prefer editing existing files to creating new ones.
- No dead code, no backward-compatibility shims, no commented-out blocks.

---

## Frontend — Angular 21

### Conventions

**Standalone components only** — no NgModules.

```typescript
@Component({
  standalone: true,
  imports: [RouterLink, DatePipe, BalanceBadgeComponent],
  templateUrl: './foo.component.html',
})
```

**Signal-based local state** — no `BehaviorSubject` for component or service state.

```typescript
readonly items = signal<Item[]>([]);
readonly total = computed(() => this.items().reduce((sum, i) => sum + i.amount, 0));
```

**HTTP stays as Observables** — subscribe and push into signals.

```typescript
this.groupsService.getAll().subscribe({ next: (g) => this.groups.set(g) });
```

**Route params via `input()`** — `withComponentInputBinding` enabled in `app.config.ts`.

```typescript
id = input.required<string>();
```

**`inject()` function** — not constructor injection.

**New template control flow** — `@if`, `@for`, `@switch`. Never `*ngIf` / `*ngFor`.

### Imports

| Symbol | From |
|--------|------|
| `DatePipe`, `NgClass`, `CurrencyPipe` | `@angular/common` |
| `RouterLink`, `RouterLinkActive` | `@angular/router` |
| `ReactiveFormsModule` | `@angular/forms` |
| `CurrencyEurPipe` (project pipe) | `@shared/pipes/currency-eur.pipe` |

All pipes and directives must appear in the component's `imports` array.

### Styling

TailwindCSS v4 (CSS-first config — no `tailwind.config.js`; wired via `@tailwindcss/postcss` in `.postcssrc.json`). Global utility classes defined in `src/styles.css`:

- `.btn-primary`, `.btn-secondary`, `.btn-ghost`
- `.card`, `.card-hover`
- `.form-input`, `.form-label`
- `.balance-positive`, `.balance-negative`, `.balance-zero`

These are authored with v4 syntax: component classes in `@layer components { … @apply … }`, single-purpose utilities via `@utility`, theme tokens via `@theme { … }`. `@apply` in a component-scoped stylesheet (not `src/styles.css`) requires `@reference "tailwindcss";` at the top of that file.

Dark mode is class-based. `ThemeService` manages `class="dark"` on `<html>`. The `dark:` variant is declared in `styles.css` via `@custom-variant dark (&:where(.dark, .dark *))` — v4 defaults to `prefers-color-scheme`, so class-based mode is opt-in.

Always use `CurrencyEurPipe` for monetary display — never `toLocaleString()` or manual formatting.

---

## Backend — Express + Node.js (TypeScript)

### Conventions

**TypeScript (strict), ESM modules.** Dev runs via `tsx watch` (no build step while developing); production is `tsc` → `dist/`, then `node dist/server.js`. Relative imports carry a `.js` extension (`import app from './app.js'`) — required by `NodeNext` resolution even though the source is `.ts`. Derive types from Zod schemas with `z.infer` rather than hand-writing parallel interfaces.

**Validation at the boundary** — every route with a request body goes through `validate(schema)` middleware. No validation logic in controllers.

```typescript
router.post('/', validate(createGroupSchema), ctrl.create);
```

**Thin controllers** — direct DB queries in controllers are fine. Extract to `services/` only when logic is shared or non-trivial (e.g., the settlement algorithm).

**Error propagation** — Express 5 automatically forwards errors thrown in (or rejected from) `async` handlers to the global error handler, so async handlers don't need a try/catch solely to call `next(err)`; let the error propagate. Catch only to add context, then `next(err)`. Never call `res.status(500)` directly in route handlers.

**Parameterised queries only** — no string interpolation in SQL.

```typescript
pool.query('SELECT * FROM users WHERE id = $1', [id]);
```

**Money** — `NUMERIC(12,2)` in the DB. `parseFloat()` on read. Round to 2 decimal places before write.

### Error Handling

Global error handler (`middleware/error.middleware.ts`) maps PostgreSQL error codes:

| Code | HTTP | Meaning |
|------|------|---------|
| `23503` | 400 | FK violation — referenced record not found |
| `23505` | 409 | Unique constraint — resource already exists |
| `23514` | 400 | Check constraint violation |

---

## Database

PostgreSQL 16. Docker locally, Neon in production.

### Schema Overview

| Table | Purpose |
|-------|---------|
| `users` | Accounts |
| `groups` | Expense groups |
| `group_members` | User ↔ group, with `is_favorite` and `role` |
| `expenses` | Expenses with `paid_by`, `amount`, `category`, `split_type` |
| `expense_splits` | Resolved monetary share per participant per expense |
| `settlements` | Recorded payments between members |

### Balance Formula

Computed in SQL, never in application code:

```
balance = SUM(expenses.amount WHERE paid_by = user)
        − SUM(expense_splits.amount WHERE user_id = user)
        + SUM(settlements.amount WHERE to_user_id = user)
        − SUM(settlements.amount WHERE from_user_id = user)
```

### Migrations

`backend/src/db/migrations/`, named `001_initial.sql`, `002_...` etc.
Runner: `pnpm migrate` (applies in alphabetical order, no rollback in MVP).

---

## Settlement Algorithm

`backend/src/services/settlement.algorithm.ts`

Pure function — no DB dependency. Takes `Map<userId, netBalance>`, returns `{ fromUserId, toUserId, amount }[]` representing the minimum transaction set. O(n log n) greedy approach.

The separation from `settlement.service.ts` (which handles DB access) exists specifically to keep the algorithm unit-testable without mocking.

---

## Testing

```bash
cd backend && pnpm exec tsc --noEmit                      # backend type check
cd backend && pnpm exec jest                              # unit tests (ts-jest, configured in Phase 7)
cd frontend && pnpm exec ng build --configuration production # type + build check
```

E2E (Playwright) — Phase 7.

---

## Deployment

| Service | Role | Notes |
|---------|------|-------|
| Vercel | Frontend | `vercel.json` SPA rewrite configured |
| Render / Railway | Backend | `pnpm install && pnpm build` (tsc) then `pnpm start`, env vars in dashboard |
| Neon | PostgreSQL | `DATABASE_URL` with `sslmode=require` |

---

## Not Yet Implemented

Check before generating code that depends on these:

- E2E tests (Playwright) — Phase 7
- GitHub Actions CI — Phase 7
- Exact / percentage expense splits — Post-MVP
- Group image upload (external URL only for now)
- OAuth — Post-MVP
- PWA — Post-MVP
- Expense filter UI (API supports filters; frontend chips not yet wired)
- Toast notification system — Phase 6
