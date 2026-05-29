# Phase 0 – Foundation & Tooling

## Context

- **Branch:** `claude/cospend-architecture-planning-NWXGO` (already created and checked out).
- **Prior phases delivered:** None. This is the first implementation phase. The repo currently contains only documentation (`CLAUDE.md`, `README.md`, `ROADMAP.md`) and loose UI prototypes (`index.html`, `dashboard.html`, `styles.css`).
- **Relevant files to read before starting:**
  - `ROADMAP.md` — Phase 0 section defines the deliverables this plan must satisfy and the rationale for each tooling choice.
  - `CLAUDE.md` — the authoritative convention source for folder layout, Angular/Express conventions, and the environment/error contracts every later phase assumes.
  - `README.md` — the "Getting Started" block documents the exact dev commands and env vars this scaffold must make true.
- **Architectural decisions (already made — do not relitigate):**
  - **Monorepo, two manifests.** `frontend/` (Angular) and `backend/` (Express) each own a `package.json`. No root workspace tooling — the two apps deploy to different hosts.
  - **Angular 21, `@angular/build:application` builder** (esbuild-based, the v17+ default). Standalone components, strict TypeScript, new control flow — all defaults in v21.
  - **TailwindCSS v4.** CSS-first configuration — no `tailwind.config.js`; wired into the Angular build via the `@tailwindcss/postcss` plugin in `.postcssrc.json`. Class-based dark mode is declared with `@custom-variant dark` (v4 defaults to `prefers-color-scheme`). Chosen to showcase the current Tailwind stack.
  - **Backend is TypeScript (strict), ESM modules.** Run directly in dev with `tsx watch` (no separate build step while developing); compiled with `tsc` to `dist/` for `pnpm start`. Handlers carry explicit `Request`/`Response` types, and Zod schemas double as the source of runtime validation *and* inferred types (`z.infer`) so validation and types can never drift. This matches how modern production Node services are built; the small one-time toolchain cost is the accepted trade-off.
  - **Environment validation via Zod on startup.** `backend/src/config/env.ts` validates `process.env` and calls `process.exit(1)` on the first invalid value, before any server starts. This is the single source of env truth for the whole backend.
  - **Environment file replacement, classic naming.** `environment.ts` (dev default) is replaced by `environment.prod.ts` in the `production` build configuration, per `ROADMAP.md`. Created manually rather than via `ng generate environments` to keep the `.prod.ts` name the roadmap specifies.
  - **No speculative infrastructure.** No `pg` pool, no migrations, no CORS, no auth deps in this phase — they arrive in the phase that first needs them. Phase 0 ships an empty folder structure and a single health endpoint as the build/run smoke signal.

---

## Target architecture

```
CoSpend/
├── .gitignore                  node_modules, dist, .angular, .env, coverage
├── docker-compose.yml          postgres:16-alpine  →  localhost:5432
├── vercel.json                 SPA rewrite: /(.*) → /index.html
├── CLAUDE.md  README.md  ROADMAP.md
│
├── frontend/                   Angular 21 SPA   →  ng serve :4200
│   ├── angular.json            production config has fileReplacements
│   ├── .postcssrc.json         { plugins: { '@tailwindcss/postcss': {} } }
│   ├── package.json
│   └── src/
│       ├── styles.css          @import 'tailwindcss'; @custom-variant dark
│       ├── environments/
│       │   ├── environment.ts        { production: false, apiUrl: …:3000/api }
│       │   └── environment.prod.ts   { production: true,  apiUrl: <Phase 8> }
│       └── app/  core/  features/  shared/   (empty, .gitkeep)
│
└── backend/                    Express 5 API (TypeScript, ESM)   →  pnpm dev :3000
    ├── package.json            "type": "module"; scripts: dev (tsx watch), build (tsc), start
    ├── tsconfig.json           strict; module/moduleResolution NodeNext; outDir dist/
    ├── .env.example            PORT, NODE_ENV, DATABASE_URL, JWT_SECRET
    └── src/
        ├── server.ts           imports config (validates) → app.listen
        ├── app.ts              express(), json(), GET /api/health
        ├── config/env.ts       Zod schema + z.infer<Env>, process.exit(1) on invalid
        ├── controllers/  services/  routes/  middleware/  validators/  db/
        │                       (empty, .gitkeep)

Data flow at end of phase:
  GET http://localhost:3000/api/health   → 200 { "status": "ok" }
  pnpm build (backend)                    → tsc exits 0, emits backend/dist/server.js
  ng build --configuration production     → exits 0, emits dist/frontend/browser
  docker compose up -d                    → cospend-db healthy on :5432
```

---

## Step 0 — Prerequisites

Confirm the toolchain before generating anything. No code is written in this step.

- **Node.js ≥ 20**: `node -v`.
- **pnpm ≥ 9** — the package manager for both apps: `pnpm -v`. If missing, enable it via Corepack (`corepack enable pnpm`) or install standalone.
- **Docker** with Compose v2: `docker --version` and `docker compose version`.
- **Angular CLI is invoked via `pnpm dlx @angular/cli@21`** — do not require a global install.
- Confirm the working branch: `git branch --show-current` returns `claude/cospend-architecture-planning-NWXGO`.

No commit.

---

## Step 1 — Root hygiene and prototype removal

Establish the root `.gitignore` and delete the loose UI prototype so the monorepo layout is clean. The prototype is no longer needed — it is removed, not relocated.

### 1.1 Create the root `.gitignore`

`/.gitignore`:

```gitignore
# Dependencies
node_modules/

# Build output
dist/
/frontend/dist/
/backend/dist/

# Angular cache
.angular/

# Environment
.env
.env.local

# Test / coverage
coverage/

# Logs
*.log
npm-debug.log*
pnpm-debug.log*

# Editor / OS
.DS_Store
Thumbs.db
.vscode/*
!.vscode/extensions.json
.idea/
```

### 1.2 Remove the UI prototype

Delete the three loose prototype files from the repo root. Use `git rm` so the removal is staged.

```bash
git rm index.html dashboard.html styles.css
```

After this, `index.html`, `dashboard.html`, and `styles.css` must not exist at the repo root.

**Commit:** `chore: add root gitignore and remove loose UI prototype`

---

## Step 2 — Scaffold the Angular 21 frontend with Tailwind v4

Generate the Angular SPA and wire TailwindCSS v4. v4 is configured through the `@tailwindcss/postcss` plugin, which Angular's esbuild builder picks up from a `.postcssrc.json` file — there is no `tailwind.config.js` in v4.

### 2.1 Generate the Angular project

Run from the repo root. The CLI creates `frontend/` with its own `package.json`.

```bash
pnpm dlx @angular/cli@21 new frontend \
  --style=css \
  --routing \
  --ssr=false \
  --skip-git \
  --package-manager=pnpm \
  --defaults
```

This produces standalone components, strict TypeScript (`strict: true`, `strictTemplates: true`), and the `@angular/build:application` builder by default. The default output path is `dist/frontend/browser`.

### 2.2 Install and configure TailwindCSS v4

TailwindCSS v4 is CSS-first: there is no `tailwind.config.js` — configuration lives in the stylesheet, and source files are detected automatically. Tailwind is wired into the Angular build through the `@tailwindcss/postcss` plugin, declared in a `.postcssrc.json` file that the esbuild builder picks up.

Run from `frontend/`:

```bash
cd frontend
pnpm add -D tailwindcss @tailwindcss/postcss
```

> Latest stable resolves to Tailwind v4 — no version pin needed. No `postcss` or `autoprefixer` packages, and no `tailwindcss init`: v4 bundles autoprefixing and `@import` handling, and has no JS config file to generate.

Create `frontend/.postcssrc.json` so the Angular build runs Tailwind through PostCSS:

```json
{
  "plugins": {
    "@tailwindcss/postcss": {}
  }
}
```

Replace the contents of `frontend/src/styles.css`. v4 collapses the three `@tailwind` directives into a single import. Because `ThemeService` drives class-based dark mode (`class="dark"` on `<html>`), the class-based `dark:` variant must be declared explicitly — v4 defaults to `prefers-color-scheme`. Component utility classes (`.btn-primary`, `.card`, etc. from `CLAUDE.md`) are added in Phase 6, not here.

```css
@import 'tailwindcss';

@custom-variant dark (&:where(.dark, .dark *));
```

### 2.3 Verify the baseline build

From `frontend/`:

```bash
pnpm exec ng build --configuration production
```

Must exit 0 and emit `dist/frontend/browser/`. Tailwind classes added to any template (verified in the acceptance checklist) must appear in the emitted CSS.

**Commit:** `feat(frontend): scaffold Angular 21 project with TailwindCSS v4`

---

## Step 3 — Angular environments and build-time file replacement

Create the environment files with the classic `environment.ts` / `environment.prod.ts` naming and wire the replacement into the `production` build configuration. `apiUrl` is the single place the frontend learns where the API lives.

### 3.1 Create the environment files

`frontend/src/environments/environment.ts` (development default):

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
};
```

`frontend/src/environments/environment.prod.ts` (production; `apiUrl` is finalized in Phase 8):

```typescript
export const environment = {
  production: true,
  apiUrl: 'https://cospend-api.onrender.com/api',
};
```

### 3.2 Wire `fileReplacements` into `angular.json`

In `frontend/angular.json`, under `projects.frontend.architect.build.configurations.production`, add a `fileReplacements` array. The `production` block already exists from the scaffold; add the array as the first key inside it:

```json
"production": {
  "fileReplacements": [
    {
      "replace": "src/environments/environment.ts",
      "with": "src/environments/environment.prod.ts"
    }
  ],
  "budgets": [
    {
      "type": "initial",
      "maximumWarning": "500kB",
      "maximumError": "1MB"
    },
    {
      "type": "anyComponentStyle",
      "maximumWarning": "4kB",
      "maximumError": "8kB"
    }
  ],
  "outputHashing": "all"
}
```

> The `budgets` and `outputHashing` keys above are the scaffold defaults — keep whatever values the generated file already has; only the `fileReplacements` array is new. Match the existing indentation of `angular.json`.

### 3.3 Prove the replacement compiles

From `frontend/`:

```bash
pnpm exec ng build --configuration production
```

Must exit 0. A production build now resolves `environment.production === true`.

**Commit:** `feat(frontend): add environment files with production file replacement`

---

## Step 4 — Scaffold the Express backend with Zod env validation

Create the `backend/` app: the documented folder structure, a Zod-validated environment module that exits on invalid config, and a single health endpoint as the run/smoke signal. TypeScript (strict), ESM modules.

### 4.1 Initialize the backend package

Run from the repo root:

```bash
mkdir backend
cd backend
pnpm init
pnpm add express zod dotenv
pnpm add -D typescript tsx @types/node @types/express
```

Every package resolves to its latest stable release — `express` is **Express 5**. `tsx` runs `.ts` directly with a built-in `watch` mode (dev), `typescript` provides `tsc` for the production build, and the `@types/*` packages supply Node and Express type definitions. No `nodemon` — `tsx watch` replaces it.

Then set the metadata and scripts in the generated `backend/package.json` (note `"type": "module"` — the backend is ESM). Leave the `dependencies` / `devDependencies` blocks exactly as `pnpm add` wrote them — do not pin or transcribe versions:

```json
{
  "name": "cospend-backend",
  "version": "0.1.0",
  "description": "CoSpend expense-sharing API",
  "type": "module",
  "main": "dist/server.js",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js"
  },
  "engines": {
    "node": ">=20"
  },
  "license": "MIT"
}
```

### 4.2 Create `tsconfig.json`

`backend/tsconfig.json` — strict, ESM (`NodeNext`), compiling `src/` to `dist/`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "sourceMap": true
  },
  "include": ["src/**/*.ts"]
}
```

> Under `NodeNext`, relative imports in source must carry a `.js` extension (e.g. `import app from './app.js'`) even though the file on disk is `app.ts` — this is the Node ESM resolution rule, not a typo.

### 4.3 Create the folder structure

Create the documented `src/` subdirectories. They are empty in this phase; a `.gitkeep` keeps them tracked. Run from `backend/`:

```bash
mkdir -p src/config src/controllers src/services src/routes src/middleware src/validators src/db
touch src/controllers/.gitkeep src/services/.gitkeep src/routes/.gitkeep src/middleware/.gitkeep src/validators/.gitkeep src/db/.gitkeep
```

`src/config` and `src/db` receive real files in this phase / Phase 1, so they do not need a `.gitkeep`.

### 4.4 Create the Zod environment module

`backend/src/config/env.ts`. Loads `.env`, validates, and exits the process on the first invalid value — nothing in the backend reads `process.env` directly. `z.infer` derives the `Env` type from the schema, so the validated config and its static type share one definition.

```typescript
import 'dotenv/config';
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
});

const parsed = envSchema.safeParse(process.env);

if (!parsed.success) {
  console.error('Invalid environment configuration:');
  console.error(parsed.error.flatten().fieldErrors);
  process.exit(1);
}

export type Env = z.infer<typeof envSchema>;
export const env: Env = parsed.data;
```

### 4.5 Create the Express app and server entry

`backend/src/app.ts` — builds and exports the app, no listening. Keeping `app` separate from `server` makes the app importable by tests in later phases.

```typescript
import express, { type Request, type Response } from 'express';

const app = express();

app.use(express.json());

app.get('/api/health', (_req: Request, res: Response) => {
  res.json({ status: 'ok' });
});

export default app;
```

`backend/src/server.ts` — importing `./config/env.js` first triggers validation before the server binds a port. (The `.js` extension on these relative imports is required by `NodeNext` resolution even though the sources are `.ts`.)

```typescript
import { env } from './config/env.js';
import app from './app.js';

app.listen(env.PORT, () => {
  console.log(`CoSpend API listening on port ${env.PORT}`);
});
```

### 4.6 Create `.env.example`

`backend/.env.example` — documents every variable the schema requires. `JWT_SECRET` must be ≥ 32 chars; the value below is a placeholder for local development only.

```bash
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://cospend:cospend@localhost:5432/cospend
JWT_SECRET=local_dev_secret_change_me_min_32_chars_long
```

### 4.7 Verify the backend boots and the health route responds

From `backend/`, create a real `.env` from the example, then start:

```bash
cp .env.example .env
pnpm dev
```

In a second shell:

```bash
curl http://localhost:3000/api/health
```

Must return `{"status":"ok"}`. Stop the server (`Ctrl-C`) before committing.

**Commit:** `feat(backend): scaffold Express app with Zod env validation and health route`

---

## Step 5 — Docker Compose PostgreSQL 16

Provide a local PostgreSQL 16 instance matching the `DATABASE_URL` in `.env.example`. No schema is created here — migrations are Phase 1.

`/docker-compose.yml`:

```yaml
services:
  db:
    image: postgres:16-alpine
    container_name: cospend-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: cospend
      POSTGRES_PASSWORD: cospend
      POSTGRES_DB: cospend
    ports:
      - '5432:5432'
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U cospend -d cospend']
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

Verify from the repo root:

```bash
docker compose up -d
docker compose ps
```

`cospend-db` must report a healthy/running state on port 5432. Leave the container running or stop it with `docker compose down`.

**Commit:** `feat: add Docker Compose PostgreSQL 16 for local development`

---

## Step 6 — Vercel SPA rewrite

Add the Vercel rewrite so client-side (HTML5 `pathLocationStrategy`) routes deep-link correctly once the frontend deploys. Deployment itself is Phase 8 — this file only declares the rewrite.

`/vercel.json`:

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

**Commit:** `feat: add vercel.json SPA rewrite for HTML5 routing`

---

## Execution order (commits)

1. `chore: add root gitignore and remove loose UI prototype`
2. `feat(frontend): scaffold Angular 21 project with TailwindCSS v4`
3. `feat(frontend): add environment files with production file replacement`
4. `feat(backend): scaffold Express app with Zod env validation and health route`
5. `feat: add Docker Compose PostgreSQL 16 for local development`
6. `feat: add vercel.json SPA rewrite for HTML5 routing`

---

## Acceptance checklist

- [ ] `git branch --show-current` returns `claude/cospend-architecture-planning-NWXGO`.
- [ ] At the repo root, `ls index.html dashboard.html styles.css 2>/dev/null` returns nothing (prototype removed).
- [ ] `git status --ignored --short node_modules` shows `node_modules/` ignored (does not appear as untracked).
- [ ] `cd frontend && pnpm exec ng build --configuration production` exits 0 and `dist/frontend/browser/index.html` exists.
- [ ] `test -f frontend/.postcssrc.json` exits 0 (Tailwind v4 PostCSS plugin wired).
- [ ] `grep -q "@import 'tailwindcss'" frontend/src/styles.css` exits 0.
- [ ] `grep -q '@custom-variant dark' frontend/src/styles.css` exits 0 (class-based dark mode declared).
- [ ] `grep -q '"replace": "src/environments/environment.ts"' frontend/angular.json` exits 0 (file replacement wired).
- [ ] `cd backend && pnpm exec tsx src/config/env.ts` run with no `.env` and a cleared environment exits non-zero and prints `Invalid environment configuration:` (validation rejects missing `DATABASE_URL`/`JWT_SECRET`).
- [ ] With a valid `backend/.env`, `cd backend && pnpm dev` starts and logs `CoSpend API listening on port 3000`.
- [ ] `curl -s http://localhost:3000/api/health` returns `{"status":"ok"}`.
- [ ] `cd backend && pnpm build` exits 0 and `backend/dist/server.js` exists (production `tsc` compile succeeds).
- [ ] `test -f backend/src/app.ts && test -f backend/src/server.ts && test -f backend/src/config/env.ts && test -f backend/tsconfig.json` exits 0.
- [ ] `ls backend/src/{controllers,services,routes,middleware,validators,db}` lists all six directories.
- [ ] `docker compose up -d && docker compose ps` shows `cospend-db` running on `0.0.0.0:5432`.
- [ ] `cat vercel.json` shows the `/(.*)` → `/index.html` rewrite.
- [ ] `git log --oneline -6` lists the six commits from the execution order in sequence.

---

## Non-goals for this phase

- **No database schema, tables, or migrations.** The `migrate.ts` runner and `001_initial.sql` are Phase 1. Docker Compose provides an empty Postgres only.
- **No `pg` dependency or connection pool.** The backend does not connect to the database in this phase; `DATABASE_URL` is validated but unused.
- **No authentication, bcrypt, JWT signing, or CORS.** `JWT_SECRET` is validated for shape only. Auth deps arrive in Phase 2; CORS is configured when the frontend first calls the backend.
- **No Angular routes, components, services, guards, or interceptors.** Only the empty `core/`, `features/`, `shared/` directories and the default scaffold component exist.
- **No global Tailwind utility classes** (`.btn-primary`, `.card`, `.form-input`, etc.). Those are defined in Phase 6 (UI Polish).
- **No Jest, Playwright, or CI configuration.** Testing tooling is Phase 7.
- **No production deployment or environment finalization.** `environment.prod.ts` `apiUrl` is a placeholder; the real backend URL and CORS origin are set in Phase 8.
- **No Angular dev proxy to the backend.** Not required until the frontend issues HTTP calls (Phase 2).
