---
name: vibe-well
version: 1.0.0
description: This skill should be used when the user wants to "develop a new feature", "start a feature", "implement a feature end-to-end", "kick off feature development", "run the dev workflow", or says things like "let's build X", "I want to add X to the product", "help me implement X from scratch". Guides the complete feature development lifecycle: requirement discussion → phased execution → delivered, documented code. Orchestrates sub-skills: project-onboard, requirement, feature-exec.
---

# Feature Workflow

Orchestrates the complete feature development lifecycle — from a raw user request to committed, documented code.

Each stage delegates to a sub-skill. The main agent's job is sequencing, gate-keeping, and communicating with the user. It does not write code or documentation directly.

## Sub-Skills

| Sub-skill | Trigger | Produces |
|---|---|---|
| `project-onboard` | Docs missing or stale | CLAUDE.md, ARCH.md, feat.md |
| `requirement` | Any new feature request | discuss.md, discuss-result.md, plan.md |
| `feature-exec` | Per-phase execution. Can be invoked standalone — auto-triggers project-onboard / requirement if dependencies are missing. | Delivered code, updated ARCH.md + feat.md, git commit |
| *(regression-test)* | Planned — release-time full regression | Test report |

---

## Workflow

```
Stage 0 · Doc Check     → project-onboard if ARCH.md missing or stale
Stage 1 · Requirement   → discuss.md → discuss-result.md → plan.md
                           🛑 Gate: user approves plan.md
Stage 2 · Mode Select   → A or C per phase
                           🛑 Gate: user confirms mode per phase
Stage 3 · Execution     → feature-exec per phase (fire-and-forget for Mode C)
                           🛑 Gate: delivery report + test evidence per phase
```

---

## Stage 0 · Doc Check

```bash
ls CLAUDE.md ARCH.md feat.md 2>/dev/null
```

- **Missing**: read `project-onboard/SKILL.md` and follow its workflow (full scan)
- **Stale** (git SHA diff > 0): read `project-onboard/SKILL.md` and follow its workflow (incremental update)
- **Current**: proceed

---

## Stage 1 · Requirement

**Read `requirement/SKILL.md` now and follow its workflow exactly.**

It will guide you through structured dialogue with the user and produce three documents:

- `discuss.md` — structured decision points (Dn) with options, recommendations, cold-water notes
- `discuss-result.md` — all Dn resolved: technical decisions knowledge base with code skeletons, schemas, tech debt
- `plan.md` — phased P1~Pn delivery plan with scope boundaries, dependencies, acceptance criteria

Output directory: `design/<YYYYMMDD>/` at the project root (or user's preference).

🛑 **Gate 1**: User approves `plan.md` (phase boundaries, sequencing, scope). This is the last point where the user makes product + technical direction decisions. After this, execution is largely autonomous.

---

## Stage 2 · Mode Selection

For each phase in `plan.md`, select execution mode:

| Phase size | Mode |
|---|---|
| S — single module, no new schema, < 1 day | **A** — main agent direct |
| M / L / XL — anything larger | **C** — agent team (fire-and-forget) |

🛑 **Gate 2**: State mode per phase with rationale. Wait for user confirmation.

---

## Stage 3 · Execution

For each phase, **read `feature-exec/SKILL.md` now and follow its workflow exactly.** It owns the complete cycle internally:

```
reviewer ↔ dev: review Pn.md (multi-round until PASS)
    │
    ▼ Pn.md approved — two tracks start in parallel
    │
    ├── Dev track: code + unit tests → reviewer reviews (multi-round)
    └── Tester track: write test cases → reviewer reviews (multi-round)
    │
    ▼ both tracks PASS
    │
tester executes test cases → failures → dev fixes → tester re-runs
    │
    ▼ all PASS
    │
dev updates ARCH.md + feat.md + git commits → delivery report
```

**Parallel execution**: phases with no dependency between them (per `plan.md` dependency graph) can have `feature-exec` invoked simultaneously.

**Main agent during Mode C**: does not intervene unless a member escalates. Escalation happens only for out-of-scope decisions or product-direction conflicts — not for technical implementation choices.

After each delivery report, main agent:
1. Updates `plan.md` status tracking (mark phase `[x]`)
2. Surfaces to user: delivery summary + any tech debt introduced

🛑 **Gate 3**: User signs off each phase delivery (delivery report + test evidence). Blocking test failures must be resolved before sign-off.

---

## Execution Modes

### Mode A — Main Agent Direct
Main agent reads all context, implements, self-reviews, tests, updates docs, commits. Used only for S-size phases with no architectural impact.

### Mode C — Agent Team
Three members (dev, reviewer, tester) declared at kickoff. They coordinate via mailbox. Main agent receives escalations and the final delivery report.

Key rules:
- Dev never reviews their own code
- Reviewer context persists across re-review rounds (same reviewer, not a new one)
- No scope expansion without escalation to main agent
- Commit is part of delivery — phase is not done until ARCH.md + feat.md + code are committed

Full kickoff and member prompt templates: `feature-exec/SKILL.md` + `references/subagent-prompts.md`

---

## Common Failure Modes

- ❌ Starting execution without an approved `plan.md`
- ❌ Main agent writing code directly in Mode C (role confusion)
- ❌ Treating `tsc` / `pnpm build` as self-test evidence
- ❌ Dev reviewing their own code or plan
- ❌ Merging without updating ARCH.md when triggers are met
- ❌ Scope expansion during implementation without escalation
- ❌ Skipping `project-onboard` when ARCH.md is missing — execution without architecture context produces unreliable output

---

## References

- `project-onboard/SKILL.md` — codebase exploration and doc generation
- `requirement/SKILL.md` — requirement discussion and document production
- `feature-exec/SKILL.md` — per-phase execution with agent team
- `references/subagent-prompts.md` — member prompt templates (dev, reviewer, tester)
