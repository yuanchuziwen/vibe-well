# Feature Workflow — 中文概览

> 本文档是 `feature-workflow/` 下各 SKILL.md 的中文翻译与说明，供人工 review 使用。
> 以英文原文为准，如有出入请修改英文版。

---

## 整体结构

```
feature-workflow/           ← 顶层编排 skill
├── project-onboard/        ← 代码库探索 + 文档生成
├── requirement/            ← 需求讨论 + 产出三份文档
├── feature-exec/           ← 单个 phase 的完整执行
└── references/
    └── subagent-prompts.md ← dev / reviewer / tester 的 kickoff prompt 模板
```

---

## 完整工作流

```
Stage 0 · 文档检查     → 没有或过期 → 跑 project-onboard
Stage 1 · 需求讨论     → discuss.md → discuss-result.md → plan.md
                          🛑 用户 approve plan.md 之后才继续
Stage 2 · 模式选择     → 每个 phase 选 A 或 C
                          🛑 用户确认每个 phase 的模式
Stage 3 · 执行         → 每个 phase 跑 feature-exec（Mode C 是 fire-and-forget）
                          🛑 每个 phase 收到交付报告 + 测试证据后用户签收
```

---

## Sub-skill 1 · project-onboard

**触发时机**：ARCH.md / feat.md 不存在或版本过期（通过 git SHA 判断）

**产出**：

| 文件 | 作用 |
|---|---|
| `ARCH.md` | 架构文档：模块边界、数据模型、不变量、请求流 |
| `feat.md` | 功能地图：所有功能点 + 入口 + 文件位置 |
| `CLAUDE.md` | 薄索引：指向以上两个文档，供 Claude Code 自动加载 |

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
3. 缺 Pn.md → Mode C 内部由 dev 写；Mode A 由主会话写并让用户确认
4. 多个 design/ 目录存在 → 询问用户用哪个

### Mode A — 主会话直接执行

适用：S 级 phase（单模块、无新 schema、< 1 天）

步骤：读文档 → 实现 → 写单测 → 自我 review → 跑验收测试 → 更新 ARCH.md + feat.md → commit

### Mode C — Agent Team（fire-and-forget）

适用：M / L / XL 级 phase

**三个成员**：dev、reviewer、tester，kickoff 时同时启动

**内部流程（双轨并行）**：

```
reviewer ↔ dev: 审 Pn.md（多轮，直到 PASS）
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
dev 更新 ARCH.md + feat.md + git commit → 交付报告 → 主会话
```

**关键规则**：
- Dev 不 review 自己的代码，永远是 reviewer 来
- Reviewer context 在多轮 review 中复用——re-review 时只验证问题是否被修复，不用从头看
- **单测是 dev 的交付门槛**：没有通过单测的代码不能送 code review
- **两条轨道都 PASS 才能执行测试**：tester 不能提前执行
- Commit 是交付的一部分：phase 未 commit 不算交付

**主会话角色**：kickoff → 等待 → 收交付报告。中间不介入，除非有成员 escalation。

**Escalation 触发条件**（只有这些才上报主会话）：
- Pn.md 有歧义，读 discuss-result.md 也无法解决
- 需要改动 Pn.md scope 之外的东西
- 代码和 discuss-result.md 某个决策冲突
- 测试失败涉及产品层面决策

---

## 三个成员的职责边界

| 成员 | 职责 | 不做 |
|---|---|---|
| dev | 写代码 + 单测，修复问题，更新 ARCH.md + feat.md，commit | 不 review 自己的代码 |
| reviewer | 审 Pn.md + 代码 + test cases，给出 ARCH.md 受影响 section 清单 | 不写代码，不写 test cases |
| tester | 写 test cases，执行，提供证据，汇报失败 | 不写代码，不在两条轨道 PASS 前执行 |

**Tester 的 test case 来源**（不是凭空想）：
1. Pn.md §Acceptance checklist → 每条至少一个 case（happy path + 关键错误路径）
2. feat.md 中被本 phase 改动文件波及的已有功能 → 回归 case
3. Pn.md §Risk notes → 高风险专项 case

**Test case 格式**：
```
TC-<n>: <名称>
类型: 新功能 / 回归 / 风险专项
入口: <URL 或操作入口>
视口: 1440×900 / 375×812 / 两者都要
步骤:
  1. <操作>
  2. <操作>
预期结果: <应该发生什么>
```

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
- 首次 project-onboard 时为空，由 feature-exec 的 tester 逐步积累
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

## 常见失败模式

- ❌ 没有 approved plan.md 就开始执行
- ❌ Mode C 里主会话直接写代码（角色混乱）
- ❌ 把 `tsc` / `pnpm build` 通过当作测试证据
- ❌ Dev review 自己的代码或方案
- ❌ 触发了 ARCH.md 更新条件却没更新
- ❌ 执行时在 Pn.md scope 之外做改动而不 escalate
- ❌ ARCH.md 缺失时跳过 project-onboard 直接执行——没有架构上下文，输出质量不可靠
