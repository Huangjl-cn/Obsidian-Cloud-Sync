Codex 的 hooks 更少、更聚焦核心 agent 生命周期；Claude Code 的 hooks 更细，覆盖配置、通知、任务、工作树和 MCP 交互等更多阶段。

**Codex 官方 hooks**

[Codex Hooks 官方文档](https://developers.openai.com/codex/hooks) 当前列出 11 个事件：

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

主要触发时机：

- `UserPromptSubmit`：提交用户输入时
- `PreToolUse` / `PostToolUse`：工具调用前后
- `PermissionRequest`：需要询问权限时
- `Stop`：当前 turn 结束时
- `SessionStart` / `SessionEnd`：主会话开始和结束
- `SubagentStart` / `SubagentStop`：子代理开始和结束
- `PreCompact` / `PostCompact`：上下文压缩前后

Codex 当前官方实现中，实际执行的是 `command` hook；`prompt` 和 `agent` 类型虽然能解析，但目前不会执行。非托管 hook 还需要在 `/hooks` 中审核并信任。

**Claude Code CLI 官方 hooks**

[Claude Code Hooks 官方文档](https://code.claude.com/docs/en/hooks) 当前列出 31 个事件：

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

Claude Code 还支持 5 种 handler：

```text
command
http
mcp_tool
prompt
agent
```

其中 `Notification` 可以区分权限提示、空闲提示、认证成功、代理需要输入、代理完成等通知类型；`StopFailure` 可以专门处理 API 错误导致的 turn 结束。

**核心对比**

| 能力 | Codex | Claude Code |
|---|---|---|
| 每轮结束 | `Stop` | `Stop`、`StopFailure` |
| 整体会话结束 | `SessionEnd` | `SessionEnd` |
| 权限请求 | `PermissionRequest` | `PermissionRequest`、`PermissionDenied` |
| 工具调用 | 调用前后 | 调用前后、失败后、批量调用后 |
| 通知事件 | 没有独立 `Notification` hook | 有，并且可按通知类型匹配 |
| 子代理 | `SubagentStart`、`SubagentStop` | 同样支持 |
| 任务系统 | 无专门事件 | `TaskCreated`、`TaskCompleted` |
| 配置变化 | 无 | `ConfigChange` |
| 文件/目录变化 | 无 | `FileChanged`、`DirectoryAdded`、`CwdChanged` |
| Git worktree | 无专门事件 | `WorktreeCreate`、`WorktreeRemove` |
| MCP 交互 | MCP 工具可通过工具 hook 匹配 | 额外支持 `Elicitation`、`ElicitationResult` |
| Handler 类型 | 实际执行主要是 command | command、HTTP、MCP tool、prompt、agent |
| 配置位置 | `~/.codex/hooks.json`、项目 `.codex/hooks.json`、`config.toml` | `~/.claude/settings.json`、项目 settings、插件、skill、agent |

对你当前的 Codex 配置来说，`PermissionRequest` 用于权限提示音，`Stop` 用于每轮结束提示音，已经是正确对应关系；`SessionEnd` 只适合播放“整个会话真正结束”的声音。