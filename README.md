# vibe-well

A Claude Code skill for the complete feature development lifecycle — from a raw idea to committed, documented code.

[中文文档](./README-zh-CN.md)

---

## What it does

vibe-well orchestrates three sub-skills that cover the full development loop:

| Sub-skill | What it does |
|---|---|
| `project-onboard` | Explores a codebase and generates ARCH.md, feat.md, test_case.md, CLAUDE.md |
| `requirement` | Turns a user request into discuss.md → discuss-result.md → plan.md |
| `feature-exec` | Executes one phase: dev + reviewer + tester work in parallel via agent team |
| *(regression-test)* | Planned — full regression at release time |

---

## How it works

```
Stage 0 · Doc Check     → project-onboard if ARCH.md missing or stale
Stage 1 · Requirement   → structured discussion → 3 documents
                           🛑 user approves plan.md
Stage 2 · Worktree      → confirm whether to use worktree (feature-level / phase-level / none)
                           🛑 user confirms strategy and branch name
Stage 3 · Mode Select   → env check → A (main agent) or C (agent team) per phase
                           🛑 user confirms
Stage 4 · Execution     → feature-exec per phase
                           🛑 user signs off delivery report
```

In **Mode C** (the default for M / L / XL phases when `TeamCreate` is available), three agents coordinate through `TeamCreate` + `SendMessage`:

```
dev writes Pn.md → reviewer reviews
    ↓ approved — parallel tracks
dev: code + unit tests → reviewer reviews
tester: write test cases → reviewer reviews
    ↓ both PASS
tester executes → dev fixes → tester re-runs
    ↓ all PASS
dev updates ARCH.md + feat.md + test_case.md + commits → delivery report
```

The main agent monitors for anomalies (member unresponsive, abnormal termination, context loss) and intervenes when needed, but does not take over member roles.

---

## Environment requirements

- **Mode C (agent team)** requires `TeamCreate` tool, available in Claude Code with experimental features enabled.
- **Mode A (main agent direct)** works in any environment (Cursor, older Claude, standard API). vibe-well auto-detects `TeamCreate` at Stage 3 and falls back to Mode A when unavailable.

---

## Install

```bash
# Global install (available across all projects)
npx reskill install github:yuanchuziwen/vibe-well -g

# Project-local install
npx reskill install github:yuanchuziwen/vibe-well
```

---

## Document system

vibe-well maintains four living documents at your project root:

| File | Purpose |
|---|---|
| `ARCH.md` | Architecture — modules, data model, invariants, request flows |
| `feat.md` | Feature map — every user-facing feature with entry point and file location |
| `test_case.md` | Functional test cases — bootstrapped from feat.md, refined by tester over time |
| `CLAUDE.md` | Thin index — auto-loaded by Claude Code, points to the above three |

All documents carry a git SHA in frontmatter. `project-onboard` detects staleness and updates incrementally by default.

Per-feature design files live in `design/<YYYYMMDD>/`:
- `discuss.md`, `discuss-result.md`, `plan.md` — written by the `requirement` skill
- `P1.md`, `P2.md`, … — written by the `dev` member during `feature-exec`

---

## Sub-skill structure

```
vibe-well/
├── SKILL.md                    ← top-level orchestrator
├── project-onboard/
│   ├── SKILL.md
│   └── references/             ← explore-guide.md, doc-templates.md
├── requirement/
│   ├── SKILL.md
│   └── references/             ← discuss-template.md, discuss-result-template.md, plan-template.md
├── feature-exec/
│   └── SKILL.md
└── references/
    └── subagent-prompts.md     ← kickoff prompt templates for dev / reviewer / tester
```

---

## License

MIT
