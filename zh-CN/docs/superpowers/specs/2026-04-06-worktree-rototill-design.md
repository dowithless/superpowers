# 工作树翻耕：检测与延缓

**日期：** 2026-04-06
**状态：** 草稿
**工单：** PRI-974
**取代：** PRI-823 (Codex 应用兼容性)

## 问题

Superpowers 对工作树管理有主张——特定路径（`.worktrees/<branch>`）、特定命令（`git worktree add`）、特定清理（`git worktree remove`）。与此同时，Claude Code、Codex 应用、Gemini CLI 和 Cursor 都提供原生工作树支持，包括各自的路径、生命周期管理和清理。

这产生了三种故障模式：

1. **重复** — 在 Claude Code 上，技能做了 `EnterWorktree`/`ExitWorktree` 已经做的事情
2. **冲突** — 在 Codex 应用上，技能试图在已管理的工作树内创建工作树
3. **幻象状态** — 技能创建的位于 `.worktrees/` 的工作树对框架不可见；框架创建的位于 `.claude/worktrees/` 的工作树对技能不可见

对于没有原生支持的框架（Codex CLI、OpenCode、Copilot standalone），superpowers 填补了实际空白。技能不应消失——它应该在存在原生支持时让路。

## 目标

1. 当存在原生工作树系统时，将任务延迟给它们
2. 继续为缺乏原生支持的框架提供工作树支持
3. 修复完成开发分支时的三个已知错误（#940、#999、#238）
4. 使工作树创建变为可选而非强制（#991）
5. 用平台中性的语言替换硬编码的 `CLAUDE.md` 引用（#1049）

## 非目标

* 按工作树的环境约定（`.worktree-env.sh`、端口偏移）— 第4阶段
* 用于路径强制的 PreToolUse 钩子 — 第4阶段
* 多仓库工作树文档 — 第4阶段
* 工作树的 Brainstorming 检查清单变更 — 第4阶段
* `.superpowers-session.json` 元数据跟踪（有趣的 PR #997 想法，但 v1 不需要）
* 将钩子符号链接进入工作树（PR #965 想法，单独关注点）

## 设计原则

### 检测状态，而非平台

使用 `GIT_DIR != GIT_COMMON` 来确定“我是否已经在工作树中？”而不是通过嗅探环境变量来识别框架。这是一个稳定的 git 原语（自 git 2.5，2015年起），在所有框架中通用有效，且随着新框架的出现无需维护。

### 声明性意图，规定性后备

技能描述目标（“确保工作在隔离的工作区中进行”）并在原生工具可用时推迟给它们。它只在框架缺乏原生工作树支持时作为后备规定具体的 git 命令。步骤1a优先，明确命名原生工具（`EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`）；步骤1b在后，提供 git 后备方案。原始规范让步骤1a保持抽象（“你了解自己的工具包”），但 TDD 证明，当步骤1a过于模糊时，智能体会锚定步骤1b的具体命令。明确命名工具和同意授权桥接对于使偏好可靠是必要的。

### 基于来源的拥有权

谁创建工作树，谁拥有其清理权。如果框架创建了它，superpowers 就不触碰它。如果 superpowers 创建了它（通过 git 后备），那么 superpowers 将其清理。启发式方法：如果工作树位于 `.worktrees/` 或 `~/.config/superpowers/worktrees/` 下，则 superpowers 拥有它。任何其他位置（`.claude/worktrees/`、`~/.codex/worktrees/`、`.gemini/worktrees/`）都归框架所有。

## 设计

### 1. `using-git-worktrees` SKILL.md 重写

该技能在创建之前增加了三个新步骤，并简化了创建流程。

#### 步骤0：检测现有隔离

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

三种结果：

| 条件 | 含义 | 操作 |
|------|------|------|
| `GIT_DIR == GIT_COMMON` | 正常仓库检出 | 继续步骤0.5 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 已在链接的工作树中 | 跳至步骤3（项目设置）。报告：“已在 `<path>` 上的隔离工作区中，分支为 `<name>`。” |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 外部管理工作树（例如 Codex 应用沙箱） | 跳至步骤3。报告：“已在 `<path>` 上的隔离工作区中（分离 HEAD，外部管理）。” |

步骤0不关心谁的创建了工作树或哪个框架正在运行。工作树就是工作树，无论来源如何。

**子模块保护：** `GIT_DIR != GIT_COMMON` 在 git 子模块内部也为 true。在得出“已在工作树中”结论之前，检查我们是否不在子模块中：

```bash
# If this returns a path, we're in a submodule, not a worktree
git rev-parse --show-superproject-working-tree 2>/dev/null
```

如果在子模块中，则视为 `GIT_DIR == GIT_COMMON`（继续步骤0.5）。

#### 步骤0.5：同意

当步骤0发现没有现有隔离时（`GIT_DIR == GIT_COMMON`），在创建前询问：

> “您希望我设置一个隔离的工作树吗？这将保护您当前的分支免受更改影响。（y/n）”

如果同意，继续步骤1。如果否决，原地工作——跳过步骤3，不建立工作树。

当步骤0检测到现有隔离时，此步骤完全跳过（没必要询问已经存在的事情）。

#### 步骤1a：原生工具（首选）

> 用户已请求隔离工作区（步骤0同意）。检查您可用的工具——您有 `EnterWorktree`、`WorktreeCreate`、`/worktree` 命令或 `--worktree` 标志吗？如果有：用户同意创建的工作树是您使用它的授权。立即使用它并跳过步骤3。

使用原生工具后，跳至步骤3（项目设置）。

**设计说明—TDD 修订：** 原始规范使用了简短的、抽象的步骤1a（“你了解自己的工具包——技能不需要命名特定工具”）。TDD 验证推翻了这一点：智能体锚定在步骤1b的具体 git 命令上，忽略了抽象指导（通过率2/6）。三个更改修复了问题（绿色和压力测试下通过率50/50）：

1. **明确命名工具** — 通过名称列出 `EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`，将决定从解释（“我有原生工具吗？”）转换为事实查找（“`EnterWorktree` 在我的工具列表中吗？”）。没有这些工具的平台上的智能体简单地检查，找不到，然后回退到步骤1b。未观察到误报。
2. **同意桥接** — “用户的同意创建的工作树是您使用它的授权”直接处理 `EnterWorktree` 的工具级护栏（“仅当用户明确请求时”）。工具描述覆盖技能指令 (Claude Code #29950)，因此技能必须将用户同意作为工具授权的框架。
3. **红色标志条目** — 在红色标志部分命名特定的反模式（“当您有原生工作树工具时使用 `git worktree add`——这是号错误”）。

文件拆分（将步骤1b放在一个单独的技能中）经过测试，证明没有必要。锚定问题已经通过步骤1a文本的质量解决，而不是通过物理分离 git 命令。使用完整的240行技能（所有 git 命令可见）的控制测试通过了20/20。

#### 步骤1b：Git 工作树后备方案

当没有原生工具可用时，手动创建工作树。

**目录选择**（优先级顺序）：

1. 检查现有 `.worktrees/` 或 `worktrees/` 目录——如果找到，则使用。如果两者都存在，`.worktrees/` 优先。
2. 检查现有 `~/.config/superpowers/worktrees/<project>/` 目录——如果找到，则使用（向后兼容传统全局路径）。
3. 检查项目的智能体指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules 或等效文件）中的工作树目录偏好。
4. 默认使用 `.worktrees/`。

没有交互式目录选择提示。不再为新用户提供全局路径（`~/.config/superpowers/worktrees/`）作为选择，但会检测并使用该位置的现有工作树，以实现向后兼容。

**安全验证**（仅限项目本地目录）：

```bash
git check-ignore -q .worktrees 2>/dev/null
```

如果未忽略，则添加到 `.gitignore` 并在继续前提交。

**创建：**

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**钩子感知：** Git 工作树不继承父仓库的钩子目录。通过步骤1b创建的工作树后，如果存在，将主仓库的钩子目录符号链接过去：

```bash
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

这可以防止在工作移至工作树时，预提交检查、linter 和其他钩子静默停止。（想法来自 PR #965。）

**沙箱回退：** 如果 `git worktree add` 因权限错误而失败，则视为受限环境。跳过创建工作，在当前目录工作，继续步骤3。

**步骤编号说明：** 当前技能有步骤1-4作为平面列表。本重新设计使用0、0.5、1a、1b、3、4。没有步骤2——它是旧的单一“创建隔离工作区”，现已拆分为1a/1b结构。实现应当干净地重新编号（例如，0 → “步骤0：检测”，0.5 → 在步骤0流程内，1a/1b →“步骤1”，3 →“步骤2”，4 →“步骤3”）或保留当前编号并附上注释。由实现者决定。

#### 步骤3-4：项目设置和基线测试（不变）

无论通过哪条路径创建的在工作空间（步骤0检测到现有、步骤1a原生工具、步骤1b git 回退，或者根本没有工作树），执行汇聚：

* **步骤3：** 自动检测并运行项目设置（`npm install`、`cargo build`、`pip install`、`go mod download` 等）
* **步骤4：** 运行测试套件。如果测试失败，报告失败并询问是否继续。

### 2. `finishing-a-development-branch` SKILL.md 重写

完成技能增加了环境检测并修复三个错误。

#### 步骤1：验证测试（不变）

运行项目的测试套件。如果测试失败，停止。不提供完成选项。

#### 步骤1.5：检测环境（新增）

重新运行与创建的步骤0相同的检测：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

三条路径：

| 状态 | 菜单 | 清理 |
|------|------|------|
| `GIT_DIR == GIT_COMMON`（正常仓库） | 标准4个选项 | 无需清理工作树 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准4个选项 | 基于来源的（见步骤5） |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 简化菜单：推送为新分支并创建 PR、保持原样、丢弃 | 无合并选项（无法从分离 HEAD 合并） |

#### 步骤2：确定基础分支（不变）

#### 步骤3：呈现选项

**正常仓库和命名分支工作树：**

1. 在本地合并回 `<base-branch>`
2. 推送并创建拉取请求
3. 保持分支不变（我稍后处理）
4. 丢弃这项工作

**分离 HEAD：**

1. 作为新分支推送并创建拉取请求
2. 保持原样（我稍后处理）
3. 丢弃这项工作

#### 步骤4：执行选择

**选项1（在本地合并）：**

```bash
# Get main repo root for CWD safety (Bug #238 fix)
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# Merge first, verify success before removing anything
git checkout <base-branch>
git pull
git merge <feature-branch>
<run tests>

# Only after merge succeeds: remove worktree, then delete branch (Bug #999 fix)
git worktree remove "$WORKTREE_PATH"  # only if superpowers owns it
git branch -d <feature-branch>
```

顺序至关重要：合并 → 验证 → 移除工作树 → 删除分支。旧技能在移除工作树之前删除了分支（失败，因为工作树仍引用该分支）。对移除工作树先行的简单修复也是错误的——如果随后合并失败，工作目录就不存在了，更改就会丢失。

**选项2（创建 PR）：**

推送分支，创建 PR。不要清理工作树——用户需要它进行 PR 迭代。（修复错误 #940：移除自相矛盾的“然后：清理工作树”的措辞。）

**选项3（保持原样）：** 无操作。

**选项4（丢弃）：** 需要键入“discard”确认。然后移除工作树（如果 superpowers 拥有它），强制删除分支。

#### 步骤5：清理（已更新）

```
if GIT_DIR == GIT_COMMON:
    # 普通仓库，没有工作树需要清理
    done

if worktree path is under .worktrees/ or ~/.config/superpowers/worktrees/:
    # Superpowers 创建了它——我们负责清理
    cd to main repo root       # Bug #238 fixes
    git worktree remove <path>

else:
    # Harness 创建了它——不要触碰
    # 如果平台提供 workspace-exit 工具，请使用它
    # 否则保留工作树原样
```

仅对选项1和4运行清理。选项2和3始终保留工作树。（修复错误 #940。）

**过期工作树清理：** 在任何 `git worktree remove` 之后，运行 `git worktree prune` 作为自我修复步骤。工作树目录可能被带外删除（例如，被框架清理、手动 `rm` 或 `.claude/` 清理），留下过期的注册信息导致令人困惑的错误。一行代码，防止无声腐败。（想法来自 PR #1072。）

### 3. 集成更新

#### `subagent-driven-development` 和 `executing-plans`

两者目前在其集成部分都将 `using-git-worktrees` 列为“必需（REQUIRED）”。更改为：

> `using-git-worktrees` — 确保工作区隔离（创建一个或验证已有的）

技能本身现在处理同意步骤（步骤 0.5）和检测步骤（步骤 0），因此调用技能无需进行门控或提示。

#### `writing-plans`

移除过时的声明“应在专用工作树中运行（由 brainstorming 技能创建）”。Brainstorming 是一个设计技能，不创建工作树。工作树提示在执行时通过 `using-git-worktrees` 进行。

### 4. 平台无关的指令文件引用

所有与工作树相关的技能中硬编码的 `CLAUDE.md` 实例均被替换为：

> “你项目的代理指令文件（CLAUDE.md, GEMINI.md, AGENTS.md, .cursorrules 或等效文件）”

这适用于步骤 1b 中的目录偏好检查。

## 错误修复（打包）

| 错误 | 问题 | 修复 | 位置 |
|-----|---------|-----|----------|
| #940 | 选项 2 的描述说“然后：清理工作树（步骤 5）”，但快速参考说保留它。步骤 5 说“仅适用于选项 1、2、4”，但常见错误部分说“仅选项 1 和 4。” | 从选项 2 中移除清理操作。步骤 5 仅适用于选项 1 和 4。 | finishing SKILL.md |
| #999 | 选项 1 在移除工作树之前删除分支。`git branch -d` 可能失败，因为工作树仍引用该分支。 | 重新排序为：合并 → 验证测试 → 移除工作树 → 删除分支。必须在移除任何内容之前成功完成合并。 | finishing SKILL.md |
| #238 | 如果当前工作目录在要移除的工作树内部，`git worktree remove` 会静默失败。 | 添加 CWD 守卫：在执行 `git worktree remove` 之前，`cd` 到主仓库根目录。 | finishing SKILL.md |

## 已解决问题

| 问题 | 解决方式 |
|-------|-----------|
| #940 | 直接修复（错误 #940） |
| #991 | 步骤 0.5 中的选择同意 |
| #918 | 步骤 0 检测 + 步骤 1.5 的 finishing 检测 |
| #1009 | 通过步骤 1a 解决 — 代理使用本地工具（例如 `EnterWorktree`），这些工具在 harness 原生路径下创建。依赖于步骤 1a 正常工作；请参阅风险。 |
| #999 | 直接修复（错误 #999） |
| #238 | 直接修复（错误 #238） |
| #1049 | 平台无关的指令文件引用 |
| #279 | 通过检测并推迟解决 — 由于我们不覆盖路径，因此原生路径得到尊重 |
| #574 | **已推迟。** 本规范中的任何内容均未触及错误所在的 brainstorming 技能。完整的修复（向 brainstorming 的检查清单添加工作树步骤）属于第 4 阶段。 |

## 风险

### 步骤 1a 是主要假设 — 已解决

步骤 1a — 代理优先使用本地工作树工具而非 git 后备方案 — 是整个设计的基础。如果代理忽略步骤 1a，而在具有本地支持的 harness 上回退到步骤 1b，则“检测并推迟”将完全失败。

**状态：** 此风险在实施过程中浮现。原始的抽象步骤 1a（“你了解自己的工具包”）在 Claude Code 上以 2/6 失败。TDD 门控按设计工作 — 它在任何技能文件被修改之前捕获了失败，从而防止了错误版本的发布。三次“重新设计”迭代确定了根本原因（代理锁定具体命令，工具说明护栏覆盖了技能指令）并制定了一个在 GREEN 和 PRESSURE 测试中以 50/50 验证的修复。详情请参见上面的步骤 1a 设计说明。

**跨平台验证：**

截至 2026-04-06，Claude Code 是唯一具有代理可调用的会话中工作树工具（`EnterWorktree`）的 harness。所有其他工具要么在代理启动前创建工作树（Codex App、Gemini CLI、Cursor），要么没有本地工作树支持（Codex CLI、OpenCode）。步骤 1a 是向前兼容的：当其他 harness 添加代理可调用的工作树工具时，代理将使它们与命名的示例匹配，并在无需技能更改的情况下使用它们。

| Harness | 当前工作树模型 | 技能机制 | 测试情况 |
|---------|----------------------|-----------------|--------|
| Claude Code | 代理可调用的 `EnterWorktree` | 步骤 1a | 50/50（GREEN + PRESSURE） |
| Codex CLI | 无本地工具（仅限 Shell） | 步骤 1b git 后备方案 | 6/6（`codex exec`） |
| Gemini CLI | 启动时使用 `--worktree` 标志，无代理工具 | 如果使用标志启动，则执行步骤 0；否则执行步骤 1b | 步骤 0：1/1，步骤 1b：1/1（`gemini -p`） |
| Cursor Agent | 面向用户的 `/worktree`，无代理工具 | 如果用户激活，则执行步骤 0；否则执行步骤 1b | 步骤 0：1/1，步骤 1b：1/1（`cursor-agent -p`） |
| Codex App | 平台管理，Detached HEAD，无代理工具 | 步骤 0 检测现有状态 | 1/1 模拟 |
| OpenCode | 仅检测（`ctx.worktree`），无代理工具 | 步骤 1b git 后备方案 | 未测试（无 CLI 访问） |

**剩余风险：**

1. 如果 Anthropic 更改 `EnterWorktree` 的工具说明，使其限制性更强（例如，“不要基于技能指令使用”），则同意桥梁断开。值得提交一个问题，请求工具说明应适应技能驱动的调用。
2. 当其他 harness 添加代理可调用的工作树工具时，它们可能使用不在步骤 1a 列表中的名称。应随着新工具的出现更新该列表。通用表述（“一个工作树或工作区隔离工具”）提供了一些前瞻性覆盖。

### 来源启发式

`.worktrees/` 或 `~/.config/superpowers/worktrees/` = 我们的，其他任何内容 = 放手 `heuristic works for every current harness. If a future harness adopts`.worktrees/`as its convention, we'd have a false positive (superpowers tries to clean up a harness-owned worktree). Similarly, if a user manually runs`git worktree add .worktrees/experiment`图标`without superpowers, we'd incorrectly claim ownership. Both are low risk — every harness uses branded paths, and manual`.worktrees/\` 创建不太可能 — 但仍值得注意。

### Detached HEAD 的 finishing

针对 Detached HEAD 工作树的简化菜单（无合并选项）对于 Codex App 的沙箱模型是正确的。如果用户因其他原因位于 Detached HEAD，此简化菜单仍然有意义 — 在没有首先创建分支的情况下，你确实无法从 Detached HEAD 合并。

## 实施说明

两个技能文件包含超出核心步骤的部分，这些部分在实施期间需要更新：

* **前言部分**（`name`，`description`）：更新以反映“检测并推迟”行为
* **快速参考表**：重写以匹配新的步骤结构和错误修复
* **常见错误部分**：更新或移除引用旧行为的条目（例如，“跳过 CLAUDE.md 检查”现在不正确）
* **危险信号部分**：更新以反映新的优先级（例如，“当步骤 0 检测到现有隔离时，切勿创建工作树”）
* **集成部分**：更新技能之间的交叉引用

本规范描述了*更改内容*；实施计划将指定对这些次要部分的具体编辑。

## 未来工作（不在此规范中）

* **第 3 阶段剩余部分：** `$TMPDIR` 目录选项（#666），用于缓存和环境继承的设置文档（#299）
* **第 4 阶段：** 用于路径执行的 PreToolUse 钩子协议（#1040），每个工作树的环境规范（#597），brainstorming 清单工作树步骤（#574），多仓库文档（#710）
