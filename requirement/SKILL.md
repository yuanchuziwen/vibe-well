---
name: requirement
version: 1.0.0
description: This skill should be used when the user wants to "clarify a requirement", "discuss a feature", "write a spec", "start requirement discussion", "analyze this request", "help me think through this feature", or says things like "I want to add X", "let's figure out how to build X", "I have a bug / refactor idea", or when the feature-workflow skill reaches Stage 1 (requirement discussion). Guides requirement clarification through structured dialogue, producing three documents: discuss.md (decision points), discuss-result.md (technical decisions knowledge base), and plan.md (phased delivery plan).
---

# Requirement

Guides requirement clarification from a raw user request to a set of three documents that any dev-member subagent can pick up cold and execute without further clarification.

## Output Documents

| Document | Purpose | Written by |
|---|---|---|
| `discuss.md` | Structured decision points (D1, D2…) with options + recommendations + cold-water analysis | This skill |
| `discuss-result.md` | Technical decisions knowledge base — all Dn resolved, with rationale, code skeletons, tech debt | This skill |
| `plan.md` | Phased delivery plan (P1~Pn) with dependencies, scope boundaries, acceptance criteria | This skill |

All three live in a directory you agree with the user — default: `design/<YYYYMMDD>/` relative to the project root.

**Not produced here**: P1.md, P2.md … task-level implementation documents. Those are written by dev-member subagents during execution.

---

## When to Invoke This Skill

- User brings: a feature idea, a bug report, a refactor proposal, a draft document, a rough spec, or just a problem statement in conversation
- User explicitly triggers: "clarify this requirement", "let's discuss", "write a spec for X"
- Called from feature-workflow Stage 1

---

## Pre-Flight

Before starting, check for project documentation:

```bash
ls CLAUDE.md ARCH.md feat.md 2>/dev/null
```

- If **ARCH.md exists**: read §1 (overview), §4 (module boundaries), §5 (data model), §7 (invariants), §8 (terminology). Use these to anchor technical options.
- If **ARCH.md is missing**: inform the user — "No ARCH.md found. You may want to run project-onboard first for richer technical context. Proceeding with what's visible in the codebase."
- Read `feat.md` if the request touches an existing feature.

---

## Workflow

### Stage 1 · Dialogue

Accept any input:
- A single sentence: "I want to add user login"
- A multi-paragraph draft doc (like `refactor-0421.md`)
- A conversation thread
- A bug description

**Your job**: extract the intent, surface what's unknown, and propose structured decision points proactively. The user should mostly be picking options, not writing prose.

**Clarification style**:
- Ask at most 2–3 short questions before generating discuss.md
- Never ask for things you can derive from the codebase or ARCH.md
- If you can propose a recommendation, do — don't make the user figure it out

**When to invoke huashu-design**: if the request involves any visible UI change (new page, new component, modified layout), say so explicitly and offer to invoke `huashu-design` for a prototype before finalizing visual decisions. UI decisions made without a prototype often need to be revisited. This is optional — user can skip.

🛑 **Gate 1**: Before writing discuss.md, briefly state what you understood and what the major open questions are. Wait for the user to confirm your understanding or correct it.

---

### Stage 2 · Write discuss.md

Generate `discuss.md` proactively — do not ask the user to write it.

**Format**: read `references/discuss-template.md` now for the full format.

Key rules:
- Each Dn covers exactly one decision that cannot be resolved without user input
- Never put a decision in discuss.md that you can resolve from the codebase, ARCH.md, or industry best practice
- Each Dn must have: options (at least 2), a recommendation with rationale, and a "cold water" note (what could go wrong)
- Options should be concrete, not abstract ("use Prisma" not "use an ORM")
- Number decisions D1, D2… in dependency order (decisions that unlock others come first)
- End with a "Total cost estimate" section: rough scope (S/M/L/XL) per decision cluster

After writing discuss.md, present it and ask the user to choose options for each Dn.

🛑 **Gate 2**: User reviews discuss.md and picks options. Resolve all Dn before proceeding. If the user changes their mind on an earlier decision after seeing a later one, update the earlier decision and note the cascade.

---

### Stage 3 · Write discuss-result.md

Once all Dn are resolved, generate `discuss-result.md`.

**Format**: read `references/discuss-result-template.md` now for the full format.

Key rules:
- One section per Dn, labeled with the decision number and title
- For each Dn: state the chosen option, the rationale, implementation notes, code skeletons (if helpful), and tech debt items introduced
- Code skeletons should be complete enough that a dev-member can implement without further design decisions — type signatures, table schemas, key algorithm pseudocode
- Tech debt items get unique IDs (TD-X) and go into a consolidated §Tech Debt section at the end
- This document is the single source of truth for all subagents — it must be self-contained. A dev-member reading this + ARCH.md should have everything they need.

🛑 **Gate 3**: Present discuss-result.md. User reviews for correctness and completeness. Minor clarifications can be handled inline; major disagreements require going back to Gate 2.

---

### Stage 4 · Write plan.md

Generate `plan.md` as a phased delivery plan.

**Format**: read `references/plan-template.md` now for the full format.

Key rules:
- Phases (P1, P2…) should be at the "meaningful delivery unit" level — e.g., "auth layer", "document editor", "AI integration". Not at the file or API level.
- Each phase must be independently releasable or at minimum independently testable
- Each phase includes: goal (one sentence), scope (in/out), deliverables, acceptance checklist, estimated size (S/M/L/XL), and risk notes
- Include a dependency graph — which phases can run in parallel, which are hard serial
- Do NOT write implementation details — that is dev-member's job. plan.md stops at what, not how.
- Capture the minimum viable path: which phases form the smallest releasable slice?

🛑 **Gate 4**: Present plan.md. User reviews phase boundaries and sequencing. Adjust if needed.

---

### Stage 5 · Finalize and Report

After Gate 4, update the frontmatter in all three documents:

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"   # generated_at
git rev-parse HEAD 2>/dev/null   # git_sha (empty if not a git repo)
```

Report to the user:
- Three documents written: `design/<date>/discuss.md`, `discuss-result.md`, `plan.md`
- Decisions resolved: D1~Dn (list the titles)
- Phases planned: P1~Pn (list with size estimates)
- Any UI design work deferred to huashu-design
- Next step: "Requirements are ready. Run feature-workflow to start implementation, or invoke feature-exec directly on P1."

---

## Working Style Notes

**Proactive, not Socratic**: generate structured proposals, don't ask the user to fill in blanks.

**Technically grounded**: anchor every recommendation in the existing codebase (ARCH.md §4, §5, §7). If you can't, say so explicitly.

**Cold-water honest**: surface risks early. A decision that looks easy may have a hidden migration cost, a concurrency edge case, or an invariant conflict. Say it in the discuss.md option analysis, not after code is written.

**Scope discipline**: the plan.md should represent what the user asked for, not what you think would be nice to have. Unsolicited scope creep goes to a "Future considerations" section, clearly labeled as out of scope.

---

## Manual Invocation

| Phrase | Behavior |
|---|---|
| "discuss this feature" / "clarify this requirement" | Full Stage 1–5 flow |
| "regenerate discuss.md" / "redo the decision list" | Restart from Stage 2 |
| "write discuss-result" / "finalize decisions" | Skip to Stage 3 (assumes Gate 2 done in conversation) |
| "write the plan" / "generate plan.md" | Skip to Stage 4 (assumes Stages 2–3 done) |

---

## References

- `references/discuss-template.md` — full discuss.md format with example Dn structure
- `references/discuss-result-template.md` — full discuss-result.md format with code skeleton examples
- `references/plan-template.md` — full plan.md format with phase structure and dependency graph
