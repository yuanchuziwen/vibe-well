# plan.md Template

Full format for the phased delivery plan generated at Stage 4.

---

```markdown
---
generated_at: YYYY-MM-DDTHH:MM:SSZ
git_sha: <sha>
feature: <one-line feature description>
discuss_result_ref: design/<YYYYMMDD>-<feature-slug>/discuss-result.md
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

**Verification strategy**: <unit-tdd | curl-acceptance | visual-snapshot | visual-diff | manual | regression-suite>
<!-- 见本文件末尾「Verification Strategy 选择指南」。默认 unit-tdd。 -->

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

**Per-phase lifecycle**（三种 Mode 共享流程骨架，只在角色独立性上不同）：

```
读上下文 + 写 Pn.md → 审查 Pn.md（多轮 PASS）
  ↓ Pn.md 批准——两条轨道并行
  ├── dev：写测试（按 verification_strategy）→ reviewer 审 → dev 写实现 → reviewer 审
  └── tester：写测试用例 → reviewer 审
  ↓ 两条轨道均 PASS
tester 执行 TC → dev 修失败 → tester 重跑
  ↓ 全 PASS
更新 ARCH.md + feat.md + test_case.md + git commit
  → 交付报告
```

各 Mode 的角色实现：
- **Mode A**（S/XS）：主 Agent 一身担三角，串行跑完
- **Mode B**（M）：主 Agent 充当 orchestrator，每轮用 `Task` 工具 spawn 一次性 subagent 扮演 dev / reviewer / tester
- **Mode C**（L/XL）：主 Agent 用 `TeamCreate` 创建团队，dev/reviewer/tester 是持久成员，互发 `SendMessage` 通信

**Parallel execution**：多个无依赖的阶段可同时执行（Mode C 用独立团队名 `p2-xxx` vs `p4-yyy`；Mode B 用 batch Task 调用；Mode A 串行不并行）。

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

---

## Verification Strategy 选择指南

每个 Phase 必须显式声明 `Verification strategy`。dev/tester 据此决定如何编写和执行测试。reviewer 收到 Pn.md 时应优先确认策略选得是否合理，再走后续审查。

| 策略 | 适用变更类型 | 对应实现 | 是否走 TDD Red/Green |
|---|---|---|---|
| `unit-tdd` | 业务逻辑、算法、纯函数、数据处理、状态机 | 单元测试驱动 | ✅ 强制（默认） |
| `curl-acceptance` | API 端点（含 unit 部分） | 单测 + curl/HTTP 验收脚本 | ✅ 强制 |
| `visual-snapshot` | UI 组件（有可断言的渲染结构） | Storybook / 快照测试 + 人工目视 | ⚠️ 可选（仅断言可测部分） |
| `visual-diff` | 纯样式变更（CSS / 主题 / 响应式调整） | 视觉对比 + 人工目视 | ❌ 跳过 |
| `manual` | 文档站、配置、shell 脚本、运维脚本、原型 | 人工运行验收清单 | ❌ 跳过 |
| `regression-suite` | 重构（不改外部行为） | 跑现有回归测试套件，无新断言 | ❌ 跳过（行为不变） |

### 选择规则

1. **优先选 `unit-tdd`**——只要变更里有可被单元测试覆盖的逻辑，就用它
2. **混合变更**：选最重的那一档（既有逻辑又有 UI → `unit-tdd`，逻辑部分按 TDD 走，UI 部分用人工验证补充）
3. **避免滥用 `manual`**：如果"懒得写测试"是主要动机，重新审视——不写测试通常意味着这块代码后续没人敢动
4. **`regression-suite` 的前提**：现有回归套件覆盖率足够。若覆盖率不足以保证行为不变，应该先补测试（升级到 `unit-tdd`）再重构

### feature-exec 的 TDD Phase 行为

- `unit-tdd` / `curl-acceptance` → 走完整 TDD Red → reviewer 审单测 → Green → reviewer 审实现
- `visual-snapshot` → Phase 3a 写可断言的快照/结构测试 + Phase 3b 写实现，reviewer 审两次；UI 视觉部分由 tester 在执行阶段人工验证
- `visual-diff` / `manual` → 跳过 Phase 3a 的 TDD Red 强制要求，dev 直接进入实现，reviewer 仍审实现质量；tester 按验收清单人工/视觉验证
- `regression-suite` → dev 跑现有测试套件作为 baseline（粘贴输出），实现完成后再跑一次确认全绿；不新增断言，但 reviewer 审"是否真的未改变行为"

无论哪种策略，**测试运行证据（命令 + 输出）** 都必须附在 review 请求里。
