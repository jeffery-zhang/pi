---
tags:
  - pi-mono
  - packages-ai
  - model
---

# Model 描述模块学习笔记

> 上级：[[index|Pi Mono 学习库]]
> 所属：[[llm-abstraction-layer|LLM 抽象层学习笔记]]
> 相关源码：`packages/ai/src/types.ts`、`packages/ai/src/models.ts`


## Model 是什么

在 `packages/ai` 里，`Model` 可以理解成一张“模型身份证”或“模型配置卡”。

AI 应用不能只知道“我要用 GPT-4”或“我要用 Claude”。程序还必须知道：

- 这个模型属于哪家公司？
- 应该用哪种 API 协议调用？
- 请求应该发到哪个地址？
- 它能不能看图片？
- 它支不支持 reasoning / thinking？
- 它支持多长上下文？
- 它一次最多能输出多少 token？
- 它的输入和输出价格是多少？

这些信息统一由 `Model` 对象描述。

核心类型位置：

- `packages/ai/src/types.ts`

简化结构：

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

---

## 一个 Model 示例

一个模型可以粗略描述成这样：

```ts
{
  id: "gpt-4o-mini",
  name: "GPT-4o Mini",
  provider: "openai",
  api: "openai-responses",
  baseUrl: "https://api.openai.com/v1",
  reasoning: false,
  input: ["text", "image"],
  contextWindow: 128000,
  maxTokens: 16384,
  cost: {
    input: 0.15,
    output: 0.6,
    cacheRead: 0,
    cacheWrite: 0
  }
}
```

这表示：

这是 OpenAI 的 `gpt-4o-mini`，需要通过 OpenAI Responses API 调用，支持文本和图片输入，最大上下文约 128k token，最大输出约 16k token，并带有对应的价格信息。

---

## 字段拆解

### `id`：模型真实 ID

```ts
id: string;
```

`id` 是传给模型服务商的模型 ID。

例子：

```ts
"gpt-4o-mini"
"claude-sonnet-4-20250514"
"gemini-2.5-flash"
```

它主要给机器用。实际请求 provider API 时，通常会用到这个字段。

---

### `name`：给人看的名字

```ts
name: string;
```

`name` 是展示用名称，常用于 UI、模型选择器、日志等地方。

区别：

```ts
id: "gpt-4o-mini"      // 给程序/API 用
name: "GPT-4o Mini"    // 给人看
```

---

### `provider`：模型属于谁

```ts
provider: Provider;
```

`provider` 表示模型供应商。

常见 provider：

```ts
"openai"
"anthropic"
"google"
"google-vertex"
"openrouter"
"groq"
"mistral"
"amazon-bedrock"
"github-copilot"
```

可以简单理解为：这个模型是谁提供或售卖的。

---

### `api`：用哪种协议调用

```ts
api: TApi;
```

这是 `Model` 中最重要、也最容易混淆的字段。

`provider` 表示“模型来自哪里”。

`api` 表示“应该怎么调用它”。

例子：

```ts
provider: "openai"
api: "openai-responses"
```

表示：这个模型来自 OpenAI，并且使用 OpenAI Responses API 调用。

再比如：

```ts
provider: "openrouter"
api: "openai-completions"
```

表示：这个模型来自 OpenRouter，但它的调用协议兼容 OpenAI Chat Completions。

所以：

- `provider` 解决“模型来源”
- `api` 解决“调用方式”

这是 pi 的 LLM 抽象层里非常重要的设计。

---

## 为什么要把 `provider` 和 `api` 分开

现实中很多平台都兼容 OpenAI API，例如：

- OpenAI
- Groq
- OpenRouter
- DeepSeek
- xAI
- Ollama
- LM Studio
- vLLM

它们可能来自不同 provider，但请求/响应协议长得很像。

如果不拆分 `provider` 和 `api`，代码里会出现大量重复 provider 实现。

拆开之后，只要某个 provider 兼容 OpenAI completions 协议，就可以复用同一个 API 实现：

```ts
api: "openai-completions"
```

这就是 LLM 抽象层的核心价值之一：用统一协议复用 provider 调用逻辑。

---

### `baseUrl`：请求发到哪里

```ts
baseUrl: string;
```

`baseUrl` 是 API 地址。

例子：

```ts
"https://api.openai.com/v1"
"https://api.anthropic.com"
"https://openrouter.ai/api/v1"
```

程序调用模型时，会把请求发到这个地址。

---

### `reasoning`：是否支持思考能力

```ts
reasoning: boolean;
```

有些模型支持 reasoning / thinking，例如：

- Claude thinking
- OpenAI reasoning models
- Gemini thinking

如果：

```ts
reasoning: true
```

表示这个模型支持思考级别设置。

上层可以进一步设置类似：

```ts
thinkingLevel: "low" | "medium" | "high"
```

如果：

```ts
reasoning: false
```

表示它是普通回答模型，不支持显式 thinking 配置。

---

### `input`：支持什么输入

```ts
input: ("text" | "image")[];
```

这个字段表示模型能接收什么类型的输入。

纯文本模型：

```ts
input: ["text"]
```

视觉模型：

```ts
input: ["text", "image"]
```

上层 UI 或 Agent 可以据此判断：

- 能不能上传图片？
- 能不能把截图传给模型？
- 能不能把工具生成的图片发回模型？

---

### `contextWindow`：上下文窗口

```ts
contextWindow: number;
```

`contextWindow` 表示模型一次请求最多能容纳多少 token。

例子：

```ts
contextWindow: 128000
```

表示这个模型一次请求最多能处理约 128k token。

这些内容都会占用上下文：

- system prompt
- 历史对话
- 工具结果
- 文件内容
- 图片相关内容
- 当前用户问题

如果超过 `contextWindow`，模型通常会报错。

因此上层的 compaction、历史裁剪、摘要记忆等能力都依赖这个字段。

可以简单理解为：

```text
contextWindow = 模型一次能看到多少内容
```

---

### `maxTokens`：最大输出长度

```ts
maxTokens: number;
```

`maxTokens` 表示模型一次最多能输出多少 token。

例子：

```ts
maxTokens: 8192
```

表示模型一次最多生成约 8192 token。

区别：

```text
contextWindow = 模型脑容量，输入和输出整体受它约束
maxTokens = 模型一次最多能说多少
```

---

### `cost`：价格信息

```ts
cost: {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
}
```

价格通常按每百万 token 计算。

例子：

```ts
cost: {
  input: 3,
  output: 15,
  cacheRead: 0.3,
  cacheWrite: 3.75
}
```

表示：

- 输入 100 万 token：3 美元
- 输出 100 万 token：15 美元
- 读缓存 100 万 token：0.3 美元
- 写缓存 100 万 token：3.75 美元

相关成本计算逻辑在：

- `packages/ai/src/models.ts`

对应函数：

```ts
calculateCost(model, usage)
```

---

## 模型数据从哪里来

模型数据主要来自：

- `packages/ai/src/models.generated.ts`

然后在这里加载：

- `packages/ai/src/models.ts`

简化逻辑：

```ts
import { MODELS } from "./models.generated.js";

const modelRegistry = new Map();

for (const [provider, models] of Object.entries(MODELS)) {
  modelRegistry.set(provider, models);
}
```

也就是说：所有内置模型先放在 `MODELS` 中，启动时再注册到 `modelRegistry`。

注意：项目规则明确说明不要直接修改 `packages/ai/src/models.generated.ts`。如果要更新模型列表，应修改生成脚本：

- `packages/ai/scripts/generate-models.ts`

---

## 如何获取模型

常用函数在：

- `packages/ai/src/models.ts`

### `getModel(provider, modelId)`

根据 provider 和 model ID 获取一个完整 `Model` 对象：

```ts
import { getModel } from "@earendil-works/pi-ai";

const model = getModel("openai", "gpt-4o-mini");
```

### `getProviders()`

列出已注册 provider：

```ts
const providers = getProviders();
```

### `getModels(provider)`

列出某个 provider 下的全部模型：

```ts
const models = getModels("openai");
```

---

## Model 在调用链里的位置

调用模型前，第一步通常是拿到 `Model`：

```ts
const model = getModel("openai", "gpt-4o-mini");
```

然后传给：

```ts
complete(model, context)
```

调用链：

```text
getModel()
   ↓
得到 Model
   ↓
complete(model, context)
   ↓
stream(model, context)
   ↓
根据 model.api 找到对应 provider 实现
   ↓
调用真实厂商 API
```

因此 `Model` 决定了：

- 用哪个模型
- 找哪个 API 实现
- 请求发到哪里
- 是否允许图片
- 是否允许 reasoning
- 如何估算 token 成本
- 上下文和输出限制是多少

---

## 一句话总结

`Model` 不是模型本身，而是模型的“说明书”。

它告诉框架：

```text
这个模型叫什么
属于哪个 provider
用哪种 API 调用
请求发到哪里
支持什么输入
上下文多大
最多输出多少
是否支持 reasoning
价格是多少
```

上层 Agent 不需要关心具体厂商细节，只需要拿到 `Model` 对象，然后调用：

```ts
complete(model, context)
```

或：

```ts
stream(model, context)
```


---

## 返回与延伸

- 返回：[[llm-abstraction-layer|LLM 抽象层学习笔记]]
- 总览：[[architecture-learning-notes|Pi Mono 框架拆解学习笔记]]
- 首页：[[index|Pi Mono 学习库]]
