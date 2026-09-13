# 桌面端 Provider 适配：Codex SDK 与 Claude Agent SDK

> Provider SDK 属于易变的基础设施细节。UI 和 Runtime 只依赖本地的 Session、Turn、Event、Approval 合约；官方 SDK 只出现在 adapter 包内。

## 1. 先澄清产品与名称

截至 2026-08，容易混淆的四个表面是：

- OpenAI **Codex SDK**：TypeScript 包 `@openai/codex-sdk`，用于在 Node 服务端控制本地 Codex thread；官方公开最小用法是 `new Codex()`、`startThread()`、`resumeThread(id)`、`thread.run()`。官方也提供稳定 Python 包 `openai-codex`，通过本地 app-server JSON-RPC 工作。
- Anthropic **Claude Agent SDK**：原“Claude Code SDK”已更名，TypeScript 包 `@anthropic-ai/claude-agent-sdk`；复用 Claude Code 的 agent loop、工具、上下文、hooks、permissions、sessions、MCP 等。
- `@anthropic-ai/sdk` 是直接调用 Messages API 的 Client SDK，需要自己实现工具循环，不等于 Claude Agent SDK。
- Anthropic Managed Agents 是托管 REST 产品，运行环境与 session 由 Anthropic 托管，也不等于本地 Agent SDK。

第三方桌面产品使用 Claude Agent SDK 时还要遵循 Anthropic 的认证与品牌规则：官方文档明确提示，未经批准不能向第三方用户转售 claude.ai 登录/额度，应使用允许的 API key 认证方式。

## 2. 适配器端口

```ts
export type AgentCapability =
  | "resume" | "stream" | "structured-output" | "tool-approval"
  | "interrupt" | "mcp" | "subagents" | "checkpoint";

export interface AgentProvider {
  readonly id: "codex-local" | "claude-agent-local" | string;
  probe(signal: AbortSignal): Promise<{
    sdkVersion: string; runtimeVersion?: string; capabilities: Set<AgentCapability>;
  }>;
  createSession(input: CreateSession): Promise<ProviderSessionRef>;
  resumeSession(ref: ProviderSessionRef): Promise<void>;
  run(input: RunInput, sink: EventSink, signal: AbortSignal): Promise<RunTerminal>;
  dispose(): Promise<void>;
}
```

`ProviderSessionRef` 是 `{provider, opaqueId, adapterVersion}`，opaque ID 不给 UI 解释。`RunInput` 使用本公司定义的 workspace grant、tool policy、output schema；adapter 负责翻译。Provider 声称支持某能力不够，`probe` 要从实际 SDK/runtime 握手获取并缓存。

```mermaid
flowchart LR
  Runtime[Runtime Core] --> Port[AgentProvider Port]
  Port --> C[CodexAdapter]
  Port --> A[ClaudeAgentAdapter]
  C --> CS[@openai/codex-sdk]
  A --> AS[@anthropic-ai/claude-agent-sdk]
  CS --> CP[Codex local process]
  AS --> AP[Claude agent process]
```

## 3. Codex TypeScript 适配示例

下面只使用官方已公开的稳定最小接口；流式事件的具体 union 随包版本锁定并在 adapter 内穷尽处理。

```ts
import { Codex } from "@openai/codex-sdk";

class CodexAdapter {
  private readonly codex = new Codex();
  private thread?: ReturnType<Codex["startThread"]>;

  constructor(
    private readonly workspace: string,
    private readonly signal: AbortSignal,
  ) {}

  async open(savedId?: string) {
    this.thread = savedId
      ? this.codex.resumeThread(savedId)
      : this.codex.startThread({
          workingDirectory: this.workspace,
          sandboxMode: "workspace-write",
        });
  }

  async run(prompt: string) {
    if (!this.thread) throw new Error("SESSION_NOT_OPEN");
    const result = await this.thread.run(prompt, { signal: this.signal });
    return {
      providerSessionId: this.thread.id,
      text: result.finalResponse,
      usage: result.usage,
    };
  }
}
```

`workingDirectory`、`sandboxMode`、`signal` 见当前 TypeScript 源码，但应以项目 lockfile 对应版本为准。若使用 `runStreamed()`，把 `thread.started`、`item.started/updated/completed`、`turn.completed/failed` 映射成统一事件；用 `assertNever` 迫使 SDK 升级时编译失败。不要把 Codex 的 `ThreadItem` 直接暴露到 UI。

Codex Python SDK 的生命周期更丰富且稳定文档明确说明其 app-server JSON-RPC、pinned runtime、sandbox presets。若企业需要显式 app-server 控制、Python 生态或更细生命周期，可把 Python Runtime 作为 sidecar；TypeScript UI 仍只看统一协议。

## 4. Claude Agent SDK 适配示例

官方 TypeScript 核心入口是 async iterable `query()`。以下字段是截至核对日的常用接口；完整消息 union、permission callback、hook 签名必须从安装版本的 TypeScript reference 生成适配，示例不作为跨版本 ABI。

```ts
import { query, type SDKMessage } from "@anthropic-ai/claude-agent-sdk";

async function* runClaude(prompt: string, cwd: string): AsyncGenerator<SDKMessage> {
  for await (const message of query({
    prompt,
    options: {
      cwd,
      maxTurns: 30,
      permissionMode: "dontAsk",
      allowedTools: ["Read", "Glob", "Grep"],
      settingSources: [],
    },
  })) {
    yield message;
  }
}
```

为什么选择 `dontAsk` 而不是让 SDK 在后台弹 CLI 提示？企业桌面端需要自己的审批 UI 和审计链。做法是默认拒绝未预授权工具，通过安装版本支持的 permission callback/hook 把请求转成 `tool.approval.requested`；用户批准后仅放行绑定参数的一次调用。**callback 与 hook 的精确类型在这里视为“示意接口”**，因为它们是 SDK 演进面，必须由 adapter contract tests 锁定。

Session 恢复、fork、streaming input、checkpoint 等能力也按 capability 暴露。不要用空字符串或静默新会话冒充 resume 成功；原 session 不存在时返回 `SESSION_NOT_FOUND`，由产品决定是否创建分支。

## 5. 统一事件映射

| 统一事件 | Codex 来源 | Claude Agent 来源 |
|---|---|---|
| `session.started` | thread started / thread id | system init / session id |
| `assistant.delta` | item update 或聚合消息 | stream/assistant content（依配置） |
| `tool.requested` | command/tool item | tool use message |
| `tool.completed` | item completed | tool result message |
| `turn.usage` | turn completed usage | result/usage |
| `turn.failed` | turn failed/error | error/result failure/iterator throw |

映射必须保留 `providerPayloadRef` 指向受限诊断存储，而不是把所有原始数据塞入公共事件。统一 `finishReason` 为枚举并保留 `providerReason`，未知值映射 `other`。usage 不存在和数值 0 不同；费用是估算值，需标模型价目版本。

## 6. 在桌面进程中的部署方式

两类 Agent SDK 都适合放 Runtime sidecar/Utility Process，不进 Renderer，也不与窗口生命周期绑定。Supervisor 启动时记录：adapter 包版本、底层 CLI/runtime 版本、二进制哈希、Node 版本、协议版本。限制环境变量白名单、cwd 与可见路径；stdout 只传 framed protocol，stderr 进入脱敏日志。

升级采用兼容矩阵而非“永远 latest”：例如 Desktop 3.4 支持 Runtime 2.8–2.9，Runtime 2.9 固定 Codex SDK X 与 Claude Agent SDK Y。安全修复通过受控 bot PR 更新 lockfile，跑回放和真实 smoke 后逐步发布。

### 两家 Session 语义不能互相冒充

Codex TypeScript thread ID 在 thread 真正启动后才可用，持久化时要等 `thread.started`/首个 run 建立 ID；调用 `run()` 多次是在同一 thread 上继续。Claude Agent SDK 的 TypeScript 核心是一次 `query()` 的消息流；官方当前 session 指南通过 result/init 中的 `session_id` 捕获 ID，并用 `resume`/`continue`/`forkSession` 等 options 工作。不同版本可用面会变化，因此统一层保存 opaque ID 和能力，不假定两者都有一个长驻 client 对象。特别注意：对话 session 保存的是模型/工具历史，不一定保存工作区文件快照；恢复前仍要检查 Git/文件状态，分支对话也不等于分支文件系统。

`settingSources: []` 代表企业 adapter 不自动加载用户或项目里的 Claude 配置，是建立可重复执行环境的常用基线；若产品允许 `.claude/` 配置、skills 或 plugins，必须把来源作为显式 capability，扫描后展示给用户，并受租户策略限制。Codex 的本地配置、AGENTS 指令同理：配置发现不是无害便利，它会改变工具、模型和网络行为。

### 原始消息到公共事件的落地算法

Adapter 为每次 run 建 `MappingContext`：公共 turnId、Provider turn/session ref、当前 message/tool map、last seq、terminal flag。收到文本增量只追加到对应 stream buffer 并发 delta；收到 message complete 时核对聚合文本/hash；工具开始先验证 schema 和 policy，再发布 requested；Provider result 到达后用 compare-and-set 写唯一 terminal。若 iterator 抛错但此前已见 Provider terminal，以 terminal 为准并把迟到异常记诊断；若无 terminal，则发布 `turn.failed(PROVIDER_STREAM_LOST)`。这可避免常见的“双终态”与“空成功”。

映射层还要做资源治理：每条消息、工具输出、累计上下文设上限；超限内容先写受控 blob，再发送引用；reasoning/内部思考类内容只按供应商政策和企业合规处理，不能默认展示或持久化。Usage 与费用直到终态才可能完整，UI 的流中估算必须标 `estimated`，最终值不可用时保持 unknown。

桌面 Supervisor 对每个 adapter 维护独立并发池与 circuit breaker。Provider SDK 内部也可能自动重试，统一层必须知道或关闭其中一层，避免“SDK 重试 × Runtime 重试”放大请求。健康探测不可用真实用户 prompt；使用低权限、自带预算的握手/最小任务，失败分类要区分认证、配置、服务过载和本地进程损坏。

Adapter 的临时目录、配置目录和 session transcript 目录应按租户/工作区分开，权限只给当前用户，并纳入配额与清理。Claude/Codex 底层进程可能从 cwd 向上发现配置，启动前要解析真实路径并明确配置根；不要让用户选中的任意子目录意外继承另一个项目或 home 级的高权限指令。登出、租户切换和卸载按数据政策清理 transcript，不能只删统一层数据库。

## 7. 生产失败模式

- **把两家“文本输出”当相同语义**：工具、usage、终止原因丢失。统一生命周期事件，不只统一字符串。
- **未知 SDK 事件默认当成功**：可能吞掉 fatal error。穷尽 switch，未知终态进入 adapter protocol error。
- **Provider 卡住不产事件**：无限占用进程。首事件、空闲、总时长三种 timeout，并区别处理中工具长任务。
- **恢复 ID 跨租户复用**：读到其他组织会话。session ref 绑定 tenant/user/provider 并加密存储。
- **并发运行共享 cwd/config**：一个 session 改变另一个权限。每 run 独立 execution context，限制全局配置读取。
- **升级 SDK 同时升级协议**：问题无法定位。adapter 内部更新与公共合约版本分离。

## 8. 练习与验收

关联：[Provider 抽象](../04-sdk/provider-abstraction.md)、[合约与事件](../04-sdk/contracts-events.md)、[兼容与测试](../04-sdk/compatibility-testing.md)。

练习：实现两个 fake provider 和一个真实 Provider smoke adapter。验收：同一 prompt 的事件都通过公共 schema；取消能到达底层；无 resume 能力时 UI 根据 capability 隐藏入口；未知事件使测试失败；日志无 prompt/key；锁定 SDK 升级后 golden trace 差异必须人工确认。

## 官方资料（核对时间：2026-08）

- [OpenAI Codex SDK](https://developers.openai.com/codex/sdk)
- [OpenAI Codex TypeScript SDK source and samples](https://github.com/openai/codex/tree/main/sdk/typescript)
- [Claude Agent SDK Overview](https://code.claude.com/docs/en/agent-sdk/overview)
- [Claude Agent SDK TypeScript Reference](https://code.claude.com/docs/en/agent-sdk/typescript)
- [Claude Agent SDK TypeScript repository](https://github.com/anthropics/claude-agent-sdk-typescript)
