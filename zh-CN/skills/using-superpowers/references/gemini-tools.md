# Gemini CLI 工具映射

技能使用 Claude Code 工具名称。当您在技能中遇到这些名称时，请使用您平台上的等效工具：

| 技能参考 | Gemini CLI 等效项 |
|-----------------|----------------------|
| `Read` (读取文件) | `read_file` |
| `Write` (创建文件) | `write_file` |
| `Edit` (编辑文件) | `replace` |
| `Bash` (运行命令) | `run_shell_command` |
| `Grep` (搜索文件内容) | `grep_search` |
| `Glob` (按名称搜索文件) | `glob` |
| `TodoWrite` (任务追踪) | `write_todos` |
| `Skill` 工具 (调用技能) | `activate_skill` |
| `WebSearch` | `google_web_search` |
| `WebFetch` | `web_fetch` |
| `Task` 工具 (调度子代理) | `@agent-name` (参见 [子代理支持](#子代理支持)) |

## 子代理支持

Gemini CLI 通过 `@` 语法原生支持子代理。使用内置的 `@generalist` 代理来分派任何任务——它可以访问所有工具，并且会遵循您提供的提示。

当技能指示分派一个命名代理类型时，使用 `@generalist` 并附带来自该技能提示模板的完整提示：

| 技能指令 | Gemini CLI 等效项 |
|-------------------|----------------------|
| `Task tool (superpowers:implementer)` | `@generalist` 并使用已填充的 `implementer-prompt.md` 模板 |
| `Task tool (superpowers:spec-reviewer)` | `@generalist` 并使用已填充的 `spec-reviewer-prompt.md` 模板 |
| `Task tool (superpowers:code-reviewer)` | `@code-reviewer` (捆绑代理) 或 `@generalist` 并使用已填充的审查提示 |
| `Task tool (superpowers:code-quality-reviewer)` | `@generalist` 并使用已填充的 `code-quality-reviewer-prompt.md` 模板 |
| `Task tool (general-purpose)` 内联提示 | `@generalist` 并使用您的内联提示 |

### 提示填充

技能提供的提示模板包含诸如 `{WHAT_WAS_IMPLEMENTED}` 或 `[FULL TEXT of task]` 之类的占位符。填充所有占位符，并将完整的提示作为消息传递给 `@generalist`。提示模板本身包含了代理的角色、审查标准和预期的输出格式——`@generalist` 将遵循它。

### 并行分派

Gemini CLI 支持并行子代理分派。当技能要求您并行分派多个独立的子代理任务时，在同一个提示中一起请求所有的 `@generalist` 或命名的子代理任务。保持存在依赖关系的任务顺序执行，但不要仅仅为了保持历史记录简洁而对独立的子代理任务进行串行化。

## 额外的 Gemini CLI 工具

这些工具在 Gemini CLI 中可用，但没有 Claude Code 的等效工具：

| 工具 | 用途 |
|------|---------|
| `list_directory` | 列出文件和子目录 |
| `save_memory` | 跨会话将事实持久化到 GEMINI.md |
| `ask_user` | 向用户请求结构化输入 |
| `tracker_create_task` | 丰富的任务管理（创建、更新、列出、可视化） |
| `enter_plan_mode` / `exit_plan_mode` | 在进行更改之前切换到只读研究模式 |
