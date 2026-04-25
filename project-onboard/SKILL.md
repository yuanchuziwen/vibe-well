---
name: project-onboard
version: 1.0.0
description: 当用户想要"初始化新项目"、"初始化项目文档"、"设置 ARCH.md"、"更新项目文档"、"同步文档与代码"、"检查文档是否最新"、"重新生成架构文档"，或任何其他技能检测到 ARCH.md / feat.md 缺失或过期时使用此技能。探索代码库，创建或更新三文件文档系统（CLAUDE.md / ARCH.md / feat.md），并通过 git SHA 跟踪文档新鲜度。
---

# Project Onboard

探索代码库并创建或更新项目文档系统。设计用于在任何功能工作流开始时调用，或在文档感觉过期时手动调用。

## 文档系统

四个文件，各有特定职责：

| 文件 | 用途 | 受众 |
|---|---|---|
| `CLAUDE.md` | 入口索引——指针 + 新鲜度状态 | Claude Code（自动加载）|
| `ARCH.md` | 架构——模块、数据模型、不变量、请求流 | 做规划/审查的 Claude Agent |
| `feat.md` | 功能地图——每个用户可见功能及其入口点和文件位置 | 做计划和测试的 Claude Agent |
| `test_case.md` | 与 feat.md 对应的功能测试用例——每个功能一组 TC | tester 成员、regression-test 技能 |

除非已有其他既定约定，四个文件均存放在项目根目录。

`test_case.md` 生成策略：
- **全量扫描（首次运行）**：从 feat.md 生成初始版本——每个功能行生成一个覆盖正常路径 + 一个关键错误路径的 TC。所有 TC 在 Notes 列标注 `bootstrapped`。这些是从路由和组件签名机械推断的，未经产品需求验证——只是起点，不代表基准事实。
- **增量更新**：不新增 TC（那是 tester 的工作）。仅维护与 feat.md 的一致性：废弃已删除功能的 TC、更新路由变更的入口点。
- **不覆盖 tester 编写的 TC** — 如果某个 TC 行在 Notes 列没有 `bootstrapped`，视为人工编写并完整保留。

## 新鲜度检测

每份文档携带一个 frontmatter 头部：

```yaml
---
generated_at: 2026-04-24T10:00:00Z
git_sha: a1b2c3d4
git_dirty: false
dirty_files: []
---
```

检查新鲜度：
1. 读取 `ARCH.md` frontmatter 中的 `git_sha`
2. 运行 `git diff <sha> HEAD --name-only -- src/` 列出上次同步后的变更文件
3. 运行 `git status --short` 检测未提交的变更
4. 分类结果：

| 状态 | 含义 |
|---|---|
| ✅ 最新 | 自 `git_sha` 以来无变更，工作目录干净 |
| ⚠️ 过期 | 自 `git_sha` 以来存在提交——列出变更文件 |
| 🔶 有未提交变更 | 存在未提交的变更——记录但不阻断 |

未提交变更：记录 `git_dirty: true` 并在 `dirty_files` 中列出文件。**不强制提交或 stash**——那是用户的决定。

## 更新策略

默认使用增量更新，在必要时升级到全量重新生成。

**增量**（快速，保留手工注释）：
- 计算 `git diff <last_sha> HEAD --name-only`
- 只重新检查 diff 中的文件
- 更新 ARCH.md 和 feat.md 中受影响的章节
- 保留所有其他内容

**全量重新生成**（在以下情况自动升级）：
- ARCH.md 或 feat.md 尚不存在
- diff 涉及超过 20 个文件
- diff 包含结构性文件：`package.json`、`pnpm-lock.yaml`、数据库 schema 文件、`tsconfig.json`、主入口点
- 用户明确请求（"重新生成文档"、"全量更新"）

## 工作流

### Step 1 — 检测状态

```bash
# 检查文档是否存在
ls ARCH.md feat.md CLAUDE.md 2>/dev/null

# 如果 ARCH.md 存在，读取其 git_sha，然后：
git diff <sha> HEAD --name-only -- src/
git status --short
```

向用户用一行报告当前状态：
- "文档缺失——将进行全量扫描"
- "文档过期——自上次同步以来 N 个文件变更，进行增量更新"
- "文档最新——未检测到变更 ✅"
- "文档最新但工作目录有未提交变更——已记录 N 个未提交文件"

🛑 **Gate**：如果文档最新，询问用户是否继续。如果过期或缺失，自动继续。

### Step 2 — 检查 test_case.md（仅增量）

进行增量更新时，检查 feat.md 中被删除或修改的行在 test_case.md 中是否有对应 TC：

```bash
# 检查 test_case.md 是否存在
ls test_case.md 2>/dev/null
```

如果存在且 feat.md 行被删除：找到匹配的 TC 并将其移入 `## 已废弃` 章节，注明当前日期和原因。

如果 feat.md 行被修改（路由变更、组件重命名）：更新 TC 的 `Entry Point` 列以匹配。

project-onboard 阶段不新增 TC——那是 feature-exec 中 tester 的工作。

---

### Step 3 — 探索（仅全量扫描）

全量扫描时，**立即读取 `references/explore-guide.md` 并按其探索顺序执行。** 关键目标摘要：

- 目录结构（前 2 层）
- `package.json` / `pnpm-workspace.yaml` — 技术栈、脚本
- 入口点（`src/index.ts`、`app.ts`、`main.ts`、`server.ts`）
- 路由文件——所有 API 端点
- 数据库 schema 文件——表、列、关系
- Service / Repository 层——核心业务逻辑边界
- 前端：组件树、页面路由
- 现有文档（`README.md`、`docs/`）
- CI/CD 配置（`.github/`、`Dockerfile`）用于部署上下文

### Step 3 — 探索（仅增量）

只读取 diff 中的文件及其直接调用方/依赖方。确认 ARCH.md 和 feat.md 中哪些章节受影响。

### Step 4 — 生成/更新文档

按此顺序生成：ARCH.md → feat.md → test_case.md → CLAUDE.md（CLAUDE.md 最后，因为它引用其他文件）。

**立即读取 `references/doc-templates.md` 获取每份文档的完整格式，再开始编写。**

关键规则：
- ARCH.md：遵循模板中的章节结构——不要发明新的顶级章节
- feat.md：细粒度，每个功能一行，按领域分组；后端和前端分开章节
- test_case.md（全量扫描）：对 feat.md 中的每一行，生成一个正常路径 TC 和一个关键错误路径 TC，全部在 Notes 列标注 `bootstrapped`。不猜测边界情况——紧贴路由/组件签名的含义。
- test_case.md（增量）：仅按 Step 2 的规则与 feat.md 变更同步——不新增 TC
- CLAUDE.md：只做精简索引——不要将 ARCH.md 或 feat.md 的内容复制进 CLAUDE.md

### Step 5 — 更新 frontmatter

写入后，在每份文件的 frontmatter 中记录当前 git 状态：

```bash
git rev-parse HEAD          # 用于 git_sha
git status --short          # 用于 git_dirty + dirty_files
date -u +"%Y-%m-%dT%H:%M:%SZ"   # 用于 generated_at
```

### Step 6 — CLAUDE.md 新鲜度摘要

CLAUDE.md 的头部块必须反映所有文档的当前状态：

```markdown
> ARCH.md       — 同步于 a1b2c3d（2026-04-24）✅
> feat.md       — 同步于 a1b2c3d（2026-04-24）✅
> test_case.md  — 24 个 TC（18 bootstrapped，6 已验证）· 最后更新 2026-04-24
```

对 test_case.md：统计 TC 总行数，计算 Notes 列中仍有 `bootstrapped` 的行数与没有的行数。这给用户一个快速信号，了解有多少测试覆盖已被真实 tester 验证。

如果 `git_dirty: true`：
```markdown
> ARCH.md — 同步于 a1b2c3d（2026-04-24）🔶 有未提交变更（生成时有 3 个未提交文件）
```

### Step 7 — 汇报

向用户说明：
- 创建或更新了哪些文档
- 变更了多少章节（增量）或总共写了多少章节（全量）
- 是否记录了 `git_dirty`
- 下一步："文档已就绪。你现在可以运行功能工作流，或在 ARCH.md / feat.md 查看文档。"

## 手动调用

用户可以随时直接触发本技能：

| 触发语 | 行为 |
|---|---|
| "更新文档" / "同步文档" | 新鲜度检查 → 过期则增量，最新则无操作 |
| "重新生成文档" / "全量更新" | 无论新鲜度如何，强制全量重新生成 |
| "检查文档是否最新" | 仅做新鲜度检查——报告状态，不写入 |
| "初始化项目文档" | 全量扫描（视为首次运行）|

## 参考文件

- `references/explore-guide.md` — 新项目的完整代码库探索顺序
- `references/doc-templates.md` — ARCH.md、feat.md、CLAUDE.md 的完整模板及章节说明
