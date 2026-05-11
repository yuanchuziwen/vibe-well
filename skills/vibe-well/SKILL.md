---
name: vibe-well
version: 2.5.0
description: 当用户想要"开发新功能"、"开始一个功能"、"端到端实现功能"、"启动开发流程"，或者说"我们来构建 X"、"我想给产品加 X"、"帮我从头实现 X"时使用此技能。引导完整的功能开发生命周期：需求讨论 → 分阶段执行 → 交付有文档的代码。编排子技能：project-onboard、requirement、feature-exec、regression-test。
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
| `regression-test` | 用户主动跑回归 / Stage 5 全量测试通过后可选触发 / hook command | 回归报告（diff 历史） |

## 按需阅读导航

本文件按 Stage 顺序写。**Agent 启动时先通读全文一次**，之后按下表按需读取扩展材料：

| 在做什么 | 应读 |
|---|---|
| 启动前判断是否恢复 / 偏好 | 本 SKILL § 启动前两节 + `references/config-schema.md` |
| Stage 0.5 复杂度筛 → XS | `references/xs-mini-plan-template.md` |
| Stage 1 需求讨论 | `../requirement/SKILL.md`（主 Agent 完全交出控制权） |
| Stage 3 模式选择细节 | `references/mode-guide.md` |
| Stage 4 执行（任意 Mode）| `../feature-exec/SKILL.md` |
| Mode B 启动 subagent | `references/role-cards/{dev,reviewer,tester}.md` |
| Mode C 启动团队 | `references/subagent-prompts.md` |
| 阶段执行中断恢复 | `../feature-exec/SKILL.md` § 恢复协议 |
| Stage 5 收尾时跑回归 | `../regression-test/SKILL.md` |

---

## 启动前：检查恢复点

进入 Stage 0 前，**先看是否有中断的阶段需要恢复**：

```bash
ls design/*/.vibe-well/state.json 2>/dev/null
```

- **找到 state.json** → 列出每个阶段当前 `stage` 字段，问用户："发现 N 个未完成阶段——\<P1: phase3b-tdd-green\>、\<P2: tester-execute\>……要继续哪个？还是开新的需求？"
- **未找到 / 用户选择新需求** → 跳过本节，按正常 Stage 0 流程

恢复执行时**直接跳到 Stage 4，调用 `feature-exec`**，传入 `resume: true` 标记和阶段编号。`feature-exec/SKILL.md` 的「恢复协议」章节会接管。

详见 `../feature-exec/SKILL.md` § 恢复协议。

---

## 启动前：读取用户偏好

进入 Stage 0 前，**先读取用户级偏好**：

```bash
ls ~/.vibe-well/config.yaml 2>/dev/null
```

- **存在**：立即读取 `references/config-schema.md` 了解字段含义，然后读 `~/.vibe-well/config.yaml` 解析 `defaults`，并合并 `projects.<当前 git 根>` 的覆盖（若有）。后续涉及 Stage 0.5 / 2 / 3 / 5 提问时，把配置值作为默认选项呈现给用户。
- **不存在**：使用 `references/config-schema.md` Schema 中标注的默认值。**不主动创建**该文件，除非用户明确要求"保存为默认"。

向用户提问时遵循"已有偏好就用偏好作默认值"的规则，例如：

```
"使用功能级 worktree（你的偏好默认值）？[Y/n]"
```

**用户当前回答始终优先**——这次用什么用户说了算，配置只是默认。仅在用户说"以后都这样" / "记住这个偏好"时才回写 `~/.vibe-well/config.yaml`。

---

## 工作流概览

```
Stage 0   · 文档检查      → ARCH.md 缺失或过期时运行 project-onboard
Stage 0.5 · 复杂度快筛    → 3 问筛选：XS 走快速通道；其余走完整流程
Stage 1   · 需求讨论      → discuss.md → discuss-result.md → plan.md
                             🛑 Gate：用户批准 plan.md
Stage 2   · Worktree 配置 → 确认是否使用 worktree 及粒度
                             🛑 Gate：用户确认 worktree 策略和分支名
Stage 3   · 模式选择      → 环境检测 → 每个阶段选择模式 A 或 C
                             🛑 Gate：用户确认每个阶段的模式
Stage 4   · 执行          → 每个阶段运行 feature-exec
                             🛑 Gate：每个阶段的交付报告 + 测试证明
Stage 5   · 收尾          → 所有阶段完成后：全量测试 + 合并/PR/保留/丢弃
                             🛑 Gate：用户选择收尾方式
```

**XS 快速通道**：Stage 0.5 判定为 XS 时，跳过 Stage 1~3，仅生成一份 `mini-plan.md`，直接进入 Stage 4 简化执行，最后走 Stage 5 收尾。详见 Stage 0.5 章节。

---

## Stage 0 · 文档检查

```bash
ls CLAUDE.md ARCH.md feat.md 2>/dev/null
```

- **文件缺失**：立即读取 `../project-onboard/SKILL.md` 并按其工作流执行（全量扫描）
- **文件存在**：读取 `../project-onboard/SKILL.md`，由它执行新鲜度判断（git SHA 对比）并决定是否需要增量更新
- 不要在顶层自行做 SHA 对比——project-onboard 内部有完整的新鲜度检测逻辑

---

## Stage 0.5 · 复杂度快筛

文档就绪后，**先用 3 个问题判断本次需求的复杂度**，避免对小变更套用重型流程。

向用户连续问（一次性给出，让用户一次回答）：

> "在开始前先做个快筛——以下三问任意一个为「是」，就走完整流程；全部为「否」，走 XS 快速通道：
>
> 1. 本次变更是否引入**新模块、新表、新 API、新依赖**？
> 2. 是否影响 **ARCH.md §4（模块边界）/ §5（数据模型）/ §7（不变量）** 任一项？
> 3. 是否需要超过 **1 小时**人工实现？"

| 用户回答 | 路径 |
|---|---|
| 全部"否" | **XS 快速通道**（见下） |
| 任一"是" | 进入 Stage 1，走完整流程 |
| 用户不确定 | 默认走完整流程（保守选择） |

**用户偏好覆盖**：如果 `~/.vibe-well/config.yaml` 中 `defaults.skip_gate_when_trivial: false`，跳过本筛选直接进入 Stage 1（用户偏好仪式感）。

### XS 快速通道

立即读取 `references/xs-mini-plan-template.md` 并按其约定执行：

1. 在 `design/<YYYYMMDD>-<feature-slug>/` 下生成 `mini-plan.md`（不生成 discuss / plan）
2. 跳过 Stage 2（不开 worktree，直接当前分支）和 Stage 3（默认 Mode A）
3. 进入简化版 Stage 4：调用 `feature-exec`，传入 `mini-plan.md` 路径和 `mode: xs` 标记
4. feature-exec 内部根据 `verification_strategy` 决定是否走 TDD
5. 完成后走 Stage 5 收尾（通常合并到当前分支，无需 PR）

**XS 通道的 escape hatch**：执行过程中如果发现实际复杂度超过 XS 范围（跨多模块、需要新 schema、超过 2 小时仍未完成、引入新依赖），主 Agent 必须停下来向用户上报，提议升级到完整流程，**不要硬撑**。

---

## Stage 1 · 需求讨论

**立即读取 `../requirement/SKILL.md` 并严格按其工作流执行。**

该技能会引导与用户进行结构化对话，产出三份文档：

- `discuss.md` — 结构化决策点（Dn），含选项、推荐、风险分析
- `discuss-result.md` — 所有 Dn 已决策：技术决策知识库，含代码骨架、数据模式、技术债
- `plan.md` — 分阶段 P1~Pn 交付计划，含范围边界、依赖关系、验收标准

输出目录：项目根目录下的 `design/<YYYYMMDD>-<feature-slug>/`（如 `design/20260505-user-auth/`）。如用户未指定，由本技能根据需求描述自动生成 slug。记录此目录的绝对路径，后续 feature-exec 启动成员时需要用到。

🛑 **Gate 1**：用户批准 `plan.md`（阶段边界、顺序、范围）。这是用户做产品和技术方向决策的最后时机，此后执行基本自主进行。

**🪝 Hook**：用户批准 `plan.md` 后立即触发 `on_plan_approved`（如已配置）。详见 `references/config-schema.md` § Hook 执行规则。

---

## Stage 2 · Worktree 配置

`plan.md` 批准后，主动询问用户。**如果已加载用户偏好**，把 `defaults.worktree_strategy` 作为默认选项呈现：

> "要为这个功能使用 git worktree 吗？（你的偏好：**功能级**）选项：
> - **不使用 worktree** — 直接在当前分支工作（最简单；适合 S 级单阶段工作）
> - **功能级 worktree** — 整个功能使用一个 worktree，所有阶段提交到同一分支（适合串行阶段或每个功能一个 PR）
> - **阶段级 worktree** — 每个阶段单独一个 worktree，各有独立分支和 PR（适合并行阶段或 M/L/XL 规模的隔离需求）
>
> 默认按你的偏好继续（功能级），还是另选？"

未加载偏好时，按原措辞询问，不附"你的偏好"提示。

**分支名**：使用 `defaults.branch_template`（默认 `feat/{slug}/p{n}-{phase_slug}`）渲染默认值并展示，用户可一键确认或修改。

根据回答操作：

| 选择 | 操作 |
|---|---|
| 不使用 worktree | 记录 `worktree: none`。全程 `<project_root>` = 原始仓库根目录。 |
| 功能级 | 现在执行一次 `git worktree add <path> -b <branch>`。记录路径和分支。所有阶段的 `<project_root>` = worktree 路径。 |
| 阶段级 | 暂不创建 worktree。每个阶段启动时执行 `git worktree add <path> -b <branch>`。每个阶段单独记录路径和分支。该阶段的 `<project_root>` = 该阶段的 worktree 路径。 |

分支命名规范（默认模板来自 `defaults.branch_template`，用户可覆盖）：
- 默认模板：`feat/{slug}/p{n}-{phase_slug}`
- 功能级 worktree 时阶段编号 `{n}` 留空、`{phase_slug}` 留空 → `feat/<slug>`
- 阶段级 worktree 时按完整模板渲染 → `feat/<slug>/p<n>-<phase-slug>`

**关于设计文件路径**：`design/<YYYYMMDD>-<feature-slug>/` 目录里的 discuss-result.md、plan.md、Pn.md 始终位于**原始仓库**，不在 worktree 内。feature-exec 启动成员时要单独传 `<design_root>`（设计目录的绝对路径），不能用 `<project_root>/<design_dir>` 拼接，否则在 worktree 场景下会找不到文件。

🛑 **Gate 2**：在创建任何 worktree 之前，先与用户确认选择和分支名。

---

## Stage 3 · 模式选择

**先检测环境能力：**

```
1. 检查主 Agent 工具列表里是否有 TeamCreate
   - 有 → 支持 Mode C
   - 无 → Mode C 不可用
2. 检查是否有 Task / generalPurpose subagent 调用能力
   - 有（Cursor / Claude Code 默认都有）→ 支持 Mode B
   - 无 → 仅支持 Mode A
```

告知用户检测结果，然后对 `plan.md` 中的每个阶段选择模式：

| 阶段规模 | 首选 | 备选（环境不支持首选时） |
|---|---|---|
| S — 单模块、无新 schema、< 1 天 | **A** — 主 Agent 直接执行 | **A** |
| M — 跨 2~3 个模块、有可测逻辑 | **B** — 单 Agent 三角色（一次性 subagent 替代团队） | **A**（拆分子任务串行）|
| L / XL — 更大规模、需要并行轨道 | **C** — Agent 团队（TeamCreate） | **B**（牺牲并行换可移植性）→ 仍不行则 **A** |

**三种模式的差异**：

| 维度 | Mode A | Mode B | Mode C |
|---|---|---|---|
| 角色独立性 | 无（主 Agent 一身担三角） | ✅ 通过子 Agent 上下文隔离 | ✅ 通过 TeamCreate 强隔离 |
| 跨轮上下文 | 主 Agent 内存 | 文档持久化（`<design_root>/.reviews/`）| reviewer 跨轮原生保持 |
| 轨道并行 | 串行 | 用 batch Task 模拟（同一轮多个 subagent 并行）| 原生并行 |
| 通信 | 自言自语 | 主 Agent 居中转交 | 成员互发 SendMessage |
| 环境要求 | 任何 | Task / subagent 工具（普及度高）| TeamCreate（Claude Code 实验功能）|

🛑 **Gate 3**：说明每个阶段的模式及理由，等待用户确认。

**降级回退**：执行中如果 Mode C 出现连续异常（成员失踪、上报循环），主 Agent 必须**停下来上报用户**，由用户决定是否切到 Mode B 或 Mode A 继续；不要自己降级。

---

## Stage 4 · 执行

每个阶段，**立即读取 `../feature-exec/SKILL.md` 并严格按其工作流执行。** 该技能内部管理完整的执行周期：

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

**并行执行**：`plan.md` 依赖图中无互相依赖的阶段可同时调用 `feature-exec`（Mode C 每个阶段独立 team_name；Mode B 用 batch Task 同回合并发；Mode A 串行）。

**主 Agent 在各 Mode 期间的角色**：
- **Mode A**：主 Agent 是唯一执行者——读取上下文、写 Pn.md、自审、写代码 + 测试、跑测试、提交
- **Mode B**：主 Agent 是 orchestrator——每轮用 `Task` spawn 一次性 subagent，自己**不写代码**；负责解析子 Agent 输出、维护 `.reviews/` 历史、决定下一轮 spawn 谁
- **Mode C**：主 Agent 是监督者——正常协作（方案审查、代码审查、测试评审）在成员之间通过 SendMessage 推进；主 Agent 只在**异常**（成员上报、长时间无响应、异常终止、循环/死锁、上下文丢失）时介入

具体行为规则见 `../feature-exec/SKILL.md` 各 Mode 段落。

每份交付报告后，主 Agent：
1. 更新 `plan.md` 状态（将阶段标记为 `[x]`）
2. 向成员发送 shutdown 并清理团队
3. 向用户展示：交付摘要 + 引入的技术债

🛑 **Gate 4**：用户签收每个阶段的交付（交付报告 + 测试证明）。阻塞性测试失败必须在签收前解决。

---

## 执行模式说明

详见 `references/mode-guide.md`。简短摘要：
- **Mode A** — 主 Agent 一身担三角，用于 S/XS
- **Mode B** — 主 Agent + 一次性 Task subagent 三角色，用于 M（不依赖 TeamCreate，Cursor/Codex 友好）
- **Mode C** — TeamCreate Agent 团队，用于 L/XL

模式选择和环境检测在 Stage 3 完成。具体执行步骤在 `../feature-exec/SKILL.md` 各模式段落。

---

## Stage 5 · 收尾

**所有 plan.md 阶段均已签收后执行。**

**Step 1 — 全量测试验证**

是否运行：
- 完整流程（走过 Stage 1）：默认运行（受 `defaults.run_full_test_on_finish`，默认 `true` 控制）
- XS 快速通道：默认跳过（受 `defaults.run_full_test_on_xs`，默认 `false` 控制）

测试命令优先级：
1. 项目脚本（`package.json` 的 `test` / `Makefile` 的 `test` 目标 / `pyproject.toml` 的 test 命令）
2. `defaults.test_command` 兜底值
3. 都没有 → 跳过 Step 1，告知用户"未发现测试命令，跳过全量测试"

**可选**：测试通过后，如果项目维护了 `test_case.md`，询问用户是否额外跑一次回归测试：
> "全量单测已通过。要顺便跑一次基于 `test_case.md` 的功能回归测试吗？（推荐策略：`risk`——只跑 Type=regression/risk 的 TC，约 N 条）[Y/n]"
>
> 选择 Y → 调用 `../regression-test/SKILL.md`，把回归报告路径附在最终交付报告里。
> 选择 n → 跳到 Step 2。

```bash
# 自动按技术栈尝试
npm test / pnpm test / pytest / go test ./... / cargo test
```

若有失败：列出失败项，告知用户"全量测试未通过，需在合并前修复"，不要继续。

若全绿或被跳过：继续 Step 2。

**Step 2 — 询问用户收尾方式**

> "所有阶段已交付，全量测试通过。接下来怎么处理这份工作？
>
> 1. **本地合并**到主分支（`main` / `master`）
> 2. **推送并创建 PR**
> 3. **保留分支**，稍后自行处理
> 4. **丢弃**本次工作
>
> 请选择？"

**Step 3 — 执行选择**

| 选项 | 操作 |
|---|---|
| 1. 本地合并 | `git checkout <base>` → `git pull` → `git merge <branch>` → 运行测试确认 → `git branch -d <branch>` → 清理 worktree |
| 2. 创建 PR | `git push -u origin <branch>` → `gh pr create` 含摘要和测试计划 → 保留 worktree 直到 PR 合并 |
| 3. 保留分支 | 报告分支名和 worktree 路径，不做任何操作 |
| 4. 丢弃 | **先请用户输入 `discard` 确认**，再 `git checkout <base>` → `git branch -D <branch>` → 清理 worktree |

**Worktree 清理**（选项 1 和 4 执行，选项 2 和 3 保留）：
```bash
git worktree remove <worktree-path>
```

不使用 worktree 时跳过清理步骤。

**🪝 Hook**：选项 1/2 执行成功后触发 `on_feature_finished`（如已配置）。选项 4 不触发。

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
- ❌ XS 通道里硬撑——发现复杂度超标却继续，最终绕过 Stage 1 的需求讨论
- ❌ 把 `manual` 当作"懒得写测试"的逃生口——`verification_strategy` 必须按变更类型选，reviewer 应在审 Pn.md 时质疑滥用
- ❌ 自动写入 `~/.vibe-well/config.yaml`——只有用户明确说"以后都这样"时才回写
- ❌ Mode B 中主 Agent 自己写代码——它是 orchestrator，所有代码改动应通过 dev subagent 完成
- ❌ Mode B 中跳过 `.reviews/` 历史——下一轮 reviewer 启动时不传历史，会反复指出同一问题
- ❌ 中断恢复时跳过 state.json 直接重做——浪费已完成的 review 轮次和测试证据
- ❌ 跨阶段复用 `.vibe-well/state.json`——每阶段必须独立，否则 checkpoint 会错乱
- ❌ 把 lint / format 等"提交前检查"配成 `on_phase_committed` hook——这类检查应该用 git pre-commit hook，vibe-well 的 hook 是 commit 之后跑的
- ❌ 用 `mode: block` hook 替代用户决策——hook 失败应该给用户选择权（忽略/修复/中止），不要让 hook 自动 abort 整个流程

---

## 参考文件

- `../project-onboard/SKILL.md` — 代码库探索和文档生成
- `../requirement/SKILL.md` — 需求讨论和文档产出
- `../feature-exec/SKILL.md` — 带 Agent 团队的按阶段执行
- `../regression-test/SKILL.md` — 用 test_case.md 跑回归测试，输出差异报告
- `references/mode-guide.md` — 三种模式（A/B/C）的语义对比和选择指南
- `references/subagent-prompts.md` — Mode C 团队成员的完整启动 prompt 模板
- `references/role-cards/{dev,reviewer,tester}.md` — Mode B 一次性 subagent 的角色卡
- `references/xs-mini-plan-template.md` — XS 快速通道的 mini-plan.md 模板
- `references/config-schema.md` — `~/.vibe-well/config.yaml` 用户偏好 + hooks schema
