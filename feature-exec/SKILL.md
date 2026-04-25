---
name: feature-exec
version: 1.0.0
description: 当用户想要"实现某个阶段"、"开始写代码"、"执行 P1"、"运行第 N 阶段"、"实现已批准的方案"，或 feature-workflow 到达执行阶段时使用此技能。接收已批准的 plan.md，驱动完整周期：dev 写 Pn.md → reviewer 审查 → dev 和 tester 并行工作 → 两条轨道合并 → tester 执行 → dev 提交。S 级变更用 Mode A（主 Agent 直接执行），其余用 Mode C（Agent 团队通过 TeamCreate + SendMessage 协调）。
---

# Feature Exec

将一个阶段从已批准的方案驱动到已提交的代码。主 Agent 的职责：
- **启动**：创建团队、spawn 三名成员
- **监控**：在团队运行期间关注异常，必要时介入
- **响应上报**：成员遇到超出权限的决策时回复
- **收尾**：收到交付报告后 shutdown 成员、清理团队

正常协作（方案审查、代码审查、测试用例评审、测试执行、失败修复）在成员之间通过 SendMessage 推进，主 Agent 不参与。

## 所需输入

| 输入 | 来源 | 是否必须 |
|---|---|---|
| `discuss-result.md` | `design/<date>/discuss-result.md` | ✅ |
| `plan.md` | `design/<date>/plan.md` | ✅ |
| `ARCH.md` | 项目根目录 | ✅ |
| `feat.md` | 项目根目录 | ✅ |
| `Pn.md` — 阶段方案 | `design/<date>/P<n>.md` | Mode A 由主 Agent 写，Mode C 由 dev 成员写 |

---

## 预检：依赖检查

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
检查主 Agent 工具列表里是否有 TeamCreate：
- 有 → 支持 Mode C
- 无（Cursor、旧版 Claude、未开启实验功能）→ 仅支持 Mode A
```

如果环境不支持 Mode C 但阶段规模为 M/L/XL，告知用户：
> "当前环境不支持 Agent 团队（TeamCreate 不可用）。此阶段规模为 \<size\>，建议：
> - 拆分为更小子任务后用 Mode A 逐个执行；或
> - 切换到支持实验功能的 Claude Code 环境后再执行。"

---

## 模式选择

| 阶段规模 | 模式 |
|---|---|
| S — 单模块、无新 schema、< 1 天 | **A** — 主 Agent 直接执行 |
| M / L / XL | **C** — Agent 团队（需要 TeamCreate 可用）|

在开始前说明模式和阶段名称。

---

## Mode A — 主 Agent 直接执行

1. 读取 ARCH.md（§2、§4、§5、§7、§8）、feat.md、discuss-result.md、Pn.md
2. 读取所有待修改文件，包括调用方和依赖方
3. 严格按 Pn.md 规定实现——不扩大范围
4. 编写单元测试；所有测试通过后才能继续
5. 自审：代码与 Pn.md 一致、无不变量被破坏、无死代码、命名符合 §8
6. 针对 Pn.md 验收清单运行浏览器 / curl 测试——粘贴证明
7. 更新 feat.md、ARCH.md（仅受影响的章节）、test_case.md（追加该阶段的 TC）
8. Git commit：代码 + 测试 + feat.md + ARCH.md + test_case.md 一次性提交

交付物：commit SHA + 测试证明。

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
    ├─── Dev 轨道 ──────────────────────────────────────┐
    │    dev 编写代码 + 单元测试                         │
    │    dev → reviewer：代码审查请求                    │
    │    reviewer ↔ dev：审查（多轮直至 PASS）           │
    │                                                   │
    └─── Tester 轨道 ────────────────────────────────────┤
         reviewer → tester：发送 Pn.md 路径，通知开始    │
         tester 编写测试用例                            │
         tester → reviewer：测试用例审查请求            │
         reviewer ↔ tester：审查（多轮直至 PASS）       │
                                                        │
    ◄──────────── 两条轨道均 PASS ───────────────────────┘
    │
reviewer → tester：通知可以执行了
tester 执行测试用例
    │
失败 → tester → dev：报告失败 → dev 修复 → tester 重跑
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

**立即读取 `../references/subagent-prompts.md` 获取各成员的完整启动提示。**

**Step 1 — 确定路径变量**

```bash
# <project_root> — 成员读写代码的目录
# - 不使用 worktree：
git rev-parse --show-toplevel
# - 功能级 worktree：使用 Stage 2 中创建的 worktree 路径
# - 阶段级 worktree：使用本阶段的 worktree 路径（如尚未创建，现在执行）：
# git worktree add <path> -b <branch>

# <design_root> — 设计文件的绝对路径（始终在原始仓库，不在 worktree 内）
# 例如 /Users/you/project/design/20260426
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
2. 用 SendMessage 向每位成员发送 `shutdown_request`（消息体：`{"type": "shutdown_request", "reason": "阶段已交付"}`）
3. 收到每位成员的 `shutdown_approved` 后，调用 `TeamDelete`
4. 向用户展示交付摘要 + 引入的技术债

Shutdown 由系统自动处理响应——成员不需要手动响应，收到 `shutdown_request` 后会自动发回 `shutdown_approved` 并终止。

---

## 关键约束

- **Dev 不审查自己的工作** — reviewer 始终是独立的成员
- **Pn.md 由 dev 编写，reviewer 审查** — 不要让同一个人既写又审
- **Reviewer 上下文跨轮次保持** — 修复后由同一个 reviewer 重新审查，不换人
- **单元测试是 dev 的交付门槛** — 没有通过单元测试的代码不进入代码审查
- **两条轨道均 PASS 后才能执行测试** — tester 在收到 reviewer 的明确通知前不执行
- **提交是交付的一部分** — 代码 + ARCH.md + feat.md + test_case.md 全部提交后阶段才算完成
- **未经授权不得扩大范围** — Pn.md §范围——包含 以外的任何改动都需先获授权
- **成员间通信必须用 SendMessage** — 成员的文字输出对队友不可见
- **主 Agent 的名字是 `team-lead`** — 成员上报时 `to="team-lead"`

---

## 参考文件

- `../references/subagent-prompts.md` — dev、reviewer、tester 的启动提示模板
- `../requirement/references/plan-template.md` — plan.md 格式（Pn.md 也参考此模板的"Phase Details"部分）
