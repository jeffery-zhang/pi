---
tags:
  - pi-mono
  - packages-ai
  - stream-events
  - assistant-message-event-stream
---

# AssistantMessageEventStream 统一流事件模块学习笔记

> 上级：[[index|Pi Mono 学习库]]
> 所属：[[llm-abstraction-layer|LLM 抽象层学习笔记]]
> 相关：[[unified-call-entry-module|stream / complete 统一调用入口模块学习笔记]]、[[context-module|Context 统一上下文模块学习笔记]]、[[tool-definition-module|Tool 统一工具定义模块学习笔记]]
> 相关源码：`packages/ai/src/types.ts`、`packages/ai/src/utils/event-stream.ts`、`packages/ai/src/providers/*`

## 统一流事件是什么

在 `packages/ai` 里，`stream()` 返回的不是普通字符串，而是一个统一事件流：

```ts
AssistantMessageEventStream
```

它的作用是：把不同 provider 的流式输出统一成一套事件协议。

你可以把它理解成：

```text
模型边生成，框架边发事件。
```

这些事件会描述：

- 模型开始响应
- 文本开始输出
- 文本增量输出
- 文本输出结束
- thinking 开始/增量/结束
- tool call 开始/增量/结束
- 最终完成
- 错误或中止

---

## 为什么需要统一流事件

不同 LLM 厂商的流式协议完全不同。

例如：

```text
OpenAI 使用自己的 SSE event
Anthropic 使用 Messages stream event
Google Gemini 使用自己的 chunk 格式
Bedrock 使用 AWS Converse stream
```

如果上层 Agent 或 UI 直接消费这些厂商事件，就必须写很多 provider-specific 逻辑。

pi 的做法是：

```text
provider-specific stream
        ↓
AssistantMessageEvent
        ↓
上层 Agent / UI 统一消费
```

这样上层只需要理解一种事件协议。

---

## 核心类型位置

统一事件类型在：

- `packages/ai/src/types.ts`

事件流实现位置：

- `packages/ai/src/utils/event-stream.ts`

核心类型：

```ts
export type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
  | { type: "error"; reason: "aborted" | "error"; error: AssistantMessage };
```

---

## AssistantMessageEventStream 是什么

`AssistantMessageEventStream` 是一个 async iterable。

也就是说可以这样消费：

```ts
const s = stream(model, context);

for await (const event of s) {
  // 逐个处理事件
}
```

同时它还有：

```ts
s.result()
```

用于获取最终 `AssistantMessage`。

因此它同时支持两种使用方式：

```text
for await (...)  实时处理事件
s.result()       等待最终消息
```

---

## EventStream 的内部机制

基础类在：

- `packages/ai/src/utils/event-stream.ts`

简化理解：

```ts
class EventStream<T, R> implements AsyncIterable<T> {
  push(event: T): void
  end(result?: R): void
  result(): Promise<R>
}
```

它内部维护：

```text
queue：已经产生但还没被消费的事件
waiting：正在等待下一个事件的消费者
done：流是否结束
finalResultPromise：最终结果
```

provider 会不断调用：

```ts
stream.push(event)
```

消费者通过：

```ts
for await (const event of stream)
```

逐个拿到事件。

---

## 事件整体生命周期

一个典型的文本回答事件顺序：

```text
start
text_start
text_delta
text_delta
text_delta
text_end
done
```

如果模型有 thinking：

```text
start
thinking_start
thinking_delta
thinking_delta
thinking_end
text_start
text_delta
text_end
done
```

如果模型调用工具：

```text
start
toolcall_start
toolcall_delta
toolcall_delta
toolcall_end
done(reason = "toolUse")
```

如果用户 abort：

```text
start
text_start
text_delta
error(reason = "aborted")
```

实际事件顺序会根据 provider 和模型能力有所变化，但最终都遵循统一协议。

---

## `start`：响应开始

```ts
{ type: "start"; partial: AssistantMessage }
```

`start` 表示模型响应开始。

它携带一个初始的 `partial` assistant message。

这个 `partial` 通常已经包含：

- `role: "assistant"`
- `provider`
- `api`
- `model`
- 初始空 content
- 初始 usage
- timestamp

上层 Agent 在收到 `start` 后，通常会把这个 partial assistant message 放进当前 context，并开始显示“模型正在回答”。

---

## `partial` 是什么

很多事件里都有：

```ts
partial: AssistantMessage
```

`partial` 表示“截至当前事件为止的 assistant message 快照”。

例如模型已经输出：

```text
你好，我是
```

那么 `partial.content` 里可能已经包含这部分文本。

下一个 delta 来了以后，`partial` 会变成：

```text
你好，我是一个 AI 助手
```

因此：

```text
partial = 当前累计状态
```

而：

```text
delta = 本次新增片段
```

---

## `contentIndex` 是什么

很多事件都有：

```ts
contentIndex: number
```

`AssistantMessage.content` 是一个数组。

它可能包含多个内容块：

```ts
content: [
  { type: "thinking", thinking: "..." },
  { type: "text", text: "..." },
  { type: "toolCall", name: "...", arguments: {...} }
]
```

`contentIndex` 表示当前事件对应的是 content 数组里的第几个块。

例如：

```ts
contentIndex: 1
```

表示当前 delta 更新的是：

```ts
partial.content[1]
```

---

## 文本事件

文本事件有三种：

```ts
text_start
text_delta
text_end
```

### `text_start`

```ts
{ type: "text_start"; contentIndex: number; partial: AssistantMessage }
```

表示一个文本内容块开始。

UI 可以在这里准备渲染一个新的文本块。

### `text_delta`

```ts
{ type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
```

表示新增了一小段文本。

例如：

```ts
{ type: "text_delta", delta: "你好" }
```

CLI 或 Web UI 通常会实时显示 `delta`。

### `text_end`

```ts
{ type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
```

表示这个文本块结束。

`content` 是该文本块的最终完整文本。

---

## Thinking 事件

Thinking 事件也有三种：

```ts
thinking_start
thinking_delta
thinking_end
```

它们用于支持有显式思考输出的模型。

### `thinking_start`

表示 thinking 内容块开始。

### `thinking_delta`

表示新增 thinking 片段。

```ts
{ type: "thinking_delta", delta: "..." }
```

### `thinking_end`

表示 thinking 内容块结束。

注意：不同 provider 对 thinking 的支持和展示规则不同。有些模型不会输出 thinking，有些会输出签名或 redacted thinking。

---

## ToolCall 事件

工具调用事件有三种：

```ts
toolcall_start
toolcall_delta
toolcall_end
```

### `toolcall_start`

表示一个工具调用内容块开始。

模型可能准备调用某个工具。

### `toolcall_delta`

```ts
{ type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
```

表示工具调用参数正在流式生成。

例如模型可能一点点生成 JSON 参数：

```json
{"path":"README.md"}
```

delta 可能是：

```text
{"path"
: "README
.md"}
```

实际 provider 会把它们累积到 `partial.content[contentIndex]` 中。

### `toolcall_end`

```ts
{ type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
```

表示工具调用生成完成。

此时可以拿到完整 `ToolCall`：

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

上层 Agent 通常会在 assistant message 完成后统一执行这些 tool calls。

---

## `done`：正常完成

```ts
{ type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
```

`done` 表示本次 assistant response 正常结束。

`reason` 有三类：

```text
stop     正常回答结束
length   达到长度限制
toolUse  模型请求调用工具
```

`message` 是最终完整的 `AssistantMessage`。

当事件流收到 `done` 时，`AssistantMessageEventStream.result()` 会解析为这个 `message`。

---

## `error`：错误或中止

```ts
{ type: "error"; reason: "aborted" | "error"; error: AssistantMessage }
```

`error` 表示本次 assistant response 异常结束。

两种原因：

```text
aborted  用户或上层主动取消
error    provider 请求失败、网络错误、解析错误等
```

`error` 字段也是一个 `AssistantMessage`，只是它的：

```ts
stopReason: "aborted" | "error"
errorMessage: string
```

当事件流收到 `error` 时，`AssistantMessageEventStream.result()` 会解析为这个 error message。

注意：这里不是 Promise reject，而是返回一个带 `stopReason` 的 assistant message。

这让上层可以把错误/中止也当作对话历史的一部分处理。

---

## `done` 和 `error` 都是终止事件

`AssistantMessageEventStream` 的构造逻辑：

```ts
(event) => event.type === "done" || event.type === "error"
```

也就是说：

```text
done  终止流并设置最终结果
error 终止流并设置最终结果
```

最终结果提取逻辑：

```ts
if (event.type === "done") return event.message;
if (event.type === "error") return event.error;
```

所以无论正常完成还是错误结束：

```ts
await stream.result()
```

都会得到一个 `AssistantMessage`。

区别在于：

```ts
message.stopReason
```

---

## stream() 消费示例

```ts
const s = stream(model, context);

for await (const event of s) {
  switch (event.type) {
    case "text_delta":
      process.stdout.write(event.delta);
      break;

    case "thinking_delta":
      // 可以渲染 thinking 区域
      break;

    case "toolcall_delta":
      // 可以显示工具参数正在生成
      break;

    case "done":
      console.log("完成", event.reason);
      break;

    case "error":
      console.error("失败", event.reason, event.error.errorMessage);
      break;
  }
}

const finalMessage = await s.result();
```

---

## Agent 层如何使用这些事件

在 `packages/agent/src/agent-loop.ts` 里，`streamAssistantResponse()` 会消费这些事件。

简化逻辑：

```text
收到 start
  -> 把 partial assistant message 加入 context
  -> emit message_start

收到 text/thinking/toolcall delta
  -> 更新 context 中最后一个 assistant partial
  -> emit message_update

收到 done/error
  -> 用 final message 替换 partial
  -> emit message_end
  -> 返回 final message
```

这解释了为什么用户 abort 后，已输出的部分内容通常会保留在 context 中：

```text
partial assistant message 在 start 后就已经进入 context，
delta 过程中持续更新，
error(aborted) 时再替换成最终 aborted message。
```

---

## Provider 层如何产生事件

每个 provider 文件都会把厂商原始流转成 `AssistantMessageEvent`。

例如：

- `packages/ai/src/providers/openai-completions.ts`
- `packages/ai/src/providers/openai-responses.ts`
- `packages/ai/src/providers/anthropic.ts`
- `packages/ai/src/providers/google.ts`
- `packages/ai/src/providers/amazon-bedrock.ts`

它们内部通常会：

```ts
const stream = new AssistantMessageEventStream();
stream.push({ type: "start", partial: output });
stream.push({ type: "text_delta", ... });
stream.push({ type: "done", message: output, reason: "stop" });
```

provider 的职责就是翻译：

```text
厂商原始事件 -> pi 统一事件
```

---

## 与 complete() 的关系

[[unified-call-entry-module|complete()]] 内部其实也是调用 `stream()`：

```ts
const s = stream(model, context, options);
return s.result();
```

所以：

```text
stream() 能看到完整事件过程
complete() 只拿最终 AssistantMessage
```

两者底层用的是同一套 `AssistantMessageEventStream`。

---

## AI 应用小白视角的类比

可以把统一流事件想象成“直播字幕系统”。

模型不是一次性把完整答案给你，而是不断发来：

```text
我开始了
我新增了一段文本
我又新增了一段文本
我准备调用工具
工具参数生成完了
我结束了
```

UI 或 Agent 就可以边收到边做事：

- CLI 逐字显示
- Web UI 实时渲染
- Agent 观察工具调用
- 用户随时 abort
- 最终把完整消息写入历史

---

## 一句话总结

`AssistantMessageEventStream` 是 `packages/ai` 的统一流式响应协议。

它把不同 provider 的流式输出统一成：

```text
start
text / thinking / toolcall 的 start-delta-end
done / error
```

上层 Agent 和 UI 不需要理解各厂商原始流，只需要消费这一套统一事件。

---

## 返回与延伸

- 返回：[[llm-abstraction-layer|LLM 抽象层学习笔记]]
- 相关：[[unified-call-entry-module|stream / complete 统一调用入口模块学习笔记]]
- 相关：[[tool-definition-module|Tool 统一工具定义模块学习笔记]]
- 相关：[[context-module|Context 统一上下文模块学习笔记]]
- 下一篇建议：Provider Registry 模块
- 总览：[[architecture-learning-notes|Pi Mono 框架拆解学习笔记]]
- 首页：[[index|Pi Mono 学习库]]
