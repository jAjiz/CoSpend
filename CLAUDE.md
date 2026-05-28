# CLAUDE.md — CoSpend Project Context

This file gives Claude Code all the context needed to work effectively on CoSpend across sessions.

---

## Session Behavior — Read This First

This project is built as a **learning exercise**, not purely for speed of delivery.

**Before writing any code, always:**
1. Explain the concept or pattern involved
2. Present the available approaches with their trade-offs
3. Ask if there is ambiguity about what the user wants
4. Wait for explicit confirmation before generating production code

**Do not** auto-generate full features, boilerplate, or scaffold entire modules unless explicitly asked.

**Do** prioritise explanations, architecture discussions, and educational material.

If asked "how should we implement X?", respond with an explanation and options — not code.
If asked "implement X", generate the minimal code needed — nothing more.

---

## Project Description

CoSpend is a full-stack expense-sharing web app. Users create groups, log shared expenses, and automatically calculate who owes what to whom using a debt-minimisation algorithm.

**Stack:**
- Frontend: Angular 21, TailwindCSS v3, TypeScript (strict)
- Backend: Node.js, Express 4, plain JavaScript
- Database: PostgreSQL 16
- Auth: JWT (stateless, long-lived access token — no refresh tokens in MVP)
- Validation: Zod (backend only)

---

## Repository Layout

```
CoSpend/
├── frontend/                Angular 21 SPA
│   └── src/app/
│       ├── core/            Singleton services, guards, interceptors, models
│       ├── features/        Lazy-loaded route components (one folder per feature)
│       └── shared/          Reusable components and pipes
├── backend/
│   └── src/
│       ├── config/          Environment validation
│       ├── controllers/     Thin route handlers — delegate logic to services
│       ├── db/              pg Pool, migrations
│       ├── middleware/       Auth, validation, error handling
│       ├── routes/          Express routers
│       ├── services/        Business logic (pure functions where possible)
│       └── validators/      Zod schemas, one file per resource
├── docker-compose.yml
├── README.md
├── ROADMAP.md
└── CLAUDE.md                ← this file
```

---

## Branch Strategy

- **Development branch:** `claude/cospend-architecture-planning-NWXGO`
- **PR:** https://github.com/jAjiz/CoSpend/pull/1
- All work goes on the development branch. Push to it; the PR updates automatically.
- Never push directly to `main` without explicit instruction.

---

## Development Workflow

Work strictly phase by phase following [ROADMAP.md](./ROADMAP.md).

For each phase:
1. Read the phase objective and deliverables together
2. Discuss and agree on implementation approach before starting
3. Implement one deliverable at a time
4. Verify it works and is understood before the next

Do not skip phases or implement future-phase features ahead of time.

---

## Frontend — Angular 21

### Core Conventions

**Always use standalone components.** No NgModules.

```typescript
@Component({
  selector: 'app-foo',
  standalone: true,
  imports: [RouterLink, DatePipe, SomeSharedComponent],
  templateUrl: './foo.component.html',
})
```

**Local state: Signals. Never BehaviorSubject for local state.**

```typescript
readonly items = signal<Item[]>([]);
readonly total = computed(() => this.items().reduce((s, i) => s + i.amount, 0));
```

**HTTP: keep as Observables.** Subscribe in the component or service, then set signals.

```typescript
this.groupsService.getAll().subscribe({ next: (g) => this.groups.set(g) });
```

**Route params via `input()`** — `withComponentInputBinding` is enabled in `app.config.ts`.

```typescript
id = input.required<string>(); // receives :id from the route
```

**Always inject with `inject()`, never via constructor.**

```typescript
private auth = inject(AuthService); // correct
```

**Template control flow — new syntax only:**

```html
@if (loading()) { ... }
@for (item of items(); track item.id) { ... }
```

Never use `*ngIf`, `*ngFor`, `*ngSwitch`.

### Imports Reference

| Thing needed | Import from |
|-------------|-------------|
| `DatePipe`, `NgClass` | `@angular/common` |
| `RouterLink`, `RouterLinkActive` | `@angular/router` |
| `ReactiveFormsModule` | `@angular/forms` |
| `CurrencyEurPipe` | `@shared/pipes/currency-eur.pipe` |

Always add pipes and directives to the component's `imports[]` array.

### Styling

TailwindCSS v3. Custom utilities in `src/styles.css`:

- `.btn-primary` / `.btn-secondary` / `.btn-ghost`
- `.card` / `.card-hover`
- `.form-input` / `.form-label`
- `.balance-positive` / `.balance-negative` / `.balance-zero`

Dark mode is class-based (`dark:` prefix). `ThemeService` adds/removes `class="dark"` on `<html>`.

### Money Formatting

Always use `CurrencyEurPipe` for monetary output. Never use `CurrencyPipe` or manual `toLocaleString()`.

---

## Backend — Express + Node.js

### Core Conventions

**No TypeScript.** Plain JavaScript with JSDoc annotations where helpful.

**Validation in `validators/`, never in controllers.** Every route receiving a body goes through `validate(schema)` middleware:

```javascript
router.post('/', validate(createGroupSchema), ctrl.create);
```

**Controllers are thin.** Direct DB queries are acceptable in controllers. Extract to `services/` only when logic is shared or complex.

**Always call `next(err)` in catch blocks** — never `res.status(500)` directly.

```javascript
} catch (err) { next(err); }
```

**Parameterised queries only.** Never string-interpolate SQL.

```javascript
pool.query('SELECT * FROM users WHERE id = $1', [id]); // correct
pool.query(`SELECT * FROM users WHERE id = '${id}'`);   // never
```

**Money:** Store as `NUMERIC(12,2)`. Always `parseFloat()` results. Round to 2 decimals before storing.

### Environment Variables

Validated at startup in `src/config/env.js` using Zod. App exits with code 1 if required vars are missing.

Required: `DATABASE_URL`, `JWT_SECRET` (≥32 chars)
Optional with defaults: `PORT`, `NODE_ENV`, `JWT_EXPIRES_IN`, `CORS_ORIGIN`

Copy `backend/.env.example` → `backend/.env` to start.

### Error Codes

The global error handler in `middleware/error.middleware.js` maps PostgreSQL error codes:
- `23503` → 400 (FK violation)
- `23505` → 409 (unique constraint)
- `23514` → 400 (check constraint)

---

## Database

PostgreSQL 16. Local via Docker; Neon in production.

### Tables

| Table | Purpose |
|-------|---------|
| `users` | Registered accounts |
| `groups` | Expense groups |
| `group_members` | Many-to-many: user ↔ group, with `is_favorite` and `role` |
| `expenses` | Individual expenses with `paid_by`, `amount`, `category` |
| `expense_splits` | One row per participant per expense — stores their share |
| `settlements` | Recorded payments between members |

### Balance Formula

```
balance = (paid by user) − (splits owed by user) + (settlements received) − (settlements sent)
```

Computed as a single SQL query — never in application code.

### Migrations

Files in `backend/src/db/migrations/`, named `001_initial.sql`, `002_...sql`.
Run with `npm run migrate`. Applied in alphabetical order.

---

## Settlement Algorithm

Location: `backend/src/services/settlement.algorithm.js`

A greedy O(n log n) algorithm. Takes a `Map<userId, netBalance>` and returns the minimum list of `{ fromUserId, toUserId, amount }` transactions.

**Deliberately kept as a pure function with no DB dependency** — this is what makes it unit-testable without mocking.

---

## Testing

```bash
# Backend unit tests
cd backend && npx jest

# Frontend build check
cd frontend && npx ng build --configuration development
```

Unit tests live in `backend/src/__tests__/`. E2E tests (Playwright) are planned for Phase 7.

---

## Deployment

| Service | Purpose |
|---------|---------|
| Vercel | Frontend — `vercel.json` configures SPA rewrite |
| Render / Railway | Backend — `npm start` as start command |
| Neon | PostgreSQL — `DATABASE_URL` with `sslmode=require` in production |

---

## What Is NOT Implemented Yet

Do not assume these exist — check before generating code that depends on them:

- E2E tests (Playwright) — Phase 7
- GitHub Actions CI — Phase 7
- Percentage / exact-amount splits — Post-MVP
- Group image upload (only external URLs accepted now)
- OAuth login — Post-MVP
- PWA manifest — Post-MVP
- Expense filter UI (API supports filters; UI chips are not wired)
- Toast notification system — Phase 6

---

## Style Rules

- No `any` in TypeScript unless unavoidable — document why
- No comments explaining *what* code does, only *why*
- No dead code after refactors
- Components: keep under ~200 lines; extract to `shared/components/` when shared
- Keep phases small: understand each deliverable before the next
