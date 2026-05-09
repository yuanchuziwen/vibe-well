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

  # Regression-test skill（可选）
  regression_default_strategy: risk   # full | risk (默认) | incremental | tagged:<domain>
  prompt_regression_on_finish: true   # Stage 5 全量测试通过后是否提示用户跑回归

# Hooks（可选）
# 在工作流的 6 个时点触发用户脚本。每个 hook 独立配置；不配置则跳过。
hooks:
  on_plan_approved:    null  # plan.md 用户批准后
  on_phase_started:    null  # feature-exec 进入 init 阶段后
  on_pn_approved:      null  # Pn.md PASS（仅 M/L/XL 阶段）
  on_phase_committed:  null  # git commit 完成、delivered 之前
  on_phase_delivered:  null  # 阶段交付报告输出后
  on_feature_finished: null  # Stage 5 收尾完成

# Hook 配置格式（任一时点）：
# hooks:
#   on_phase_committed:
#     command: "scripts/post-commit.sh"   # 必填—相对 git 仓库根的命令；shell 执行
#     mode:    warn                        # warn (默认) | block
#     timeout_ms: 60000                    # 可选—默认 60s
# Hook 进程的 stdin 自动注入 JSON 上下文：
#   { phase: {...}, mode: "A|B|C|xs", commit_sha?, design_root, context_ref? }
# Hook 退出码：0 视为成功；非 0：mode=warn 警告并继续，mode=block 阻断主流程

# 项目覆盖（可选）
# 当前 git 仓库根路径作为 key 时，覆盖上面的 defaults / hooks
projects:
  "/Users/me/proj-a":
    worktree_strategy: phase
    branch_template: "ai/{slug}/p{n}"
    hooks:
      on_phase_committed:
        command: "make ci-fast"
        mode: warn
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

## Hook 执行规则

主 Agent / feature-exec 在每个 hook 时点检查：

```
1. 合并后的 hooks.<event> 是否为 null/缺失？是 → 跳过
2. 是 → 取 command + mode + timeout_ms（默认 60000）
3. 在 git 仓库根目录用 shell 执行 command，stdin 注入 JSON：
   {
     "event":       "on_phase_committed",
     "phase":       { "num": 1, "name": "auth-layer", "size": "M" },
     "mode":        "B",
     "commit_sha":  "<sha or null>",
     "design_root": "<abs path>",
     "context_ref": "<abs path or null>",
     "timestamp":   "<ISO 8601>"
   }
4. 等待最多 timeout_ms ms：
   - 退出码 0：日志记录"hook <event> 通过"，继续
   - 退出码非 0：
     * mode=warn  → 打印 stderr 摘要 + 继续
     * mode=block → 打印 stderr 全文 + 询问用户：忽略 / 修复后重跑 / 中止
   - 超时：当作非 0 处理
5. Hook 输出到 <design_root>/.vibe-well/hooks.log（每行 JSON：event/start/end/exit）
```

**关键约束**：
- Hook 失败**不应自动修复**——主 Agent 不要替用户判断"小问题"
- Hook 命令**不接受参数列表**——所有参数走 stdin JSON，避免 shell 转义问题
- `mode=block` 只用于真正不能跳过的检查（如 lint / format）；其它一律 warn

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
