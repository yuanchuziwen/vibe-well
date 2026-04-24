# vibe-well

A Claude Code skill for the complete feature development lifecycle — from a raw idea to committed, documented code.

[中文文档](./README-zh-CN.md)

---

## What it does

vibe-well orchestrates four sub-skills that cover the full development loop:

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
Stage 2 · Mode Select   → A (main agent) or C (agent team) per phase
                           🛑 user confirms
Stage 3 · Execution     → feature-exec per phase (fire-and-forget for Mode C)
                           🛑 user signs off delivery report
```

In **Mode C** (the default for anything non-trivial), three agents coordinate via mailbox with no main-agent involvement:

```
reviewer ↔ dev: review Pn.md
    ↓ approved — parallel tracks
dev: code + unit tests → reviewer reviews
tester: write test cases → reviewer reviews
    ↓ both PASS
tester executes → dev fixes → tester re-runs
    ↓ all PASS
dev updates ARCH.md + feat.md + test_case.md + commits
```

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

---

## Sub-skill structure

```
vibe-well/
├── SKILL.md                  ← top-level orchestrator
├── project-onboard/
│   └── SKILL.md
├── requirement/
│   └── SKILL.md
├── feature-exec/
│   └── SKILL.md
└── references/
    └── subagent-prompts.md   ← kickoff prompt templates for dev / reviewer / tester
```

---

## License

MIT
