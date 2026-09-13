# LLM 系统原理：Agent Runtime 的模型基础

## 1. 先理解推理约束

设计 Agent Runtime 不需要重新实现 Transformer，但要知道推理方式如何影响成本、延迟、状态和失败边界。模型在有限上下文中按 token 自回归生成结果，不会自动记住上一次调用，也不是确定性函数。输入相同，模型版本、采样配置或服务端实现不同，输出仍可能不同。

把一次调用抽象为：

```text
output ~ Model(modelVersion, normalizedContext, tools, sampling, providerState)
```

这里的 `~` 表示从概率分布采样，而不是严格等号。Runtime 想获得可恢复性，必须把**输入、模型版本、工具定义、采样配置、外部状态和产生的事件**记录下来；不能假设“重放一次模型调用就会得到相同结果”。

## 2. Token 是预算、延迟和缓存的共同单位

Token 不是字符。不同 tokenizer 对中文、代码、JSON、空格的切分不同；同一段上下文在不同模型上的 token 数可能不同。架构影响包括：

- `context window` 一般包含输入、工具 schema、部分推理/输出预算，不能把标称窗口全部用于历史消息。
- Tool schema 很大时，每个 Step 都可能重复支付输入成本。
- 工具返回 2 MB 日志并不意味着模型能有效使用；需要裁剪、结构化或保存为 Artifact 后只传摘要和引用。
- Token 估算器只能用于预规划，Provider 返回的 usage 才是计费与归因事实。

建立显式预算，而不是调用前才发现超限：

```ts
type ContextPart = {
  id: string;
  kind: 'policy' | 'task' | 'history' | 'retrieval' | 'tool-result';
  priority: number;
  estimatedTokens: number;
  required: boolean;
  content: string;
};

function fitContext(parts: readonly ContextPart[], budget: number): ContextPart[] {
  const required = parts.filter((p) => p.required);
  const requiredTokens = required.reduce((n, p) => n + p.estimatedTokens, 0);
  if (requiredTokens > budget) {
    throw new Error(`required context exceeds budget: ${requiredTokens}/${budget}`);
  }

  let remaining = budget - requiredTokens;
  const optional = parts
    .filter((p) => !p.required)
    .sort((a, b) => b.priority - a.priority)
    .filter((part) => {
      if (part.estimatedTokens > remaining) return false;
      remaining -= part.estimatedTokens;
      return true;
    });
  return [...required, ...optional];
}
```

生产版本还要处理“部分压缩”：一个检索块可缩短后纳入，而不是只能全选或全丢；并记录每个 part 的 provenance 和淘汰原因，便于 Eval 分析。

## 3. Prefill 与 Decode 决定延迟体验

推理通常可理解为两个阶段：

1. **Prefill**：处理全部输入，建立注意力/KV 状态。上下文越长，首 token 延迟通常越高。
2. **Decode**：逐 token 生成输出。生成越长，完成延迟和成本越高。

因此要分别观测：

```text
queue_latency   = provider_accepted_at - request_created_at
time_to_first_token = first_delta_at - provider_accepted_at
decode_duration = last_delta_at - first_delta_at
end_to_end      = run_completed_at - run_created_at
```

只看端到端平均值会掩盖问题：可能是客户端队列、网络、过长 Context、模型 decode、Tool 等待或审批等待。Trace 中应把 `model.call`、`tool.execute`、`approval.wait` 分成不同 Span。

### Prompt Cache 的架构含义

许多 Provider 支持某种形式的 prompt/context caching，但命中规则、价格和生命周期不同。稳定做法是：

- 把长期稳定的 system policy、tool definition 放在前缀；动态用户内容放后面。
- 对序列化进行规范化，避免无意义字段顺序或时间戳破坏前缀一致性。
- 在 Adapter 内读写 Provider-specific cache hint，核心 SDK 只暴露通用的 `cachePolicy` 和 usage 事件。
- 观测 cache read/write tokens 和命中率，不把缓存当作正确性依赖。

## 4. 采样参数不是“创造力旋钮”那么简单

`temperature`、`top_p` 等控制候选 token 分布，但不同 Provider/模型的支持和语义不完全一致。对 Runtime 来说：

- 结构化提取、路由、权限判定应尽量使用低方差配置和 schema 校验。
- 创意生成可允许更高方差，但仍需要安全和内容约束。
- 不要同时任意调整多个采样参数，导致实验无法归因。
- “temperature = 0”也不保证跨时间、跨硬件和跨版本 bit-for-bit 一致。

模型配置应被版本化：

```ts
interface ModelProfile {
  id: string;                 // e.g. coding-high-quality-v3
  provider: string;
  model: string;              // pinned model id when available
  reasoning?: 'low' | 'medium' | 'high';
  sampling?: { temperature?: number; topP?: number };
  maxOutputTokens: number;
  policyVersion: string;
  rollout: { percentage: number; tenantAllowlist?: string[] };
}
```

业务只引用 `profile.id`，控制面把它解析成 Provider 配置。这样可以 canary、回滚、比较 Eval，而不让模型名散落在业务代码。

## 5. Context Window、Memory 与 Provider Session

三个概念常被混淆：

- **Context Window**：一次推理实际输入给模型的 token 序列。
- **Runtime Memory**：Event Store、KV、向量库、Artifact 等可跨 Step/Run 读取的数据。
- **Provider Session/Thread**：Provider 管理的会话句柄，可能保存消息或服务端状态，生命周期和可导出性由 Provider 决定。

企业平台不能只保存 Provider Session ID。否则审计、迁移、离线 Eval、数据删除和 Provider 退出都会受制于外部黑盒。推荐保留 canonical event/message state，并把 Provider ID 当作可选映射。

## 6. Embedding 与生成模型承担不同任务

Embedding 把内容映射为向量，适合语义召回、聚类、去重和近邻检索；生成模型用于根据上下文预测输出。向量相似并不等于“事实相关”或“权限允许”。RAG 仍需：

- metadata filter：tenant、ACL、时间、文档类型。
- lexical + vector hybrid retrieval：代码符号、错误码、专有名词常需关键词召回。
- reranking：对初步候选重新排序。
- provenance：模型回答能追溯到具体文档版本和片段。
- injection defense：检索内容是低信任数据，不是 system instruction。

## 7. 能力比模型名更稳定

统一 SDK 不应写满 `if (model === '...')`。维护可验证的 Capability：

```ts
type Capability =
  | { name: 'input.text' }
  | { name: 'input.image'; maxImages: number }
  | { name: 'tool.call'; parallel: boolean }
  | { name: 'output.schema'; dialect: 'json-schema-2020-12' | 'subset' }
  | { name: 'stream.resume'; mode: 'cursor' | 'session' }
  | { name: 'context.window'; tokens: number };

function requireCapability(caps: readonly Capability[], name: Capability['name']) {
  const cap = caps.find((item) => item.name === name);
  if (!cap) throw new Error(`capability not available: ${name}`);
  return cap;
}
```

Capability 来自 Adapter 的静态声明、运行时探测和控制面覆盖。一次 Run 开始后保存 snapshot，避免执行中配置改变造成语义漂移。

## 8. 常见架构错误

- 把超长上下文当成检索质量的替代品，成本和噪声同时上升。
- 认为模型“知道”系统当前文件或数据库状态，却没有把事实放入 Context 或 Tool。
- 将模型输出直接当成授权判断；模型可辅助分类，最终授权必须由确定性 Policy Engine 执行。
- 认为重试同一 prompt 可重放一次 Run；它只能产生另一次概率性决策。
- 用总 token 数做唯一成本指标，忽略 cache、reasoning、tool、网络和人工审批成本。
- 把 Provider Session 当永久记忆和审计 source of truth。

## 9. 关联知识

- **编译器**：Context Assembler 类似编译管线，把多种来源规范化、检查、优化并生成模型输入。
- **操作系统调度**：token、时间、tool 次数、并发是 Run 的资源配额，需要 budget 和 preemption。
- **数据库**：Provider Session 不是系统记录；canonical events 才能支持审计、迁移和回放。
- **控制理论**：Agent Loop 是带噪声的反馈系统，需要终止条件、约束和观测，不能无限开放循环。

## 10. 练习与验收

1. 对一条真实 Agent 请求分解 system、tools、history、retrieval、output 五类 token 预算。
2. 写一个 Model Registry，禁止业务直接引用 provider/model 字符串。
3. 模拟 Context 从 8k 增到 64k，分别记录 TTFT、输出质量和成本；解释结果而非只报数。
4. 设计实验验证 prompt cache 命中，确保未命中时正确性不受影响。

验收：能在不谈营销型号的情况下，解释某个模型调用为何慢、贵、不稳定，以及 Runtime 能控制哪些部分、不能控制哪些部分。

## 延伸阅读

- [Attention Is All You Need（原始论文）](https://arxiv.org/abs/1706.03762)
- [OpenAI API：向后兼容与固定模型版本](https://developers.openai.com/api/reference/overview#backward-compatibility)
- [Anthropic 文档：Context windows](https://docs.anthropic.com/en/docs/build-with-claude/context-windows)
- [Anthropic 文档：Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)

下一章：[模型接口、流式与结构化输出 →](../../docs/01-foundations/model-io.md)
