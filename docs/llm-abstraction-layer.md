---
tags:
  - pi-mono
  - packages-ai
  - llm-abstraction
---

# LLM 抽象层学习笔记：`packages/ai`

> 上级：[[index|Pi Mono 学习库]]
> 来源：[[architecture-learning-notes|Pi Mono 框架拆解学习笔记]]
> 下一步：[[model-description-module|Model 描述模块学习笔记]]、[[context-module|Context 统一上下文模块学习笔记]]、[[tool-definition-module|Tool 统一工具定义模块学习笔记]]、[[unified-call-entry-module|stream / complete 统一调用入口模块学习笔记]]、[[stream-event-module|AssistantMessageEventStream 统一流事件模块学习笔记]]


## 定位

`packages/ai` 是 pi-mono 中最底层的 LLM 抽象层。

它的目标是：把不同 LLM 厂商 API 抽象成一套统一的模型、消息、工具和事件协议。

它本身不是完整 Agent。它不负责执行工具、不负责会话管理、不负责 UI，也不负责代码编辑工作流。它负责的是：

- 模型描述
- 消息协议
- 工具 schema
- provider 注册
- provider 调用
- 流式事件标准化
- token / cost 统计
- provider-specific 差异适配

---

## 1. Model：模型描述

详细拆解：[[model-description-module|Model 描述模块学习笔记]]

核心类型：

- `packages/ai/src/types.ts`

`Model` 描述一个具体模型：

```ts
interface Model<TApi extends Api> {
  id: string;
  name: string;
  api: TApi;
  provider: Provider;
  baseUrl: string;
  reasoning: boolean;
  input: ("text" | "image")[];
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };
  contextWindow: number;
  maxTokens: number;
}
```

关键字段：

- `provider`：模型供应商，例如 `openai`、`anthropic`、`google`
- `api`：调用协议，例如：
  - `openai-completions`
  - `openai-responses`
  - `anthropic-messages`
  - `google-generative-ai`
- `id`：模型 ID，例如 `gpt-4o`、`claude-sonnet-*`
- `reasoning`：是否支持 reasoning / thinking
- `input`：支持文本、图片，或两者
- `contextWindow`：上下文窗口大小
- `maxTokens`：最大输出 token 数
- `cost`：每百万 tokens 的价格

重要理解：

`provider` 和 `api` 是分开的。

多个 provider 可以复用同一个 API 协议。例如 OpenRouter、Groq、DeepSeek、Ollama-compatible API 都可能复用类似 OpenAI completions 的协议。

---

## 2. Context：统一上下文

详细拆解：[[context-module|Context 统一上下文模块学习笔记]]

调用模型时传入统一 `Context`：

```ts
interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

它包含三部分：

1. `systemPrompt`
2. `messages`
3. `tools`

消息类型统一为：

```ts
type Message = UserMessage | AssistantMessage | ToolResultMessage;
```

### UserMessage

```ts
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;
}
```

用户消息可以是纯文本，也可以是文本和图片组成的内容块。

### AssistantMessage

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

助手消息可以包含：

- 普通文本：`TextContent`
- reasoning / thinking：`ThinkingContent`
- 工具调用：`ToolCall`

### ToolResultMessage

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

工具结果也支持文本和图片。

---

## 3. Tool：统一工具定义

详细拆解：[[tool-definition-module|Tool 统一工具定义模块学习笔记]]

工具定义使用 TypeBox schema：

```ts
interface Tool<TParameters extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: TParameters;
}
```

示例：

```ts
const tool = {
  name: "get_weather",
  description: "Get weather for a city",
  parameters: Type.Object({
    city: Type.String(),
  }),
};
```

重要理解：

`packages/ai` 只定义工具 schema，不负责真正执行工具。

真正执行工具的是更上一层：

- `packages/agent`
- `packages/coding-agent`

`packages/ai` 负责：

1. 把统一工具 schema 转成 provider 能理解的格式。
2. 接收模型返回的 tool call。
3. 把 provider-specific tool call 转成统一 `ToolCall`。

---

## 4. stream / complete：统一调用入口

详细拆解：[[unified-call-entry-module|stream / complete 统一调用入口模块学习笔记]]

核心文件：

- `packages/ai/src/stream.ts`

主要导出：

```ts
stream(model, context, options)
complete(model, context, options)

streamSimple(model, context, options)
completeSimple(model, context, options)
```

### `stream()`

`stream()` 返回一个事件流：

```ts
const s = stream(model, context);

for await (const event of s) {
  // 处理 text_delta / toolcall_delta / done 等事件
}
```

### `complete()`

`complete()` 是 `stream()` 的简化封装：

```ts
const message = await complete(model, context);
```

其内部逻辑可以理解为：

```ts
const s = stream(model, context, options);
return s.result();
```

也就是说：

`complete()` = 启动 `stream()`，然后等待最终 `AssistantMessage`。

---

## 5. AssistantMessageEventStream：统一流事件

详细拆解：[[stream-event-module|AssistantMessageEventStream 统一流事件模块学习笔记]]

核心文件：

- `packages/ai/src/utils/event-stream.ts`
- `packages/ai/src/types.ts`

核心事件：

```ts
start
text_start
text_delta
text_end
thinking_start
thinking_delta
thinking_end
toolcall_start
toolcall_delta
toolcall_end
done
error
```

不同 provider 的真实流式协议完全不同：

- OpenAI SSE
- Anthropic Messages stream
- Google Gemini stream
- Bedrock Converse stream

但 `packages/ai` 会把它们统一成同一套事件。

因此上层 Agent 不需要理解每个厂商的流式协议差异，只需要消费统一 `AssistantMessageEvent`。

---

## 6. Provider Registry：根据 `api` 找实现

核心文件：

- `packages/ai/src/api-registry.ts`
- `packages/ai/src/providers/register-builtins.ts`

调用流程：

```ts
stream(model, context)
  -> 根据 model.api 找到 provider implementation
  -> 调用对应 provider.stream()
  -> 返回统一 AssistantMessageEventStream
```

简化理解：

```ts
const provider = getApiProvider(model.api);
return provider.stream(model, context, options);
```

内置 provider 在这里注册：

- `packages/ai/src/providers/register-builtins.ts`

每种 API 协议有一个 provider 实现文件，例如：

- `packages/ai/src/providers/openai-completions.ts`
- `packages/ai/src/providers/openai-responses.ts`
- `packages/ai/src/providers/anthropic.ts`
- `packages/ai/src/providers/google.ts`
- `packages/ai/src/providers/amazon-bedrock.ts`

---

## 最小调用链

整体工作流可以理解为：

```text
getModel(provider, modelId)
        ↓
Model
        ↓
Context = systemPrompt + messages + tools
        ↓
stream(model, context)
        ↓
根据 model.api 找 provider 实现
        ↓
provider 把 Context 转成厂商请求
        ↓
厂商返回流式响应
        ↓
provider 转成统一 AssistantMessageEvent
        ↓
最终得到 AssistantMessage
```

---

## 最小示例

```ts
import { getModel, complete, Type } from "@earendil-works/pi-ai";

const model = getModel("openai", "gpt-4o-mini");

const result = await complete(model, {
  systemPrompt: "You are a helpful assistant.",
  messages: [
    {
      role: "user",
      content: "Hello",
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
});

console.log(result.content);
```

---

## 当前阶段推荐阅读顺序

先读这些文件：

1. `packages/ai/src/types.ts`
2. `packages/ai/src/stream.ts`
3. `packages/ai/src/models.ts`
4. `packages/ai/src/api-registry.ts`
5. `packages/ai/src/providers/register-builtins.ts`

读完后应能理解：

- 模型如何表示
- 消息如何表示
- 工具如何表示
- 统一调用入口是什么
- provider 如何注册和分发
- 流式事件如何标准化

下一步可以继续深入某个具体 provider，例如 Anthropic 或 OpenAI，观察它如何把统一 `Context` 转成厂商 API 请求。
