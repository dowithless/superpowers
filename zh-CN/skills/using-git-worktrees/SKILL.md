---
name: using-git-worktrees
description: 在开始需要从当前工作空间隔离的功能开发，或执行实施计划之前使用——确保通过原生工具或git worktree回退存在一个隔离的工作空间
---

# 使用 Git Worktrees

## 概述

确保工作在隔离的工作区内进行。优先使用平台的原生工作树工具。仅在没有原生工具可用时，回退到手动使用 git 工作树。

**核心原则：** 首先检测现有隔离。然后使用原生工具。最后回退到 git。永远不要和基础设施对抗。

**开始时声明：** "我正在使用 using-git-worktrees 技能来设置一个隔离的工作空间。"

## 第0步：检测现有隔离

**在创建任何内容前，检查是否已在隔离工作区内。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**子模块保护：** `GIT_DIR != GIT_COMMON` 在 git 子模块中也为真。在判断 "已在工作树中" 前，验证你不在子模块中：

```bash
# If this returns a path, you're in a submodule, not a worktree — treat as normal repo
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON` （且不是子模块）：** 你已处于链接的工作树中。跳转到第3步（项目设置）。不要重复创建工作树。

附带分支状态报告：

* 在分支上："已在隔离工作区 `<path>` 的分支 `<name>` 上。"
* 分离 HEAD："已在隔离工作区 `<path>`（分离 HEAD，外部管理）。在完成时需要创建分支。"

**如果 `GIT_DIR == GIT_COMMON`（或在子模块中）：** 你处于正常的仓库检出中。

用户是否已在你的指令中指定了他们的工作树偏好？如果没有，在创建工作树前征得同意：

> "您希望我建立一个隔离工作树吗？它可以保护当前分支免受更改影响。"

如果已声明偏好则遵守，无需询问。如果用户拒绝，则在原地工作并跳转到第3步。

## 第1步：创建隔离工作空间

**你有两种机制。按此顺序尝试。**

### 1a. 原生工作树工具（首选）

用户已请求隔离工作空间（第0步同意）。你是否有办法创建工作树？也许是一个工具，如 `EnterWorktree`、`WorktreeCreate`、`/worktree` 命令或 `--worktree` 标志。如果有，使用它并跳转到第3步。

原生工具能自动处理目录放置、分支创建和清理。当你有原生工具时使用 `git worktree add` 会产生基础设施无法看到或管理的虚假状态。

只有在没有原生工作树工具可用时，才继续执行第1b步。

### 1b. Git 工作树回退

**仅当第1a步不适用时使用**——你没有可用的原生工作树工具。使用 git 手动创建工作树。

#### 目录选择

按此优先级顺序。用户显式偏好始终优先于观察到的文件系统状态。

1. **检查指令中是否有声明的 worktree 目录偏好。** 如果用户已指定，则使用，无需询问。

2. **检查是否存在现有的项目本地 worktree 目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # 首选（隐藏）
   ls -d worktrees 2>/dev/null      # 备选
   ```
   如果找到，则使用。如果两者都存在，`.worktrees` 优先。

3. **检查是否存在现有的全局目录：**
   ```bash
   project=$(basename "$(git rev-parse --show-toplevel)")
   ls -d ~/.config/superpowers/worktrees/$project 2>/dev/null
   ```
   如果找到，则使用（向后兼容传统全局路径）。

4. **如果没有其他引导可用**，则默认使用项目根目录下的 `.worktrees/`。

#### 安全检查（仅项目本地目录）

**创建 worktree 前必须验证目录是否被忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 将其添加到 .gitignore，提交更改，然后继续。

**为何关键：** 防止意外地将 worktree 内容提交到仓库。

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

**沙盒回退：** 如果 `git worktree add` 因权限错误（沙盒拒绝）失败，告知用户沙盒阻止了 worktree 创建，你将在当前目录中工作。然后在原地运行项目设置和基线测试。

## 第3步：项目设置

自动检测并运行适当的设置：

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

## 第4步：验证干净的基线

运行测试以确保工作空间从干净状态开始：

```bash
# Use project-appropriate command
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，询问是继续还是调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree 准备就绪于 <full-path>
测试通过（<N> 项测试，0 失败）
准备实施 <feature-name>
```

## 快速参考

| 情况 | 操作 |
|-----------|--------|
| 已在链接工作树中 | 跳过创建（第0步） |
| 在子模块中 | 视为普通仓库（第0步保护） |
| 有原生工作树工具可用 | 使用它（第1a步） |
| 无原生工具 | Git 工作树回退（第1b步） |
| 存在 `.worktrees/` | 使用它（验证已忽略） |
| 存在 `worktrees/` | 使用它（验证已忽略） |
| 两者都存在 | 使用 `.worktrees/` |
| 两者都不存在 | 检查指令文件，然后默认使用 `.worktrees/` |
| 全局路径存在 | 使用它（向后兼容） |
| 目录未被忽略 | 添加到 .gitignore + 提交 |
| 创建时权限错误 | 沙盒回退，原地工作 |
| 基线期间测试失败 | 报告失败并询问 |
| 没有 package.json/Cargo.toml | 跳过依赖安装 |

## 常见错误

### 与基础设施对抗

* **问题：** 当平台已经提供隔离时使用 `git worktree add`
* **修复：** 第0步检测现有隔离。第1a步优先使用原生工具。

### 跳过检测

* **问题：** 在现有工作树内创建嵌套工作树
* **修复：** 始终先执行第0步

### 跳过忽略验证

* **问题：** Worktree 内容被跟踪，污染 git status
* **修复：** 创建项目本地 worktree 前始终使用 `git check-ignore`

### 假设目录位置

* **问题：** 导致不一致，违反项目约定
* **修复：** 按优先级顺序：现有 > 全局遗留 > 指令文件 > 默认

### 在测试失败的情况下继续

* **问题：** 无法区分新错误与预先存在的问题
* **修复：** 报告失败，获取明确的继续许可

## 红色警报

**绝不要：**

* 在第0步检测到现有隔离时创建工作树
* 拥有原生工作树工具时使用 `git worktree add` 例如 `EnterWorktree`。这是 #1 错误——如果有，就用它。
* 跳过第1a步直接跳到第1b步的 git 命令
* 未验证目录是否被忽略就创建工作树（项目本地）
* 跳过基线测试验证
* 未询问就继续执行失败测试

**始终要：**

* 首先执行第0步检测
* 优先使用原生工具而不是 git 回退
* 按目录优先级：现有 > 全局遗留 > 指令文件 > 默认
* 验证项目本地目录已忽略
* 自动检测并运行项目设置
* 验证干净的测试基线
