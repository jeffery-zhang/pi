---
tags:
  - pi-mono
  - packages-ai
  - stream
  - complete
  - llm-call-entry
---

# stream / complete 统一调用入口模块学习笔记

> 上级：[[index|Pi Mono 学习库]]
> 所属：[[llm-abstraction-layer|LLM 抽象层学习笔记]]
> 相关：[[model-description-module|Model 描述模块学习笔记]]、[[context-module|Context 统一上下文模块学习笔记]]、[[tool-definition-module|Tool 统一工具定义模块学习笔记]]
> 相关源码：`packages/ai/src/stream.ts`、`packages/ai/src/api-registry.ts`、`packages/ai/src/types.ts`

## 统一调用入口是什么

在 `packages/ai` 里，真正给上层使用的模型调用入口主要是四个函数：

```ts
stream(model, context, options)
complete(model, context, options)

streamSimple(model, context, options)
completeSimple(model, context, options)
```

核心文件：

- `packages/ai/src/stream.ts`

它们的作用是：

```text
上层只需要传入 Model + Context，
不用关心底层到底是 OpenAI、Anthropic、Google 还是 Bedrock。
```

也就是说，统一调用入口把不同 provider 的调用方式包成了一套统一 API。

---

## 为什么需要统一调用入口

不同模型厂商的 SDK 和接口不一样：

```text
OpenAI 有自己的请求格式
Anthropic 有自己的请求格式
Google Gemini 有自己的请求格式
Bedrock 又是 AWS SDK 风格
```

如果上层 Agent 直接调用这些厂商 API，就需要到处写判断：

```ts
if (provider === "openai") {
  // OpenAI 调用方式
} else if (provider === "anthropic") {
  // Anthropic 调用方式
} else if (provider === "google") {
  // Google 调用方式
}
```

这样会让 Agent 层非常混乱。

pi 的做法是：

```text
上层统一调用 stream() / complete()
底层根据 model.api 自动分发到具体 provider 实现
```

---

## 最核心的源码

`packages/ai/src/stream.ts` 的简化逻辑如下：

```ts
export function stream(model, context, options) {
  const provider = resolveApiProvider(model.api);
  return provider.stream(model, context, options);
}

export async function complete(model, context, options) {
  const s = stream(model, context, options);
  return s.result();
}
```

这段代码说明了两个重点：

1. `stream()` 根据 `model.api` 找到真正的 provider 实现。
2. `complete()` 只是 `stream()` 的便捷封装。

---

## `stream()`：流式调用入口

```ts
stream(model, context, options)
```

`stream()` 返回一个 `AssistantMessageEventStream`。

也就是说，它不会只返回最终文本，而是边生成边返回事件。

使用方式：

```ts
const s = stream(model, context);

for await (const event of s) {
  if (event.type === "text_delta") {
    process.stdout.write(event.delta);
  }
}

const finalMessage = await s.result();
```

可以理解为：

```text
stream() = 打开一个模型输出流，边接收边处理
```

适合：

- CLI 里逐字输出
- Web UI 里实时显示
- 观察 thinking delta
- 观察 tool call 参数流式生成
- 构建响应式 Agent UI

---

## `complete()`：一次性调用入口

```ts
complete(model, context, options)
```

`complete()` 返回最终 `AssistantMessage`。

使用方式：

```ts
const message = await complete(model, context);
```

它内部本质是：

```ts
const s = stream(model, context, options);
return s.result();
```

可以理解为：

```text
complete() = 调用 stream()，但不关心中间过程，只等最终结果
```

适合：

- 脚本任务
- 单次问答
- 不需要实时 UI 的场景
- 测试和简单集成

---

## `stream()` 和 `complete()` 的区别

| 函数 | 返回 | 适合场景 |
| --- | --- | --- |
| `stream()` | 事件流 | 实时输出、UI、Agent 观察工具调用过程 |
| `complete()` | 最终 `AssistantMessage` | 简单调用、脚本、测试 |

更直观地说：

```text
stream()   = 看模型边想边说
complete() = 等模型说完再看结果
```

---

## `streamSimple()` 和 `completeSimple()`

除了 `stream()` / `complete()`，还有：

```ts
streamSimple(model, context, options)
completeSimple(model, context, options)
```

它们使用 `SimpleStreamOptions`。

主要区别是：`SimpleStreamOptions` 提供统一 reasoning 配置。

相关类型：

```ts
export interface SimpleStreamOptions extends StreamOptions {
  reasoning?: ThinkingLevel;
  thinkingBudgets?: ThinkingBudgets;
}
```

也就是说，上层可以用统一方式表达：

```ts
reasoning: "low" | "medium" | "high"
```

然后具体 provider 再把它转换成自己的参数格式。

例如：

```ts
const s = streamSimple(model, context, {
  reasoning: "medium",
});
```

可以理解为：

```text
streamSimple / completeSimple
= 带统一 reasoning 抽象的调用入口
```

---

## 调用入口的输入：Model + Context + Options

统一调用入口的三个输入是：

```text
Model
Context
Options
```

### Model

来自 [[model-description-module|Model 描述模块]]。

它决定：

- 调哪个模型
- 属于哪个 provider
- 用哪个 API 协议
- 请求发到哪个 baseUrl
- 是否支持 reasoning / image

### Context

来自 [[context-module|Context 统一上下文模块]]。

它包含：

- `systemPrompt`
- `messages`
- `tools`

### Options

`options` 是本次调用的运行参数。

核心类型是：

```ts
export interface StreamOptions {
  temperature?: number;
  maxTokens?: number;
  signal?: AbortSignal;
  apiKey?: string;
  transport?: Transport;
  cacheRetention?: CacheRetention;
  sessionId?: string;
  onPayload?: (payload, model) => unknown;
  onResponse?: (response, model) => void;
  headers?: Record<string, string>;
  timeoutMs?: number;
  maxRetries?: number;
  maxRetryDelayMs?: number;
  metadata?: Record<string, unknown>;
}
```

---

## 常见 Options 解释

### `temperature`

控制输出随机性。

```ts
temperature: 0.2
```

通常：

- 越低越稳定
- 越高越发散

---

### `maxTokens`

限制本次最大输出 token。

```ts
maxTokens: 4096
```

注意它是本次调用参数，不同于 [[model-description-module|Model]] 里的 `maxTokens` 能力上限。

---

### `signal`

用于取消请求。

```ts
const controller = new AbortController();

const s = stream(model, context, {
  signal: controller.signal,
});

controller.abort();
```

适合用户按 Escape、取消任务、关闭页面等场景。

---

### `apiKey`

显式传入 API key。

```ts
complete(model, context, {
  apiKey: process.env.OPENAI_API_KEY,
});
```

如果不传，provider 通常会尝试从环境变量或 auth storage 中获取。

---

### `transport`

选择传输方式。

```ts
transport: "sse" | "websocket" | "websocket-cached" | "auto"
```

不是所有 provider 都支持。

---

### `cacheRetention`

控制 prompt cache 保留偏好。

```ts
cacheRetention: "none" | "short" | "long"
```

具体如何映射到 provider，由各 provider 自己决定。

---

### `sessionId`

给支持 session-aware caching 的 provider 使用。

例如 OpenAI Codex 这类 provider 可能通过 `sessionId` 做连接复用、缓存或路由。

---

### `onPayload`

请求发送前观察或替换 provider payload。

```ts
onPayload: (payload, model) => {
  console.log(payload);
}
```

适合调试、审计、扩展 provider 行为。

---

### `onResponse`

收到 HTTP response 后、消费 body 前触发。

```ts
onResponse: (response, model) => {
  console.log(response.status, response.headers);
}
```

适合观察响应状态、headers、rate-limit 信息。

---

## 根据 `model.api` 分发 provider

`stream()` 并不知道 OpenAI、Anthropic、Google 怎么调用。

它只做一件事：

```text
根据 model.api 找 provider implementation
```

相关逻辑：

```ts
function resolveApiProvider(api: Api) {
  const provider = getApiProvider(api);
  if (!provider) {
    throw new Error(`No API provider registered for api: ${api}`);
  }
  return provider;
}
```

然后：

```ts
const provider = resolveApiProvider(model.api);
return provider.stream(model, context, options);
```

所以 `model.api` 是分发关键。

例如：

```text
model.api = "openai-responses"
  -> 使用 OpenAI Responses provider

model.api = "anthropic-messages"
  -> 使用 Anthropic Messages provider

model.api = "google-generative-ai"
  -> 使用 Google provider
```

---

## Provider 从哪里注册

provider 注册在：

- `packages/ai/src/api-registry.ts`
- `packages/ai/src/providers/register-builtins.ts`

`api-registry.ts` 提供：

```ts
registerApiProvider(provider)
getApiProvider(api)
getApiProviders()
unregisterApiProviders(sourceId)
clearApiProviders()
```

`register-builtins.ts` 注册内置 provider。

关键点：内置 provider 是懒加载的。

也就是说，一开始只注册入口包装，真正 provider 模块在首次调用时再 import。

这样可以减少启动成本，也避免某些 Node-only provider 在浏览器场景中过早加载。

---

## 完整调用链

```text
开发者调用 complete(model, context)
        ↓
complete() 调用 stream(model, context)
        ↓
stream() 读取 model.api
        ↓
getApiProvider(model.api)
        ↓
找到对应 provider.stream()
        ↓
provider 把统一 Context 转成厂商请求
        ↓
发送请求到 model.baseUrl
        ↓
provider 接收厂商流式响应
        ↓
转成统一 AssistantMessageEventStream
        ↓
complete() 等待 stream.result()
        ↓
返回最终 AssistantMessage
```

---

## `AssistantMessageEventStream` 的位置

`stream()` 返回的是：

```ts
AssistantMessageEventStream
```

它是一个 async iterable。

你可以：

```ts
for await (const event of s) {
  // 逐个处理事件
}
```

也可以：

```ts
const finalMessage = await s.result();
```

因此它同时支持两种使用方式：

```text
实时消费事件
等待最终结果
```

详细事件协议后续会单独拆解。

---

## 最小 complete 示例

```ts
import { complete, getModel } from "@earendil-works/pi-ai";

const model = getModel("openai", "gpt-4o-mini");

const message = await complete(model, {
  systemPrompt: "You are a helpful assistant.",
  messages: [
    {
      role: "user",
      content: "你好，介绍一下自己",
      timestamp: Date.now(),
    },
  ],
});

console.log(message.content);
```

---

## 最小 stream 示例

```ts
import { getModel, stream } from "@earendil-works/pi-ai";

const model = getModel("openai", "gpt-4o-mini");

const s = stream(model, {
  messages: [
    {
      role: "user",
      content: "用一句话解释 AI Agent",
      timestamp: Date.now(),
    },
  ],
});

for await (const event of s) {
  if (event.type === "text_delta") {
    process.stdout.write(event.delta);
  }
}

const finalMessage = await s.result();
```

---

## 和 Agent 层的关系

`packages/ai` 的统一调用入口只负责“一次 LLM 调用”。

它不负责：

- 自动执行工具
- 多轮 tool loop
- 管理消息历史
- compaction
- session 持久化
- 用户交互

这些由更上层负责：

```text
packages/agent
  用 stream()/complete() 驱动 Agent loop
  处理 tool call
  执行工具
  把 tool result 放回 Context

packages/coding-agent
  在 Agent loop 上构建 CLI、SDK、RPC、TUI、session、extensions
```

所以可以这样理解：

```text
packages/ai 的 stream/complete
  = 单次模型调用入口

packages/agent 的 Agent / agentLoop
  = 多轮 Agent 工作循环
```

---

## AI 应用小白视角的类比

可以把 `stream()` / `complete()` 想象成“统一客服窗口”。

你只需要提交：

```text
我要找哪个模型：Model
我要给它看的材料：Context
这次调用的要求：Options
```

统一窗口会自动判断：

```text
这个模型该去 OpenAI 窗口？
还是 Anthropic 窗口？
还是 Google 窗口？
```

然后帮你把材料翻译成对应厂商能看懂的格式。

你不用自己挨个适配每家厂商。

---

## 一句话总结

`stream()` / `complete()` 是 `packages/ai` 的统一模型调用入口。

它们让上层只需要关心：

```text
Model + Context + Options
```

而不需要关心：

```text
OpenAI 怎么调
Anthropic 怎么调
Google 怎么调
Bedrock 怎么调
```

`stream()` 适合实时消费事件，`complete()` 适合等待最终结果。

---

## 返回与延伸

- 返回：[[llm-abstraction-layer|LLM 抽象层学习笔记]]
- 相关：[[model-description-module|Model 描述模块学习笔记]]
- 相关：[[context-module|Context 统一上下文模块学习笔记]]
- 相关：[[tool-definition-module|Tool 统一工具定义模块学习笔记]]
- 下一篇：[[stream-event-module|AssistantMessageEventStream 统一流事件模块学习笔记]]
- 总览：[[architecture-learning-notes|Pi Mono 框架拆解学习笔记]]
- 首页：[[index|Pi Mono 学习库]]
