---
tags:
  - pi-mono
  - packages-ai
  - tools
  - tool-calling
---

# Tool 统一工具定义模块学习笔记

> 上级：[[index|Pi Mono 学习库]]
> 所属：[[llm-abstraction-layer|LLM 抽象层学习笔记]]
> 相关：[[context-module|Context 统一上下文模块学习笔记]]、[[model-description-module|Model 描述模块学习笔记]]
> 相关源码：`packages/ai/src/types.ts`、`packages/ai/src/utils/validation.ts`、`packages/ai/src/utils/typebox-helpers.ts`

## Tool 是什么

在 `packages/ai` 里，`Tool` 可以理解成一张“工具说明书”。

它告诉模型：

```text
你可以调用什么工具？
这个工具是干什么的？
调用时必须传哪些参数？
参数分别是什么类型？
```

核心类型位置：

- `packages/ai/src/types.ts`

源码结构：

```ts
export interface Tool<TParameters extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: TParameters;
}
```

可以简单理解为：

```text
Tool = 工具名 + 工具说明 + 参数 schema
```

---

## 为什么 LLM 需要 Tool 定义

普通聊天时，模型只能根据已有知识回答。

但 AI Agent 需要能做事，例如：

- 读取文件
- 搜索代码
- 执行命令
- 查询数据库
- 调用外部 API
- 创建工单
- 获取当前时间

模型本身不会真的读取文件或执行命令。它只能提出一个“工具调用请求”。

因此框架需要先告诉模型：

```text
你有哪些工具可以调用，以及调用格式是什么。
```

这就是 `Tool` 的作用。

---

## 一个最小 Tool 示例

```ts
import { Type } from "@earendil-works/pi-ai";

const getTimeTool = {
  name: "get_time",
  description: "Get the current time",
  parameters: Type.Object({}),
};
```

这个工具告诉模型：

- 工具名叫 `get_time`
- 用来获取当前时间
- 不需要任何参数

---

## 一个带参数的 Tool 示例

```ts
import { Type } from "@earendil-works/pi-ai";

const readFileTool = {
  name: "read_file",
  description: "Read a file from disk",
  parameters: Type.Object({
    path: Type.String({ description: "Path of the file to read" }),
  }),
};
```

这个工具告诉模型：

```text
可以调用 read_file 工具。
调用时必须传入 path。
path 必须是字符串。
```

模型如果想读取 README.md，就可能生成：

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

---

## Tool 的三个字段

### `name`：工具名

```ts
name: string;
```

`name` 是模型调用工具时使用的名字。

例子：

```ts
"read_file"
"get_time"
"search_web"
"run_sql"
```

工具名应该稳定、清晰、短小。

模型最终返回的 `ToolCall.name` 会和这个字段匹配。

---

### `description`：工具说明

```ts
description: string;
```

`description` 是给模型看的说明，不是给用户看的 UI 文案。

它会影响模型什么时候选择这个工具。

例如：

```ts
description: "Read a file from disk by path. Use this when you need exact file contents."
```

比下面这种更有帮助：

```ts
description: "Read file"
```

因为模型需要通过 description 判断：

- 这个工具适合什么场景？
- 什么时候应该用？
- 什么时候不应该用？

---

### `parameters`：参数 schema

```ts
parameters: TParameters;
```

`parameters` 定义工具调用参数。

pi 使用 TypeBox 来描述参数 schema。

常见例子：

```ts
Type.Object({
  path: Type.String(),
  maxBytes: Type.Optional(Type.Number()),
})
```

它表示工具参数应该长这样：

```ts
{
  path: string;
  maxBytes?: number;
}
```

---

## TypeBox 是什么

TypeBox 是一个用 TypeScript 写 JSON Schema 的库。

为什么这里需要 schema？

因为模型生成的工具参数本质上是 JSON。

框架需要检查：

- 参数是不是对象？
- 必填字段有没有缺？
- 字段类型对不对？
- 枚举值是否合法？
- 数组和嵌套对象是否符合要求？

所以 `Tool.parameters` 不是普通 TypeScript 类型，而是运行时也能检查的 schema。

---

## Tool 定义不等于 Tool 执行

这是最重要的分工。

在 `packages/ai` 里，`Tool` 只描述工具，不执行工具。

它负责：

1. 告诉模型有哪些工具。
2. 告诉模型工具参数格式。
3. 帮 provider 把工具定义转换成厂商 API 格式。
4. 帮上层校验模型返回的 tool call 参数。

它不负责：

- 读取文件
- 执行 bash
- 调用数据库
- 发 HTTP 请求
- 把工具结果写回上下文

真正执行工具的是更上层：

- `packages/agent`
- `packages/coding-agent`

可以理解为：

```text
packages/ai 的 Tool
  = 工具说明书

packages/agent 的 AgentTool
  = 工具说明书 + execute 函数
```

---

## Tool 如何放进 Context

Tool 会通过 [[context-module|Context]] 传给模型。

```ts
const context = {
  systemPrompt: "You are a helpful assistant.",
  messages: [
    {
      role: "user",
      content: "读取 README.md",
      timestamp: Date.now(),
    },
  ],
  tools: [readFileTool],
};
```

调用模型时：

```ts
complete(model, context)
```

provider 会把 `context.tools` 转成对应厂商的工具格式。

---

## ToolCall：模型返回的工具调用请求

当模型决定使用工具时，会在 `AssistantMessage.content` 里返回 `ToolCall`。

核心类型：

```ts
export interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, any>;
  thoughtSignature?: string;
}
```

字段含义：

- `type: "toolCall"`：说明这是工具调用内容块
- `id`：本次工具调用 ID
- `name`：要调用的工具名
- `arguments`：模型生成的工具参数
- `thoughtSignature`：Google 特定的思考上下文签名，普通场景先不用关心

例子：

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

---

## ToolResultMessage：工具执行结果

工具执行完之后，上层 Agent 会把结果写回上下文，形成 `ToolResultMessage`。

```ts
export interface ToolResultMessage<TDetails = any> {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: TDetails;
  isError: boolean;
  timestamp: number;
}
```

例子：

```ts
{
  role: "toolResult",
  toolCallId: "call_123",
  toolName: "read_file",
  content: [
    { type: "text", text: "# Pi Agent Harness Mono Repo..." }
  ],
  isError: false,
  timestamp: Date.now()
}
```

注意：`toolCallId` 必须对应之前的 `ToolCall.id`。

这样模型才知道：

```text
这是我刚才那个工具调用的结果。
```

---

## 工具参数校验

工具参数校验逻辑在：

- `packages/ai/src/utils/validation.ts`

主要函数：

```ts
validateToolCall(tools, toolCall)
validateToolArguments(tool, toolCall)
```

`validateToolCall` 做两件事：

1. 根据 `toolCall.name` 找到对应工具。
2. 用该工具的 `parameters` schema 校验 `toolCall.arguments`。

简化理解：

```ts
const args = validateToolCall(tools, toolCall);
```

如果校验成功，返回校验后的参数。

如果失败，会抛出错误，例如：

```text
Tool "read_file" not found
```

或者：

```text
Validation failed for tool "read_file":
  - path: Expected string
```

这一步很重要，因为模型生成的 JSON 不一定可靠。

---

## 参数类型转换

`validation.ts` 中还做了一些参数转换。

例如模型可能把数字写成字符串：

```ts
{ maxBytes: "1000" }
```

schema 要求数字：

```ts
maxBytes: Type.Number()
```

校验逻辑会尝试转换成：

```ts
{ maxBytes: 1000 }
```

这种容错可以让工具调用更稳定。

---

## StringEnum：更兼容的枚举参数

辅助函数在：

- `packages/ai/src/utils/typebox-helpers.ts`

导出：

```ts
StringEnum(values, options)
```

例子：

```ts
import { StringEnum, Type } from "@earendil-works/pi-ai";

const weatherTool = {
  name: "get_weather",
  description: "Get current weather for a location",
  parameters: Type.Object({
    location: Type.String(),
    units: StringEnum(["celsius", "fahrenheit"], {
      default: "celsius",
    }),
  }),
};
```

为什么不用 `Type.Enum`？

因为有些 provider，特别是 Google，不支持某些复杂 JSON Schema 形式，例如 `anyOf` / `const` 模式。

`StringEnum` 生成的是更通用的：

```json
{
  "type": "string",
  "enum": ["celsius", "fahrenheit"]
}
```

兼容性更好。

---

## 一次完整工具调用流程

以读取文件为例：

```text
1. 开发者定义 Tool
   read_file(path: string)

2. Tool 放进 Context.tools

3. complete(model, context) 或 stream(model, context)

4. provider 把 Tool 转成厂商 API 格式

5. 模型决定调用工具，返回 ToolCall
   name = read_file
   arguments = { path: "README.md" }

6. 上层 Agent 校验参数
   validateToolCall(tools, toolCall)

7. 上层 Agent 真正执行工具
   读取 README.md

8. 上层 Agent 生成 ToolResultMessage

9. ToolResultMessage 放回 Context.messages

10. 再次调用模型

11. 模型根据工具结果生成最终回答
```

---

## Tool 在各层的分工

```text
packages/ai
  定义 Tool schema
  标准化 ToolCall
  校验工具参数
  转换 provider 工具格式

packages/agent
  维护工具调用循环
  执行工具
  把工具结果写回消息历史

packages/coding-agent
  提供具体代码工具
  read / write / edit / bash / grep / find / ls
  提供 extension 注册自定义工具
```

---

## AI 应用小白视角的类比

可以把工具调用想象成“填表申请服务”。

`Tool` 是表格模板：

```text
服务名：read_file
说明：读取文件
必填字段：path，必须是字符串
```

模型填表：

```text
我要调用 read_file
path = README.md
```

框架检查表格：

```text
path 有没有？
path 是不是字符串？
工具名是否存在？
```

检查通过后，真正干活的人是 Agent，不是模型。

Agent 读取文件后，把结果交回模型。

---

## 最小示例

```ts
import { Type, complete, getModel } from "@earendil-works/pi-ai";

const model = getModel("openai", "gpt-4o-mini");

const tools = [
  {
    name: "get_time",
    description: "Get the current time",
    parameters: Type.Object({}),
  },
];

const context = {
  messages: [
    {
      role: "user",
      content: "现在几点？",
      timestamp: Date.now(),
    },
  ],
  tools,
};

const message = await complete(model, context);
```

注意：这个例子只会让模型知道有 `get_time` 工具，并可能返回一个 `ToolCall`。

如果要真的执行 `get_time`，还需要上层 Agent 或你自己写执行逻辑。

---

## 一句话总结

`Tool` 是给模型看的工具说明书。

它告诉模型：

```text
有什么工具
工具做什么
调用工具时参数应该怎么写
```

但 `Tool` 本身不执行任何动作。

真正执行工具、把结果写回上下文、继续下一轮模型调用，是 `packages/agent` 和 `packages/coding-agent` 的职责。

---

## 返回与延伸

- 返回：[[llm-abstraction-layer|LLM 抽象层学习笔记]]
- 相关：[[context-module|Context 统一上下文模块学习笔记]]
- 下一篇：[[unified-call-entry-module|stream / complete 统一调用入口模块学习笔记]]
- 上一篇：[[model-description-module|Model 描述模块学习笔记]]
- 总览：[[architecture-learning-notes|Pi Mono 框架拆解学习笔记]]
- 首页：[[index|Pi Mono 学习库]]
