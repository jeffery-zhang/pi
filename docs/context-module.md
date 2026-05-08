---
tags:
  - pi-mono
  - packages-ai
  - context
  - messages
---

# Context 统一上下文模块学习笔记

> 上级：[[index|Pi Mono 学习库]]
> 所属：[[llm-abstraction-layer|LLM 抽象层学习笔记]]
> 相关：[[model-description-module|Model 描述模块学习笔记]]、[[tool-definition-module|Tool 统一工具定义模块学习笔记]]
> 相关源码：`packages/ai/src/types.ts`

## Context 是什么

在 `packages/ai` 里，`Context` 可以理解成每次调用 LLM 时，框架交给模型的一整个“对话包”。

它告诉模型：

```text
你是谁？
之前发生了什么？
用户现在说了什么？
你可以使用哪些工具？
工具之前返回了什么结果？
```

核心类型位置：

- `packages/ai/src/types.ts`

源码结构：

```ts
export interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

可以简单理解为：

```text
Context = 系统提示词 + 对话历史 + 可用工具
```

---

## Context 的三个组成部分

```text
Context
├── systemPrompt
├── messages
└── tools
```

---

## `systemPrompt`：告诉模型它是谁

```ts
systemPrompt?: string;
```

`systemPrompt` 是系统提示词。

它通常用来定义模型身份、行为边界和工作规则。

例子：

```ts
systemPrompt: "You are a helpful coding assistant."
```

意思是：你是一个有帮助的 coding assistant。

更复杂的例子：

```ts
systemPrompt: `
You are a senior TypeScript engineer.
Be concise.
Never modify files unless explicitly asked.
`
```

它不是普通用户消息，而是更高优先级的角色设定。

---

## `messages`：对话历史

```ts
messages: Message[];
```

`messages` 是上下文里最重要的部分。它保存整个对话历史。

消息类型统一为：

```ts
type Message = UserMessage | AssistantMessage | ToolResultMessage;
```

也就是三类消息：

```text
Message
├── UserMessage        用户说的话
├── AssistantMessage   模型说的话 / 工具调用
└── ToolResultMessage  工具执行结果
```

---

## UserMessage：用户消息

```ts
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;
}
```

最简单的用户消息：

```ts
{
  role: "user",
  content: "帮我解释这个项目",
  timestamp: Date.now()
}
```

`role: "user"` 表示这是用户输入。

用户消息可以是字符串：

```ts
content: "Hello"
```

也可以是内容块数组：

```ts
content: [
  { type: "text", text: "这张图里是什么？" },
  { type: "image", data: "...base64...", mimeType: "image/png" }
]
```

因此它支持：

- 文本
- 图片
- 文本 + 图片

---

## AssistantMessage：模型消息

```ts
interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: Api;
  provider: Provider;
  model: string;
  usage: Usage;
  stopReason: StopReason;
  timestamp: number;
}
```

这是模型返回的消息。

它不只是文本，也可能包含：

```text
AssistantMessage.content
├── TextContent       普通回答
├── ThinkingContent   思考内容
└── ToolCall          工具调用请求
```

普通回答示例：

```ts
{
  role: "assistant",
  content: [
    { type: "text", text: "这个项目分为五层..." }
  ],
  provider: "anthropic",
  api: "anthropic-messages",
  model: "claude-sonnet-...",
  usage: {...},
  stopReason: "stop",
  timestamp: Date.now()
}
```

如果模型想调用工具，则可能返回：

```ts
content: [
  {
    type: "toolCall",
    id: "call_123",
    name: "read_file",
    arguments: {
      path: "README.md"
    }
  }
]
```

这表示：模型不是直接回答，而是请求执行 `read_file` 工具。

---

## ToolResultMessage：工具结果消息

```ts
interface ToolResultMessage {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  isError: boolean;
  timestamp: number;
}
```

当模型请求调用工具后，上层 Agent 会真正执行工具，然后把结果塞回上下文。

例如模型调用：

```ts
{
  type: "toolCall",
  id: "call_123",
  name: "read_file",
  arguments: {
    path: "README.md"
  }
}
```

工具执行后，返回：

```ts
{
  role: "toolResult",
  toolCallId: "call_123",
  toolName: "read_file",
  content: [
    {
      type: "text",
      text: "# Pi Agent Harness Mono Repo..."
    }
  ],
  isError: false,
  timestamp: Date.now()
}
```

注意：`toolCallId` 必须对应之前的工具调用 ID。

这样模型才知道：

```text
这是我刚才请求的那个工具的结果。
```

---

## `tools`：告诉模型有哪些工具能用

```ts
tools?: Tool[];
```

`tools` 是当前可用工具列表。

例子：

```ts
tools: [
  {
    name: "read_file",
    description: "Read a file from disk",
    parameters: Type.Object({
      path: Type.String()
    })
  }
]
```

这表示模型可以调用一个叫 `read_file` 的工具，并且必须传入：

```ts
{
  path: string
}
```

重要理解：

`packages/ai` 只告诉模型“有什么工具”和“参数长什么样”，不负责执行工具。

真正执行工具的是：

- `packages/agent`
- `packages/coding-agent`

---

## Context 为什么要统一

不同厂商的上下文格式不一样。

OpenAI 可能需要类似：

```ts
messages: [
  { role: "system", content: "..." },
  { role: "user", content: "..." }
]
```

Anthropic 可能需要类似：

```ts
system: "...",
messages: [
  { role: "user", content: [...] }
]
```

Google Gemini 又是另一套格式。

如果上层 Agent 直接适配每个厂商，代码会很乱。

所以 pi 先定义自己的统一格式：

```ts
Context
```

再由各个 provider 负责转换：

```text
统一 Context
   ↓
OpenAI request / Anthropic request / Google request
```

这样上层 Agent 永远只需要构造一种 Context。

---

## 一次完整工具调用中的 Context 演化

假设用户说：

```text
读取 README.md 并总结
```

初始 Context：

```ts
{
  systemPrompt: "You are a coding assistant.",
  messages: [
    {
      role: "user",
      content: "读取 README.md 并总结",
      timestamp: 1
    }
  ],
  tools: [
    read_file_tool
  ]
}
```

模型返回 tool call：

```ts
{
  role: "assistant",
  content: [
    {
      type: "toolCall",
      id: "call_1",
      name: "read_file",
      arguments: {
        path: "README.md"
      }
    }
  ],
  stopReason: "toolUse",
  ...
}
```

Agent 把它加入 `messages`：

```ts
messages: [
  userMessage,
  assistantToolCallMessage
]
```

然后 Agent 执行工具，加入工具结果：

```ts
messages: [
  userMessage,
  assistantToolCallMessage,
  {
    role: "toolResult",
    toolCallId: "call_1",
    toolName: "read_file",
    content: [
      { type: "text", text: "...README 内容..." }
    ],
    isError: false,
    timestamp: 3
  }
]
```

然后再次调用模型。

模型现在能看到：

```text
用户让它读取 README
它自己请求了 read_file
工具返回了 README 内容
```

于是它生成最终总结。

---

## Context 和 Agent 的关系

在 `packages/ai` 里，`Context` 只是数据结构。

它不决定：

- 是否继续下一轮
- 是否执行工具
- 工具怎么执行
- 什么时候压缩历史
- 什么时候停止

这些由更上层负责：

- `packages/agent`
- `packages/coding-agent`

分工可以理解为：

```text
packages/ai
  负责定义 Context，并把 Context 发给模型

packages/agent
  负责维护 Context，执行工具，循环调用模型

packages/coding-agent
  负责把 Context 用在真实 coding agent 产品里
```

---

## Context 最小例子

### 无工具、纯文本

```ts
const context = {
  systemPrompt: "You are a helpful assistant.",
  messages: [
    {
      role: "user",
      content: "你好，介绍一下自己",
      timestamp: Date.now(),
    },
  ],
};
```

### 有工具

```ts
const context = {
  systemPrompt: "You are a helpful assistant.",
  messages: [
    {
      role: "user",
      content: "现在几点？",
      timestamp: Date.now(),
    },
  ],
  tools: [
    {
      name: "get_time",
      description: "Get current time",
      parameters: Type.Object({}),
    },
  ],
};
```

### 带图片

```ts
const context = {
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "这张图里有什么？" },
        {
          type: "image",
          data: "base64...",
          mimeType: "image/png",
        },
      ],
      timestamp: Date.now(),
    },
  ],
};
```

---

## 一句话总结

`Context` 是每次发给 LLM 的统一上下文包。

它包含：

```text
systemPrompt：模型身份和规则
messages：用户、模型、工具结果组成的历史
tools：模型当前可以调用的工具说明
```

它的价值是：

上层 Agent 只构造一种 Context，各个 provider 自己负责把它转换成 OpenAI、Anthropic、Google 等厂商需要的格式。

---

## 返回与延伸

- 返回：[[llm-abstraction-layer|LLM 抽象层学习笔记]]
- 上一篇：[[model-description-module|Model 描述模块学习笔记]]
- 下一篇：[[tool-definition-module|Tool 统一工具定义模块学习笔记]]
- 总览：[[architecture-learning-notes|Pi Mono 框架拆解学习笔记]]
- 首页：[[index|Pi Mono 学习库]]
