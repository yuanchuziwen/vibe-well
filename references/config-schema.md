# vibe-well 用户偏好配置

文件位置：`~/.vibe-well/config.yaml`（用户级，跨项目）

可选；不存在则使用本文件中标注的"默认值"。主 SKILL 启动时读取，问问题时把对应值作为默认选项。

## Schema

```yaml
# ~/.vibe-well/config.yaml

defaults:
  # Stage 0.5 复杂度快筛
  skip_gate_when_trivial: true        # 默认 true—XS 全否时直接走快速通道；false—始终走完整流程

  # Stage 2 worktree
  worktree_strategy: feature          # none | feature | phase
  branch_template: "feat/{slug}/p{n}-{phase_slug}"
  # 可用占位符：{slug}（feature-slug）、{n}（阶段编号）、{phase_slug}（阶段名 slug）

  # Stage 3 模式选择
  prefer_mode_b_over_c: false         # true—M 级优先用 Mode B 而非 Mode C（适合不支持 TeamCreate 的环境，留给后续 Mode B 落地后启用；当前未实现 Mode B 时此项无效）

  # Stage 4 测试命令兜底
  test_command: "pnpm test"           # 找不到测试框架时使用；空字符串则跳过

  # 全量测试触发条件
  run_full_test_on_xs: false          # XS 完成后是否跑全量测试（默认 false，节省时间）
  run_full_test_on_finish: true       # Stage 5 收尾前是否跑全量测试（默认 true）

# 项目覆盖（可选）
# 当前 git 仓库根路径作为 key 时，覆盖上面的 defaults
projects:
  "/Users/me/proj-a":
    worktree_strategy: phase
    branch_template: "ai/{slug}/p{n}"
```

## 读取规则

```
1. 检查 ~/.vibe-well/config.yaml 是否存在
   - 不存在：使用本文件 Schema 中标注的"默认值"
   - 存在：读取 defaults 节
2. 检查 projects.<当前 git 根> 是否存在覆盖
   - 存在：用项目覆盖深度合并到 defaults 之上
3. 用户在交互中的回答始终覆盖配置（一次性，不持久化）
```

## 提示语规范

向用户提问时把缓存值作为默认选项，括号注明来源：

```
"使用功能级 worktree（你的偏好默认值）？[Y/n]"
"分支命名 feat/user-auth/p1-auth-layer（按你的模板）？[Y/n]"
```

## 何时主动写入配置

主 SKILL **不自动写入** `~/.vibe-well/config.yaml`。用户明确说"以后都这样" / "保存为默认" / "记住这个偏好"时，才写入对应字段。每次写入后向用户确认变更。

## 创建配置文件

如果用户首次运行 vibe-well 且想配置偏好，可建议：

```bash
mkdir -p ~/.vibe-well
cat > ~/.vibe-well/config.yaml <<'EOF'
defaults:
  skip_gate_when_trivial: true
  worktree_strategy: feature
  branch_template: "feat/{slug}/p{n}-{phase_slug}"
  test_command: "pnpm test"
EOF
```
