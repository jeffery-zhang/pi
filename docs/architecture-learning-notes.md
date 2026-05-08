---
tags:
  - pi-mono
  - architecture
  - learning
---

# Pi Mono 框架拆解学习笔记

> 上级：[[index|Pi Mono 学习库]]
> 相关：[[llm-abstraction-layer|LLM 抽象层学习笔记]]、[[model-description-module|Model 描述模块学习笔记]]


## 目的

本文档用于记录对当前 fork 的 `pi-mono` 项目的分层理解，作为后续深入学习和自定义 AI Agent 扩展的基础资料。

## 当前粗略结论：5 层核心能力

`pi-mono` 是一个可嵌入、可扩展的 AI Agent harness。它不是单一 CLI 工具，而是由底层 LLM 抽象、Agent 运行时、Coding Agent 产品层、UI 层和扩展系统组合而成。

---

## 1. LLM 抽象层：`packages/ai`

主文档：[[llm-abstraction-layer|LLM 抽象层学习笔记]]

这一层负责把不同模型供应商的 API 统一成同一套接口。

核心能力：

- 统一多 provider / 多模型调用接口
- 流式与非流式生成：`stream()`、`complete()`
- Tool / function calling 标准化
- 文本和图片输入
- Tool result 中返回文本或图片
- Thinking / reasoning 统一抽象
- Token 和 cost 统计
- 上下文序列化
- 跨 provider handoff
- API key 和 OAuth 认证支持
- 内置 provider 支持 OpenAI、Anthropic、Google、Vertex、Mistral、Groq、Cloudflare、OpenRouter、Bedrock、GitHub Copilot 等

关键文件：

- `packages/ai/src/index.ts`
- `packages/ai/src/types.ts`
- `packages/ai/src/stream.ts`
- `packages/ai/src/providers/`
- `packages/ai/src/models.generated.ts`

理解重点：

这一层不是完整 Agent，而是“模型访问与消息协议标准化层”。如果后续要接入自定义模型或 provider，主要会动这一层。

---

## 2. Agent 运行时核心：`packages/agent`

这一层是最干净的 Agent 基座。

核心能力：

- `Agent` 类管理 Agent state、prompt、continue、abort 和事件订阅
- `agentLoop()` 提供低层 Agent 循环
- 处理 LLM 调用、tool calls、tool results 和自动续轮
- 支持工具并发或顺序执行
- 支持 `beforeToolCall` / `afterToolCall` hook
- 支持 steering / follow-up 队列
- 支持上下文转换：`transformContext`、`convertToLlm`
- 输出标准事件流：
  - `agent_start`
  - `turn_start`
  - `message_start`
  - `message_update`
  - `message_end`
  - `tool_execution_start`
  - `tool_execution_update`
  - `tool_execution_end`
  - `turn_end`
  - `agent_end`

关键文件：

- `packages/agent/src/agent.ts`
- `packages/agent/src/agent-loop.ts`
- `packages/agent/src/types.ts`
- `packages/agent/README.md`

理解重点：

如果目标是构建一个“自己的 AI Agent 内核”，这一层最值得先读。它定义了 Agent 如何围绕模型、消息、工具和事件运转。

---

## 3. Coding Agent 产品层：`packages/coding-agent`

这一层是在 Agent 核心上构建出的完整 coding agent 应用。

核心能力：

- `pi` CLI
- Interactive TUI 模式
- Print / JSON / RPC 模式
- SDK 嵌入能力
- 会话持久化 JSONL
- 会话树、branch、fork、clone
- 自动和手动 compaction
- 模型选择、登录、设置管理
- 内置代码工具：
  - `read`
  - `write`
  - `edit`
  - `bash`
  - `grep`
  - `find`
  - `ls`
- Extension 系统
- Skills、prompt templates、themes、pi packages

关键文件：

- `packages/coding-agent/src/core/agent-session.ts`
- `packages/coding-agent/src/core/sdk.ts`
- `packages/coding-agent/src/core/tools/`
- `packages/coding-agent/src/core/extensions/`
- `packages/coding-agent/src/modes/`
- `packages/coding-agent/docs/sdk.md`
- `packages/coding-agent/docs/extensions.md`
- `packages/coding-agent/docs/rpc.md`

理解重点：

这一层是最适合“基于现成能力做自己的 Agent 产品”的入口。它已经包含会话、工具、扩展、配置、模型认证、CLI/RPC 等工程化能力。

---

## 4. UI 层：`packages/tui` 与 `packages/web-ui`

UI 层分为终端 UI 和 Web UI 两部分。

### Terminal UI：`packages/tui`

核心能力：

- 终端组件系统
- Editor
- Select list
- Markdown rendering
- Keybindings
- Overlay
- Diff rendering
- Terminal image support
- 终端差量渲染

关键文件：

- `packages/tui/src/`
- `packages/tui/README.md`

### Web UI：`packages/web-ui`

核心能力：

- 浏览器端 Agent / Chat 组件
- `ChatPanel`
- `AgentInterface`
- `MessageList`
- `ModelSelector`
- API key dialog
- 文档和图片附件处理
- JS REPL / document extraction tools

关键文件：

- `packages/web-ui/src/`
- `packages/web-ui/README.md`

理解重点：

如果后续要做自己的交互壳，可以选择复用 `tui` 做 CLI/TUI，也可以复用 `web-ui` 做浏览器界面。

---

## 5. 扩展与集成层

这一层是 pi 的可塑性来源。

核心扩展点：

- SDK：`createAgentSession()`
- 自定义工具：`defineTool()` / `registerTool()`
- 自定义 provider：`registerProvider()`
- 自定义命令：`registerCommand()`
- 事件拦截：
  - `tool_call`
  - `before_agent_start`
  - `context`
  - `before_provider_request`
- 自定义 UI：extension 的 `ctx.ui.custom()`
- RPC 模式：`pi --mode rpc`
- 会话管理：`SessionManager`
- 资源加载：extensions、skills、prompts、AGENTS.md context files

重要示例目录：

- `packages/coding-agent/examples/sdk/`
- `packages/coding-agent/examples/extensions/`

理解重点：

后续自定义 Agent 不一定要直接改内核。可以先用 extension 和 SDK 快速验证设计，再决定是否下沉到 fork 内部实现。

---

## 初步学习路径建议

1. 先读 `packages/agent`，理解最小 Agent 循环。
2. 再读 `packages/coding-agent/src/core/sdk.ts`，理解如何组合出完整 session。
3. 再读内置工具目录 `packages/coding-agent/src/core/tools/`，理解工具规范。
4. 再读 extension 文档和 examples，理解如何无侵入扩展。
5. 最后根据目标选择 UI 层：TUI、Web UI 或 RPC 外接自定义界面。

## 当前判断

- 想做最小可控 Agent：从 `packages/agent` + `packages/ai` 开始。
- 想快速做自己的 Agent 产品：从 `packages/coding-agent` SDK 和 extension 系统开始。
- 想先低风险实验：优先写 `.pi/extensions/` extension，不直接改 core。
