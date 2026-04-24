---
name: project-onboard
version: 1.0.0
description: This skill should be used when the user wants to "onboard a new project", "initialize project docs", "set up ARCH.md", "update project documentation", "sync docs with code", "check if docs are up to date", "regenerate architecture docs", or when any other skill detects that ARCH.md / feat.md are missing or stale. Explores the codebase, creates or updates the three-file documentation system (CLAUDE.md / ARCH.md / feat.md), and tracks doc freshness via git SHA.
---

# Project Onboard

Explores a codebase and creates or updates the project documentation system. Designed to be called at the start of any feature workflow, or manually when docs feel stale.

## Document System

Four files, each with a specific role:

| File | Purpose | Audience |
|---|---|---|
| `CLAUDE.md` | Entry index — pointers + freshness status | Claude Code (auto-loaded) |
| `ARCH.md` | Architecture — modules, data model, invariants, request flows | Claude agents doing planning/review |
| `feat.md` | Feature map — every user-facing feature with entry point + file location | Claude agents doing plan + test |
| `test_case.md` | Functional test cases mirroring feat.md — one TC set per feature | tester member, regression-test skill |

All four live at the project root unless a different convention is already established.

`test_case.md` generation strategy:
- **Full scan (first run)**: generate a bootstrapped version from feat.md — one TC per feature row covering happy path + one key error path. Mark every TC with `bootstrapped` in the Notes column. These are mechanically inferred from routes and components, not verified against product requirements — they are a starting point, not ground truth.
- **Incremental update**: do not add new TCs (that is tester's job). Only maintain consistency with feat.md: deprecate TCs for removed features, update entry points for changed routes.
- **Never overwrite tester-authored TCs** — if a TC row does not have `bootstrapped` in Notes, treat it as human-authored and preserve it exactly.

## Freshness Detection

Each document carries a frontmatter header:

```yaml
---
generated_at: 2026-04-24T10:00:00Z
git_sha: a1b2c3d4
git_dirty: false
dirty_files: []
---
```

To check freshness:
1. Read the `git_sha` from `ARCH.md` frontmatter
2. Run `git diff <sha> HEAD --name-only -- src/` to list changed files since last sync
3. Run `git status --short` to detect uncommitted changes
4. Classify result:

| Status | Meaning |
|---|---|
| ✅ Current | No changes since `git_sha`, working tree clean |
| ⚠️ Stale | Commits exist since `git_sha` — list changed files |
| 🔶 Dirty | Uncommitted changes exist — note but do not block |

Uncommitted changes: record `git_dirty: true` and list files in `dirty_files`. Do NOT force a commit or stash — that is the user's decision.

## Update Strategy

Use incremental update by default. Escalate to full regeneration when needed.

**Incremental** (fast, preserves manual annotations):
- Compute `git diff <last_sha> HEAD --name-only`
- Only re-examine files in the diff
- Update affected sections in ARCH.md and feat.md
- Preserve all other content

**Full regeneration** (escalate automatically when):
- ARCH.md or feat.md does not exist yet
- Diff touches > 20 files
- Diff includes structural files: `package.json`, `pnpm-lock.yaml`, DB schema files, `tsconfig.json`, main entry point
- User explicitly requests it ("regenerate docs", "full update")

## Workflow

### Step 1 — Detect state

```bash
# Check doc existence
ls ARCH.md feat.md CLAUDE.md 2>/dev/null

# If ARCH.md exists, read its git_sha then:
git diff <sha> HEAD --name-only -- src/
git status --short
```

Report the current state to the user in one line:
- "Docs missing — will do full scan"
- "Docs stale — N files changed since last sync, doing incremental update"
- "Docs current — no changes detected ✅"
- "Docs current but working tree is dirty — N uncommitted files noted"

🛑 **Gate**: If docs are current, ask the user whether to proceed. If stale or missing, proceed automatically.

### Step 2 — Check test_case.md (incremental only)

When doing an incremental update, check if any feat.md rows being removed or changed have corresponding TCs in test_case.md:

```bash
# Check if test_case.md exists
ls test_case.md 2>/dev/null
```

If it exists and feat.md rows are being removed: find the matching TCs and move them to the `## Deprecated` section with the current date and reason.

If feat.md rows are being modified (route changed, component renamed): update the TC `Entry Point` column to match.

Do not add new TCs during project-onboard — that is tester's job during feature-exec.

---

### Step 3 — Explore (full scan only)

For full scan, explore the project systematically. Read `references/explore-guide.md` for the full exploration sequence. Key targets:

- Directory structure (top 2 levels)
- `package.json` / `pnpm-workspace.yaml` — tech stack, scripts
- Entry points (`src/index.ts`, `app.ts`, `main.ts`, `server.ts`)
- Route files — all API endpoints
- DB schema files — tables, columns, relations
- Service / repository layer — core business logic boundaries
- Frontend: component tree, page routes
- Existing docs (`README.md`, `docs/`)
- CI/CD config (`.github/`, `Dockerfile`) for deployment context

### Step 3 — Explore (incremental only)

Read only the files in the diff plus their direct callers/dependents. Identify which sections of ARCH.md and feat.md are affected.

### Step 4 — Generate / update documents

Generate in this order: ARCH.md → feat.md → test_case.md → CLAUDE.md (CLAUDE.md is last because it references the others).

Full formats and templates: `references/doc-templates.md`

Key rules:
- ARCH.md: follow the section structure from the template — do not invent new top-level sections
- feat.md: fine-grained, one row per feature, grouped by domain; backend and frontend in separate sections
- test_case.md (full scan): for each row in feat.md, generate one happy-path TC and one key error-path TC. Mark all with `bootstrapped` in Notes. Do not guess edge cases — stay close to what the route/component signature implies.
- test_case.md (incremental): only sync with feat.md changes per Step 2 — do not add new TCs
- CLAUDE.md: thin index only — never duplicate content from ARCH.md or feat.md into CLAUDE.md

### Step 5 — Update frontmatter

After writing, record the current git state in each file's frontmatter:

```bash
git rev-parse HEAD          # for git_sha
git status --short          # for git_dirty + dirty_files
date -u +"%Y-%m-%dT%H:%M:%SZ"   # for generated_at
```

### Step 6 — CLAUDE.md freshness summary

CLAUDE.md's header block must reflect the current status of all docs:

```markdown
> ARCH.md       — synced at a1b2c3d (2026-04-24) ✅
> feat.md       — synced at a1b2c3d (2026-04-24) ✅
> test_case.md  — 24 TCs (18 bootstrapped, 6 verified) · last updated 2026-04-24
```

For test_case.md: count total TC rows, count how many still have `bootstrapped` in Notes vs those without. This gives the user a quick signal of how much of the test coverage has been validated by a real tester.

If `git_dirty: true`:
```markdown
> ARCH.md — synced at a1b2c3d (2026-04-24) 🔶 dirty (3 uncommitted files at generation time)
```

### Step 7 — Report

Tell the user:
- What was created or updated
- How many sections changed (incremental) or total sections written (full)
- Whether `git_dirty` was recorded
- Next step: "Docs are ready. You can now run the feature workflow, or review the docs at ARCH.md / feat.md."

## Manual Invocation

Users can trigger this skill directly at any time:

| Phrase | Behavior |
|---|---|
| "update docs" / "sync docs" | Freshness check → incremental if stale, no-op if current |
| "regenerate docs" / "full update" | Force full regeneration regardless of freshness |
| "check if docs are up to date" | Freshness check only — report status, do not write |
| "initialize project docs" | Full scan (treat as first run) |

## References

- `references/explore-guide.md` — full codebase exploration sequence for new projects
- `references/doc-templates.md` — ARCH.md, feat.md, CLAUDE.md full templates with section descriptions
