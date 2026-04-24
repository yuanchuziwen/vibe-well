---
name: feature-exec
version: 1.0.0
description: This skill should be used when the user wants to "implement a phase", "start coding", "execute P1", "run phase N", "implement the approved plan", or when feature-workflow reaches the execution stage. Takes an approved Pn.md and drives the complete cycle: reviewer approves plan → dev and tester work in parallel → both tracks merge → tester executes → dev commits. Mode A for small changes (main agent direct), Mode C for everything else (agent team via mailbox, fire-and-forget).
---

# Feature Exec

Drives one phase from approved plan to committed code. The main agent's role is kickoff and receiving the delivery report. Everything in between runs inside the agent team via mailbox.

## Inputs Required

| Input | Source | Required? |
|---|---|---|
| `Pn.md` — phase plan | `design/<date>/P<n>.md` | ✅ |
| `discuss-result.md` | `design/<date>/discuss-result.md` | ✅ |
| `ARCH.md` | project root | ✅ |
| `feat.md` | project root | ✅ |

---

## Pre-flight: Dependency Check

Before selecting mode, verify all dependencies are present. Run in order:

```bash
# 1. Project docs
ls ARCH.md feat.md CLAUDE.md 2>/dev/null

# 2. Requirement docs — look for the most recent design/ directory
# If multiple exist, the user should specify which one; default to the newest
ls -dt design/*/ 2>/dev/null | head -1 | xargs -I{} ls {}discuss-result.md {}plan.md 2>/dev/null
```

| Missing | Action |
|---|---|
| `ARCH.md` or `feat.md` | Invoke `project-onboard` (full scan), then continue |
| `discuss-result.md` or `plan.md` | Invoke `requirement` skill, then continue |
| `Pn.md` | Dev member writes it during Mode C internal flow (see below); or main agent writes it for Mode A using `../requirement/references/plan-template.md` and confirms with user before proceeding |

If multiple `design/` directories exist, report them to the user and ask which one to use before proceeding.

If multiple things are missing, resolve in order: project-onboard → requirement → then proceed.

Report the dependency check result in one line before stating the mode:
- "All dependencies present — proceeding with P<n>"
- "ARCH.md missing — running project-onboard first"
- "No requirement docs found — running requirement skill first"

---

## Mode Selection

| Phase size | Mode |
|---|---|
| S — single module, no new schema, < 1 day | **A** — main agent direct |
| M / L / XL | **C** — agent team |

State mode and phase name before starting.

---

## Mode A — Main Agent Direct

1. Read ARCH.md (§2, §4, §5, §7, §8), feat.md, discuss-result.md, Pn.md
2. Read every file to be modified including callers and dependents
3. Implement exactly what Pn.md specifies — no scope expansion
4. Write unit tests; all must pass before proceeding
5. Self-review: code matches Pn.md, no invariant broken, no dead code, naming matches §8
6. Run browser / curl tests against Pn.md acceptance checklist — paste evidence
7. Update feat.md and ARCH.md (affected sections only)
8. Git commit: code + tests + feat.md + ARCH.md in one commit

Deliver: commit SHA + test evidence.

---

## Mode C — Agent Team

### Internal Flow

```
Pn.md ready
    │
reviewer ↔ dev: review Pn.md (multi-round until PASS)
    │
    ▼ Pn.md approved
    │
    ├─── Dev track ──────────────────────────────────┐
    │    dev writes code + unit tests                │
    │    reviewer ↔ dev: review code + tests         │
    │    (multi-round until PASS)                    │
    │                                                │
    └─── Tester track ───────────────────────────────┤
         reviewer sends approved Pn.md to tester     │
         tester writes test cases                    │
         reviewer ↔ tester: review test cases        │
         (multi-round until PASS)                    │
                                                     │
    ◄────────────── both tracks PASS ────────────────┘
    │
tester executes test cases
    │
failures → dev fixes → tester re-runs (multi-round)
    │
all PASS
    │
dev updates ARCH.md + feat.md + test_case.md (from tester) + git commits
    │
dev → main agent: delivery report
```

The two tracks (dev and tester) run in parallel after Pn.md is approved. Reviewer participates in both tracks simultaneously via separate mailbox threads. The merge point is when both code review and test cases review have passed — neither can skip ahead.

---

### Kickoff

Before spawning, read these files so you can paste their contents:
- `ARCH.md`
- `feat.md`
- `design/<date>/discuss-result.md`
- `design/<date>/P<n>.md`

Spawn three members simultaneously. Full kickoff prompt templates: `../references/subagent-prompts.md`

After kickoff, main agent does not participate until receiving the delivery report or an escalation.

---

### Escalation Protocol

Members escalate to the main agent only for decisions outside their authorization:

| Situation | Who escalates | Message |
|---|---|---|
| Pn.md ambiguity not resolvable from discuss-result.md | dev | "Ambiguity in §X: [option A] vs [option B]. Which?" |
| Implementation requires change outside Pn.md scope | dev | "Need to touch [X] outside scope. Reason: [Y]. Proceed?" |
| Code conflicts with a discuss-result.md decision | reviewer | "Code deviates from D<n>: [conflict]. Deliberate?" |
| Test failure indicates a product-level issue | tester | "Acceptance item [X] fails in a way that may need a product decision: [Y]" |

Do **not** escalate: technical choices within scope, naming consistent with §8, fixing reviewer blocking issues.

---

### Delivery Report

Dev sends to main agent when phase is complete:

```
✅ P<n> · <name> delivered

Commit: <sha>
Files changed: N
  - <path>: <one-line summary>

Pn.md review: PASS (<N> rounds)
Code + unit tests review: PASS (<N> rounds)
Test cases review: PASS (<N> rounds)
Browser / functional test:
  - <test case>: PASS / FAIL / SKIP (<reason>)

ARCH.md updated: §X, §Y / no update needed
feat.md updated: <rows changed> / no update needed
test_case.md updated: TC-<n> ~ TC-<m> added, TC-<x> deprecated / no change

Deviations from plan: none / <list with reason>
Tech debt introduced: TD-X <description> / none
```

Main agent marks the phase `[x]` in `plan.md` and surfaces the summary to the user.

---

## Key Constraints

- **Dev never reviews their own work** — reviewer is always a separate member
- **Reviewer context persists across rounds** — same reviewer re-reviews after fixes, not a new one
- **Unit tests are dev's delivery gate** — code without passing unit tests is not ready for code review
- **Test execution only starts after both tracks PASS** — tester does not execute until code review and test case review are both done
- **Commit is part of delivery** — phase is not complete until code + ARCH.md + feat.md + test_case.md are in one commit
- **No scope expansion without escalation** — any change outside Pn.md §Scope — in must be authorized first

---

## References

- `../references/subagent-prompts.md` — kickoff prompt templates for dev, reviewer, tester
- `../requirement/references/plan-template.md` — Pn.md format
