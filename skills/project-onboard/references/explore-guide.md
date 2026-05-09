# Codebase Exploration Guide

Step-by-step sequence for full-scan exploration of an unknown project. Follow this order — it's designed to build understanding progressively rather than randomly reading files.

---

## Phase 1 — Orientation (5 minutes)

Goal: understand what kind of project this is and where things live.

```bash
# Top-level structure
ls -la
find . -maxdepth 2 -type f -name "*.json" -o -name "*.yaml" -o -name "*.toml" | grep -v node_modules | grep -v .git

# Tech stack
cat package.json            # or pnpm-workspace.yaml for monorepos
cat tsconfig.json 2>/dev/null
cat Dockerfile 2>/dev/null
```

Read in order:
1. `README.md` — project purpose and setup
2. `package.json` — dependencies reveal the stack (hono vs express vs nextjs, prisma vs drizzle, etc.)
3. Top-level `src/` or `app/` directory listing

At this point you should be able to answer: monorepo or single package? Backend framework? Frontend framework? ORM or raw SQL?

---

## Phase 2 — Entry Points

Goal: find where requests come in and where the app starts.

**Backend:**
```bash
# Common entry point locations
cat src/index.ts 2>/dev/null
cat src/app.ts 2>/dev/null
cat src/server.ts 2>/dev/null
cat src/main.ts 2>/dev/null
# For monorepos
ls apps/backend/src/ 2>/dev/null
```

Look for: HTTP server setup, middleware registration, route mounting. This gives you the complete list of route prefixes.

**Frontend:**
```bash
# Next.js App Router
ls src/app/ 2>/dev/null
ls app/ 2>/dev/null
# Next.js Pages Router
ls pages/ 2>/dev/null
# Vite / SPA
cat src/main.tsx 2>/dev/null
cat src/App.tsx 2>/dev/null
```

---

## Phase 3 — Routes and API Surface

Goal: enumerate all API endpoints.

```bash
# Find all route files
find src -name "*.route.ts" -o -name "*.routes.ts" -o -name "route.ts" | grep -v node_modules
find src/routes -type f 2>/dev/null
find src/api -type f 2>/dev/null

# For Next.js API routes
find src/app/api -name "route.ts" 2>/dev/null
find pages/api -type f 2>/dev/null
```

Read each route file. For each handler, note:
- HTTP method + path
- Auth required? (middleware check)
- What service/repository it calls
- What it returns

---

## Phase 4 — Data Model

Goal: understand the database schema and relations.

```bash
# Raw SQL migrations
find . -name "*.sql" | grep -v node_modules
find src/db -type f 2>/dev/null
find migrations -type f 2>/dev/null
find drizzle -type f 2>/dev/null

# Prisma
cat prisma/schema.prisma 2>/dev/null

# Drizzle
find src -name "schema.ts" | grep -v node_modules
```

Read schema files first, then migration files in chronological order. Build a mental model of tables → relations → key constraints.

---

## Phase 5 — Service / Business Logic Layer

Goal: understand module boundaries and what each service owns.

```bash
find src/services -type f 2>/dev/null
find src/lib -type f 2>/dev/null
find src/core -type f 2>/dev/null
```

For each service file, skim the exported functions:
- What does it accept?
- What does it return?
- Does it call the DB directly or through a repository?
- Does it call external APIs?

This is where you identify candidate invariants (e.g. "services never touch DB directly").

---

## Phase 6 — Frontend Component Tree

Goal: map pages to components to files.

```bash
# App Router pages
find src/app -name "page.tsx" | grep -v node_modules
find src/app -name "layout.tsx" | grep -v node_modules

# Components
ls src/components/ 2>/dev/null
find src/components -name "*.tsx" | grep -v node_modules | head -30
```

For each page, note:
- URL route (derived from directory name in App Router)
- Main component rendered
- Key child components

---

## Phase 7 — Auth and Middleware

Goal: understand the security boundary.

```bash
find src -name "middleware.ts" -o -name "auth.ts" -o -name "*.middleware.ts" | grep -v node_modules
cat src/middleware.ts 2>/dev/null
```

Identify: how is auth enforced? JWT / session? Which routes are public vs protected? This is almost always an invariant.

---

## Phase 8 — Background Jobs and External Integrations

```bash
find src -name "*.job.ts" -o -path "*/jobs/*" | grep -v node_modules
find src -name "*.cron.ts" | grep -v node_modules
grep -r "cron\|schedule\|queue\|webhook" src --include="*.ts" -l 2>/dev/null | grep -v node_modules | head -10
```

Also check for external API clients (Stripe, SendGrid, S3, etc.) — these are often failure points and worth noting in ARCH.md.

---

## Phase 9 — Existing Docs

Read last to cross-check your understanding against what was previously documented.

```bash
ls docs/ 2>/dev/null
cat docs/*.md 2>/dev/null
cat ARCHITECTURE.md 2>/dev/null
cat ARCH.md 2>/dev/null
```

If docs exist, treat them as a hint, not ground truth — the code is the source of truth.

---

## Synthesis Checklist

Before writing ARCH.md and feat.md, confirm you can answer:

- [ ] What does this project do in one sentence?
- [ ] What is the full list of API route prefixes?
- [ ] What are the DB tables and their primary relations?
- [ ] What are the 3–5 most important service boundaries?
- [ ] What are the obvious invariants (auth, DB access pattern, etc.)?
- [ ] What are the frontend pages and their routes?
- [ ] Are there any background jobs or external integrations?

If any box is unchecked, read more files before writing.

---

## Incremental Exploration (for partial updates)

When updating based on a git diff:

```bash
git diff <last_sha> HEAD --name-only -- src/
```

For each changed file:
1. Read the file
2. Identify which ARCH.md sections it affects (§2 quick ref / §4 modules / §5 data model / §6 flows / §7 invariants)
3. Identify which feat.md rows it affects (added/removed/changed routes or components)
4. Read direct callers and dependents to catch cascade effects

Only write to the sections that are actually affected. Preserve all other content exactly.
