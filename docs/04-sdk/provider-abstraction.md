# Provider 抽象：端口适配器、能力协商与韧性

> 抽象的目标不是隐藏差异，而是把差异放在一个可测试的位置。公共 Runtime 定义稳定语义；Adapter 把每家 SDK 的会话、流、工具、错误、取消和 usage 翻译进来。

## 1. 反腐层的边界

```mermaid
flowchart LR
  Core[Runtime Core] --> P[AgentProvider Port]
  P --> R[Provider Registry/Router]
  R --> CA[Codex Adapter]
  R --> AA[Claude Agent Adapter]
  R --> FA[Fake Adapter]
  CA --> OSDK[@openai/codex-sdk]
  AA --> CSDK[@anthropic-ai/claude-agent-sdk]
```

只有 adapter 包能 import 官方 SDK。上游只认识公共 `ProviderSession`、`ProviderEvent`、`ProviderError`。这条依赖规则要由 ESLint/依赖图测试执行，不依赖口头约定。

```ts
export interface ProviderPort {
  describe(ctx: TenantContext, signal: AbortSignal): Promise<ProviderDescriptor>;
  open(input: ProviderOpenInput, signal: AbortSignal): Promise<ProviderSession>;
  run(
    session: ProviderSession,
    input: ProviderRunInput,
    sink: (event: ProviderEvent) => Promise<void>,
    signal: AbortSignal,
  ): Promise<ProviderTerminal>;
  cancel?(session: ProviderSession, turnRef: string): Promise<"ack" | "unsupported">;
  close(session: ProviderSession): Promise<void>;
}
```

Sink 返回 Promise 很关键：adapter 必须等待下游接收，才能形成背压。若供应商流不可暂停，adapter 使用有界缓冲，超限时主动取消而不是耗尽内存。

## 2. 规范化输入，不做字符串透传

`ProviderRunInput` 应包括：有来源标记的 content、system/developer policy、workspace grant、允许工具清单、审批策略、模型选择意图、推理/成本预算、结构化输出 schema、deadline 与幂等键。每个字段有三种翻译结果：支持、可降级、拒绝。

例如公共意图 `execution: {filesystem: "workspace-write", network: "deny"}`：Codex adapter 可翻译到其 sandbox/网络设置；Claude Agent adapter 翻译到 permission mode、allowed tools 与本地沙箱部署策略。若某 adapter 无法保证 network deny，必须返回 `CAPABILITY_CONSTRAINT_UNSATISFIED`，不能只在 prompt 写“请勿联网”。

所有自由扩展都经过 registry：

```ts
type ExtensionValue = { schemaId: string; value: unknown };

function translateExtensions(input: Record<string, ExtensionValue>) {
  for (const [key, ext] of Object.entries(input)) {
    const schema = extensionRegistry.get(key, ext.schemaId);
    if (!schema) throw new ProviderInputError("UNKNOWN_EXTENSION");
    schema.parse(ext.value);
  }
}
```

## 3. 能力描述与探测

```ts
type ProviderDescriptor = {
  id: string;
  adapterVersion: string;
  runtimeVersion?: string;
  models: Array<{ id: string; contextWindow?: number; traits: string[] }>;
  capabilities: Array<{
    id: string;
    version: string;
    constraints?: Record<string, unknown>;
  }>;
  observedAt: string;
  ttlMs: number;
};
```

不要把模型列表和上下文长度硬编码进前端。启动时从企业控制面 + Provider/runtime probe 得到描述符，签名或经受信通道发送，按 TTL 缓存；真正 run 时重新校验关键策略。`describe()` 失败不应清空上一份缓存，而是标 stale 并禁止依赖新能力的操作。

能力是交集：Provider 实现能力、当前模型、SDK/runtime 版本、部署形态、租户策略与用户 grant。比如 SDK 支持 shell，但 Web Runtime 未配置安全执行环境，则 `tool.shell` 是 unavailable；策略禁止时是 policy-denied，UI 要能解释二者差异。

## 4. 两个 Adapter 的最小骨架

OpenAI 官方截至核对日给出的 TypeScript Codex 稳定入口是 `startThread/resumeThread/run`：

```ts
import { Codex } from "@openai/codex-sdk";

async function openCodex(saved?: string) {
  const sdk = new Codex();
  const thread = saved ? sdk.resumeThread(saved) : sdk.startThread();
  return { sdk, thread };
}

async function runCodex(s: Awaited<ReturnType<typeof openCodex>>, prompt: string) {
  const out = await s.thread.run(prompt);
  return { ref: s.thread.id, text: out.finalResponse, usage: out.usage };
}
```

在真实 adapter 中使用安装版本支持的 streamed API 映射 item/turn 事件；公共文档不假设尚未进入官方稳定页面的控制方法。无法 cooperative cancel 时用 capability 明示并由 sidecar supervisor 执行 forced cancel。

Claude Agent SDK 的稳定核心形式是 `query()` async iterable：

```ts
import { query } from "@anthropic-ai/claude-agent-sdk";

async function runClaude(prompt: string, cwd: string, emit: (x: unknown) => Promise<void>) {
  for await (const message of query({
    prompt,
    options: {
      cwd,
      maxTurns: 20,
      allowedTools: ["Read", "Glob", "Grep"],
      permissionMode: "dontAsk",
      settingSources: [],
    },
  })) await emit(message);
}
```

`dontAsk` 用于避免后台 CLI 交互；需要写操作时，应按锁定版本的 permission callback/hook API 转给统一审批服务。Callback 签名属于版本适配细节，adapter 应直接使用锁定依赖中的官方类型。对 `assistant/system/user/result` 等消息 union 做穷尽映射。

## 5. Session 与身份映射

公共 session 记录 `{tenantId, providerId, opaqueProviderSessionId, adapterMajor, createdBy}`。Provider session ID 加密存储且从不由前端任意提供；resume 前查询归属。Adapter 升 major 后若旧 session 无法读取，提供显式 migration/fork 结果，不能悄悄创建空 session。

同 session 的 turn 默认串行，因为多数 Agent 上下文有顺序语义。若产品允许分支，创建新公共 session 并记录 `forkedFrom(sessionId, seq)`。并发读请求也要确认 Provider 是否能安全复用同一底层进程。

认证实现独立于 session：企业服务端优先 workload/service identity；桌面 BYOK 使用 OS vault 引用。Adapter 接收 credential lease，而不是长期 key；lease 限 provider/tenant/audience/TTL，调用结束释放。认证失败不自动尝试其他租户凭据。

## 6. 错误翻译与重试

Adapter 建立表驱动映射：认证、权限、参数、限流、过载、网络、timeout、内容策略、SDK protocol、runtime crash。保留安全的 `providerCode` 供诊断，但路由器只按公共 code 决策。

自动重试必须同时满足：错误明确 transient；操作可幂等或能按 provider turn ID 查询；仍在 deadline/budget；未被取消。退避使用 full jitter 并尊重 `retry-after`。流已经产生工具副作用后，不允许从头重试整个 turn。

Circuit breaker 以 `provider + region + model + errorClass` 分桶，避免一个模型过载拖垮所有调用。半开探测使用低成本请求；breaker 状态不应成为全局永久单例，需有 TTL 与多实例协调。

## 7. 路由、回退与预算

路由输入包括：required capabilities、数据地域/合规、模型允许列表、租户额度、延迟目标、成本上限与健康度。先硬约束过滤，再按策略排序。Provider fallback 仅适用于业务允许的场景：

- 新的无副作用 turn 可回退；已有会话通常不能无损迁移上下文。
- 工具 schema、系统策略、结构化输出必须在目标 Provider 可表达。
- 用户和审计记录要能看见实际 Provider；不能暗中把代码发往另一地域。
- 已产生 token/工具事件后切换 Provider 需创建新 branch，不拼接成同一个 turn。

```ts
function choose(candidates: ProviderDescriptor[], req: RouteRequirements) {
  return candidates
    .filter(p => satisfiesHardConstraints(p, req))
    .sort((a, b) => score(b, req) - score(a, req))[0]
    ?? (() => { throw new NoProviderError(explainMissing(req, candidates)); })();
}
```

### Context、Tool 与结构化输出的差异处理

Provider 对 system/developer/user 角色、缓存、图片/文件、tool result、并行工具和结构化输出的支持并不一致。先把公共 Context 表示成有来源和策略的 block，再由 adapter 编译；adapter 返回 `TranslationReport`，列出精确映射、被截断内容、使用的降级和拒绝原因。安全相关内容（权限、数据区域、网络禁用）不允许降级为自然语言提示；展示格式类意图可以在明确标记后降级。

上下文预算器先保留系统策略、待决 tool call/result 配对与最近用户输入，再按来源优先级选择历史；任何 compaction 都产生可审计事件和摘要版本。Provider 自带 context management 仍不能替代企业侧预算，因为企业要在数据发送前执行 DLP、地域与附件大小策略。缓存 key 包含 tenant、provider、model、policy hash 和内容 hash，禁止跨租户复用潜在敏感前缀。

Tool 名称、schema dialect、并行调用能力通过 adapter 编译。若 Provider 只接受有限名称字符，建立可逆映射表并放在 run context，不能简单截断造成两个工具同名。Tool result 超过限制则变 blob/摘要，并保留完整结果在授权存储；摘要是非权威数据，后续写操作需要重新读取权威来源。

结构化输出优先使用 Provider 原生 schema 能力，但终态仍由 Runtime 用自己的 schema validator 校验。Provider 不支持时可选择拒绝或有限“文本 JSON + 修复”降级，结果 descriptor 标 `native:false`；高风险自动化通常应要求原生/确定性验证能力。枚举、数值精度、nullable 和 additionalProperties 是跨 SDK 常见差异，必须有专门 fixture。

### 成本、限额与公平性

预算不是 prompt 建议。Runtime 在 run 前检查租户并发、日/月额度、最大 turn/工具数；流中按 reported/estimated usage 更新，达到硬上限请求取消并记录 `budget_exceeded`。Provider SDK 的 max-turns 与 Runtime 外层预算同时存在时取更严者。队列按租户做加权公平，防止一个大型索引任务占满所有 adapter worker。

模型价格和 rate limit 会变化，由控制面版本化下发并记录 pricing version。路由器不能为了省钱自动选择不符合数据政策的 Provider；成本只在硬约束通过后参与评分。对未报告 usage 的 Provider 使用保守估算并标 estimated，不把缺失当零。

## 8. 生产失败模式

- **Adapter 返回 Provider 原始错误对象**：序列化泄密且上游耦合。转换成公共错误，原始内容进受限诊断。
- **只看包版本，不看底层 CLI/runtime**：同 SDK 在不同机器行为不一。握手并记录双版本和 hash。
- **fallback 绕过地域策略**：可用性恢复但造成合规事件。硬约束永远先于健康评分。
- **全局复用一个 abort/controller**：取消 A turn 连带 B。每 turn 独立 execution context。
- **映射未知事件为 text**：隐藏协议变化。unknown 诊断 + 安全失败 + adapter 升级门禁。
- **重试 tool use**：重复提交或重复写文件。tool-call 幂等键、结果账本、明确人工恢复。

## 9. 关联知识、练习与验收

关联：[桌面 Provider 适配](../02-desktop/provider-adapters.md)、[合约事件](contracts-events.md)、[兼容测试](compatibility-testing.md)。

练习：实现 `FakeProviderPort`、Codex/Claude 的最小 smoke adapter 和基于能力的 router。验收：Provider 包之外无法 import 官方 SDK；相同 fixture 产出相同公共事件不变量；缺关键 sandbox 能力时拒绝而非 prompt 降级；认证失败不 fallback；限流按 deadline 退避；升级 SDK 后 unknown union 分支令 CI 失败。

## 官方延伸阅读（核对时间：2026-08）

- [OpenAI Codex SDK](https://developers.openai.com/codex/sdk)
- [OpenAI Codex TypeScript source](https://github.com/openai/codex/tree/main/sdk/typescript)
- [Claude Agent SDK Overview](https://code.claude.com/docs/en/agent-sdk/overview)
- [Claude Agent SDK TypeScript Reference](https://code.claude.com/docs/en/agent-sdk/typescript)
- [Claude Agent SDK Sessions](https://code.claude.com/docs/en/agent-sdk/sessions)
