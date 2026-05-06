# Worktree Rototill 实现计划

> **对于智能代理工作者：** 必需的子技能：使用 superpowers：subagent-driven-development（推荐）或 superpowers：executing-plans 来按任务实现此计划。步骤使用复选框 (`- [ ]`) 语法进行追踪。

**目标：** 使 superpowers 在可用时优先使用原生工作树系统，不可用时回退到手动 git worktree，并修复三个已知的完成阶段错误。

**架构：** 重写两个技能文件 (`using-git-worktrees`，`finishing-a-development-branch`)，三个文件进行单行集成更新 (`executing-plans`，`subagent-driven-development`，`writing-plans`)。核心更改是添加检测 (`GIT_DIR != GIT_COMMON`) 和原生工具优先的创建路径。这些是 Markdown 技能指令文件，而非应用程序代码——"测试"是使用 testing-skills-with-subagents TDD 框架的代理行为测试。

**技术栈：** Markdown（技能文件），bash（测试脚本），Claude Code CLI（`claude -p` 用于无头测试）

**规格：** `docs/superpowers/specs/2026-04-06-worktree-rototill-design.md`

***

### 任务 1：关卡 — 第 1a 步（原生工具偏好）的 TDD 验证

第 1a 步是整个设计的承重假设。如果代理不优先使用原生工作树工具而非 `git worktree add`，则规格失败。请首先进行验证，然后再触碰任何技能文件。

**文件：**

* 创建：`tests/claude-code/test-worktree-native-preference.sh`
* 读取：`skills/using-git-worktrees/SKILL.md`（当前版本，用于 RED 基线）
* 读取：`tests/claude-code/test-helpers.sh`（用于 `run_claude`，`assert_contains` 等）
* 读取：`skills/writing-skills/testing-skills-with-subagents.md`（TDD 框架）

**此任务是一个关卡。** 如果 GREEN 阶段在 2 次 REFACTOR 迭代后失败，则应停止。不要继续执行任务 2。请报告问题——创建方法需要重新设计。

* \[ ] **第 1 步：编写 RED 基线测试脚本**

创建将在不包含和包含更新技能文本的情况下运行场景的测试脚本。RED 阶段针对当前技能运行（该技能没有第 1a 步）。

```bash
#!/usr/bin/env bash
# Test: Does the agent prefer native worktree tools (EnterWorktree) over git worktree add?
# Framework: RED-GREEN-REFACTOR per testing-skills-with-subagents.md
#
# RED:   Current skill has no native tool preference. Agent should use git worktree add.
# GREEN: Updated skill has Step 1a. Agent should use EnterWorktree on Claude Code.

set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

# Pressure scenario: realistic implementation task where agent needs isolation
SCENARIO='IMPORTANT: This is a real task. Choose and act.

You need to implement a small feature (add a "version" field to package.json).
This should be done in an isolated workspace to protect the main branch.

You have the using-git-worktrees skill available. Set up the isolated workspace now.
Do NOT actually implement the feature — just set up the workspace and report what you did.

Respond with EXACTLY what tool/command you used to create the workspace.'

echo "=== Worktree Native Preference Test ==="
echo ""

# Phase selection
PHASE="${1:-red}"

if [ "$PHASE" = "red" ]; then
    echo "--- RED PHASE: Running WITHOUT Step 1a (current skill) ---"
    echo "Expected: Agent uses 'git worktree add' (no native tool awareness)"
    echo ""

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"
    mkdir -p .worktrees

    output=$(run_claude "$SCENARIO" 120)

    echo "Agent output:"
    echo "$output"
    echo ""

    # RED expectation: agent uses git worktree add (current behavior)
    if echo "$output" | grep -qi "EnterWorktree"; then
        echo "[UNEXPECTED] Agent used EnterWorktree WITHOUT Step 1a — skill may not be needed"
        echo "Investigate: is Claude Code's default behavior already correct?"
    else
        echo "[RED CONFIRMED] Agent did NOT use EnterWorktree"
        assert_contains "$output" "git worktree" "Agent used git worktree (expected in RED)"
    fi

    cleanup_test_project "$test_dir"

elif [ "$PHASE" = "green" ]; then
    echo "--- GREEN PHASE: Running WITH Step 1a (updated skill) ---"
    echo "Expected: Agent uses EnterWorktree instead of git worktree add"
    echo ""

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"

    output=$(run_claude "$SCENARIO" 120)

    echo "Agent output:"
    echo "$output"
    echo ""

    # GREEN expectation: agent uses EnterWorktree
    assert_contains "$output" "EnterWorktree" "Agent used native EnterWorktree tool"
    assert_not_contains "$output" "git worktree add" "Agent did NOT fall back to git worktree add"

    cleanup_test_project "$test_dir"

elif [ "$PHASE" = "pressure" ]; then
    echo "--- PRESSURE PHASE: Multiple pressures to skip native tools ---"
    echo ""

    PRESSURE_SCENARIO='IMPORTANT: This is a real task. Choose and act.

You need to set up an isolated workspace URGENTLY. Production is impacted and
the fix needs to go in a worktree. You know git worktree add works reliably.
The .worktrees/ directory already exists and is gitignored.

You have the using-git-worktrees skill. Set up the workspace NOW.
Speed matters. Report EXACTLY what tool/command you used.'

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"
    mkdir -p .worktrees
    echo ".worktrees/" >> .gitignore

    output=$(run_claude "$PRESSURE_SCENARIO" 120)

    echo "Agent output:"
    echo "$output"
    echo ""

    # Should STILL use EnterWorktree even under pressure
    assert_contains "$output" "EnterWorktree" "Agent used native tool even under time pressure"
    assert_not_contains "$output" "git worktree add" "Agent resisted falling back to git despite pressure"

    cleanup_test_project "$test_dir"
fi

echo ""
echo "=== Test Complete ==="
```

* \[ ] **第 2 步：运行 RED 阶段 — 确认代理当前使用 git worktree add**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh red`

预期结果：`[RED CONFIRMED] Agent did NOT use EnterWorktree` — 代理使用 `git worktree add`，因为当前技能没有原生工具偏好。

逐字记录代理的确切输出和任何理由。这是技能必须修复的基线失败。

* \[ ] **第 3 步：如果 RED 已确认，继续执行。编写第 1a 步技能文本。**

创建一个临时的技能测试版本，仅包含第 1a 步的添加内容（最小更改以隔离变量）。在技能创建指令的顶部，现有目录选择过程之前，添加此部分：

```markdown
## 步骤 1：创建隔离工作空间

**你有两种机制。请按此顺序尝试。**

### 1a. 原生工作树工具（推荐）

如果你的平台提供了工作树或工作空间隔离工具，请使用它。你对自己的工具集了如指掌 —— 此技能无需指定具体工具。原生工具会自动处理目录放置、分支创建和清理工作。

使用原生工具后，请跳至步骤 3（项目设置）。

### 1b. Git 工作树备用方案

如果不存在原生工具，请使用 git 手动创建工作树。
```

* \[ ] **第 4 步：运行 GREEN 阶段 — 确认代理现在使用 EnterWorktree**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh green`

预期结果：`[PASS] Agent used native EnterWorktree tool`

如果失败：逐字记录代理的确切输出和理由。这是一个 REFACTOR 信号——第 1a 步文本需要修订。尝试最多 2 次 REFACTOR 迭代。如果 2 次迭代后仍然失败，请停止并报告。

* \[ ] **第 5 步：运行 PRESSURE 阶段 — 确认代理在压力下坚持回退方案**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh pressure`

预期结果：`[PASS] Agent used native tool even under time pressure`

如果失败：逐字记录理由。在第 1a 步文本中添加明确的计数器（例如，一个红色标志条目："当你的平台提供原生工作树工具时，永远不要使用 git worktree add"）。重新运行。

* \[ ] **第 6 步：提交测试脚本**

```bash
git add tests/claude-code/test-worktree-native-preference.sh
git commit -m "test: add RED/GREEN validation for native worktree preference (PRI-974)

Gate test for Step 1a — validates agents prefer EnterWorktree over
git worktree add on Claude Code. Must pass before skill rewrite."
```

***

### 任务 2：重写 `using-git-worktrees` SKILL.md

完全重写创建工作技能。完全替换现有文件。

**文件：**

* 修改：`skills/using-git-worktrees/SKILL.md`（完全重写，219 行 → 约 210 行）

**依赖于：** 任务 1 GREEN 通过。

* \[ ] **第 1 步：编写完整的新 SKILL.md**

将 `skills/using-git-worktrees/SKILL.md` 的全部内容替换为：

````markdown
---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - ensures an isolated workspace exists via native tools or git worktree fallback
---

# 使用 Git Worktrees

## 概述

确保工作在隔离的工作区中执行。优先使用平台原生的 worktree 工具。仅在无原生工具可用时回退到手动 git worktrees。

**核心原则：** 先检测现有隔离。然后使用原生工具。最后回退到 git。永远不要与框架对抗。

**开始时声明：** "我正在使用 git-worktrees 技能来设置隔离的工作区。"

## 第0步：检测现有隔离状态

**在创建任何内容之前，请检查是否已经在隔离的工作区中。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)

````

**子模块保护：** `GIT_DIR != GIT_COMMON` 在 git 子模块内部也为真。在得出结论"已在工作树中"之前，请验证您不在子模块中：

```bash
# If this returns a path, you're in a submodule, not a worktree — proceed to Step 1
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不是子模块）：** 您已在链接的工作树中。跳到第 3 步（项目设置）。不要创建另一个工作树。

使用分支状态报告：

* 在分支上："已在 `<path>` 的隔离工作区中，位于分支 `<name>`。"
* 分离 HEAD："已在 `<path>` 的隔离工作区中（分离 HEAD，外部管理）。完成时需要创建分支。"

**如果 `GIT_DIR == GIT_COMMON`（或在子模块中）：** 您处在正常的仓库检查中。

您的指令中是否已经指明了用户的工作树偏好？如果没有，请在进行工作树创建前请求同意：

> "您是否希望我设置一个隔离的工作树？它可以保护您当前的分支不受更改影响。"

尊重任何已有的声明偏好，无需再次询问。如果用户拒绝同意，则就地工作并跳到第 3 步。

## 第 1 步：创建隔离工作区

**您有两种机制。请按此顺序尝试。**

### 1a. 原生工作树工具（首选）

如果您的平台提供了工作树或工作区隔离工具，请使用它。您了解自己的工具包——技能不需要命名特定工具。原生工具会自动处理目录放置、分支创建和清理。

使用原生工具后，跳到第 3 步（项目设置）。

### 1b. Git 工作树回退

如果原生工具不可用，请使用 git 手动创建工作树。

#### 目录选择

按以下优先级顺序操作：

1. **检查现有目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # 首选（隐藏）
   ls -d worktrees 2>/dev/null      # 备选
   ```
   如果找到，使用该目录。如果两者都存在，`.worktrees` 胜出。

2. **检查现有全局目录：**
   ```bash
   project=$(basename "$(git rev-parse --show-toplevel)")
   ls -d ~/.config/superpowers/worktrees/$project 2>/dev/null
   ```
   如果找到，则使用它（与旧版全局路径向后兼容）。

3. **检查您的指令中是否有工作树目录偏好。** 如果指定了，则无需询问直接使用。

4. **默认为 `.worktrees/`。**

#### 安全检查（仅限项目本地目录）

**必须在创建工作树之前验证目录是否被忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 添加到 .gitignore，提交更改，然后继续。

**为何关键：** 防止意外将工作树内容提交到仓库中。

全局目录（`~/.config/superpowers/worktrees/`）无需验证。

#### 创建工作树

```bash
project=$(basename "$(git rev-parse --show-toplevel)")

# Determine path based on chosen location
# For project-local: path="$LOCATION/$BRANCH_NAME"
# For global: path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

#### 钩子感知

Git 工作树不会继承父仓库的钩子目录。创建工作树后，如果存在钩子，则从主仓库创建符号链接：

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

这可以防止项目迁移到工作树时，预提交检查、linter 和其他钩子静默停止。

**沙盒回退：** 如果 `git worktree add` 因权限错误（沙盒拒绝）而失败，则将其视为受限环境。跳过创建，在当前目录中运行设置和基线测试，据此报告。

## 第 3 步：项目设置

自动检测并运行相应的设置：

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## 第 4 步：验证干净的基线

运行测试以确保工作区启动时是干净的：

```bash
# Use project-appropriate command
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，询问是继续还是调查。

**如果测试通过：** 报告就绪。

### 报告

```
工作目录已就绪于 <完整路径>
测试通过（<N> 个测试，0 个失败）
已准备好实现 <功能名称>
```

## 快速参考

| 情况 | 操作 |
|-----------|--------|
| 已在链接的工作树中 | 跳过创建（第 0 步） |
| 在子模块中 | 视为普通仓库（第 0 步保护） |
| 原生工作树工具可用 | 使用它（第 1a 步） |
| 无原生工具 | Git 工作树回退（第 1b 步） |
| `.worktrees/` 存在 | 使用它（验证被忽略） |
| `worktrees/` 存在 | 使用它（验证被忽略） |
| 两者都存在 | 使用 `.worktrees/` |
| 都不存在 | 检查指令文件，然后默认为 `.worktrees/` |
| 全局路径存在 | 使用它（向后兼容） |
| 目录未被忽略 | 添加到 .gitignore + 提交 |
| 创建时权限错误 | 沙盒回退，就地工作 |
| 基线测试期间失败 | 报告失败 + 询问 |
| 没有 package.json/Cargo.toml | 跳过依赖安装 |

## 常见错误

### 对抗工具链

* **问题：** 在平台已提供隔离的情况下使用 `git worktree add`
* **修复：** 第 0 步检测现有隔离。第 1a 步优先使用原生工具。

### 跳过检测

* **问题：** 在现有工作树内部创建嵌套工作树
* **修复：** 在创建任何内容之前始终运行第 0 步

### 跳过忽略验证

* **问题：** 工作树内容被追踪，污染 git status
* **修复：** 在创建项目本地工作树之前始终使用 `git check-ignore`

### 假设目录位置

* **问题：** 造成不一致，违反项目约定
* **修复：** 遵循优先级：现有 > 指令文件 > 默认

### 在测试失败的情况下继续

* **问题：** 无法区分新错误与预先存在的问题
* **修复：** 报告失败，获得明确许可才能继续

## 红色标志

**绝不：**

* 在第 0 步检测到现有隔离时创建工作树
* 在原生工作树工具可用时使用 git 命令
* 创建工作树而不验证它是否被忽略（项目本地）
* 跳过基线测试验证
* 在测试失败的情况下不问继续

**始终：**

* 首先运行第 0 步检测
* 优先使用原生工具而非 git 回退
* 遵循目录优先级：现有 > 指令文件 > 默认
* 验证项目本地目录是否被忽略
* 自动检测并运行项目设置
* 验证干净的测试基线
* 通过 1b 创建工作树后创建钩子符号链接

## 集成

**由以下调用：**

* **subagent-driven-development** - 确保隔离的工作区（创建一个或验证现有）
* **executing-plans** - 确保隔离的工作区（创建一个或验证现有）
* 任何需要隔离工作区的技能

**与以下配对使用：**

* **finishing-a-development-branch** - 工作完成后进行清理必需

````
- [ ] **第二步：验证文件读取正确**

运行：`wc -l skills/using-git-worktrees/SKILL.md`

预期：大约 200-220 行。检查是否存在任何 Markdown 格式问题。

- [ ] **第三步：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat: rewrite using-git-worktrees with detect-and-defer (PRI-974)

Step 0: GIT_DIR != GIT_COMMON detection (skip if already isolated)
Step 0 consent: opt-in prompt before creating worktree (#991)
Step 1a: native tool preference (short, first, declarative)
Step 1b: git worktree fallback with hooks symlink and legacy path compat
Submodule guard prevents false detection
Platform-neutral instruction file references (#1049)"
````

***

### 任务 3：重写 `finishing-a-development-branch` SKILL.md

完全重写完成工作技能。添加环境检测，修复三个错误，添加基于来源的清理。

**文件：**

* 修改：`skills/finishing-a-development-branch/SKILL.md`（完全重写，201 行 → 约 220 行）

* \[ ] **第 1 步：编写完整的新 SKILL.md**

将 `skills/finishing-a-development-branch/SKILL.md` 的全部内容替换为：

````markdown
---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# 完成开发分支

## 概述

通过提供明确的选项并处理所选工作流程，引导完成开发工作。

**核心原则：** 验证测试 → 检测环境 → 呈现选项 → 执行选择 → 清理。

**开始时说明：** "我正在使用完成开发分支技能来完成这项工作。"

## 流程

### 步骤 1：验证测试

**在提供选项之前，请确保测试通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
````

**如果测试失败：**

```
测试失败（<N> 个失败）。必须在完成前修复：

[显示失败]

测试通过之前，不能继续合并/PR。
```

停止。不要继续执行第 2 步。

**如果测试通过：** 继续执行第 2 步。

### 第 2 步：检测环境

**在显示选项前确定工作区状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

此决定显示哪个菜单以及清理工作的方式：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 无需清理工作树 |
| `GIT_DIR != GIT_COMMON`，已命名分支 | 标准 4 选项 | 基于来源（参见第 6 步） |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 精简 3 选项（无合并） | 无需清理（外部管理） |

### 第 3 步：确定基础分支

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或询问："此分支是从主分支分裂出来的 - 是否正确？"

### 第 4 步：呈现选项

**普通仓库和已命名分支工作树 — 精确呈现以下 4 个选项：**

```
实施完成。您希望怎么做？

1. 将变更合并到 <base-branch> 本地分支
2. 提交并创建拉取请求
3. 保持当前分支不变（稍后处理）
4. 放弃此操作

请选择选项？
```

**分离 HEAD — 精确呈现以下 3 个选项：**

```
实现完成。您处于分离的 HEAD 状态（外部管理工作区）。

1. 作为新分支推送，并创建拉取请求
2. 保持原样（稍后自行处理）
3. 放弃此工作

选哪个选项？
```

**不加解释** - 保持选项简洁。

### 第 5 步：执行选择

#### 选项 1：本地合并

```bash
# Get main repo root for CWD safety
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# Merge first — verify success before removing anything
git checkout <base-branch>
git pull
git merge <feature-branch>

# Verify tests on merged result
<test command>

# Only after merge succeeds: remove worktree, then delete branch
# (See Step 6 for worktree cleanup)
git branch -d <feature-branch>
```

然后：清理工作树（第 6 步）

#### 选项 2：推送并创建 PR

```bash
# Push branch
git push -u origin <feature-branch>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

**不要清理工作树** — 用户需要保持活动状态以迭代 PR 反馈。

#### 选项 3：保持现状

报告："正在保留分支 <名称>。工作树保留在 <路径>。"

**不要清理工作树。**

#### 选项 4：丢弃

**先确认：**

```
这将永久删除：
- 分支 <名称>
- 所有提交：<提交列表>
- 路径 <路径> 下的工作树

输入 'discard' 以确认。
```

等待确切确认。

如果确认：

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理工作树（第 6 步），然后强制删除分支：

```bash
git branch -D <feature-branch>
```

### 第 6 步：清理工作区

**仅适用于选项1和4。** 选项2和3始终保留工作树。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 普通仓库，无需清理工作树。已完成。

**如果工作树路径位于 `.worktrees/` 或 `~/.config/superpowers/worktrees/` 下：** 此工作树由 Superpowers 创建 — 我们有清理权。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # Self-healing: clean up any stale registrations
```

**否则：** 此工作区由宿主环境（测试平台）所有。**切勿**删除。如果你所在的平台提供了退出工作区的工具，请使用它。否则，请将工作区留在原地。

## 快速参考

| 选项 | 合并 | 推送 | 保留工作树 | 清理分支 |
|------|------|------|----------|----------|
| 1. 本地合并 | 是 | - | - | 是 |
| 2. 创建PR | - | 是 | 是 | - |
| 3. 保持原样 | - | - | 是 | - |
| 4. 丢弃 | - | - | - | 是（强制） |

## 常见错误

**跳过测试验证**

* **问题:** 合并损坏的代码，创建失败的PR
* **修复:** 在提供选项前始终先验证测试

**开放性问题**

* **问题:** “下一步我该做什么？” 是一个含糊的问题
* **修复:** 精确提供4个结构化的选项（若是处于已分离头指针状态，则提供3个）

**清理选项2的工作树**

* **问题:** 移除用户进行PR迭代所需的工作树
* **修复:** 仅清理选项1和4

**在移除工作树前删除分支**

* **问题:** `git branch -d` 失败，因为工作树仍引用该分支
* **修复:** 先合并，然后移除工作树，最后删除分支

**从其内部运行 git worktree remove**

* **问题:** 当CWD位于正在被移除的工作树内部时，命令静默失败
* **修复:** 在 `git worktree remove` 之前始终 `cd` 到主仓库根目录

**清理测试平台拥有的工作树**

* **问题:** 移除由测试平台创建的工作树会导致幻影状态
* **修复:** 仅清理位于 `.worktrees/` 或 `~/.config/superpowers/worktrees/` 下的工作树

**丢弃操作无确认**

* **问题:** 意外删除工作成果
* **修复:** 要求键入 “discard” (丢弃) 以进行确认

## 红色标志

**绝不：**

* 在测试失败的情况下继续进行
* 在未验证合并结果测试的情况下进行合并
* 未经确认即删除工作
* 无明确请求就进行强制推送
* 在确认合并成功前就移走工作树
* 清理不是你创建的工作树（溯源检查）
* 在工作树内部运行 `git worktree remove`

**始终：**

* 提供选项前验证测试
* 显示菜单前检测环境
* 精确呈现4个选项（对于已分离头指针状态则为3个）
* 获取“选项4”的键入确认
* 仅清理位于选项1和4的工作树
* 在移除工作树前 `cd` 到主仓库根目录
* 移除后运行 `git worktree prune`

## 集成

**由以下调用：**

* **subagent-driven-development**（第7步） - 所有任务完成后
* **executing-plans**（第5步） - 所有批次完成后

**与以下配对使用：**

* **using-git-worktrees** - 清理该技能创建的工作树

````

- [ ] **步骤 2：验证文件读取正确**

运行：`wc -l skills/finishing-a-development-branch/SKILL.md`

预期：大约 210-230 行。

- [ ] **步骤 3：提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat: rewrite finishing-a-development-branch with detect-and-defer (PRI-974)

Step 2: environment detection (GIT_DIR != GIT_COMMON) before presenting menu
Detached HEAD: reduced 3-option menu (no merge from detached HEAD)
Provenance-based cleanup: .worktrees/ = ours, anything else = hands off
Bug #940: Option 2 no longer cleans up worktree
Bug #999: merge -> verify -> remove worktree -> delete branch
Bug #238: cd to main repo root before git worktree remove
Stale worktree pruning after removal (git worktree prune)"
````

***

### 任务4：集成更新

对引用 `using-git-worktrees` 的三个文件进行单行更改。

**文件：**

* 修改: `skills/executing-plans/SKILL.md:68`

* 修改: `skills/subagent-driven-development/SKILL.md:268`

* 修改: `skills/writing-plans/SKILL.md:16`

* \[ ] **第1步：更新 executing-plans 集成行**

在 `skills/executing-plans/SKILL.md` 中，将第68行从：

```markdown
- **superpowers:using-git-worktrees** - 必需：在开始前设置独立的工作空间
```

更改为：

```markdown
- **superpowers:using-git-worktrees** - 确保独立的工作区（创建一个或验证现有工作区）
```

* \[ ] **第2步：更新 subagent-driven-development 集成行**

在 `skills/subagent-driven-development/SKILL.md` 中，将第268行从：

```markdown
- **superpowers:using-git-worktrees** - 必需：在开始前设置独立的工作空间
```

更改为：

```markdown
- **superpowers:using-git-worktrees** - 确保独立的工作区（创建一个或验证现有工作区）
```

* \[ ] **第3步：更新 writing-plans 上下文行**

在 `skills/writing-plans/SKILL.md` 中，将第16行从：

```markdown
**上下文：** 应在专用工作树（由brainstorming技能创建）中运行。
```

更改为：

```markdown
**上下文：** 如果在隔离工作树中进行操作，该工作树应已通过执行时的 using-git-worktrees 技能创建。
```

* \[ ] **第4步：全部提交三者**

```bash
git add skills/executing-plans/SKILL.md skills/subagent-driven-development/SKILL.md skills/writing-plans/SKILL.md
git commit -m "fix: update worktree integration references across skills (PRI-974)

Remove REQUIRED language from executing-plans and subagent-driven-development.
Consent and detection now live inside using-git-worktrees itself.
Fix stale 'created by brainstorming' claim in writing-plans."
```

***

### 任务5：端到端验证

验证整个重写后的技能是否能协同工作。运行现有的测试套件并进行手动验证。

**文件：**

* 读取: `tests/claude-code/run-skill-tests.sh`

* 读取: `skills/using-git-worktrees/SKILL.md` (验证最终状态)

* 读取: `skills/finishing-a-development-branch/SKILL.md` (验证最终状态)

* \[ ] **第1步：运行现有测试套件**

运行: `cd tests/claude-code && bash run-skill-tests.sh`

预期: 所有现有测试通过。如果有任何测试失败，请进行调查——集成更改（任务4）可能破坏了内容断言。

* \[ ] **第2步：重新运行第1a步的GREEN测试**

运行：`cd tests/claude-code && bash test-worktree-native-preference.sh green`

预期: 通过 — 代理仍使用最终的技能文本与EnterWorktree交互（而不仅仅是任务1中第1a步的最小化新增）。

* \[ ] **第3步：手动验证 — 从头到尾阅读两个重写后的技能**

完整阅读 `skills/using-git-worktrees/SKILL.md` 和 `skills/finishing-a-development-branch/SKILL.md`。检查：

1. 没有提及旧行为（硬编码的 `CLAUDE.md`、交互式目录提示、“REQUIRED” 语言）
2. 每向文件中步骤编号一致
3. 快速参考表格与论述相符
4. 跨章节的集成环节引用正确
5. 无Markdown格式问题

* \[ ] **第4步：验证git状态是否干净**

运行: `git status`

预期: 工作树干净。所有更改已通过任务1-4提交。

* \[ ] **第5步：如果需要任何修复，进行最终提交**

如果手动验证发现了问题，修复它们并提交：

```bash
git add -A
git commit -m "fix: address review findings in worktree skill rewrite (PRI-974)"
```

如果未发现问题，跳过此步骤。
