下面按“事件、触发时机、配置方式、执行能力、实际使用场景”整理。内容以本次查询到的两家官方文档为准：

- [OpenAI Codex Hooks 官方文档](https://developers.openai.com/codex/hooks)
- [Anthropic Claude Code Hooks 官方文档](https://code.claude.com/docs/en/hooks)

## 一、总体结论

Codex 和 Claude Code 的 hooks 核心思想相同：

```text
生命周期事件
    -> matcher 匹配
        -> 执行 command / HTTP / 其他 handler
```

两者都支持：

- 用户提交提示词时执行
- 工具调用前后执行
- 请求权限时执行
- 当前一轮结束时执行
- 整个会话结束时执行
- 子代理启动和结束时执行
- 上下文压缩前后执行

区别是：

- Codex 的 hooks 数量更少，主要覆盖核心 agent loop。
- Claude Code 的 hooks 更细，额外覆盖通知、任务、团队、工作树、配置变更、文件变化、MCP elicitation 等场景。
- Codex 当前实际执行的 handler 主要是 `command`。
- Claude Code 支持 `command`、HTTP、MCP tool、prompt、agent 等多种 handler。

## 二、Codex Hooks

Codex 官方目前列出 11 个 hook 事件：

```text
SessionStart
UserPromptSubmit
PreToolUse
PermissionRequest
PostToolUse
PreCompact
PostCompact
SubagentStart
SubagentStop
Stop
SessionEnd
```

### 1. 会话生命周期

| 事件             | 触发时机          | 典型用途                  |
| -------------- | ------------- | --------------------- |
| `SessionStart` | 会话开始或恢复时      | 加载项目说明、环境变量、工作区上下文    |
| `SessionEnd`   | 主会话真正结束时      | 保存笔记、清理临时文件、播放退出提示音   |
| `Stop`         | 当前一轮 turn 结束时 | 播放完成音、运行测试、检查是否满足完成条件 |

Codex 官方明确区分：

```text
Stop       = 当前一轮回答结束
SessionEnd = 整个主会话结束
```

`SessionEnd` 不会在每轮回答后触发，也不会对子代理触发。官方文档列出的结束场景包括正常关闭 Codex、归档或删除仍打开的会话，以及会话长时间没有连接客户端等情况。

### 2. 用户输入相关

| 事件                 | 触发时机           | 说明               |
| ------------------ | -------------- | ---------------- |
| `UserPromptSubmit` | 用户提交一条消息时      | 可以记录用户输入、注入额外上下文 |
| `Stop`             | Codex 完成当前轮输出时 | 最适合“每轮完成提示音”     |

`UserPromptSubmit` 发生在模型处理用户输入之前，而 `Stop` 发生在当前 turn 停止之后。

### 3. 工具调用相关

| 事件                  | 触发时机      | 典型用途              |
| ------------------- | --------- | ----------------- |
| `PreToolUse`        | 工具调用之前    | 阻止危险命令、修改参数、做策略检查 |
| `PermissionRequest` | 工具需要用户授权时 | 播放权限提示音、记录授权请求    |
| `PostToolUse`       | 工具调用完成之后  | 检查结果、记录日志、触发后处理   |

Codex 的 matcher 可以匹配工具名称，例如：

```text
Bash
apply_patch
Edit
Write
mcp__context7__query_docs
```

Codex 官方文档还说明，MCP 工具和其他本地工具也可以通过工具名称进入 `PreToolUse` 和 `PostToolUse` hook。

需要注意，Codex 当前没有单独列出以下事件：

```text
PostToolUseFailure
PostToolBatch
PermissionDenied
StopFailure
Notification
```

也就是说，Codex 的工具失败、API 错误、批量工具完成等场景，没有 Claude Code 那么细的专门 hook 事件。

### 4. 上下文压缩相关

| 事件 | 触发时机 |
|---|---|
| `PreCompact` | 上下文压缩之前 |
| `PostCompact` | 上下文压缩完成之后 |

matcher 可以匹配：

```text
manual
auto
```

适合用来：

- 压缩前保存状态
- 压缩后重新加载项目说明
- 记录上下文窗口变化
- 恢复模型需要的关键上下文

### 5. 子代理相关

| 事件 | 触发时机 |
|---|---|
| `SubagentStart` | 子代理启动时 |
| `SubagentStop` | 子代理结束时 |

可以根据子代理类型进行 matcher 匹配。

### 6. Codex hook 配置位置

Codex 可以从以下位置加载 hooks：

```text
~/.codex/hooks.json
~/.codex/config.toml
<项目>/.codex/hooks.json
<项目>/.codex/config.toml
```

插件也可以提供 hooks。

多个来源的 hooks 会合并执行，不是简单的高优先级覆盖低优先级。官方文档建议同一层不要同时使用 `hooks.json` 和 `config.toml` 内嵌 hooks，否则会产生合并警告。

### 7. Codex handler 能力

Codex 当前 hook 配置形式类似：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe ..."
          }
        ]
      }
    ]
  }
}
```

官方文档说明：

- 当前实际执行的是 `type = "command"`
- `prompt` 和 `agent` 类型目前只会被解析，不会执行
- Windows 可以使用 `commandWindows`
- command hook 通过标准输入接收 JSON
- 多个匹配的 command hook 会并发启动
- 非托管 hook 需要在 `/hooks` 中审核和信任
- hook 配置发生变化后，需要重新信任新的 hook hash

## 三、Claude Code CLI Hooks

Claude Code 官方当前列出 31 个 hook 事件：

```text
SessionStart
Setup
UserPromptSubmit
UserPromptExpansion
PreToolUse
PermissionRequest
PermissionDenied
PostToolUse
PostToolUseFailure
PostToolBatch
Notification
MessageDisplay
SubagentStart
SubagentStop
TaskCreated
TaskCompleted
Stop
StopFailure
TeammateIdle
InstructionsLoaded
ConfigChange
CwdChanged
DirectoryAdded
FileChanged
WorktreeCreate
WorktreeRemove
PreCompact
PostCompact
Elicitation
ElicitationResult
SessionEnd
```

### 1. Claude Code 的三个主要时间粒度

官方文档把 Claude Code hooks 分成三个主要节奏：

```text
每个会话：
    SessionStart
    SessionEnd

每个 turn：
    UserPromptSubmit
    Stop
    StopFailure

每个工具调用：
    PreToolUse
    PostToolUse
```

其他事件会在具体状态变化时触发。

### 2. 会话和初始化事件

| 事件 | 触发时机 |
|---|---|
| `SessionStart` | 会话启动或恢复时 |
| `Setup` | 使用 `--init-only`、`--init` 或 `--maintenance` 等初始化模式时 |
| `SessionEnd` | 会话终止时 |

Claude Code 的 `SessionEnd` 和 Codex 的 `SessionEnd` 作用基本一致，都是整个会话结束，而不是当前一轮完成。

### 3. 用户输入事件

| 事件 | 触发时机 |
|---|---|
| `UserPromptSubmit` | 用户提交提示词时 |
| `UserPromptExpansion` | 用户输入的命令或 slash command 被展开成真正 prompt 时 |
| `MessageDisplay` | Claude 正在展示助手消息文本时 |
| `InstructionsLoaded` | 加载 `CLAUDE.md` 或规则文件时 |

`InstructionsLoaded` 是 Claude Code 一个比较特殊的能力。它不仅会在会话启动时触发，也可能在会话中懒加载某个规则文件时触发。

### 4. 工具和权限事件

| 事件 | 触发时机 |
|---|---|
| `PreToolUse` | 工具调用之前 |
| `PermissionRequest` | 工具需要权限决定时 |
| `PermissionDenied` | 自动权限分类器拒绝工具调用时 |
| `PostToolUse` | 工具成功执行之后 |
| `PostToolUseFailure` | 工具执行失败之后 |
| `PostToolBatch` | 一批并行工具调用全部结束后 |

Claude Code 在工具 hook 方面比 Codex 更细：

```text
PreToolUse
PostToolUse
PostToolUseFailure
PostToolBatch
```

因此可以分别处理：

- 工具执行之前的安全检查
- 成功后的日志
- 失败后的恢复
- 批量并行工具全部结束后的汇总

`PermissionDenied` 还可以让 hook 返回 `{ "retry": true }`，告诉模型尝试重新执行被拒绝的工具调用。

### 5. 通知事件

| 事件 | 触发时机 |
|---|---|
| `Notification` | Claude Code 发送通知时 |

官方列出的通知类型包括：

```text
permission_prompt
idle_prompt
auth_success
elicitation_dialog
elicitation_complete
elicitation_response
agent_needs_input
agent_completed
```

这一点是 Claude Code 和 Codex 的重要差异。

Claude Code 有一个独立的 `Notification` hook，可以直接监听：

```text
权限提示
代理需要输入
代理完成
空闲提示
认证成功
```

Codex 没有同名的 hook 事件。Codex 的内置通知由：

```toml
[tui]
notifications = false
```

控制；如果需要自定义声音，则使用 `PermissionRequest` 和 `Stop` hooks。

### 6. 子代理、任务和团队事件

| 事件 | 触发时机 |
|---|---|
| `SubagentStart` | 子代理启动 |
| `SubagentStop` | 子代理结束 |
| `TaskCreated` | 创建任务 |
| `TaskCompleted` | 任务标记完成 |
| `TeammateIdle` | 团队代理即将空闲 |

Claude Code 原生提供任务和团队相关 hook，Codex 当前官方事件列表中没有对应的：

```text
TaskCreated
TaskCompleted
TeammateIdle
```

### 7. 当前 turn 结束和错误结束

| 事件 | 触发时机 |
|---|---|
| `Stop` | Claude 正常完成回答 |
| `StopFailure` | turn 因 API 错误结束 |

如果你的需求是“只要一轮结束就播放声音”，Claude Code 可以配置：

```text
Stop       -> 正常完成声音
StopFailure -> 错误结束声音
```

Codex 目前只有 `Stop`，没有独立的 `StopFailure`。

### 8. 配置和环境变化事件

Claude Code 还提供：

| 事件 | 触发时机 |
|---|---|
| `ConfigChange` | 配置文件在会话中发生变化 |
| `CwdChanged` | 当前工作目录发生变化 |
| `DirectoryAdded` | 通过 `/add-dir` 或 SDK 添加目录 |
| `FileChanged` | 被监控的文件发生变化 |
| `WorktreeCreate` | 创建 Git worktree |
| `WorktreeRemove` | 删除 Git worktree |

这些事件适合做：

- 自动重新加载环境
- 重新执行 direnv
- 更新项目上下文
- worktree 创建后安装依赖
- worktree 删除前清理资源

Codex 当前没有这些专门事件。

### 9. 压缩和 MCP 交互事件

| 事件 | 触发时机 |
|---|---|
| `PreCompact` | 上下文压缩前 |
| `PostCompact` | 上下文压缩后 |
| `Elicitation` | MCP 服务请求用户输入时 |
| `ElicitationResult` | 用户完成 MCP 输入后、结果返回 MCP 服务前 |

Claude Code 对 MCP 交互的生命周期暴露得更细。Codex 当前可以匹配 MCP 工具本身，但没有单独的 `Elicitation` 和 `ElicitationResult` hook。

### 10. Claude Code hook 配置位置

Claude Code 支持的配置来源包括：

```text
~/.claude/settings.json
.claude/settings.json
.claude/settings.local.json
组织管理策略
插件 hooks/hooks.json
skill 或 agent 的 frontmatter
```

其中：

- `~/.claude/settings.json`：用户全局配置
- `.claude/settings.json`：项目共享配置
- `.claude/settings.local.json`：项目本地、不提交到 Git
- 插件 hooks：插件启用时生效
- skill/agent hooks：对应 skill 或 agent 活跃时生效

Claude Code 的不同配置层会合并 hooks，而不是简单覆盖。

### 11. Claude Code handler 能力

Claude Code 官方支持 5 种 handler：

```text
command
http
mcp_tool
prompt
agent
```

含义如下：

| Handler    | 作用                       |
| ---------- | ------------------------ |
| `command`  | 执行 shell 或 PowerShell 命令 |
| `http`     | 向 HTTP endpoint 发送 POST  |
| `mcp_tool` | 调用已连接 MCP server 的工具     |
| `prompt`   | 让 Claude 模型对当前事件做一次判断    |
| `agent`    | 启动一个可以使用工具的子代理进行判断       |

Claude Code 还提供异步/background hook 能力，适合：

- 不阻塞主流程的日志
- 文件变化后自动运行测试
- 后台通知
- 耗时较长的检查

## 四、两者的核心对比

| 对比项 | Codex | Claude Code |
|---|---|---|
| 官方事件数量 | 11 个 | 31 个 |
| 核心生命周期 | 支持 | 支持 |
| 每轮结束 | `Stop` | `Stop`、`StopFailure` |
| 整个会话结束 | `SessionEnd` | `SessionEnd` |
| 权限请求 | `PermissionRequest` | `PermissionRequest`、`PermissionDenied` |
| 工具失败事件 | 没有专门事件 | `PostToolUseFailure` |
| 批量工具结束 | 没有 | `PostToolBatch` |
| 通知事件 | 没有独立通知 hook | `Notification` |
| 任务事件 | 没有 | `TaskCreated`、`TaskCompleted` |
| 团队空闲 | 没有 | `TeammateIdle` |
| 配置变化 | 没有 | `ConfigChange` |
| 文件变化 | 没有专门事件 | `FileChanged` |
| 工作树生命周期 | 没有专门事件 | `WorktreeCreate`、`WorktreeRemove` |
| MCP 用户输入流程 | 没有单独事件 | `Elicitation`、`ElicitationResult` |
| 工具前后检查 | `PreToolUse`、`PostToolUse` | 同样支持，且粒度更细 |
| Handler | 实际主要是 command | command、HTTP、MCP、prompt、agent |
| 异步 hook | command async 尚未支持 | 官方提供 async/background hook |
| 配置格式 | JSON、TOML | JSON |
| Windows 命令 | 支持 `commandWindows` | 官方示例支持 PowerShell |
| Hook 信任 | 非托管 hook 需要 `/hooks` 审核 | 通过设置层、插件和策略管理 |
| 适合定位 | 轻量生命周期自动化 | 复杂 agent 工作流编排 |


## 五、最重要的概念区别

可以用下面这张简化图记忆：

```text
Codex:

SessionStart
    |
UserPromptSubmit
    |
PreToolUse
    |
PermissionRequest
    |
PostToolUse
    |
Stop                 <- 当前一轮结束
    |
SessionEnd           <- 整个主会话结束
```

```text
Claude Code:

SessionStart
    |
UserPromptSubmit
    |
PreToolUse
    |
PermissionRequest / PermissionDenied
    |
PostToolUse / PostToolUseFailure
    |
PostToolBatch
    |
Stop / StopFailure   <- 当前一轮结束
    |
SessionEnd           <- 整个会话结束
```

Claude Code 在这条主链路之外，还额外监听：

```text
Notification
SubagentStart / SubagentStop
TaskCreated / TaskCompleted
ConfigChange
FileChanged
WorktreeCreate / WorktreeRemove
Elicitation / ElicitationResult
```

下面这段可以直接接在前六节后面，作为：

## 七、Hooks 实践：官方推荐用法与典型场景

这里的“官方推荐”分为两类：

- 官方文档直接给出的示例和适用场景。
- 根据官方事件能力整理出的工程实践建议。

### 7.1 Hooks 的基本使用原则

Hooks 最适合处理确定性、重复性、可自动验证的工作，例如：

- 工具调用前的安全检查。
- 工具调用后的格式化、测试和审计。
- 会话开始时加载项目上下文。
- 每轮结束时发送通知或生成摘要。
- 需要用户授权时播放声音。
- 对提示词、文件路径和敏感信息进行检查。

Hooks 不适合承载复杂的业务逻辑，也不应该替代 `AGENTS.md`、`CLAUDE.md` 或普通提示词。

可以按照下面的思路区分：

| 目标               | 适合的 Hook 类型                                   |
| ---------------- | --------------------------------------------- |
| 阻止危险操作           | `PreToolUse`                                  |
| 监听权限请求           | `PermissionRequest` 或 Claude 的 `Notification` |
| 工具执行后处理          | `PostToolUse`                                 |
| 每轮对话完成后处理        | `Stop`                                        |
| 整个会话退出时清理        | `SessionEnd`                                  |
| 会话启动、恢复或压缩后恢复上下文 | `SessionStart`                                |
| 后台执行耗时任务         | Claude Code 的异步 `PostToolUse`                 |

Hooks 应该尽量满足以下要求：

- 执行时间短。
- 可以重复执行，不会产生严重副作用。
- 不依赖多个 Hook 的执行顺序。
- 不把密钥、令牌和完整敏感提示词写入日志。
- 对用户输入、文件路径和 Shell 参数进行校验。
- 出错时不能破坏正常对话，除非这个 Hook 本身就是安全阻断策略。

Codex 和 Claude Code 都可能并行执行多个匹配的 Hook，因此不要假设 Hook 会严格按照配置文件中的顺序执行。

---

### 7.2 Codex 的推荐实践

#### `SessionStart`：加载项目上下文

`SessionStart` 在会话启动、恢复、清空上下文或压缩后重新建立上下文时触发。

Codex 官方支持根据以下来源进行匹配：

- `startup`
- `resume`
- `clear`
- `compact`

适合的场景：

- 输出当前项目的构建方式。
- 加载项目约定、测试命令和目录说明。
- 压缩上下文后重新注入必要规则。
- 检查当前分支、运行环境或工具版本。
- 加载自动生成的项目摘要。

Codex 的 `SessionStart` 可以通过标准输出或 `additionalContext` 给模型增加开发者上下文。

实践建议：

- 只输出模型真正需要的内容。
- 不要每次启动都扫描整个仓库。
- 不要把完整日志直接注入上下文。
- `compact` 场景下只恢复关键状态，不要重新输出整份项目文档。

---

#### `UserPromptSubmit`：提示词审计和上下文处理

这个 Hook 在用户提交消息后、模型开始处理前触发。

Codex 官方给出的典型用途包括：

- 将对话发送到日志或分析系统。
- 检查用户是否误粘贴了 API Key。
- 根据当前目录附加额外提示。
- 对输入内容进行组织或脱敏。

适合的实际场景：

- 检测 `sk-`、`ctx7sk-` 等疑似密钥格式。
- 将用户输入记录到本地审计日志。
- 根据仓库类型自动附加测试命令。
- 对企业内部项目的敏感词进行提醒。

如果只是日志记录，建议只做旁路处理，不要阻塞用户正常提问。只有明确的安全策略才应该拒绝提示词。

---

#### `PreToolUse`：工具调用前的安全闸门

`PreToolUse` 是 Codex 中最重要的策略控制点之一。它在工具真正执行前触发，可以检查：

- Shell 命令。
- 文件编辑。
- `apply_patch`。
- MCP 工具调用。
- 其他本地工具调用。

适合的场景：

- 阻止危险删除命令。
- 阻止访问项目目录外的敏感文件。
- 检查命令是否包含明显的密钥泄露风险。
- 限制某些 MCP 工具只能访问特定资源。
- 对支持的工具输入进行修改后再执行。
- 对高风险操作增加人工审批。

Codex 官方示例包括：在工具调用前识别危险命令并拒绝执行。

实践建议：

- Matcher 尽量精确，不要对所有工具执行复杂脚本。
- 安全规则应该确定、可解释、可测试。
- 不要仅依赖字符串匹配构建完整安全边界。
- 自动允许操作的条件必须比自动拒绝更加严格。
- 对路径进行规范化后再判断，避免通过 `..` 绕过限制。

对于真正的安全阻断，Hook 应该明确返回拒绝结果；对于普通审计，则只记录日志，不改变工具调用。

---

#### `PermissionRequest`：权限请求提醒和审计

`PermissionRequest` 只会在 Codex 即将向用户请求授权时触发，例如：

- Shell 命令需要升级权限。
- 某个操作需要额外授权。
- 需要用户确认访问外部资源。

它适合：

- 播放提示音。
- 发送桌面通知。
- 记录权限请求日志。
- 根据工具名称实施企业审批策略。
- 对特殊命令进行自动拒绝。

需要注意，`PermissionRequest` 不是所有工具调用都会触发。只有确实需要向用户询问权限时，它才会执行。

---

#### `PostToolUse`：工具执行后的验证

`PostToolUse` 适合做工具执行后的工作，例如：

- 记录工具调用结果。
- 对修改后的文件运行格式化。
- 检查生成文件是否符合规范。
- 对 MCP 返回结果进行审计。
- 生成简短的后续上下文。

适合的实践：

- 编辑 TypeScript 文件后运行格式化检查。
- 修改配置文件后验证 TOML、JSON 或 YAML 格式。
- 写入数据库或外部系统后记录操作结果。
- 工具执行后生成一条简短状态消息。

Codex 当前的命令 Hook 主要是同步执行，因此不建议在这里直接运行很长的完整测试套件。耗时任务应该尽量缩小范围，或交给外部后台任务处理。

---

#### `PreCompact` 和 `PostCompact`：上下文压缩保护

这两个 Hook 适合处理上下文压缩前后的状态保存和恢复。

`PreCompact` 可以：

- 保存当前任务状态。
- 生成工作摘要。
- 记录已完成和未完成的事项。
- 保存重要的临时上下文。

`PostCompact` 可以：

- 重新加载任务摘要。
- 恢复项目约定。
- 提醒模型当前正在处理的文件和目标。

一个实用模式是：

```text
PreCompact  ->  保存任务摘要
压缩上下文
PostCompact ->  重新加载任务摘要
```

不要把完整 transcript 当作稳定 API 使用。更可靠的做法是由 Hook 自己维护简洁、结构化的状态文件。

---

#### `SubagentStart` 和 `SubagentStop`：子代理协作

`SubagentStart` 适合给子代理注入额外规则，例如：

- 先阅读项目测试约定。
- 只能修改指定目录。
- 必须运行指定验证命令。
- 输出结果时必须包含文件路径和测试状态。

`SubagentStop` 适合：

- 收集子代理摘要。
- 记录子代理完成状态。
- 验证子代理是否产生了预期文件。
- 统一记录子代理的耗时和结果。

不建议使用它们强制子代理执行复杂的长流程，否则容易造成代理之间相互等待。

---

#### `Stop`：每轮对话结束后的处理

`Stop` 表示当前一轮模型处理结束，而不是整个 Codex 会话退出。

Codex 官方明确给出的适用场景是：

- 当前轮结束后执行自定义验证。
- 检查是否满足质量标准。
- 保存当前轮摘要。
- 发送完成通知。

适合的场景：

- 播放任务完成提示音。
- 记录本轮修改的文件。
- 运行轻量检查。
- 生成简短工作摘要。
- 将完成状态发送到外部通知系统。

不建议在 `Stop` 中每次都执行完整构建或全量测试，因为这会让每轮对话都变慢。

---

#### `SessionEnd`：整体会话结束后的清理

`SessionEnd` 表示主会话整体结束，和 `Stop` 的层级不同。

适合：

- 保存最终任务摘要。
- 清理临时文件。
- 关闭会话级资源。
- 生成最终审计记录。
- 播放“整个会话退出”的声音。

它不会在每一轮对话后触发，也不会用于子代理会话。

Codex 官方文档特别强调，`SessionEnd` 的时间预算很短，默认约 1 秒，最长支持到 3 秒。因此它不适合运行长时间任务，也不应该承担必须完成的大型数据保存工作。

推荐区分：

```text
Stop       -> 当前一轮对话完成
SessionEnd -> 整个主会话退出
```

---

### 7.3 Claude Code 的推荐实践

Claude Code 的 Hook 生命周期比 Codex 更丰富，除了会话、工具和轮次，还覆盖了任务、团队、工作目录、配置变化、MCP 交互和 Worktree。

Claude Code 支持的 Handler 类型也更多：

- `command`：Shell 或 PowerShell 命令。
- `http`：向 HTTP 地址发送事件。
- `mcp_tool`：调用已连接的 MCP 工具。
- `prompt`：让一个快速模型判断是否满足条件。
- `agent`：启动一个具备读取和搜索能力的子代理。
- `async`：让命令在后台执行，不阻塞当前工具调用。

---

#### `SessionStart` 和 `Setup`：环境初始化

`SessionStart` 适合：

- 会话启动时同步共享技能。
- 检查依赖是否安装。
- 加载项目状态。
- 恢复会话上下文。
- 压缩后重新加载必要规则。

Claude 官方示例包括：

- 在会话开始时同步共享技能仓库。
- 同步后通过 `reloadSkills` 让技能立即生效。

`Setup` 更适合初始化和维护场景，例如：

- `--init-only` 初始化。
- CI 或脚本模式准备环境。
- 安装项目所需工具。
- 创建开发目录。

实践上，可以把“一次性环境准备”放到 `Setup`，把“每次会话都需要的上下文”放到 `SessionStart`。

---

#### `UserPromptSubmit` 和 `UserPromptExpansion`：输入控制

这两个事件可以用于：

- 提示词审计。
- 记录用户输入。
- 为特定项目附加规则。
- 校验自定义命令展开结果。
- 阻止不符合策略的命令展开。

不建议对普通用户输入设置过于严格的阻断规则。更好的方式是：

- 对疑似密钥进行提醒或脱敏。
- 对危险操作进行后续 `PreToolUse` 检查。
- 只对明确的命令扩展规则进行阻止。

---

#### `PreToolUse`：命令和文件操作拦截

Claude 官方提供了阻止危险 Shell 命令的示例，例如：

- 阻止破坏性删除。
- 阻止访问敏感路径。
- 阻止修改不允许修改的文件。
- 阻止未经批准的网络或系统操作。

Claude Code 除了 `matcher` 外，还支持 `if` 条件，可以进一步限制参数范围，例如：

```text
只对 Bash(git push *)
只对 Edit(*.env)
只对 Bash(rm *)
```

因此 Claude Code 可以同时使用：

- `matcher` 过滤工具类型。
- `if` 过滤工具参数和命令模式。

这对于减少误拦截非常有用。

---

#### `PermissionRequest`、`PermissionDenied`：权限决策和失败重试

`PermissionRequest` 适合：

- 记录用户授权请求。
- 对特定工具增加审批策略。
- 播放声音。
- 发送桌面通知。

`PermissionDenied` 适合：

- 记录被自动模式拒绝的工具调用。
- 检查是否可以安全重试。
- 对满足条件的操作返回 `retry: true`。

权限相关 Hook 必须非常谨慎。自动放行权限的规则如果过宽，可能绕过用户原本看到的安全确认。

---

#### `PostToolUse`：修改后自动格式化和测试

这是 Claude Code 最适合自动化工程检查的事件之一。

官方推荐示例包括：

- 文件写入或编辑后运行测试。
- 编辑 TypeScript 或 JavaScript 文件后执行测试。
- 工具调用成功后进行安全扫描。
- 使用 MCP 工具对修改后的文件进行审计。

Claude Code 支持异步 Hook，因此可以：

```text
Write/Edit
    |
    v
PostToolUse
    |
    v
后台运行 npm test
    |
    v
下一轮对话时返回测试结果
```

异步 Hook 的特点：

- 不阻塞当前工具调用。
- 不能阻止工具执行。
- 不能返回工具权限决策。
- 结果会在后续对话中通过上下文或系统消息展示。

适合用异步 Hook 执行：

- 单元测试。
- lint。
- 类型检查。
- 安全扫描。
- 文档生成。
- 代码统计。

---

#### `PostToolUseFailure` 和 `PostToolBatch`：失败处理和批量处理

`PostToolUseFailure` 适合：

- 记录失败命令。
- 保存错误上下文。
- 对失败操作发送通知。
- 针对特定错误给出修复建议。

`PostToolBatch` 在一批并行工具调用全部完成后触发，适合：

- 汇总多个文件的修改结果。
- 在批量编辑完成后统一运行测试。
- 避免每个文件单独触发一次昂贵检查。
- 生成一次性的批量审计结果。

如果一次操作会修改很多文件，`PostToolBatch` 通常比每个 `PostToolUse` 都运行完整检查更合适。

---

#### `Notification`：通知、权限和完成提示

Claude Code 的 `Notification` 可以根据通知类型进行匹配，例如：

- `permission_prompt`：需要用户授权。
- `agent_completed`：代理完成。
- `agent_needs_input`：代理等待用户输入。
- `idle_prompt`：代理处于空闲状态。
- `elicitation_dialog`：MCP 请求用户输入。

因此，Claude Code 中更推荐使用：

```text
Notification(permission_prompt) -> 权限提示音
Notification(agent_completed)   -> 代理完成提示
```

如果需要严格表达“当前一轮模型回复结束”，则使用 `Stop`。

在 Windows 上，官方文档建议使用 PowerShell Handler，并通过终端序列或外部脚本发送通知。

---

#### `Stop` 和 `StopFailure`：轮次完成与异常结束

`Stop` 表示 Claude Code 当前一轮响应结束，适合：

- 播放完成音。
- 记录本轮修改。
- 做轻量质量检查。
- 生成当前任务摘要。

`StopFailure` 表示当前轮因为 API 错误等原因结束，适合：

- 播放错误提示音。
- 记录错误类型。
- 发送失败通知。
- 收集 API 或网络错误信息。

Claude Code 还支持 `prompt` 类型的 `Stop` Hook，让一个快速模型判断任务是否真的完成。例如：

```text
当前任务是否已经完成？
如果没有完成，请说明还缺少什么。
```

这种方式适合语义型质量检查，但不适合作为唯一的安全边界，因为它：

- 会增加模型调用成本。
- 可能产生额外延迟。
- 可能因为判断结果反复触发，形成循环。
- 不如确定性的脚本检查稳定。

---

#### `SubagentStart`、`SubagentStop`、`TaskCompleted`：任务协作

Claude Code 的任务和团队 Hook 比 Codex 更完整。

适合的场景：

- `SubagentStart`：给子代理注入角色规则。
- `SubagentStop`：记录子代理结果。
- `TaskCreated`：登记任务、分配负责人。
- `TaskCompleted`：验证任务产物是否存在。
- `TeammateIdle`：发现队友空闲时提醒或分配后续任务。

例如，一个代码审查子代理完成后，可以自动检查：

- 是否生成审查报告。
- 是否包含严重程度。
- 是否包含文件路径。
- 是否记录测试缺口。

---

#### `InstructionsLoaded`、`ConfigChange`、`CwdChanged`：环境变化响应

这些是 Claude Code 中很实用、但 Codex 当前没有完全对应事件的能力。

典型场景：

- `InstructionsLoaded`：记录哪些 `CLAUDE.md` 或规则文件被加载。
- `ConfigChange`：审计权限配置或 MCP 配置变化。
- `CwdChanged`：切换目录后重新加载目录环境。
- `DirectoryAdded`：新增项目目录后更新索引。
- `FileChanged`：监视特定配置或规则文件的变化。

例如，进入不同目录后，可以重新加载：

- Python 虚拟环境。
- Node.js 版本。
- `.env` 路径。
- 项目专用测试命令。
- 目录级别的开发规范。

`FileChanged` 应该只监视明确的文件名，不要对整个仓库进行宽泛监听。

---

#### `WorktreeCreate` 和 `WorktreeRemove`：隔离工作区

这两个 Hook 适合：

- 创建临时开发环境。
- 准备独立依赖。
- 生成非 Git 类型的工作副本。
- 删除工作区时清理资源。
- 为子代理准备隔离目录。

Claude 官方还给出了使用 Worktree Hook 管理非标准版本控制系统工作副本的示例。

适合大型项目、并行子代理以及需要隔离构建环境的场景。

---

#### `PreCompact` 和 `PostCompact`：压缩前后保存状态

用途和 Codex 类似：

- `PreCompact` 保存任务摘要。
- `PostCompact` 恢复上下文。
- 保存当前工作目录、分支和未完成任务。
- 重新加载项目规则。

如果对话很长，这是比依赖完整聊天记录更可靠的状态管理方式。

---

#### `Elicitation` 和 `ElicitationResult`：MCP 用户输入审计

这两个事件用于 MCP 服务器向用户请求输入的场景。

适合：

- 验证用户输入格式。
- 审计 MCP 收集了哪些信息。
- 拦截敏感数据。
- 在用户提交前提示风险。
- 记录外部服务请求。

这对于需要用户填写账号、路径、部署参数或外部系统凭据的 MCP 工具特别有用。

---

### 7.4 Codex 和 Claude Code 的场景对照

| 实践目标 | Codex | Claude Code |
|---|---|---|
| 权限请求提示音 | `PermissionRequest` | `Notification` 匹配 `permission_prompt` |
| 当前一轮完成提示音 | `Stop` | `Stop` |
| 整个会话退出提示音 | `SessionEnd` | `SessionEnd` |
| 阻止危险命令 | `PreToolUse` | `PreToolUse`，可配合 `if` |
| 工具调用后审计 | `PostToolUse` | `PostToolUse` |
| 工具失败处理 | 通常由脚本自行判断 | `PostToolUseFailure` |
| 批量工具调用完成后统一处理 | 能力较有限 | `PostToolBatch` |
| 启动时加载上下文 | `SessionStart` | `SessionStart` 或 `Setup` |
| 子代理开始和结束 | `SubagentStart`、`SubagentStop` | `SubagentStart`、`SubagentStop` |
| 后台运行测试 | 当前命令 Hook 不支持真正异步 | `PostToolUse` 支持 `async` |
| MCP 安全扫描 | `PreToolUse` / `PostToolUse` | `mcp_tool` / `PreToolUse` / `PostToolUse` |
| 团队任务管理 | 没有同等丰富的事件 | `TaskCreated`、`TaskCompleted`、`TeammateIdle` |
| 工作区生命周期 | 能力较有限 | `WorktreeCreate`、`WorktreeRemove` |
| 配置和目录变化 | 能力较有限 | `ConfigChange`、`CwdChanged`、`FileChanged` |
| MCP 用户输入 | 主要通过工具 Hook 处理 | `Elicitation`、`ElicitationResult` |

总体上：

- Codex 的 Hook 体系更集中在工具调用、权限、上下文压缩和会话生命周期。
- Claude Code 的 Hook 体系覆盖了更完整的开发工作流，包括任务、团队、目录、配置、Worktree 和 MCP 用户交互。
- Codex 适合构建轻量、确定性的个人自动化。
- Claude Code 更适合构建完整的工程工作流和后台自动化。

---

### 7.5 一套实用的最小 Hook 组合

对于个人开发环境，建议先使用以下组合：

#### Codex

```text
SessionStart
    -> 加载项目约定和压缩后恢复的摘要

PermissionRequest
    -> 播放权限请求声音

PreToolUse
    -> 阻止明确危险的命令

PostToolUse
    -> 记录工具结果或执行轻量检查

Stop
    -> 播放每轮完成声音，保存简短摘要

SessionEnd
    -> 可选：保存最终会话记录和清理临时文件
```

#### Claude Code

```text
SessionStart
    -> 同步技能、加载项目上下文

PreToolUse
    -> 阻止危险命令和敏感路径访问

PostToolUse
    -> 文件修改后异步运行测试

Notification(permission_prompt)
    -> 播放权限请求声音

Stop
    -> 播放每轮完成声音

StopFailure
    -> 播放异常结束声音

SessionEnd
    -> 保存最终状态和清理资源
```

---

### 7.6 不建议使用 Hooks 的场景

以下情况通常不适合放进 Hook：

- 每轮都执行完整构建。
- 每次工具调用都扫描整个仓库。
- 使用模型 Hook 判断简单的确定性条件。
- 对所有工具调用执行复杂网络请求。
- 通过 Hook 自动批准所有权限。
- 将完整 transcript、密钥或环境变量输出给模型。
- 依赖多个 Hook 的执行顺序。
- 直接修改用户文件但没有幂等设计。
- 用 `SessionEnd` 保存唯一的重要状态。
- 用一个 Hook 同时承担通知、安全、测试和数据同步。

更稳妥的做法是把职责拆开：

```text
安全判断       -> PreToolUse
权限提醒       -> PermissionRequest / Notification
工具后检查     -> PostToolUse
耗时测试       -> Claude async Hook 或外部后台任务
每轮通知       -> Stop
最终清理       -> SessionEnd
```

核心判断可以概括为：

> 想阻止工具，用 `PreToolUse`；想响应权限，用 `PermissionRequest` 或 `Notification`；想处理当前一轮，用 `Stop`；想处理整个会话，用 `SessionEnd`；想恢复上下文，用 `SessionStart`；想运行耗时检查，优先使用 Claude Code 的异步 `PostToolUse`。