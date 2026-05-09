---
name: regression-test
version: 1.0.0
description: 当用户想要"跑回归测试"、"验证当前实现没破坏历史"、"verify nothing broke"、"run regression"，或在阶段交付/收尾时希望对历史 TC 做退化检查时使用此技能。读取 test_case.md，按选定策略执行子集 TC，输出差异报告（新增 PASS / 新增 FAIL / 转折 TC）。
---

# Regression Test

把 `test_case.md` 当作长期回归测试集，对当前代码状态执行选定 TC，输出与历史报告的差异。**这个 skill 不写新功能、不写新 TC**——它只跑已有的 TC，然后告诉你哪些坏了。

## 何时触发

| 场景 | 入口 |
|---|---|
| 用户主动 | "跑回归" / "verify nothing broke" / "run regression on auth" |
| Stage 5 收尾 Step 1 | 主 SKILL 询问"全量测试 vs 回归测试"时选择本 skill |
| `on_phase_committed` / `on_feature_finished` hook | hook command 调用此 skill |
| feature-exec 阶段交付前 | 主 Agent 询问用户是否触发 |

## 必读输入

| 文件 | 用途 |
|---|---|
| `<project_root>/test_case.md` | 全部 TC 表格（按 domain/area 分组） |
| `<project_root>/ARCH.md` §7 | 不变量列表，作为通用必检项 |
| `<project_root>/.vibe-well/regression-reports/` | 历史回归报告目录（用于差异对比） |

## 输出

| 产物 | 路径 |
|---|---|
| 报告文件 | `<project_root>/.vibe-well/regression-reports/<YYYYMMDD-HHMMSS>.md` |
| 控制台摘要 | 一行：`Regression: N/M PASS · K FAIL · J SKIP · D 转折` |

报告 frontmatter 字段：

```yaml
---
ran_at: 2026-05-09T17:30:00Z
git_sha: <short>
strategy: risk
selected_tcs: [TC-001, TC-002, TC-011, TC-022, ...]
prev_report: 2026-05-08T11:00:00Z   # null if first run
---
```

## 流程

### Step 1 — 启动检查

1. **`test_case.md` 不存在** → 提示"项目尚未维护 test_case.md，先运行 project-onboard 或在 feature-exec 阶段写 TC"，退出
2. **空 TC 表（仅有 frontmatter）** → 报告"无可执行 TC"，退出
3. **创建 reports 目录**：`mkdir -p <project_root>/.vibe-well/regression-reports`

### Step 2 — 选择策略

向用户确认（除非已通过参数显式指定）：

> "选择回归策略：
> 1. **full** — 全部非 Deprecated TC（最稳，最慢）
> 2. **risk**（默认）— Type=regression 或 Type=risk 的 TC（推荐用于阶段交付）
> 3. **incremental** — 用 `git diff` 推断近期变更涉及的 domain，只跑相关 TC
> 4. **tagged:\<domain\>** — 指定 domain（如 `tagged:auth`、`tagged:dashboard`）"

### Step 3 — 解析 TC 集合

按策略筛选：

| 策略 | 筛选规则 |
|---|---|
| `full` | 所有非 Deprecated TC |
| `risk` | `Type` 列等于 `regression` 或 `risk` 的 TC |
| `incremental` | `git diff <prev-tag-or-sha>..HEAD --name-only` 取改动文件 → 反向查 ARCH.md §4/§5 推断涉及的 domain → 取该 domain 下所有 TC |
| `tagged:<x>` | `<x>` 等于 test_case.md 中的二级标题（domain 名）或 feature 列前缀 |

把筛选结果列给用户确认（前 10 条 + 总数），用户可手动剔除。

### Step 4 — 执行

**复用** `../vibe-well/references/role-cards/tester.md` 的执行规则：
- 脚本模式 vs 交互模式判断（看是否有 `playwright.config.*`）
- 视觉验证字段决定截图策略
- 失败时附 screenshot + trace + 堆栈

**关键差异**：
- **不写新 TC**——只读 test_case.md 已有的
- **执行单位**：每条 TC 跑完立即写一行结果到内存表
- **失败不修代码**——回归测试发现的失败属于"信息上报"，由用户决定是否进入 fix 流程
- **进度心跳**：每 10 条 TC 或每 60 秒输出一次进度

### Step 5 — 差异分析

读取 `prev_report`（按 `regression-reports/` 目录下时间戳最新一份）：

| 状态 | 含义 |
|---|---|
| **新增 PASS** | 上次报告里这条 TC 不存在（新写的 TC）或 SKIP，本次 PASS |
| **新增 FAIL** | 上次 PASS 本次 FAIL（**回归**！需重点上报）|
| **稳定 PASS** | 上次和本次都 PASS（默认收起，仅总数）|
| **稳定 FAIL** | 上次和本次都 FAIL（已知问题，提示"是否需要修复"）|
| **转折** | 上次 FAIL 本次 PASS（**修好了**，值得标记）|

### Step 6 — 写报告

```markdown
---
ran_at: <ISO8601>
git_sha: <short>
strategy: <full|risk|incremental|tagged:xxx>
selected_tcs: [<list>]
prev_report: <prev timestamp or null>
summary:
  total: N
  pass: N
  fail: N
  skip: N
  new_fail: N      # 回归数
  new_pass: N
  recovered: N     # 转折数
---

# Regression Report — <timestamp>

## 摘要

跑了 N 条 TC（策略：<x>），<P> PASS / <F> FAIL / <S> SKIP。

**回归（新增 FAIL）**：<N> 条
**转折（恢复 PASS）**：<N> 条

## 回归详情（如有）

### TC-XXX — <名称>
- 上次状态：PASS（<prev_timestamp>）
- 本次状态：FAIL
- 入口：<URL/cmd>
- 失败步骤：<step>
- 实际表现：<observed>
- 证据：[screenshot](path) · [trace](path)

## 完整 TC 结果表

| TC | 名称 | Type | 上次 | 本次 | 备注 |
|---|---|---|---|---|---|
| TC-001 | ... | feature | PASS | PASS | |
| TC-022 | ... | regression | PASS | FAIL | 新增 FAIL！|

## 建议下一步

- 若有新增 FAIL：建议进入 feature-exec 修复流程（用 fix-failure 任务标签）
- 若全绿且策略=incremental：可考虑切到 full 做全面回归
- 若 N 天内未跑过 full：建议安排一次（命令略）
```

### Step 7 — 收尾

- 控制台输出一行摘要 + 报告路径
- 如有新增 FAIL，**询问用户是否立即进入 feature-exec 修复**：
  - 是 → 把失败 TC 列表打包传给 feature-exec，作为 mini-plan
  - 否 → 退出

## 与其它 skill 的边界

- **tester 角色（feature-exec 中）**：写新 TC + 跑当前阶段 TC
- **regression-test（本 skill）**：跑历史 TC，做差异对比

简单记法：tester 看的是"这阶段的功能"，regression 看的是"历史不退化"。

## 关键约束

- **不修改 test_case.md**——本 skill 是只读使用者，TC 增删归 tester 角色管
- **不写产品代码**——回归测试发现失败时上报，由用户决定后续
- **报告必须对比 prev_report**——首次运行时显式标注 "first run, no prev to compare"
- **incremental 策略需要 git history**——非 git 仓库或浅克隆下退化为 risk

## 参考文件

- `../vibe-well/references/role-cards/tester.md` — 执行 TC 的脚本/交互模式规则（直接复用）
- `../vibe-well/references/config-schema.md` — 用户偏好（如默认策略，可在 config.yaml 加 `regression.default_strategy`）
- `../project-onboard/references/doc-templates.md` § test_case.md — TC 表格 schema
