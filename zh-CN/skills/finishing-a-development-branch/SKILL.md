---
name: finishing-a-development-branch
description: 当实现完成、所有测试通过且需要决定如何集成工作时使用 - 通过提供合并、拉取请求或清理的结构化选项，指导开发工作的完成
---

# 完成开发分支

## 概述

通过呈现清晰选项并处理所选工作流，指导完成开发工作。

**核心原则：** 验证测试 → 检测环境 → 提供选项 → 执行选择 → 清理。

**开始时声明：** "我正在使用完成开发分支技能来完成这项工作。"

## 流程

### 步骤 1：验证测试

**在呈现选项之前，验证测试通过：**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**

```
测试失败（<N> 个失败项）。完成前必须修复：

[显示失败详情]

测试通过前无法进行合并/拉取请求操作。
```

停止。不要继续到步骤 2。

**如果测试通过：** 继续到步骤 2。

### 步骤 2：检测环境

**在提供选项之前确定工作区状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这将决定显示哪个菜单以及清理工作的方式：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（正常仓库） | 标准4选项 | 无需清理工作树 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准4选项 | 基于来源（见步骤6） |
| `GIT_DIR != GIT_COMMON`，分离头指针 | 精简3选项（无合并） | 不清理（外部管理） |

### 步骤 3：确定基础分支

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或询问："此分支是从 main 分支分出来的 - 是否正确？"

### 步骤 4：提供选项

**正常仓库和命名分支工作树 — 仅展示以下4个选项：**

```
实施完成。您想做什么？

1. 在本地合并回 <base-branch>
2. 推送并创建 Pull Request
3. 保持分支原样（我稍后处理）
4. 丢弃此项工作

请选择哪个选项？
```

**分离头指针 — 仅展示以下3个选项：**

```
实现完成。您处于分离头指针状态（外部管理工作区）。

1. 推送为新分支并创建拉取请求
2. 保持现状（稍后处理）
3. 放弃本次工作

选择哪个选项？
```

**不要添加解释** - 保持选项简洁。

### 步骤 5：执行选择

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

# Only after merge succeeds: cleanup worktree (Step 6), then delete branch
```

然后：清理工作树（步骤6），然后删除分支：

```bash
git branch -d <feature-branch>
```

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

**不要清理工作树** — 用户需要它活跃以迭代 PR 反馈。

#### 选项 3：保持原样

报告："保持分支 <名称>。工作树保留在 <路径>。"

**不要清理工作树。**

#### 选项 4：丢弃

**首先确认：**

```
这将永久删除：
- 分支 <name>
- 所有提交：<commit-list>
- 位于 <path> 的工作树

输入 'discard' 以确认。
```

等待确切的确认。

如果确认：

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理工作树（步骤6），然后强制删除分支：

```bash
git branch -D <feature-branch>
```

### 步骤 6：清理工作区

**仅对选项1和4执行。** 选项2和3始终保留工作树。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 正常仓库，无需清理工作树。完成。

**如果工作树路径位于 `.worktrees/`、`worktrees/` 或 `~/.config/superpowers/worktrees/` 下：** Superpowers 创建了此工作树 — 我们负责清理。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # Self-healing: clean up any stale registrations
```

**否则：** 主机环境（测试框架）拥有此工作区。**不要**移除它。如果你的平台提供了一个工作区退出工具，请使用它。否则，保持工作区不变。

## 快速参考

| 选项 | 合并 | 推送 | 保留工作树 | 清理分支 |
|--------|-------|------|---------------|----------------|
| 1. 本地合并 | 是 | - | - | 是 |
| 2. 创建 PR | - | 是 | 是 | - |
| 3. 保持原样 | - | - | 是 | - |
| 4. 丢弃 | - | - | - | 是（强制） |

## 常见错误

**跳过测试验证**

* **问题：** 合并损坏的代码，创建失败的 PR
* **修复：** 在提供选项之前始终验证测试

**开放式问题**

* **问题：** “我接下来应该做什么？”表述含糊
* **修复：** 精确展示4个结构化选项（分离头指针则展示3个）

**为选项2清理工作树**

* **问题：** 用户迭代 PR 所需的工作树被移除
* **修复：** 仅对选项1和4进行清理

**在移除工作树之前删除分支**

* **问题：** 由于工作树仍引用该分支，`git branch -d` 失败
* **修复：** 先合并，再移除工作树，最后删除分支

**从工作树内部运行 git worktree remove**

* **问题：** 当当前工作目录位于即将移除的工作树内部时，命令会静默失败
* **修复：** 在 `git worktree remove` 之前，始终 `cd` 返回主仓库根目录

**清理测试框架拥有的工作树**

* **问题：** 移除由测试框架创建工作树会导致幻影状态
* **修复：** 只清理位于 `.worktrees/`、`worktrees/` 或 `~/.config/superpowers/worktrees/` 下的工作树

**丢弃时无确认**

* **问题：** 意外删除工作
* **修复：** 要求输入 "discard" 确认

## 危险信号

**切勿：**

* 在测试失败的情况下继续操作
* 未在结果上验证测试即进行合并
* 未经确认删除工作
* 未经明确请求强制推送
* 在确认合并成功前移除工作树
* 清理不是你创建的工作树（来源检查）
* 从工作树内部运行 `git worktree remove`

**始终：**

* 提供选项前先验证测试
* 展示菜单前检测环境
* 精确展示4个选项（分离头指针则展示3个）
* 对选项4获取输入确认
* 仅对选项1和4清理工作树
* 工作树移除前先 `cd` 到主仓库根目录
* 移除后运行 `git worktree prune`
