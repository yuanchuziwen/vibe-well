# XS Mini Plan Template

XS 快速通道的最小计划模板。**适用场景**：变更不引入新模块/新表/新 API/新依赖，不影响 ARCH.md 核心章节，预估 ≤ 1 小时实现。

不写 discuss.md / discuss-result.md / plan.md / Pn.md。本文件即为唯一规划产出，写在 `design/<YYYYMMDD>-<feature-slug>/mini-plan.md`。

---

```markdown
---
generated_at: YYYY-MM-DDTHH:MM:SSZ
git_sha: <sha>
mode: xs
verification_strategy: <unit-tdd | curl-acceptance | visual-snapshot | visual-diff | manual | regression-suite>
---

# Mini Plan: <一句话描述变更>

## 意图
<1~2 句话说明为什么要做这个变更>

## 修改点（≤ 5 项）
- `<file:line 范围>`：<改什么>
- `<file:line 范围>`：<改什么>

## 验收（每项 ≤ 2 分钟可验证）
- [ ] <可机械验证的检查>
- [ ] <可机械验证的检查>

## 范围 — 不做
- <明确不在本次变更内的事项，避免范围蔓延>

## 风险
<None / 一句话说明>
```

---

## 执行约定

走 XS 通道时：

1. **跳过 Gate 1~3**（不做需求讨论 / worktree / 模式选择对话）
2. 默认在当前分支直接工作（不开 worktree）
3. 默认 Mode A（主 Agent 直接执行）
4. **TDD 是否使用看 `verification_strategy`**：
   - `unit-tdd` → 按 feature-exec Mode A 流程跑 TDD Red/Green
   - 其他策略 → 按对应策略验证后直接 commit
5. 完成后只走 **Gate 4（交付确认）+ Gate 5（收尾）**
6. 仍需要更新 feat.md（如有用户可见变化）和 test_case.md（如新增测试）；ARCH.md 通常无需更新（XS 的定义就是不动架构）

## 何时退出 XS 通道升级到完整流程

执行过程中如果发现：
- 实际改动跨越多个模块
- 需要新增/修改数据 schema
- 实现时间超过 2 小时仍未完成
- 引入了新依赖

→ **停下来上报用户**："这个变更不再属于 XS 范围，建议升级到完整流程（Stage 1 起走）"。不要硬撑。
