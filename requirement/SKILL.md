---
name: requirement
version: 1.0.0
description: 当用户想要"明确需求"、"讨论一个功能"、"写规格文档"、"开始需求讨论"、"分析这个需求"、"帮我想清楚这个功能"，或者说"我想加 X"、"我们来想想怎么做 X"、"我有个 bug / 重构想法"，或 feature-workflow 到达 Stage 1 时使用此技能。通过结构化对话引导需求澄清，产出三份文档：discuss.md（决策点）、discuss-result.md（技术决策知识库）、plan.md（分阶段交付计划）。
---

# Requirement

从用户的原始需求出发，通过结构化对话引导，产出三份文档，使任何 dev 成员子 Agent 都能无需进一步澄清、直接拿到文档后开始执行。

## 输出文档

| 文档 | 用途 | 编写人 |
|---|---|---|
| `discuss.md` | 结构化决策点（D1、D2……），含选项、推荐、风险分析 | 本技能 |
| `discuss-result.md` | 技术决策知识库——所有 Dn 已决策，含决策理由、代码骨架、技术债 | 本技能 |
| `plan.md` | 分阶段交付计划（P1~Pn），含依赖关系、范围边界、验收标准 | 本技能 |

三份文档存放在与用户商定的目录——默认：项目根目录下的 `design/<YYYYMMDD>/`。

**此处不产出**：P1.md、P2.md…… 任务级实现文档。这些由 dev 成员子 Agent 在执行阶段编写。

---

## 何时调用本技能

- 用户提出：功能想法、bug 报告、重构提案、草稿文档、粗略规格，或只是在对话中描述一个问题
- 用户明确触发："明确这个需求"、"我们来讨论"、"为 X 写规格"
- 从 feature-workflow Stage 1 调用

---

## 预检

开始前检查项目文档：

```bash
ls CLAUDE.md ARCH.md feat.md 2>/dev/null
```

- **ARCH.md 存在**：读取 §1（概述）、§4（模块边界）、§5（数据模型）、§7（不变量）、§8（术语）。用这些内容锚定技术选项。
- **ARCH.md 缺失**：告知用户——"未找到 ARCH.md。建议先运行 project-onboard 以获取更丰富的技术上下文。当前将基于可见代码库继续。"
- 如果需求涉及已有功能，读取 `feat.md`。

---

## 工作流

### Stage 1 · 对话

接受任何输入：
- 一句话："我想加用户登录"
- 多段草稿文档（如 `refactor-0421.md`）
- 一段对话
- 一个 bug 描述

**你的任务**：提取意图、发现未知问题、主动提出结构化决策点。用户应该主要是在选择选项，而不是写大段文字。

**澄清风格**：
- 生成 discuss.md 前最多提 2~3 个简短问题
- 不要问可以从代码库或 ARCH.md 推导出来的问题
- 如果可以给出推荐，直接给——不要让用户自己想

**当需求涉及 UI 变更**（新页面、新组件、修改布局）时，检查可用的设计工具：

```bash
ls ~/.agents/skills/huashu-design/ 2>/dev/null && echo "huashu-design:yes" || echo "huashu-design:no"
ls ~/.agents/skills/playground/ ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/playground/ 2>/dev/null | grep -q playground && echo "playground:yes" || echo "playground:no"
```

按优先级使用第一个可用选项：

| 可用工具 | 使用时机 | 操作 |
|---|---|---|
| `huashu-design` | 视觉方向不明确——存在多种合理风格 | 提议调用 `huashu-design` 生成 3 个差异化的高保真方向供用户选择，再确定 discuss.md。 |
| `playground`（无 huashu-design）| 需要对比布局 / 组件变体 | 提议生成一个带实时控件的交互式单文件 HTML 探索器，让用户可视化比较。将选定方向作为 Dn 决策嵌入 discuss.md。 |
| 两者均无 | — | 在 discuss.md 相关 Dn 中用文字描述 UI 方向（布局、视觉风格、关键交互、视口行为）。注明："无可用设计工具——UI 方向以文字描述。" |

此步骤始终可选——用户可以跳过，直接使用文字描述继续。

🛑 **Gate 1**：编写 discuss.md 前，简要陈述你理解的内容和主要待解问题，等待用户确认或纠正。

---

### Stage 2 · 编写 discuss.md

主动生成 `discuss.md`——不要让用户来写。

**格式**：立即读取 `references/discuss-template.md` 获取完整格式。

关键规则：
- 每个 Dn 对应一个没有用户输入就无法决策的问题
- 不要将可以从代码库、ARCH.md 或行业最佳实践推导出的决策放入 discuss.md
- 每个 Dn 必须包含：至少 2 个选项、含理由的推荐，以及"泼冷水"说明（可能出错的地方）
- 选项应具体，不能抽象（"使用 Prisma" 而非 "使用 ORM"）
- 按依赖顺序编号 D1、D2……（能解锁其他决策的先写）
- 末尾附"总体成本估算"部分：按决策群组粗估规模（S/M/L/XL）

编写完 discuss.md 后，展示给用户并请其为每个 Dn 选择选项。

🛑 **Gate 2**：用户审查 discuss.md 并做出选择，所有 Dn 解决后再继续。如果用户在看到后续决策后改变了对早期决策的看法，更新早期决策并说明级联影响。

---

### Stage 3 · 编写 discuss-result.md

所有 Dn 解决后，生成 `discuss-result.md`。

**格式**：立即读取 `references/discuss-result-template.md` 获取完整格式。

关键规则：
- 每个 Dn 对应一个章节，用决策编号和标题标注
- 每个 Dn 包含：选定选项、决策理由、实现说明、代码骨架（如有帮助）、引入的技术债
- 代码骨架应足够完整，使 dev 成员无需额外设计决策即可实现——类型签名、表 schema、关键算法伪代码
- 技术债项目获得唯一 ID（TD-X），并汇总到末尾的 §技术债 章节
- 本文档是所有子 Agent 的唯一权威来源——必须自包含。dev 成员读取本文档 + ARCH.md 后应拥有所需的一切。

🛑 **Gate 3**：展示 discuss-result.md，用户审查正确性和完整性。小的澄清可以就地处理；重大分歧需要回到 Gate 2。

---

### Stage 4 · 编写 plan.md

生成 `plan.md` 作为分阶段交付计划。

**格式**：立即读取 `references/plan-template.md` 获取完整格式。

关键规则：
- 阶段（P1、P2……）应在"有意义的交付单元"层面——如"认证层"、"文档编辑器"、"AI 集成"。不要细化到文件或 API 层面。
- 每个阶段必须可独立发布，或至少可独立测试
- 每个阶段包含：目标（一句话）、范围（包含/排除）、交付物、验收清单、规模估算（S/M/L/XL）、风险说明
- 包含依赖图——哪些阶段可以并行，哪些必须串行
- **不要写实现细节**——那是 dev 成员的工作。plan.md 只说明 what，不说明 how。
- 明确最小可行路径：哪些阶段构成最小可发布切片？

🛑 **Gate 4**：展示 plan.md，用户审查阶段边界和顺序，如需要则调整。

---

### Stage 5 · 收尾和汇报

Gate 4 通过后，更新三份文档的 frontmatter：

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"   # generated_at
git rev-parse HEAD 2>/dev/null   # git_sha（非 git 仓库时为空）
```

向用户汇报：
- 三份文档已写入：`design/<date>/discuss.md`、`discuss-result.md`、`plan.md`
- 已决策：D1~Dn（列出标题）
- 已规划：P1~Pn（列出阶段和规模估算）
- 延迟到 huashu-design 的 UI 设计工作（如有）
- 下一步："需求已就绪。运行 feature-workflow 开始实现，或直接对 P1 调用 feature-exec。"

---

## 工作风格说明

**主动而非苏格拉底式**：生成结构化提案，不要让用户填写空白。

**技术扎根**：每个推荐都应锚定在现有代码库（ARCH.md §4、§5、§7）。如果无法做到，明确说明。

**诚实泼冷水**：早期暴露风险。看起来简单的决策可能隐藏迁移成本、并发边界情况或不变量冲突。在 discuss.md 选项分析中说，而不是在代码写完之后再说。

**范围纪律**：plan.md 应代表用户要求的内容，而不是你认为应该有的东西。未被请求的范围扩展放入"未来考虑"章节，并明确标注为超出范围。

---

## 手动调用

| 触发语 | 行为 |
|---|---|
| "讨论这个功能" / "明确这个需求" | 完整 Stage 1~5 流程 |
| "重新生成 discuss.md" / "重做决策列表" | 从 Stage 2 重新开始 |
| "写 discuss-result" / "确定决策" | 跳到 Stage 3（假设对话中已完成 Gate 2）|
| "写计划" / "生成 plan.md" | 跳到 Stage 4（假设 Stage 2~3 已完成）|

---

## 参考文件

- `references/discuss-template.md` — discuss.md 完整格式及 Dn 结构示例
- `references/discuss-result-template.md` — discuss-result.md 完整格式及代码骨架示例
- `references/plan-template.md` — plan.md 完整格式及阶段结构和依赖图
