# plan.md Template

Full format for the phased delivery plan generated at Stage 4.

---

```markdown
---
generated_at: YYYY-MM-DDTHH:MM:SSZ
git_sha: <sha>
feature: <one-line feature description>
discuss_result_ref: design/<date>/discuss-result.md
---

# Delivery Plan: <Feature Name>

> **Context**: All technical decisions are in `discuss-result.md`.
> This document answers: what phases, in what order, with what scope boundaries.
>
> **This document does not contain implementation details.**
> Dev-member writes P1.md, P2.md… with file-level plans after reading this + discuss-result.md.

---

## Overview

| Phase | Name | Decisions covered | Size | Risk | Depends on |
|---|---|---|---|---|---|
| P1 | [name] | D1, D2 | S | low | — |
| P2 | [name] | D3 | M | medium | P1 |
| P3 | [name] | D4, D5 | L | high | P1, P2 |
| P4 | [name] | D6 | M | low | P1 |

**Size key**: S ≈ 0.5–1 day · M ≈ 1–2.5 days · L ≈ 3–5 days · XL ≈ 1+ week
**Total estimate**: ~ [range]

---

## Dependency Graph

```
          P1
          │
    ┌─────┼─────┐
    │           │
    P2          P4      ← P2 and P4 can run in parallel after P1
    │
    P3
```

**Hard dependencies** (phase cannot start until prerequisite is complete):
- P2 requires P1: [reason]
- P3 requires P1 and P2: [reason]

**Can run in parallel**:
- P2 and P4 after P1: [reason they don't conflict]

---

## Minimum Viable Path

The smallest releasable slice:

> P1 → P2 → P3

Delivers: [what capability this enables]
Deferred: P4 (reason: [lower priority / can be added later without rework])

---

## Phase Details

### P1 · <Name> (S)

**Goal**: [one sentence — what capability or foundation this phase delivers]

**Scope — in**:
- [concrete deliverable]
- [concrete deliverable]

**Scope — out** (explicitly deferred):
- [thing not done here] — deferred to P2 / future
- [thing not done here]

**Deliverables**:
- [artifact: e.g., "auth middleware", "users table + migration", "login/logout endpoints"]

**Acceptance checklist** (functional, not compilational):
- [ ] [mechanically verifiable check — e.g., "Unauthenticated request to /api/projects returns 401"]
- [ ] [check]
- [ ] [check]

**Risk notes**:
- [risk if any — migration safety, external dependency, concurrency concern]

---

### P2 · <Name> (M)

**Goal**: [one sentence]

**Scope — in**:
- [deliverable]

**Scope — out**:
- [deferred item]

**Deliverables**:
- [artifact]

**Acceptance checklist**:
- [ ] [check]
- [ ] [check]

**Risk notes**:
- [risk]

---

### P3 · <Name> (L)

[same structure]

---

### P4 · <Name> (M)

[same structure]

---

## Execution Notes

**Per-phase lifecycle** (applies to every phase in Mode C):

```
主 Agent 创建团队 + spawn dev / reviewer / tester
  → dev 写 P<n>.md（本模板 Phase Details 部分）
  → reviewer 审查 P<n>.md（多轮直至 PASS）
  → Pn.md 批准后两条轨道并行：
      - dev：写代码 + 单元测试 → reviewer 审查代码
      - tester：写测试用例 → reviewer 审查测试用例
  → 两条轨道都 PASS
  → tester 执行测试 → dev 修复失败 → tester 重跑
  → dev 更新 ARCH.md + feat.md + test_case.md + git commit
  → dev 向主 Agent 发交付报告 → 主 Agent shutdown 团队
```

Mode A（S 级小阶段）：由主 Agent 直接按上述流程串行执行，不 spawn 团队。

**Parallel execution**: 多个无依赖的阶段可同时 spawn 各自的团队，每个团队独立运行。注意团队名需要区分（如 `p2-xxx` 和 `p4-yyy`）。

---

## Status Tracking

Update after each phase completes:

- [ ] P1 · <name>
- [ ] P2 · <name>
- [ ] P3 · <name>
- [ ] P4 · <name>
```

---

## Writing Notes

### Phase Granularity

**Right granularity** — a "meaningful delivery unit":
- Delivers an end-to-end capability (even if minimal): e.g., "user can log in and their data is associated"
- Can be independently tested by a human
- Represents 0.5–5 days of work
- Roughly aligns with a PR or a deploy

**Too fine** (API-level — dev-member's job, not yours):
- "Add the `POST /api/users` endpoint"
- "Write the UserRepository.insert method"
- "Create the users table migration"

These belong inside P1.md written by dev-member, not in plan.md.

**Too coarse** (hard to review or test as a unit):
- "Build the whole auth system"
- "Complete the backend"

### Scope Discipline

Every phase needs an explicit "Scope — out" section. This prevents scope creep from bleeding into the wrong phase and makes it possible to assess whether a phase is complete.

Common things to explicitly defer:
- Error handling edge cases beyond the happy path
- Permission / access control (often comes in a later phase)
- Performance optimization
- Cross-device / multi-user synchronization
- Analytics / usage tracking
- Admin tooling

### Acceptance Checklist Rules

Checks must be **mechanically verifiable** — something a person can do in under 2 minutes with curl, a browser, or a SQL query. Not:
- ~~"Code quality is good"~~ (not verifiable)
- ~~"Tests pass"~~ (compilational, not functional)
- ~~"Feature works correctly"~~ (too vague)

Yes:
- "`curl -X POST /api/login` with valid credentials returns 200 with a session cookie"
- "Unauthenticated request to `/api/projects` returns 401"
- "`SELECT COUNT(*) FROM users` shows the test user was created"
- "Browser at 1440×900: clicking 'Create Project' opens the modal without console errors"
