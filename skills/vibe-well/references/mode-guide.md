# 执行模式说明

主 SKILL Stage 3 选定模式后，按本节理解模式语义。Stage 3 的环境检测和模式选择表见主 SKILL，本文件聚焦"每种模式各自怎么跑"。

## Mode A — 主 Agent 直接执行
主 Agent 读取所有上下文、实现、自审、测试、更新文档、提交。用于 S/XS 级阶段，或者 Mode B/C 都不可用的环境。

具体步骤：见 `../../feature-exec/SKILL.md` § Mode A。

## Mode B — 单 Agent 三角色（B-orchestrate）
主 Agent 充当 orchestrator/team-lead，每轮用 `Task` 工具 spawn 一次性 subagent 扮演 dev / reviewer / tester。subagent 上下文隔离实现"角色独立性"，跨轮上下文通过 `<design_root>/.reviews/` 历史文档保持。无需 TeamCreate，Cursor / Codex / 标准 Claude 都能跑。用于 M 级阶段，L/XL 退而求其次。

关键规则：
- 每次 spawn 一个 subagent 只做一件事（写 Pn / 审 Pn / 写 TC / 跑 TC / 修 fail）
- 每次 reviewer spawn 必须传入"前 N 轮反馈历史"，避免反复
- 角色卡在 `role-cards/{dev,reviewer,tester}.md`，不要把内容塞进 prompt
- 主 Agent 自己**不写代码**——它只 orchestrate

具体步骤：见 `../../feature-exec/SKILL.md` § Mode B。

## Mode C — Agent 团队
通过 `TeamCreate` 创建团队，用 `Agent` 工具的 `team_name` + `name` 参数同时 spawn 三名成员（dev、reviewer、tester）。成员之间通过 `SendMessage` 按名字寻址通信。主 Agent 的名字是 `team-lead`，成员上报时 `to` 参数写 `team-lead`。用于 L/XL 级阶段。

具体步骤：见 `../../feature-exec/SKILL.md` § Mode C。完整成员启动 prompt 模板：`subagent-prompts.md`。

## 三种模式的共同约束

- Dev 不审查自己的代码
- Reviewer 跨轮一致——同一个上下文/同一份历史
- 未经主 Agent 授权不得扩大范围
- 提交是交付的一部分——ARCH.md + feat.md + test_case.md + 代码全部提交后阶段才算完成

## 模式选择速查

| 阶段规模 | 首选 | 备选 |
|---|---|---|
| S / XS | A | A |
| M | B（环境支持时）/ C | A 拆子任务 |
| L / XL | C / B（无 TeamCreate 时）| B → A |

详细环境检测和决策表见主 SKILL.md § Stage 3。
