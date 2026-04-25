---
name: vibe-well
version: 1.0.0
description: 当用户想要"开发新功能"、"开始一个功能"、"端到端实现功能"、"启动开发流程"，或者说"我们来构建 X"、"我想给产品加 X"、"帮我从头实现 X"时使用此技能。引导完整的功能开发生命周期：需求讨论 → 分阶段执行 → 交付有文档的代码。编排子技能：project-onboard、requirement、feature-exec。
---

# 功能工作流

编排完整的功能开发生命周期——从用户原始需求到已提交的有文档的代码。

每个阶段委托给一个子技能。主 Agent 的职责是排序、把关和与用户沟通，不直接写代码或文档。

## 子技能

| 子技能 | 触发条件 | 产出 |
|---|---|---|
| `project-onboard` | 文档缺失或过期 | CLAUDE.md、ARCH.md、feat.md、test_case.md |
| `requirement` | 任何新功能请求 | discuss.md、discuss-result.md、plan.md |
| `feature-exec` | 按阶段执行。可单独调用——依赖缺失时自动触发 project-onboard / requirement。 | 交付代码、更新 ARCH.md + feat.md + test_case.md、git commit |
| *(regression-test)* | 计划中——发布时全量回归 | 测试报告 |

---

## 工作流概览

```
Stage 0 · 文档检查      → ARCH.md 缺失或过期时运行 project-onboard
Stage 1 · 需求讨论      → discuss.md → discuss-result.md → plan.md
                           🛑 Gate：用户批准 plan.md
Stage 2 · Worktree 配置 → 确认是否使用 worktree 及粒度
                           🛑 Gate：用户确认 worktree 策略和分支名
Stage 3 · 模式选择      → 环境检测 → 每个阶段选择模式 A 或 C
                           🛑 Gate：用户确认每个阶段的模式
Stage 4 · 执行          → 每个阶段运行 feature-exec
                           🛑 Gate：每个阶段的交付报告 + 测试证明
```

---

## Stage 0 · 文档检查

```bash
ls CLAUDE.md ARCH.md feat.md 2>/dev/null
```

- **缺失**：立即读取 `project-onboard/SKILL.md` 并按其工作流执行（全量扫描）
- **过期**（git SHA diff > 0）：读取 `project-onboard/SKILL.md` 并按其工作流执行（增量更新）
- **最新**：继续

---

## Stage 1 · 需求讨论

**立即读取 `requirement/SKILL.md` 并严格按其工作流执行。**

该技能会引导与用户进行结构化对话，产出三份文档：

- `discuss.md` — 结构化决策点（Dn），含选项、推荐、风险分析
- `discuss-result.md` — 所有 Dn 已决策：技术决策知识库，含代码骨架、数据模式、技术债
- `plan.md` — 分阶段 P1~Pn 交付计划，含范围边界、依赖关系、验收标准

输出目录：项目根目录下的 `design/<YYYYMMDD>/`（或用户指定路径）。记录此目录的绝对路径，后续 feature-exec 启动成员时需要用到。

🛑 **Gate 1**：用户批准 `plan.md`（阶段边界、顺序、范围）。这是用户做产品和技术方向决策的最后时机，此后执行基本自主进行。

---

## Stage 2 · Worktree 配置

`plan.md` 批准后，主动询问用户：

> "要为这个功能使用 git worktree 吗？选项：
> - **不使用 worktree** — 直接在当前分支工作（最简单；适合 S 级单阶段工作）
> - **功能级 worktree** — 整个功能使用一个 worktree，所有阶段提交到同一分支（适合串行阶段或每个功能一个 PR）
> - **阶段级 worktree** — 每个阶段单独一个 worktree，各有独立分支和 PR（适合并行阶段或 M/L/XL 规模的隔离需求）
>
> 请选择？"

根据回答操作：

| 选择 | 操作 |
|---|---|
| 不使用 worktree | 记录 `worktree: none`。全程 `<project_root>` = 原始仓库根目录。 |
| 功能级 | 现在执行一次 `git worktree add <path> -b <branch>`。记录路径和分支。所有阶段的 `<project_root>` = worktree 路径。 |
| 阶段级 | 暂不创建 worktree。每个阶段启动时执行 `git worktree add <path> -b <branch>`。每个阶段单独记录路径和分支。该阶段的 `<project_root>` = 该阶段的 worktree 路径。 |

分支命名规范（建议，允许用户覆盖）：
- 功能级：`feat/<feature-slug>`
- 阶段级：`feat/<feature-slug>/p<n>-<phase-slug>`

**关于设计文件路径**：`design/<date>/` 目录里的 discuss-result.md、plan.md、Pn.md 始终位于**原始仓库**，不在 worktree 内。feature-exec 启动成员时要单独传 `<design_root>`（设计目录的绝对路径），不能用 `<project_root>/<design_dir>` 拼接，否则在 worktree 场景下会找不到文件。

🛑 **Gate 2**：在创建任何 worktree 之前，先与用户确认选择和分支名。

---

## Stage 3 · 模式选择

**先检测环境能力：**

```
主 Agent 工具列表里是否有 TeamCreate？
- 有 → 支持 Mode C
- 无（Cursor、旧版 Claude、未开启实验功能）→ 仅支持 Mode A
```

告知用户检测结果，然后对 `plan.md` 中的每个阶段选择模式：

| 阶段规模 | 推荐模式（TeamCreate 可用）| 备选模式（TeamCreate 不可用）|
|---|---|---|
| S — 单模块、无新 schema、< 1 天 | **A** — 主 Agent 直接执行 | **A** |
| M / L / XL — 更大规模 | **C** — Agent 团队 | **A**（需拆分为更小子任务逐个执行，或改由用户在支持 TeamCreate 的环境中重跑）|

🛑 **Gate 3**：说明每个阶段的模式及理由，等待用户确认。

---

## Stage 4 · 执行

每个阶段，**立即读取 `feature-exec/SKILL.md` 并严格按其工作流执行。** 该技能内部管理完整的执行周期：

```
dev 编写 Pn.md → reviewer 审查 Pn.md（多轮直至 PASS）
    │
    ▼ Pn.md 批准 — 两条轨道并行启动
    │
    ├── Dev 轨道：写代码 + 单元测试 → reviewer 审查（多轮）
    └── Tester 轨道：编写测试用例 → reviewer 审查（多轮）
    │
    ▼ 两条轨道均 PASS
    │
tester 执行测试用例 → 失败 → dev 修复 → tester 重跑
    │
    ▼ 全部 PASS
    │
dev 更新 ARCH.md + feat.md + test_case.md + git commit → 交付报告
```

**并行执行**：`plan.md` 依赖图中无互相依赖的阶段可同时调用 `feature-exec`（注意：每个阶段需要独立的团队名）。

**主 Agent 在 Mode C 期间的角色**：正常协作（方案审查、代码审查、测试用例评审、测试执行、失败修复）在成员之间通过 SendMessage 推进，主 Agent 不参与。但主 Agent 需要**监控异常**（成员上报、长时间无响应、异常终止、循环/死锁、上下文丢失）并在必要时介入。介入的具体规则见 `feature-exec/SKILL.md` 中的"主 Agent 监控与介入"章节。

每份交付报告后，主 Agent：
1. 更新 `plan.md` 状态（将阶段标记为 `[x]`）
2. 向成员发送 shutdown 并清理团队
3. 向用户展示：交付摘要 + 引入的技术债

🛑 **Gate 4**：用户签收每个阶段的交付（交付报告 + 测试证明）。阻塞性测试失败必须在签收前解决。

---

## 执行模式说明

### Mode A — 主 Agent 直接执行
主 Agent 读取所有上下文、实现、自审、测试、更新文档、提交。用于 S 级阶段，或者 TeamCreate 不可用的环境。

### Mode C — Agent 团队
通过 `TeamCreate` 创建团队，用 `Agent` 工具的 `team_name` + `name` 参数同时 spawn 三名成员（dev、reviewer、tester）。成员之间通过 `SendMessage` 按名字寻址通信。主 Agent 的名字是 `team-lead`，成员上报时 `to` 参数写 `team-lead`。

关键规则：
- Dev 不审查自己的代码
- Reviewer 上下文跨轮次保持——同一个 reviewer 在修复后重新审查，不换人
- 未经主 Agent 授权不得扩大范围
- 提交是交付的一部分——ARCH.md + feat.md + test_case.md + 代码全部提交后阶段才算完成

完整启动和成员提示模板：`feature-exec/SKILL.md` + `references/subagent-prompts.md`

---

## 常见失败模式

- ❌ 未经批准的 `plan.md` 就开始执行
- ❌ Mode C 中主 Agent 直接写代码（角色混乱）
- ❌ 将 `tsc` / `pnpm build` 当作自测证明
- ❌ Dev 审查自己的代码或方案
- ❌ 触发更新条件时未更新 ARCH.md 就合并
- ❌ 实现过程中未上报擅自扩大范围
- ❌ ARCH.md 缺失时跳过 `project-onboard`——没有架构上下文的执行会产生不可靠的输出
- ❌ 成员异常时主 Agent 擅自 spawn 替换或直接接手——应停下来由用户决策
- ❌ 在不支持 TeamCreate 的环境里硬跑 Mode C——会退化成多个孤立的子 Agent

---

## 参考文件

- `project-onboard/SKILL.md` — 代码库探索和文档生成
- `requirement/SKILL.md` — 需求讨论和文档产出
- `feature-exec/SKILL.md` — 带 Agent 团队的按阶段执行
- `references/subagent-prompts.md` — 成员提示模板（dev、reviewer、tester）
