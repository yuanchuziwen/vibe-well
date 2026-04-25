# vibe-well — 中文概览

[English](./README.md)

## 安装

```bash
# 全局安装（所有项目可用）
npx reskill install github:yuanchuziwen/vibe-well -g

# 项目本地安装
npx reskill install github:yuanchuziwen/vibe-well
```

---

> 本文档是各 SKILL.md 的中文说明，供人工 review 使用。
> 以对应 SKILL.md 为准，如有出入请以 SKILL.md 为基准修改。

---

## 整体结构

```
vibe-well/                      ← 顶层编排 skill
├── project-onboard/            ← 代码库探索 + 文档生成
│   └── references/             ← explore-guide.md、doc-templates.md
├── requirement/                ← 需求讨论 + 产出三份文档
│   └── references/             ← discuss-template.md、discuss-result-template.md、plan-template.md
├── feature-exec/               ← 单个 phase 的完整执行
└── references/
    └── subagent-prompts.md     ← dev / reviewer / tester 的 kickoff prompt 模板
```

---

## 完整工作流

```
Stage 0 · 文档检查      → 没有或过期 → 跑 project-onboard
Stage 1 · 需求讨论      → discuss.md → discuss-result.md → plan.md
                           🛑 用户 approve plan.md 之后才继续
Stage 2 · Worktree 配置 → 不用 / 功能级 / 阶段级
                           🛑 用户确认 worktree 策略和分支名
Stage 3 · 模式选择      → 环境检测 + 每个 phase 选 A 或 C
                           🛑 用户确认每个 phase 的模式
Stage 4 · 执行          → 每个 phase 跑 feature-exec
                           🛑 每个 phase 收到交付报告 + 测试证据后用户签收
```

---

## 环境要求

- **Mode C（Agent 团队）** 需要 `TeamCreate` 工具，在开启实验功能的 Claude Code 中可用。
- **Mode A（主 Agent 直接执行）** 在任何环境都能跑（Cursor、旧版 Claude、标准 API）。vibe-well 会在 Stage 3 自动检测 `TeamCreate`，不可用时降级到 Mode A。

---

## Sub-skill 1 · project-onboard

**触发时机**：ARCH.md / feat.md 不存在或版本过期（通过 git SHA 判断）

**产出**：

| 文件 | 作用 |
|---|---|
| `ARCH.md` | 架构文档：模块边界、数据模型、不变量、请求流 |
| `feat.md` | 功能地图：所有功能点 + 入口 + 文件位置 |
| `test_case.md` | 功能测试用例：首次扫描时从 feat.md 机械推断（标注 `bootstrapped`），后续由 tester 精炼 |
| `CLAUDE.md` | 薄索引：指向以上三个文档，供 Claude Code 自动加载 |

**新鲜度判断**：
- ✅ 当前：git SHA 无变化，working tree 干净
- ⚠️ 过期：有 commit 变更未同步
- 🔶 脏：有未提交变更（记录但不阻塞）

**更新策略**：
- **增量更新**（默认）：只重新看 diff 里的文件，只更新受影响的 section
- **全量重新生成**（自动升级条件）：文档不存在 / diff 超 20 个文件 / 改了 package.json、schema 文件、入口文件 / 用户主动要求

---

## Sub-skill 2 · requirement

**触发时机**：用户提出任何功能需求、bug 修复、重构想法

**产出三份文档**（存放在 `design/<YYYYMMDD>/`）：

| 文档 | 内容 |
|---|---|
| `discuss.md` | 结构化决策点 D1~Dn，每个含：选项、推荐、冷水分析（风险） |
| `discuss-result.md` | 所有 Dn 决策落地：技术方案知识库，含代码骨架、DB schema、技术债 |
| `plan.md` | 分阶段交付计划 P1~Pn：scope 边界、依赖关系、验收条件 |

**关键设计原则**：
- Agent 主动提议，用户只需选选项，不用写文字
- 每个 Dn 必须是"不靠用户输入无法决定的事"——可以从代码库或行业经验推断的不应该列出来
- 冷水分析要诚实：迁移风险、并发问题、未来后悔点都要说清楚
- plan.md 的粒度是"有意义的交付单元"（如"登录模块"），不是 API 级别
- P1.md、P2.md 等任务级文档由 dev-member 在执行时自己写，不是需求阶段的产出

**UI 设计辅助**（当需求涉及 UI 变更时）：
- 如有 `huashu-design` skill：提议产出 3 个差异化高保真方向供用户选
- 如有 `playground` skill：提议生成交互式 HTML 探索器
- 两者都没有：在 discuss.md 相关 Dn 中用文字描述 UI 方向

**四个 Gate**：
1. Agent 复述理解 + 主要开放问题 → 用户确认
2. discuss.md 写完 → 用户逐一选 Dn 选项
3. discuss-result.md 写完 → 用户 review 正确性
4. plan.md 写完 → 用户 review 阶段划分和顺序

---

## Sub-skill 3 · feature-exec

**触发时机**：有 approved plan.md 的情况下，执行某个 phase

**自检依赖**（按顺序）：
1. 缺 ARCH.md / feat.md → 自动触发 project-onboard
2. 缺 discuss-result.md / plan.md → 自动触发 requirement
3. 多个 design/ 目录存在 → 询问用户用哪个

### Mode A — 主 Agent 直接执行

适用：S 级 phase（单模块、无新 schema、< 1 天），或 TeamCreate 不可用的环境

步骤：写 Pn.md（如不存在）→ 读文档 → 实现 → 写单测 → 自我 review → 跑验收测试 → 更新 ARCH.md + feat.md + test_case.md → commit

### Mode C — Agent 团队（需 TeamCreate 可用）

适用：M / L / XL 级 phase

**启动**：主 Agent 执行 `TeamCreate` → 用 `Agent(team_name, name, ...)` 同时 spawn dev / reviewer / tester 三个成员。

**通信**：成员间用 `SendMessage` 按名字寻址（`dev` / `reviewer` / `tester`）。主 Agent 的名字是 `team-lead`。

**内部流程（双轨并行）**：

```
dev 写 Pn.md → reviewer 审 Pn.md（多轮，直到 PASS）
    │
    ▼ Pn.md 通过 —— 两条轨道同时开始
    │
    ├── Dev 轨道：写代码 + 单测 → reviewer 审代码（多轮）
    └── Tester 轨道：写 test cases → reviewer 审 test cases（多轮）
    │
    ▼ 两条轨道都 PASS
    │
tester 执行 test cases → 失败 → dev 修复 → tester 回归
    │
    ▼ 全部通过
    │
dev 更新 ARCH.md + feat.md + test_case.md + git commit → 交付报告 → 主 Agent shutdown 团队
```

**关键规则**：
- Pn.md 由 dev 写，reviewer 审——不要让同一个成员既写又审
- Dev 不 review 自己的代码，永远是 reviewer 来
- Reviewer context 在多轮 review 中复用——re-review 时只验证问题是否被修复，不用从头看
- **单测是 dev 的交付门槛**：没有通过单测的代码不能送 code review
- **两条轨道都 PASS 才能执行测试**：tester 不能提前执行
- Commit 是交付的一部分：phase 未 commit 不算交付

**主 Agent 监控与介入**：
- 正常情况下不介入，成员之间通过 SendMessage 推进
- 收到成员上报（歧义、超范围改动、产品决策问题）→ 转述给用户，把用户决策回复给上报的成员
- 识别异常（长时间无响应、异常终止、循环消息、上下文丢失）→ 先用 SendMessage 轻度纠偏；纠偏 2~3 次无效或成员终止 → **停下来向用户汇报现状和候选方案，由用户决策**
- **禁止**未经用户同意 spawn 替换成员或直接接手成员工作

---

## 三个成员的职责边界

| 成员 | 职责 | 不做 |
|---|---|---|
| dev | 写 Pn.md、写代码 + 单测，修复问题，更新 ARCH.md + feat.md + test_case.md，commit | 不 review 自己的代码或方案 |
| reviewer | 审 Pn.md + 代码 + test cases，给出 ARCH.md 受影响 section 清单 | 不写代码、不写 test cases、不直接 escalate |
| tester | 写 test cases，执行，提供证据，汇报失败 | 不写代码，不在两条轨道 PASS 前执行 |

**Tester 的 test case 来源**（不是凭空想）：
1. Pn.md §Acceptance checklist → 每条至少一个 case（happy path + 关键错误路径）
2. feat.md 中被本 phase 改动文件波及的已有功能 → 回归 case
3. Pn.md §Risk notes → 高风险专项 case

**Test case 格式**：
```
TC-<n>: <名称>
类型: 新功能 / 回归 / 风险
入口: <URL 或操作入口>
视口: 1440×900 / 375×812 / 两者（仅前端 TC 需要）
步骤:
  1. <操作>
预期结果: <应该发生什么>
```

TC 编号 tester 启动时读取 test_case.md 当前最大编号，从下一个开始。

---

## 文档更新规则

phase 完成后，由 dev 负责更新，全部在一个 commit 里：

| 文件 | 更新触发条件 | 谁提供内容 |
|---|---|---|
| `feat.md` | 有 route 或 component 新增 / 修改 / 删除 | dev 自己判断 |
| `ARCH.md` | 见下方触发列表 | reviewer 在 code review 时给出受影响 section 清单 |
| `test_case.md` | 每个 phase 结束都更新 | tester 提供格式化的 TC 行，dev 写入 |

**test_case.md 的定位**：
- 与 feat.md 结构对应，每个 feature 有若干 TC
- 粒度：功能级（TC 名称 + 入口 + 简要步骤 + 预期结果），不存截图和完整 curl 命令
- 首次 project-onboard 时从 feat.md 机械推断，标注 `bootstrapped`，由 feature-exec 的 tester 逐步验证和精炼
- 废弃的 TC（feature 被删除或改变）移入 `## Deprecated` 区，不删除，保留可追溯性
- 是未来 regression-test skill 的输入来源

**ARCH.md 更新触发列表**（任意一项命中就必须更新）：
- §2 Quick Reference 里的文件有变化
- §4 模块边界发生变化
- §5 数据模型（表、列）有变化
- §6 请求流有变化
- §7 不变量新增、修改或删除
- §8 术语新增
- §9 技术债解决或新增

---

## Worktree 策略

| 选项 | 适用场景 |
|---|---|
| **不使用** | S 级单 phase、不需要独立 PR |
| **功能级** | 整个功能一个 worktree、所有 phase 同一分支、整体一个 PR |
| **阶段级** | 每个 phase 独立 worktree 和分支，每个 phase 一个 PR，适合并行阶段 |

**关键约束**：`design/<date>/` 目录下的设计文件始终在原始仓库根目录，不在 worktree 内。feature-exec 启动成员时会同时传 `<project_root>`（代码目录）和 `<design_root>`（设计文件绝对路径）。

---

## 常见失败模式

- ❌ 没有 approved plan.md 就开始执行
- ❌ Mode C 里主 Agent 直接写代码（角色混乱）
- ❌ 把 `tsc` / `pnpm build` 通过当作测试证据
- ❌ Dev review 自己的代码或方案
- ❌ 触发了 ARCH.md 更新条件却没更新
- ❌ 执行时在 Pn.md scope 之外做改动而不 escalate
- ❌ ARCH.md 缺失时跳过 project-onboard 直接执行——没有架构上下文，输出质量不可靠
- ❌ 成员异常时主 Agent 擅自 spawn 替换或直接接手——应停下来由用户决策
- ❌ 在不支持 TeamCreate 的环境里硬跑 Mode C——会退化成多个孤立的子 Agent
