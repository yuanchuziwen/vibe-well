---
name: feature-exec
version: 1.1.0
description: 当用户想要"实现某个阶段"、"开始写代码"、"执行 P1"、"运行第 N 阶段"、"实现已批准的方案"，或 feature-workflow 到达执行阶段时使用此技能。接收 plan.md 的 P<n>（完整流程）或 mini-plan.md（XS 快速通道），驱动完整周期。四种模式按规模选：XS=简化 Mode A（直接读 mini-plan 执行），S=Mode A（主 Agent 直接），M=Mode B（Task subagent 三角色，无 TeamCreate 依赖），L/XL=Mode C（Agent 团队通过 TeamCreate + SendMessage 协调）。
---

# Feature Exec

将一个阶段从已批准的方案驱动到已提交的代码。主 Agent 的职责：
- **启动**：根据 Mode 直接执行 / spawn Task subagent / 创建 TeamCreate 团队
- **监控**：执行期间关注异常，必要时介入
- **响应上报**：subagent 或团队成员遇到超出权限的决策时回复
- **收尾**：commit + 写交付报告 + 清理（Mode C 还要 TeamDelete）

## 章节导航

按需读取，不必每次通读全文：

| 章节 | 何时读 |
|---|---|
| § 所需输入 | 启动前确认前置依赖 |
| § 预检：依赖检查 | 任何模式启动前 |
| § 环境检测 + 模式选择 | 主 SKILL 已检测过则跳过 |
| § Mode A | 选 A 时 |
| § Mode B | 选 B 时（包含 Task subagent 启动模板） |
| § Mode C | 选 C 时（包含 Step 0~4 启动步骤、上报机制、交付报告） |
| § Hook 时点 | 检查 4 个生命周期 hook（A/B/C 共用表）|
| § 恢复协议 | 中断后恢复时（state.json 状态机 + 9 stage 重启表）|
| § 参考文件 | 末尾外链 |

## 所需输入

**完整流程**（来自 plan.md 的 P<n>）：

| 输入 | 来源 | 是否必须 |
|---|---|---|
| `discuss-result.md` | `design/<date>/discuss-result.md` | ✅ |
| `plan.md` | `design/<date>/plan.md` | ✅ |
| `ARCH.md` | 项目根目录 | ✅ |
| `feat.md` | 项目根目录 | ✅ |
| `Pn.md` — 阶段方案 | `design/<date>/P<n>.md` | Mode A 由主 Agent 写，Mode C 由 dev 成员写 |

**XS 快速通道**（来自主 SKILL Stage 0.5）：

| 输入 | 来源 | 是否必须 |
|---|---|---|
| `mini-plan.md` | `design/<date>/mini-plan.md` | ✅ |
| `ARCH.md` | 项目根目录 | ✅（用于命名/边界对齐，不要求修改） |
| `feat.md` | 项目根目录 | ✅ |

XS 模式下不读 `discuss-result.md` / `plan.md` / `Pn.md`；mini-plan.md 自包含。

---

## 预检：依赖检查

**先判断本次调用是完整流程还是 XS 模式**：

- 调用方传入 `mode: xs` 或显式提供 `mini-plan.md` 路径 → 走 **XS 预检**
- 否则 → 走 **完整流程预检**

### XS 预检

```bash
# 仅需项目文档 + mini-plan.md
ls ARCH.md feat.md CLAUDE.md 2>/dev/null
ls "<design_root>/mini-plan.md" 2>/dev/null
```

| 缺失内容 | 处理方式 |
|---|---|
| `ARCH.md` 或 `feat.md` | 调用 `project-onboard`（全量扫描），然后继续 |
| `mini-plan.md` | 由调用方（主 SKILL Stage 0.5）按 `../vibe-well/references/xs-mini-plan-template.md` 生成，本技能不自动创建 |

跳过 `discuss-result.md` / `plan.md` / `Pn.md` 检查——XS 模式不读这些。

### 完整流程预检

选择模式前，按顺序验证所有依赖是否存在：

```bash
# 1. 项目文档
ls ARCH.md feat.md CLAUDE.md 2>/dev/null

# 2. 需求文档——查找最新的 design/ 目录
ls -dt design/*/ 2>/dev/null | head -1 | xargs -I{} ls {}discuss-result.md {}plan.md 2>/dev/null
```

| 缺失内容 | 处理方式 |
|---|---|
| `ARCH.md` 或 `feat.md` | 调用 `project-onboard`（全量扫描），然后继续 |
| `discuss-result.md` 或 `plan.md` | 调用 `requirement` 技能，然后继续 |
| `Pn.md` | Mode C 由 dev 成员在团队内编写（见下方内部流程）；Mode A 由主 Agent 使用 `../requirement/references/plan-template.md` 编写并与用户确认 |

如果存在多个 `design/` 目录，向用户报告并询问使用哪个。

如果多项缺失，按顺序解决：project-onboard → requirement → 继续。

在说明模式前用一行报告结果：
- "所有依赖已就绪——继续执行 P\<n\>"
- "ARCH.md 缺失——先运行 project-onboard"
- "未找到需求文档——先运行 requirement 技能"

---

## 环境检测

如果顶层 SKILL 已经检测过，跳过。否则：

```
1) 检查 TeamCreate 是否存在 → 决定 Mode C 是否可用
2) 检查 Task / generalPurpose subagent 调用能力 → 决定 Mode B 是否可用
   (Cursor / Claude Code 默认都有 Task 工具)
3) 都没有 → 仅 Mode A
```

如果环境不支持 Mode C 但阶段规模为 M/L/XL，告知用户：
> "当前环境不支持 Agent 团队（TeamCreate 不可用）。建议：
> - **Mode B**（推荐）：单 Agent 三角色，用一次性 subagent 替代团队——M 级直接可用，L/XL 牺牲并行性后也可用
> - **Mode A**：拆分为更小子任务后逐个执行
> - 或切换到支持 TeamCreate 的 Claude Code 环境继续 Mode C"

---

## 模式选择

| 阶段规模 | 首选 | 备选（首选不可用时） |
|---|---|---|
| S / XS — 单模块、无新 schema、< 1 天 | **A** — 主 Agent 直接执行 | **A** |
| M — 跨 2~3 模块、有可测逻辑 | **B** — 单 Agent 三角色 | **A**（拆子任务串行）|
| L / XL — 大规模、需要并行轨道 | **C** — Agent 团队 | **B** → **A** |

在开始前说明模式和阶段名称。

---

## Mode A — 主 Agent 直接执行

0. **🪝 Hook**：触发 `on_phase_started`（如已配置）
1. 读取 ARCH.md（§2、§4、§5、§7、§8）、feat.md、discuss-result.md、Pn.md
2. 读取所有待修改文件，包括调用方和依赖方
3. **根据 Pn.md `Verification strategy` 选择实现路径**（详见 `../requirement/references/plan-template.md` 末尾「Verification Strategy 选择指南」）：

   | 策略 | 步骤 3 行为 | 步骤 4 行为 |
   |---|---|---|
   | `unit-tdd` / `curl-acceptance` | **TDD Red**：为验收清单每项写失败单测，运行确认全失败（粘贴输出）| **TDD Green**：写最小实现让所有单测通过（粘贴通过输出）|
   | `visual-snapshot` | 编写可断言的快照/结构测试，运行确认失败 | 写实现让快照测试通过 + 人工目视确认 UI |
   | `visual-diff` | 不写自动化测试，记录变更前后的截图 baseline | 实现样式变更，输出对比截图 |
   | `manual` | 不写自动化测试，准备验收脚本/手动步骤清单 | 实现 + 按清单逐项执行 |
   | `regression-suite` | 跑现有测试套件得到 baseline（粘贴绿色输出）| 完成重构后再跑一次确认全绿、行为不变 |

4. 见上表步骤 4 行为
5. 自审：
   - 代码与 Pn.md 一致、无不变量被破坏、无死代码、命名符合 §8
   - 少实现检查：Pn.md §Acceptance checklist 每项都有对应实现，**且按所选策略有相应验证证据**
   - 过度实现检查：无 Scope — out 以外的改动
6. 针对 Pn.md 验收清单运行浏览器 / curl / 手工验收——粘贴证明（截图、curl 输出、命令日志均可）
7. **调试铁律**（步骤 4 或 6 有失败时）：先找根因再修复，每次只改一个变量；3 次修复仍失败则停下来质疑设计，向用户上报
8. 更新 feat.md、ARCH.md（仅受影响的章节）、test_case.md（追加该阶段的 TC，标注验证策略）
9. Git commit：代码 + 测试（如有）+ feat.md + ARCH.md + test_case.md 一次性提交
10. **🪝 Hook**：commit 完成后触发 `on_phase_committed`（如已配置）；交付报告输出后触发 `on_phase_delivered`

**XS 快速通道下的简化**：当 feature-exec 被传入 `mode: xs` 标记时（来自主 SKILL Stage 0.5）：
- 输入文件用 `mini-plan.md` 替代 `Pn.md` + `discuss-result.md`
- 步骤 8 通常只需更新 feat.md（mini-plan 默认不动 ARCH）
- 仍按 `verification_strategy` 选择步骤 3/4 路径

交付物：commit SHA + 验证证据（按策略类型输出对应证据）。

---

## Mode B — 单 Agent 三角色（B-orchestrate）

主 Agent 是 orchestrator/team-lead；每轮用 `Task` 工具 spawn 一次性 subagent 扮演 dev / reviewer / tester；角色独立性靠 subagent 上下文隔离实现，无需 TeamCreate。

### 输入与输出
- 输入：与 Mode C 相同（`Pn.md` 由 dev subagent 在第一轮写）
- 输出：与 Mode C 相同（commit SHA + 验证证据 + 文档更新）

### 角色卡
所有 dev / reviewer / tester 的行为约定写在 `../vibe-well/references/role-cards/{dev,reviewer,tester}.md`。

**spawn 时只引用角色卡路径，不要把内容复制进 prompt**——节省 token，也避免不同轮次角色定义漂移。

### 跨轮上下文持久化目录

第一次进入 Mode B 时创建：
```bash
mkdir -p <design_root>/.reviews
```

每轮 reviewer 的输出落到 `<design_root>/.reviews/<task>-round<n>.md`：
- `<task>` 取值：`pn`、`tdd-red`、`tdd-green`、`code`、`tc`
- `<n>` 从 1 开始

下一轮 reviewer subagent 启动时，主 Agent 在 prompt 头加一行：
> "前 N 轮反馈在 `<design_root>/.reviews/<task>-round1.md` 至 `round<n-1>.md`，先读取再审本轮。"

### 内部流程

```
主 Agent: 初始化 .reviews/ + 读上下文
    │
    ▼
[Round 1] Task(dev, 任务=write-pn)
    → dev subagent 写 <design_root>/Pn.md
    → 返回 "DONE，产出 Pn.md"
    │
    ▼
[Round 1] Task(reviewer, 任务=review-pn)
    → reviewer 读 Pn.md + ARCH + discuss-result
    → 返回 "PASS" 或 "ISSUES（清单）"
    → 主 Agent 把返回内容存到 .reviews/pn-round1.md
    │
    ▼ ISSUES → 重复 Task(dev, 修订 Pn) → Task(reviewer, round 2，传入 round1 反馈)
    ▼ 直至 PASS
    │
    ▼ Pn.md 批准——并行启动两条轨道（用 batch Task 调用同时 spawn）
    │
    ├── Dev 轨道:
    │     [3a] Task(dev, 任务=tdd-red) → 失败单测
    │            → Task(reviewer, 任务=review-tdd-red, 传入 .reviews/tdd-red-round*)
    │            → 多轮直至 PASS
    │     [3b] Task(dev, 任务=tdd-green) → 实现 + 通过输出
    │            → Task(reviewer, 任务=review-code, 传入 .reviews/code-round*)
    │            → 多轮直至 PASS
    │
    └── Tester 轨道:
          Task(tester, 任务=write-tc) → TC 列表
            → Task(reviewer, 任务=review-tc, 传入 .reviews/tc-round*)
            → 多轮直至 PASS
    │
    ▼ 两轨道均 PASS
    │
[Execute] Task(tester, 任务=execute-tc)
    → 返回 PASS/FAIL/SKIP 报告
    │
    ▼ 有 FAIL → Task(dev, 任务=fix-failure, 传入 tester 报告)
              → Task(tester, 任务=rerun-failures)
              → 循环（3 次仍失败 → 主 Agent 上报用户）
    ▼ 全部 PASS
    │
[Wrap-up] 主 Agent 自己更新 ARCH/feat/test_case + commit
    → 或 spawn Task(dev, 任务=final-commit) 完成（推荐：让 dev 做 commit 保持纪律）
    │
    ▼ 主 Agent 输出交付报告 + 清理 .reviews/ 是否保留由用户决定
```

### Task subagent 启动模板

**前提**：先按 Mode C 「Step 0 — 写 exec-context.json」 在 `<design_root>/.vibe-well/exec-context.json` 写入本阶段上下文（Mode B 也用这份 schema，`team.members` 仅 `lead` 字段必填，其余可省略；`mode` 字段填 `"B"`）。

每次 spawn 用以下精简格式：

```
你扮演 <role>（dev / reviewer / tester），任务标签：<task-label>

# 第一步：读上下文
1. 读取 <design_root>/.vibe-well/exec-context.json — 这里有 project_root、design_root、phase、verification_strategy 等所有变量
2. 读取角色卡 <project_root>/.../role-cards/<role>.md — 你的行为指南，必须严格遵守

# 仅 reviewer 任务：附上历史反馈
前几轮反馈历史在 <design_root>/.reviews/<task>-round*.md
（如果是 round 1，跳过这条）

# 任务具体说明
<本轮任务描述，如 "为验收清单 1~3 写失败单测，运行 context.test_command 确认全失败">

# 输出要求
按角色卡末尾的"输出格式"严格返回。结束时把关键产出文件路径列在输出末尾。
```

**为什么这样写**：原本要在 prompt 里展开 8 个变量（project_root、design_root、phase_num、phase_name、verification_strategy、dev_name、reviewer_name、tester_name），现在主 Agent 只需在 context.json 写一次，所有 subagent 共享同一份真相。出错率显著降低。

### 主 Agent 在 Mode B 期间的职责

- **不写代码**——只 orchestrate 和转交
- **解析 subagent 输出**：把每个 subagent 的"产出文件"信息记录到内存，下次 spawn 时引用其路径
- **写 .reviews/ 历史**：reviewer 输出收到后立刻 append 到 `<design_root>/.reviews/<task>-round<n>.md`
- **轮次管理**：每个任务的 `<n>` 独立计数（pn 的 round 数和 tdd-red 的 round 数互不影响）
- **失败兜底**：subagent 输出格式错误（缺字段、不按角色卡格式）→ 主 Agent 重试一次；连续 2 次失败 → 上报用户
- **范围控制**：dev 返回 `NEEDS-SCOPE-EXPANSION` 时，主 Agent 转给用户决策，**不要自行授权**

### 并行的实现

Phase 3a/3b 中 dev 轨道和 tester 轨道可并行——用单个 message 中包含两个 Task tool call 实现。例如：

```
[在同一回合中]
Task(subagent_type="generalPurpose", description="dev 写失败单测", prompt=...)
Task(subagent_type="generalPurpose", description="tester 写 TC", prompt=...)
```

两者结果会被批量返回。**reviewer 不能并行**：每次 review 必须读上一轮历史，需要串行。

### 关键约束（Mode B 专属）

- **角色卡是契约**：subagent 偏离角色卡（如 reviewer 自己改代码）→ 主 Agent 拒绝输出，重 spawn 一次
- **reviewer 跨轮历史是必读**：第 N 轮 reviewer prompt 必须显式引用 `.reviews/<task>-round<1..n-1>.md`
- **subagent 不感知"团队"**：不要在 prompt 里说"另一个成员"——它们是一次性的，只知道自己的任务
- **commit 是主 Agent 或 final-commit dev 任务的职责**——**不要**让普通任务的 dev subagent 自己 commit（会导致提交散乱）
- **`<design_root>/.reviews/` 的处理**：阶段交付后是否保留由用户决定（默认保留以便审计；用户嫌占空间可手动删）

---

## Mode C — Agent 团队

### 内部流程

```
主 Agent 创建团队 + spawn 3 名成员
    │
    ▼ dev 成员根据 plan.md + discuss-result.md 编写 Pn.md
    │
reviewer → dev：审查 Pn.md（多轮直至 PASS）
    │
    ▼ Pn.md 批准
    │
    ├─── Dev 轨道 ──────────────────────────────────────────────┐
    │    Phase 3a：按 Pn.md verification_strategy 准备验证基线   │
    │      · unit-tdd / curl-acceptance：写失败单测 (TDD Red)    │
    │      · visual-snapshot：写快照/结构测试，确认失败          │
    │      · visual-diff / manual：跳过 3a，记录截图或手工步骤   │
    │      · regression-suite：跑现有测试得到全绿 baseline       │
    │    dev → reviewer：3a 产出审查请求（附测试或基线证据）     │
    │    reviewer ↔ dev：审查测试/基线设计（多轮直至 PASS）      │
    │                                                           │
    │    Phase 3b：dev 写最小实现 + 运行验证                     │
    │      · TDD 类策略：让 3a 单测全绿（粘贴通过输出）          │
    │      · visual-* / manual：实现 + 按对应方式验证            │
    │      · regression-suite：实现后重跑测试确认仍全绿          │
    │    dev → reviewer：代码质量审查请求（附验证证据）          │
    │    reviewer ↔ dev：审查实现质量（多轮直至 PASS）           │
    │                                                           │
    └─── Tester 轨道 ────────────────────────────────────────────┤
         reviewer → tester：发送 Pn.md 路径，通知开始           │
         tester 编写功能测试用例                                │
         tester → reviewer：测试用例审查请求                    │
         reviewer ↔ tester：审查（多轮直至 PASS）               │
                                                                │
    ◄──────────── 两条轨道均 PASS ─────────────────────────────┘
    │
reviewer → tester：通知可以执行了
tester 执行测试用例（每个 TC 完成后向 team-lead 发进度心跳）
    │
失败 → tester → dev：报告失败 → dev 找根因修复 → tester 重跑
（3 次修复仍失败 → dev → team-lead 上报，等待用户决策）
    │
全部 PASS
    │
tester → dev：发送 test_case.md 格式化行
dev 更新 ARCH.md + feat.md + test_case.md + git commit
dev → 主 Agent（team-lead）：交付报告
    │
主 Agent shutdown 团队 + TeamDelete
```

---

### 启动步骤

**立即读取 `../vibe-well/references/subagent-prompts.md` 获取各成员的完整启动提示。**

**Step 0 — 写 exec-context.json**（推荐，让所有成员只需读一行 JSON 即可获得全部上下文）

在 spawn 任何成员之前，先在 `<design_root>/.vibe-well/exec-context.json` 写入本阶段的执行上下文：

```json
{
  "schema_version": 1,
  "phase":   { "num": 1, "name": "auth-layer", "size": "M" },
  "mode":    "C",
  "paths": {
    "project_root": "/Users/me/proj",
    "design_root":  "/Users/me/proj/design/20260509-user-auth"
  },
  "branch":   "feat/user-auth/p1-auth-layer",
  "worktree": "/Users/me/proj-worktrees/p1-auth-layer",
  "team": {
    "team_name": "p1-auth-layer",
    "members":   { "dev": "dev", "reviewer": "reviewer", "tester": "tester", "lead": "team-lead" }
  },
  "verification_strategy": "unit-tdd",
  "test_command":  "pnpm test"
}
```

**Schema 字段说明**：

| 字段 | 含义 | 谁填 |
|---|---|---|
| `schema_version` | 1（首版本，未来兼容性预留）| 主 Agent |
| `phase.{num,name,size}` | 来自 plan.md 阶段块 | 主 Agent |
| `mode` | A / B / C / xs | Stage 3 决策结果 |
| `paths.project_root` | worktree 或仓库根（见 Step 1）| Stage 2 决策结果 |
| `paths.design_root` | 设计文件目录（始终原仓库）| Stage 1 输出 |
| `branch` | 当前分支名 | Stage 2 决策结果 |
| `worktree` | worktree 路径（不使用时为 null）| Stage 2 决策结果 |
| `team.team_name` | Mode C 用；其它模式为 null | 本步骤决定 |
| `team.members` | Mode C 用；Mode B 仅 `lead` 字段；Mode A null | 本步骤决定 |
| `verification_strategy` | 来自 Pn.md（Mode C 由 dev 写入后由主 Agent 回填）| Pn.md 批准后回填 |
| `test_command` | 项目脚本 / `defaults.test_command` | 启动检测 |

**为什么需要这个文件**：
- 成员 prompt 不再需要 8 个手工替换的占位符——只需引用 context 文件路径
- 跨阶段 / 跨 mode 共用同一份 schema，恢复协议（state.json）和 context 解耦
- Pn.md 批准后回填 `verification_strategy` 给 tester / reviewer 直接读，不再靠 prompt 串传

下面的 Step 1~4 是各字段值的来源说明——**实际操作中按 Step 0 一次性写好**。

**Step 1 — 确定路径变量**

```bash
# <project_root> — 成员读写代码的目录
# - 不使用 worktree：
git rev-parse --show-toplevel
# - 功能级 worktree：使用 Stage 2 中创建的 worktree 路径
# - 阶段级 worktree：使用本阶段的 worktree 路径（如尚未创建，现在执行）：
# git worktree add <path> -b <branch>

# <design_root> — 设计文件的绝对路径（始终在原始仓库，不在 worktree 内）
# 例如 /Users/you/project/design/20260505-user-auth
```

- `<project_root>` — 成员读写代码和项目文档（ARCH.md、feat.md、test_case.md）的绝对路径
- `<design_root>` — 本功能设计文件的绝对路径，用于读取 discuss-result.md、plan.md、Pn.md
- `<phase_num>` — 阶段编号（如 `1`、`2`）
- `<phase_name>` — 阶段名称（来自 plan.md）

**Step 2 — 确定成员名**

默认：`dev` / `reviewer` / `tester` / `team-lead`。串行执行（一次只跑一个 phase 团队）时直接用默认值即可。

需要自定义的场景：
- **并行多个 phase 团队**：虽然 `team_name` 已经区分（如 `p2-xxx` / `p4-yyy`），默认成员名在各自 team 内不会冲突；但如果你或用户希望看消息时一眼区分来源，可以把成员名带上 phase 后缀（`dev-p2` / `dev-p4`）。
- **用户显式指定**：遵循用户偏好。

**关键约束**：同一个 team 里，所有成员 prompt 引用的名字必须一致。`<dev_name>` / `<reviewer_name>` / `<tester_name>` / `<team_lead_name>` 这四个变量在启动 prompt 里贯穿全文，spawn 时必须统一替换成实际使用的名字，否则成员之间会找不到彼此。

确定下来后记录：
- `<dev_name>` = "dev" 或 "dev-p<n>"
- `<reviewer_name>` = 同上
- `<tester_name>` = 同上
- `<team_lead_name>` = "team-lead"（主 Agent 默认名，一般不改）

**Step 3 — 创建团队**

```
TeamCreate(
  team_name: "p<n>-<phase-slug>",
  description: "P<n> · <phase name> 实现团队"
)
```

团队创建后，主 Agent 的名字默认是 `team-lead`。成员用 `SendMessage(to="<team_lead_name>", ...)` 上报给主 Agent。

**Step 4 — 同时 spawn 三名成员**

每个成员带 `team_name` 和 `name` 参数，`name` 对应 Step 2 里确定的值：

```
Agent(subagent_type="general-purpose", team_name="p<n>-<phase-slug>", name="<reviewer_name>",
      prompt=<reviewer 提示，替换 project_root/design_root/phase_num/phase_name + 四个成员名变量>)

Agent(subagent_type="general-purpose", team_name="p<n>-<phase-slug>", name="<dev_name>",
      prompt=<dev 提示，替换同上>)

Agent(subagent_type="general-purpose", team_name="p<n>-<phase-slug>", name="<tester_name>",
      prompt=<tester 提示，替换同上>)
```

成员间通过名字寻址：`<dev_name>` / `<reviewer_name>` / `<tester_name>` / `<team_lead_name>`。

**Step 5 — 进入监控状态**

Spawn 完成后，**结束当前轮次**。成员消息会作为新的对话轮自动送达；主 Agent 不需要轮询或调用任何"等待"工具。

---

### 主 Agent 监控与介入

#### 成员 idle 的正常情况

成员每一轮结束后自动 idle，这是正常的——他们在等待其他成员的消息。单次 idle 不代表有问题。

#### 成员发来消息的三种类型

主 Agent 收到成员 SendMessage 时需要区分：

| 类型 | 识别特征 | 主 Agent 处理 |
|---|---|---|
| **进度心跳** | `<tester_name>` 执行 TC 期间的消息，格式开头是"进度：TC-\<n\>/\<总数\>" | **只记录，不回复、不打扰用户**；把心跳视为"tester 仍在推进"的信号，重置"长时间无响应"计时 |
| **交付报告** | `<dev_name>` 发来的 ✅ 交付报告（见后文格式）| 按"交付报告"段落处理：标记 plan.md、shutdown 团队、向用户展示摘要 |
| **上报** | 需要产品/范围决策的消息（见"上报协议"）| 转述给用户，获得决策后 SendMessage 回复成员 |

#### 需要介入的异常信号

| 信号 | 描述 |
|---|---|
| **长时间无响应** | 其他成员已发消息但该成员 idle 不响应；或全体 idle 但阶段未完成。**注意**：tester 执行阶段应有进度心跳，若心跳也停了超过合理时间才算异常 |
| **异常终止** | 收到 `teammate_terminated` 通知但阶段未完成 |
| **循环/死锁** | 两成员互相等对方；或某条消息格式不符合协议导致对方无法推进 |
| **上下文丢失** | 成员回复显示它忘记自己的角色（开始自行写方案、跳过审查等）|

#### 介入的两层处理

**第一层 — 轻度纠偏**

用 SendMessage 向相关成员发明确指令，例如：
- "你忘记在回复中加 PASS / ISSUES 标签，请补上"
- "你应该先等 reviewer 通知再开始测试执行，请先等待"
- "请用 SendMessage 联系队友，不要只在文字输出里说"

成员上下文仍在，通常一两条消息就能拉回轨道。

**第二层 — 向用户上报决策**

轻度纠偏 2~3 次无效，或出现以下任一情况时，**停下来向用户汇报并请用户决策**，不要擅自处理：
- 成员异常终止（`teammate_terminated`），进行中的工作状态只在其上下文里
- 成员完全无响应（发消息也不回）
- 多成员同时陷入问题
- 成员坚持走错方向

向用户报告时要提供：
- 哪个成员、当前在流程哪一步
- 已发生的关键消息摘要
- 可见产出（新文件、未提交的修改、临时 commit）
- 推测原因

让用户在几个选项之间选择：
- 继续等 / 主动重发消息
- 从现状恢复重试（保留现有工作，重新 spawn 替换成员时手动把现状同步给新成员）
- 降级为 Mode A 由主 Agent 接管
- 回到某个检查点重开
- 放弃本阶段

**禁止行为**：
- **不要**未经用户同意就重新 spawn 替换成员——新成员没有进行中的上下文（代码草稿、已审轮次、已跑 TC），硬替换会导致返工或遗漏
- **不要**直接接手成员的角色继续推进——这会破坏 Mode C 的角色边界，且主 Agent 没有该成员积累的上下文

---

### 上报协议

成员仅在遇到超出自身权限的决策时才用 SendMessage 向 `team-lead` 发送上报消息：

| 情况 | 上报人 | 消息示例 |
|---|---|---|
| Pn.md / plan.md 存在歧义且无法从 discuss-result.md 解决 | dev | "§X 存在歧义：[选项 A] vs [选项 B]，请确认。" |
| 实现需要改动 Pn.md 范围外的内容 | dev | "需要改动范围外的 [X]，原因：[Y]，是否继续？" |
| 代码与 discuss-result.md 的决策冲突 | reviewer | "代码偏离了 D\<n\>：[冲突内容]，是否有意为之？" |
| 测试失败涉及产品级问题 | tester | "验收项 [X] 的失败方式可能需要产品决策：[Y]" |

**不上报**：范围内的技术选择、符合 ARCH.md §8 的命名、修复 reviewer 提出的阻塞问题。

**主 Agent 回复上报**：收到上报后，主 Agent 将问题转述给用户、获得用户的决策，然后用 SendMessage 回复给上报的成员。成员收到回复后按决策继续。回复时必须带明确的行动指示（"按选项 A 实现"、"扩大范围到 X 已授权"、"接受当前偏离"），不要只回一段描述。

---

### 交付报告

阶段完成时，dev 用 SendMessage 向 `team-lead` 发送：

```
✅ P<n> · <name> 已交付

Commit：<sha>
变更文件：N
  - <路径>：<一行说明>

Pn.md 审查：PASS（<N> 轮）
代码 + 单元测试审查：PASS（<N> 轮）
测试用例审查：PASS（<N> 轮）
浏览器 / 功能测试：
  - <测试用例>：PASS / FAIL / SKIP（<原因>）

ARCH.md 更新：§X、§Y / 无需更新
feat.md 更新：<变更行数> / 无需更新
test_case.md 更新：TC-<n> ~ TC-<m> 新增，TC-<x> 废弃 / 无变化

偏离方案：无 / <列表及原因>
引入技术债：TD-X <描述> / 无
```

主 Agent 收到交付后：
1. 在 `plan.md` 中将该阶段标记为 `[x]`
2. 用 SendMessage 向每位成员发送 `shutdown_request`（消息体：`{"type": "shutdown_request", "reason": "阶段已交付"}`），然后直接调用 `TeamDelete`
3. 向用户展示交付摘要 + 引入的技术债

注意：shutdown_approved 由系统自动处理，主 Agent 无需等待回复再调用 TeamDelete。

---

## 关键约束

- **Dev 不审查自己的工作** — reviewer 始终是独立的成员
- **Pn.md 由 dev 编写，reviewer 审查** — 不要让同一个人既写又审
- **Reviewer 上下文跨轮次保持** — 修复后由同一个 reviewer 重新审查，不换人
- **TDD 顺序按 verification_strategy 决定** — `unit-tdd` / `curl-acceptance` / `visual-snapshot` 必须先写失败测试再写实现，不得先实现后补测试；`visual-diff` / `manual` / `regression-suite` 按对应路径走
- **reviewer 必须先确认 verification_strategy 选择合理**——审 Pn.md 时把策略合理性作为首要审查项
- **验证证据必须附上** — dev 每次请求 review 时必须附测试命令输出 / 截图 / 手工验收日志，不能只口头说"通过了"
- **基线产出是进入代码质量审查的门槛** — Phase 3a 产出未经 reviewer PASS，不得进入 Phase 3b
- **两条轨道均 PASS 后才能执行测试** — tester 在收到 reviewer 的明确通知前不执行
- **提交是交付的一部分** — 代码 + ARCH.md + feat.md + test_case.md 全部提交后阶段才算完成
- **未经授权不得扩大范围** — Pn.md §范围——包含 以外的任何改动都需先获授权
- **调试先找根因** — 修复失败前先复现、追根因；3 次修复失败向 team-lead 上报，不继续猜测
- **成员间通信必须用 SendMessage** — 成员的文字输出对队友不可见
- **主 Agent 的名字是 `team-lead`** — 成员上报时 `to="team-lead"`

---

## Hook 时点（三种模式共用）

合并后的 `~/.vibe-well/config.yaml` `hooks.<event>` 字段在以下 4 个时点检查并执行（schema + 执行规则见 `../vibe-well/references/config-schema.md`）：

| 时点 | Mode A | Mode B | Mode C | XS |
|---|---|---|---|---|
| `on_phase_started` | Step 0（在读 ARCH 之前） | 主 Agent 写完 exec-context.json 后、第一个 Task() 之前 | TeamCreate 之后、spawn 成员之前 | feature-exec 入口处 |
| `on_pn_approved` | Pn.md 自审通过后 | reviewer subagent 返回 PASS 后 | reviewer 把 Pn.md 标记为 PASS 后 | 不触发（XS 用 mini-plan，无 Pn）|
| `on_phase_committed` | 步骤 9 完成后 | dev subagent `final-commit` 返回后 | dev → team-lead 交付前 | commit 完成后 |
| `on_phase_delivered` | 步骤 10 输出交付报告后 | 主 Agent 输出交付报告后 | TeamDelete 之前 | 输出交付报告后 |

**Hook 失败处理**：
- `mode: warn`（默认）→ 打印警告继续
- `mode: block` → 询问用户：忽略 / 修复后重跑 / 中止阶段

**Hook 不阻断 commit 本身**——`on_phase_committed` 在 commit 之后才跑，避免与 git 自身 pre-commit hook 冲突。需要"提交前检查"的请用 git pre-commit hook，不要塞到 vibe-well。

---

## 恢复协议

任何 Mode（A / B / C / XS）执行过程中如果中断（Claude 重启、用户 Ctrl+C、subagent 失踪、context 爆炸），下次回到这个阶段时按本节恢复。

### state.json 状态机

每个阶段的执行状态持久化到 `<design_root>/.vibe-well/state.json`：

```json
{
  "schema_version": 1,
  "phase":  { "num": 1, "name": "auth-layer", "size": "M" },
  "mode":   "B",
  "stage":  "phase3a-tdd-red",
  "round":  { "pn": 2, "tdd-red": 1, "code": 0, "tc": 1 },
  "checkpoints": {
    "pn_md_approved":     "2026-05-09T03:50:00Z",
    "tdd_red_approved":   null,
    "tdd_green_approved": null,
    "test_cases_ready":   null,
    "tests_passed":       null,
    "committed":          null,
    "delivered":          null
  },
  "last_message_at":     "2026-05-09T04:12:00Z",
  "last_subagent_task":  "tdd-red round 1",
  "team_name":           null,
  "context_ref":         ".vibe-well/exec-context.json"
}
```

**stage 枚举值**（11 个）：

```text
init  →  write-pn  →  review-pn  →  phase3a-tdd-red  →  phase3b-tdd-green
                          ↓ (并行 tester 轨道)
                          tester-write-tc  →  tester-review-tc
                                                ↓ (两轨道汇合)
                                                tester-execute  →  fixing  →  committing  →  delivered
```

XS 模式只用其中：`init → phase3b-tdd-green`（按 verification_strategy 决定是否走 3a） `→ committing → delivered`。

### 写入时机

| 事件 | 主 Agent 应更新 |
|---|---|
| 阶段启动 | 创建 state.json，stage=`init` |
| Pn.md 写完 | stage=`review-pn`, round.pn 自增 |
| Pn.md PASS | checkpoints.pn_md_approved=now |
| TDD Red PASS | checkpoints.tdd_red_approved=now |
| TDD Green PASS | checkpoints.tdd_green_approved=now |
| TC PASS | checkpoints.test_cases_ready=now |
| 测试全 PASS | checkpoints.tests_passed=now |
| commit 完成 | checkpoints.committed=now |
| 交付报告输出 | checkpoints.delivered=now |
| 任何 subagent 返回 | last_message_at=now, last_subagent_task=... |

### 恢复入口

用户回来后说"继续 P\<n\>"或"恢复"时：

```bash
ls <design_root>/.vibe-well/state.json 2>/dev/null
```

- **不存在** → 没有进行中的阶段，按正常流程重开
- **存在** → 读取，按 `stage` 字段决定从哪步重启：

| state.stage | 恢复动作 |
|---|---|
| `init` | 当作首次启动 |
| `write-pn` 或 `review-pn` | 重新 Task(dev / reviewer)，传入 `<design_root>/Pn.md`（如已存在）和 `.reviews/pn-round*` 历史（Mode B）|
| `phase3a-tdd-red` | 检查测试文件是否已存在；存在则跳到 review，否则重 spawn dev |
| `phase3b-tdd-green` | 同上 |
| `tester-write-tc` / `tester-review-tc` | 重 spawn tester / reviewer，传入历史 |
| `tester-execute` | 重 spawn tester 执行；保留已 PASS 的 TC，只跑剩余 |
| `fixing` | 把上次 tester 报来的 FAIL 详情重新交给 dev |
| `committing` | 检查 git status；如已 commit 但 state 没更新，标记 committed 跳到 delivered |
| `delivered` | 已完成，提示用户该阶段已交付 |

### 恢复时主 Agent 的强制动作

1. **先报告状态**：把 state.json 内容用一行摘要展示给用户（"P1 进度：Pn.md PASS（2 轮），TDD Red 已完成，准备进入 Green"）
2. **询问是否继续**：用户可能想看一下当前产出再决定
3. **同步必要文件**：如果中断时 dev/tester 的产出已落到磁盘但未通过 review，先把这些文件路径列给用户看，再决定是否丢弃重写
4. **从 last_message_at 推断异常**：如果与现在时差 > 24 小时，提醒用户"这个阶段中断很久了，磁盘上的代码可能已被你手动改动，请决定从哪一步重启"

### Mode 特定的恢复细节

- **Mode A**：state 主要靠主 Agent 自己读，没有外部成员
- **Mode B**：恢复时所有 `.reviews/<task>-round*.md` 必须保留——它们是 reviewer 跨轮上下文的唯一载体
- **Mode C**：成员 ID（team_name）在 state.json，但成员上下文在 Claude Code 团队系统里。如果团队已被销毁（典型重启场景），**必须重 spawn 整个团队**，并在新成员的启动 prompt 里加："恢复执行——当前 stage=\<X\>，已通过：\<checkpoints\>。请先读 state.json 决定从哪步开始"

### 清理

阶段交付（`delivered`）后：

- `state.json` 默认保留——便于审计和后续 regression-test 引用
- `.reviews/` 目录默认保留（Mode B）
- 用户可手动 `rm -rf <design_root>/.vibe-well/` 清理

下个阶段启动时**新建独立的** `state.json`，不复用上一阶段的。

---

## 参考文件

- `../vibe-well/references/subagent-prompts.md` — dev、reviewer、tester 的启动提示模板（Mode C 用）
- `../vibe-well/references/role-cards/{dev,reviewer,tester}.md` — 角色卡（Mode B 用，一次性 subagent 启动时引用）
- `../requirement/references/plan-template.md` — plan.md 格式（Pn.md 也参考此模板的"Phase Details"部分；末尾「Verification Strategy 选择指南」是各模式共用）
