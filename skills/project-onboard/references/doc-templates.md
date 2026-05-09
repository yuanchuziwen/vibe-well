# Document Templates

Full templates for ARCH.md, feat.md, and CLAUDE.md.

---

## ARCH.md

```markdown
---
generated_at: YYYY-MM-DDTHH:MM:SSZ
git_sha: <sha>
git_dirty: false
dirty_files: []
---

# Architecture: <Project Name>

> **Scope**: All changes to code / DB / frontend / deployment config must reference this document.
> Only documentation-only changes (*.md, comments, typos) are exempt.

---

## §1 Project Overview

One paragraph: what this project does, who uses it, and the core technical approach.

**Tech stack**: [e.g. Next.js 14 / Hono / PostgreSQL / Bedrock]
**Deployment**: [e.g. Vercel (frontend) + Railway (backend) + Supabase (DB)]

---

## §2 Quick Reference — File Locations

Use this table to find the file responsible for any area. Update whenever a file's responsibility changes.

| Area | File(s) | Notes |
|---|---|---|
| API entry point | `src/index.ts` | |
| Auth routes | `src/routes/auth.ts` | |
| DB schema | `src/db/schema.sql` | |
| Env config | `.env.example` | |
| Frontend pages | `src/app/` | Next.js App Router |
| Background jobs | `src/jobs/` | |

---

## §3 Repository Structure

Top-level directory map. Update when directories are added or removed.

```
<project-root>/
├── src/
│   ├── routes/       # API route handlers
│   ├── services/     # Business logic (no direct DB access)
│   ├── db/           # DB client, schema, migrations, repositories
│   ├── jobs/         # Background/cron jobs
│   └── lib/          # Shared utilities
├── frontend/         # or app/ for Next.js
├── docs/
├── .github/
└── ...
```

---

## §4 Core Module Boundaries

For each module: what it owns, what it accepts as input, what it returns, and what it must not do.

### <Module Name> (e.g. AuthService)

- **Owns**: [what data / responsibility]
- **Input**: [parameters / events it receives]
- **Output**: [what it returns / emits]
- **Must not**: [explicit constraints — e.g. "must not access DB directly, goes through UserRepository"]
- **File**: `src/services/auth.ts`

[Repeat for each core module]

---

## §5 Data Model

### Tables

| Table | Key Columns | Notes |
|---|---|---|
| `users` | `id uuid PK`, `email text UNIQUE`, `created_at` | |
| `projects` | `id uuid PK`, `user_id FK → users`, `name text`, `created_at` | |

### Relations

- `projects.user_id` → `users.id` (many-to-one)
- [add relations]

### Migrations

| Migration | Date | What it does |
|---|---|---|
| `001_init.sql` | 2026-01-01 | Initial schema |
| [add entries] | | |

---

## §6 Typical Data Flows

Describe the most important request paths end-to-end. These are the flows most likely to break when you change something.

### <Flow Name> (e.g. Create Project)

```
POST /api/projects
  → auth middleware (verifies JWT)
  → ProjectsRoute.create()
    → ProjectService.create(userId, name)
      → ProjectRepository.insert({ userId, name })
        → DB: INSERT INTO projects ...
      ← returns Project
    ← returns Project
  ← 201 { id, name, created_at }
```

[Add one flow per major feature]

---

## §7 Key Invariants

Rules that must never be broken. Reference these by number (e.g. "INV-3") in plans and reviews.

| ID | Invariant | Why |
|---|---|---|
| INV-1 | Every API route except `/auth/*` requires a valid JWT | Security boundary |
| INV-2 | Services never access the DB directly — always through repositories | Testability + separation |
| INV-3 | All DB mutations go through migrations, never via raw DDL in code | Reproducibility |
| [add] | | |

---

## §8 Terminology

Key terms used consistently across the codebase and docs.

| Term | Definition |
|---|---|
| Project | A user's workspace containing documents and chat history |
| Document | A single file or text the user uploads for analysis |
| [add] | |

---

## §9 Tech Debt

Track known debt here. Mark resolved items inline.

| ID | Description | Severity | Added |
|---|---|---|---|
| TD-1 | [description] | high/med/low | YYYY-MM-DD |
| [add] | | | |
```

---

## feat.md

```markdown
---
generated_at: YYYY-MM-DDTHH:MM:SSZ
git_sha: <sha>
git_dirty: false
dirty_files: []
---

# Feature Map: <Project Name>

Fine-grained map of every user-facing feature. Used by plan and test stages.
Format: feature name / entry point (route or component) / primary file:function / notes.

---

## Backend Features

### <Domain> (e.g. Auth)

| Feature | HTTP Method + Route | File:Function | Notes |
|---|---|---|---|
| User login | `POST /api/auth/login` | `src/routes/auth.ts:login` | Returns JWT |
| User register | `POST /api/auth/register` | `src/routes/auth.ts:register` | Sends verification email |
| Refresh token | `POST /api/auth/refresh` | `src/routes/auth.ts:refresh` | |

### <Domain> (e.g. Projects)

| Feature | HTTP Method + Route | File:Function | Notes |
|---|---|---|---|
| Create project | `POST /api/projects` | `src/routes/projects.ts:create` | Requires auth |
| List projects | `GET /api/projects` | `src/routes/projects.ts:list` | Paginated |
| Get project | `GET /api/projects/:id` | `src/routes/projects.ts:getById` | |
| Delete project | `DELETE /api/projects/:id` | `src/routes/projects.ts:remove` | Cascades to documents |

[Add one section per domain]

---

## Frontend Features

### <Page/Area> (e.g. Dashboard)

| Feature | Component | File | Page Route | Notes |
|---|---|---|---|---|
| View project list | `ProjectList` | `src/app/dashboard/page.tsx` | `/dashboard` | |
| Create project modal | `CreateProjectModal` | `src/components/CreateProjectModal.tsx` | (modal) | |

### <Page/Area> (e.g. Project Detail)

| Feature | Component | File | Page Route | Notes |
|---|---|---|---|---|
| View documents | `DocumentList` | `src/app/projects/[id]/page.tsx` | `/projects/:id` | |
| Upload document | `UploadButton` | `src/components/UploadButton.tsx` | (inline) | |

---

## Background Jobs / Scheduled Tasks

| Job | Trigger | File:Function | Notes |
|---|---|---|---|
| [job name] | cron / event | `src/jobs/xxx.ts:run` | |
```

---

## test_case.md

```markdown
---
generated_at: YYYY-MM-DDTHH:MM:SSZ
git_sha: <sha>
---

# Test Cases: <Project Name>

Functional-level test cases for every user-facing feature. Mirrors feat.md structure.
One row per TC. Steps are brief — enough to reproduce, not a full script.
Maintained by tester: append after each phase, mark deprecated cases inline.

---

## Backend

### <Domain> (e.g. Auth)

| TC | Feature | Type | Entry Point | Steps (brief) | Expected |
|---|---|---|---|---|---|
| TC-001 | User login — happy path | feature | `POST /api/auth/login` | POST with valid email+password | 200 + JWT in response |
| TC-002 | User login — wrong password | feature | `POST /api/auth/login` | POST with wrong password | 401 `invalid_credentials` |
| TC-003 | User login — missing field | feature | `POST /api/auth/login` | POST with no password field | 400 `validation_error` |
| TC-004 | Refresh token — valid | feature | `POST /api/auth/refresh` | POST with valid refresh token | 200 + new JWT |

### <Domain> (e.g. Projects)

| TC | Feature | Type | Entry Point | Steps (brief) | Expected |
|---|---|---|---|---|---|
| TC-010 | Create project | feature | `POST /api/projects` | POST with name, auth header | 201 + project object |
| TC-011 | Create project — unauthenticated | regression | `POST /api/projects` | POST without auth header | 401 |

---

## Frontend

### <Page/Area> (e.g. Dashboard)

| TC | Feature | Type | Route | Viewport | Steps (brief) | Expected |
|---|---|---|---|---|---|---|
| TC-020 | View project list | feature | `/dashboard` | both | Open dashboard | Project cards rendered |
| TC-021 | Create project modal | feature | `/dashboard` | 1440×900 | Click "New Project" | Modal opens, form visible |
| TC-022 | Create project modal | regression | `/dashboard` | 375×812 | Click "New Project" | Modal opens, not clipped |

---

## Deprecated

Cases marked deprecated are kept for traceability. Do not execute.

| TC | Deprecated in | Reason |
|---|---|---|
| TC-005 | P3 (2026-05-01) | Login flow replaced by SSO — TC-050 supersedes |
```

---

## CLAUDE.md

```markdown
# <Project Name>

> ARCH.md       — synced at <sha> (<date>) [✅ / ⚠️ stale / 🔶 dirty]
> feat.md       — synced at <sha> (<date>) [✅ / ⚠️ stale / 🔶 dirty]
> test_case.md  — <N> TCs (<M> bootstrapped, <K> verified) · last updated <date>

---

## What is this project?

One sentence.

## Key documents

| Document | Purpose |
|---|---|
| [`ARCH.md`](./ARCH.md) | Architecture — modules, data model, invariants, flows |
| [`feat.md`](./feat.md) | Feature map — all features with entry points and file locations |
| [`test_case.md`](./test_case.md) | Functional test cases — one TC set per feature, bootstrapped + tester-verified |
| [`docs/dev-workflow.md`](./docs/dev-workflow.md) | Development workflow rules (source of truth) |

## Before touching any code

1. Read this file ✓
2. Read `ARCH.md` — especially §7 Invariants and the section covering your area
3. Read `feat.md` if your change affects existing features
4. Read `test_case.md` if your change may affect existing test coverage
5. Read the files you plan to modify, including callers and dependents

## Quick orientation

- **API entry**: [file path]
- **Frontend entry**: [file path]
- **DB schema**: [file path]
- **Env vars**: see `.env.example`
- **Run locally**: `[command]`
- **Run tests**: `[command]`
```
