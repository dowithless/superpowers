# Superpowers

Superpowers 是一套针对编码代理的完整软件开发方法论，构建于一组可组合技能和确保代理使用这些技能的初始指令之上。

## 快速上手

为你的代理安装 Superpowers：[Claude Code](#claude-code)、[Codex CLI](#codex-cli)、[Codex App](#codex-app)、[Factory Droid](#factory-droid)、[Gemini CLI](#gemini-cli)、[OpenCode](#opencode)、[Cursor](#cursor)、[GitHub Copilot CLI](#github-copilot-cli)。

## 工作原理

它从您启动编程智能体的那一刻开始。一旦它发现您正在构建某些东西，它*不会*直接跳入尝试编写代码。相反，它会退一步，询问您真正想要实现的目标。

一旦它从对话中梳理出需求规格，它会以足够简短、便于您实际阅读和消化的块状形式展示给您。

在您确认设计之后，您的智能体会制定一个足够清晰的实施计划，即使是一个品味不佳、缺乏判断力、没有项目背景且厌恶测试的热心初级工程师也能遵循。它强调真正的红/绿测试驱动开发、YAGNI（您不会需要它）和 DRY 原则。

接下来，一旦您说“开始”，它会启动一个*子智能体驱动开发*过程，让智能体们处理每个工程任务，检查和评审他们的工作，并持续推进。Claude 通常能够自主工作数小时而不偏离您制定的计划。

其中还有更多内容，但这是系统的核心。由于技能会自动触发，您无需做任何特殊操作。您的编程智能体就拥有了 Superpowers。

## 赞助

如果 Superpowers 帮助您完成了能赚钱的事情，并且您有意愿，如果您能考虑[赞助我的开源工作](https://github.com/sponsors/obra)，我将不胜感激。

谢谢！

* Jesse

## 安装

不同工具的安装方式各有差异。如果使用多个工具，请为每个工具分别安装 Superpowers。

### Claude Code

Superpowers 可通过[官方 Claude 插件市场](https://claude.com/plugins/superpowers)获取

#### 官方市场

* 通过 Anthropic 官方市场安装插件：

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers 市场

Superpowers 市场提供 Superpowers 及其他一些与 Claude Code 相关的插件。

* 注册市场：

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

* 从该市场安装插件：

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Codex CLI

Superpowers 可通过 [官方 Codex 插件市场](https://github.com/openai/plugins) 获取。

* 打开插件搜索界面：

  ```bash
  /plugins
  ```

* 搜索 Superpowers：

  ```bash
  superpowers
  ```

* 选择 `Install Plugin`。

### Codex App

Superpowers 可通过 [官方 Codex 插件市场](https://github.com/openai/plugins) 获取。

* 在 Codex 应用中，点击侧边栏的"插件"。
* 你应在"编码"部分看到 `Superpowers`。
* 点击 Superpowers 旁的 `+`，然后按照提示操作。

### Factory Droid

* 注册市场：

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

* 安装插件：

  ```bash
  droid plugin install superpowers@superpowers
  ```

### Gemini CLI

* 安装扩展：

  ```bash
  gemini extensions install https://github.com/obra/superpowers
  ```

* 后续更新：

  ```bash
  gemini extensions update superpowers
  ```

### OpenCode

OpenCode 使用自己的插件安装系统；即使你已在其他工具中使用，也请为 OpenCode 单独安装 Superpowers。

* 告知 OpenCode：

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

* 详细文档：[docs/README.opencode.md](docs/README.opencode.md)

### Cursor

* 在 Cursor 代理聊天中，通过市场安装：

  ```text
  /add-plugin superpowers
  ```

* 或在插件市场中搜索 "superpowers"。

### GitHub Copilot CLI

* 注册市场：

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

* 安装插件：

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

## 基本工作流程

1. **头脑风暴** - 在编写代码前激活。通过提问完善粗略想法，探索替代方案，分部分呈现设计以供验证。保存设计文档。

2. **使用 Git 工作树** - 在设计批准后激活。在新分支上创建隔离的工作区，运行项目设置，验证干净的测试基线。

3. **编写计划** - 在批准设计后激活。将工作分解成小块任务（每个 2-5 分钟）。每个任务都有确切的文件路径、完整代码、验证步骤。

4. **子智能体驱动开发** 或 **执行计划** - 在计划制定后激活。为每个任务派遣新的子智能体，进行两阶段评审（规范符合性，然后是代码质量），或者分批执行并设置人工检查点。

5. **测试驱动开发** - 在实施过程中激活。强制执行 RED-GREEN-REFACTOR 循环：编写失败测试，观察其失败，编写最小化代码，观察其通过，提交。删除在测试之前编写的代码。

6. **请求代码审查** - 在任务之间激活。对照计划进行审查，按严重程度报告问题。关键问题会阻止进展。

7. **完成开发分支** - 在任务完成时激活。验证测试，呈现选项（合并/PR/保留/丢弃），清理工作树。

**智能体在任何任务前都会检查相关技能。** 这是强制性的工作流程，而非建议。

## 包含内容

### 技能库

**测试**

* **测试驱动开发** - RED-GREEN-REFACTOR 循环（包含测试反模式参考）

**调试**

* **系统化调试** - 4 阶段根本原因分析过程（包含根本原因追溯、深度防御、条件等待技术）
* **完成前验证** - 确保问题真正解决

**协作**

* **头脑风暴** - 苏格拉底式设计完善
* **编写计划** - 详细的实施计划
* **执行计划** - 带检查点的批量执行
* **派遣并行智能体** - 并发子智能体工作流
* **请求代码审查** - 预审查清单
* **接收代码审查** - 响应反馈
* **使用 Git 工作树** - 并行开发分支
* **完成开发分支** - 合并/PR 决策工作流
* **子智能体驱动开发** - 带两阶段评审（规范符合性，然后是代码质量）的快速迭代

**元技能**

* **编写技能** - 遵循最佳实践创建新技能（包含测试方法）
* **使用 superpowers** - 技能系统介绍

## 理念

* **测试驱动开发** - 始终先写测试
* **系统化优于临时性** - 流程优于猜测
* **降低复杂性** - 以简洁为主要目标
* **证据优于断言** - 在宣布成功前进行验证

阅读 [原始发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 贡献

以下是 Superpowers 的一般贡献流程。请注意，我们一般不接受新技能的贡献，且对技能的任何更新都必须在所有支持的编码代理上有效。

1. Fork 仓库
2. 切换到 'dev' 分支
3. 为你的工作创建一个分支
4. 遵循 `writing-skills` 技能来创建并测试新技能或修改技能
5. 提交 Pull 请求，务必填写拉取请求模板。

查看 `skills/writing-skills/SKILL.md` 获取完整指南。

## 更新

Superpowers 的更新在一定程度上依赖于编码代理，但通常是自动完成的。

## 许可证

MIT 许可证 - 详见 LICENSE 文件

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的其他成员共同构建。

* **Discord**: [加入我们](https://discord.gg/35wsABTejz)获取社区支持、提问以及分享你使用 Superpowers 构建的项目
* **Issues**: https://github.com/obra/superpowers/issues
* **发布公告**: [注册](https://primeradiant.com/superpowers/)以获取新版本通知
