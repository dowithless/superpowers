# Codex 工具映射

技能使用 Claude Code 工具名称。当您在技能中遇到这些时，请使用您平台的等效工具：

| 技能参考 | Codex 等价项 |
|-----------------|------------------|
| `Task` 工具（分发子代理） | `spawn_agent`（参见[子代理分发需要多代理支持](#子代理分发需要多代理支持)） |
| 多次 `Task` 调用（并行） | 多次 `spawn_agent` 调用 |
| 任务返回结果 | `wait_agent` |
| 任务自动完成 | `close_agent` 释放插槽 |
| `TodoWrite`（任务跟踪） | `update_plan` |
| `Skill` 工具（调用技能） | 技能原生加载——只需遵循指示 |
| `Read`、`Write`、`Edit`（文件） | 使用原生文件工具 |
| `Bash`（运行命令） | 使用原生 shell 工具 |

## 子代理分发需要多代理支持

添加到您的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这为诸如 `dispatching-parallel-agents` 和 `subagent-driven-development` 之类的技能启用了 `spawn_agent`、`wait_agent` 和 `close_agent`。

遗留说明：`rust-v0.115.0` 之前的 Codex 版本将生成的代理等待暴露为 `wait`。当前 Codex 对生成的代理使用 `wait_agent`。`wait` 这个名称现在属于代码模式下的 `exec/wait`，通过 `cell_id` 恢复已挂起的执行单元格；它不是生成的代理结果工具。

## 环境检测

创建 worktree 或完成分支的技能应在继续之前，使用只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

* `GIT_DIR != GIT_COMMON` → 已在链接的 worktree 中（跳过创建）
* `BRANCH` 为空 → 分离的 HEAD（无法从沙盒中分支/推送/PR）

有关每个技能如何使用这些信号的示例，请参见 `using-git-worktrees` 第 0 步和 `finishing-a-development-branch` 第 1 步。

## Codex App 完成

当沙盒阻止分支/推送操作时（在外部管理的 worktree 中处于分离的 HEAD 状态），代理将提交所有工作并通知用户使用 App 的原生控件：

* **"创建分支"** —— 命名分支，然后通过 App UI 提交/推送/PR
* **"移交到本地"** —— 将工作转移到用户的本地检出

代理仍然可以运行测试、暂存文件，并输出建议的分支名称、提交消息和 PR 描述供用户复制。
