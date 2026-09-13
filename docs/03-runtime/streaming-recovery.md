# Streaming、取消、背压与断点恢复

事件流必须在慢 UI、断网和进程崩溃时保留执行语义。下面区分连接断开与任务取消，并实现可续传序号、有限缓冲和恢复检查点。

## 1. Streaming 不只是逐字显示

Agent Runtime 同时产生多类流：模型文字增量、推理/进度提示、工具调用参数、工具日志、审批请求、usage 与最终状态。它们的可靠性要求不同：

| 类别 | 示例 | 是否必须持久化 | 能否丢弃/合并 |
| --- | --- | --- | --- |
| 控制事件 | approval_requested、run_failed | 必须 | 不可 |
| 业务结果 | tool_succeeded、final_committed | 必须 | 不可 |
| 内容增量 | assistant_text_appended | 视恢复承诺 | 可按顺序合并，不可乱序 |
| 临时进度 | “正在分析…”、百分比 | 通常不必 | 可保留最新值 |
| 原始供应商 delta | tool args token、内部协议帧 | 调试采样 | UI 前先规范化 |

先定义产品承诺：重连后只保证拿到最终结果，还是保证恢复到最后一个已显示字符？若承诺后者，就要把文字增量按块持久化，不能仅转发 provider socket。

## 2. 统一事件信封与序列

```ts
type RunEventType =
  | "run_started" | "model_started" | "assistant_text_appended"
  | "tool_started" | "tool_progress" | "tool_finished"
  | "approval_requested" | "checkpoint_saved"
  | "run_completed" | "run_failed" | "run_cancelled";

interface RunEvent<T = unknown> {
  eventId: string;
  runId: string;
  seq: number;                 // 单 Run 严格递增，由持久层分配
  type: RunEventType;
  timestamp: string;
  correlationId?: string;      // 一次 model/tool operation
  durable: boolean;
  payload: T;
  schemaVersion: 1;
}
```

`seq` 是恢复游标，不用 wall clock 排序。`eventId` 用于消费方去重。一次数据库事务同时追加业务事件并推进 Run headSeq，之后才发布到 broker。即使 broker 重复投递，客户端也能以 `(runId, seq)` 去重。

常见 API：`GET /runs/:id/events?after=42` 或 SSE `Last-Event-ID: 42`。服务端先从 Event Store 补历史，再切到实时订阅；切换窗口要防事件漏失，通常先记录订阅水位、读到该水位，再消费其后的实时事件。

## 3. Delta 聚合与提交边界

供应商可能每几个字符发一个 delta。每个 token 写数据库会造成 IOPS 风暴；完全不写又无法恢复。折中方案是按 50–200 ms、1–4 KB 或语义边界合并为 `assistant_text_appended`，每块带 `messageId/chunkIndex/text`。

工具参数 delta 只能在内存拼接；收到完整 tool call 并通过 JSON/schema 校验后，才产生持久 `tool_call_proposed`。半截 JSON 不可进入执行器。

`run_completed` 必须在所有文字块、usage 和最终消息索引提交之后写入。UI 看到 completed 后，应能仅从持久层重建完整结果。若用户在中途取消，可将已提交文字标记 `partial=true`；不要把 partial 冒充完整答案。

## 4. Backpressure：慢消费者不能拖垮 Runtime

Electron renderer 被挂起、远程浏览器网络变慢或 DevTools 卡顿时，生产者仍可能持续吐 token。无限数组会耗尽内存；无脑丢消息会破坏文本。

策略按事件分类：

- terminal/control/result：lossless，队列满时让生产者等待或落盘后让客户端重放；
- text delta：在相同 messageId 上合并连续块；
- progress：只保留最新；
- debug log：可采样/截断，并生成 `logs_truncated` 标记。

```ts
type StreamItem =
  | { kind: "text"; messageId: string; text: string }
  | { kind: "progress"; operationId: string; value: number }
  | { kind: "control"; event: RunEvent };

export class BoundedStream implements AsyncIterable<StreamItem> {
  #items: StreamItem[] = [];
  #bytes = 0;
  #waitReader?: () => void;
  #waitWriters: Array<() => void> = [];
  #iteratorClaimed = false;
  #closed = false;

  constructor(
    private readonly maxItems = 128,
    private readonly maxBytes = 1 * 1024 * 1024,
  ) {}

  #size(item: StreamItem): number {
    return new TextEncoder().encode(JSON.stringify(item)).byteLength;
  }

  async #waitForSpace(): Promise<void> {
    await new Promise<void>(resolve => this.#waitWriters.push(resolve));
  }

  #notifyWriters(): void {
    for (const wake of this.#waitWriters.splice(0)) wake();
  }

  async push(item: StreamItem): Promise<void> {
    if (this.#closed) throw new Error("stream closed");
    const size = this.#size(item);
    if (size > this.maxBytes) throw new Error("single stream item exceeds byte budget");

    if (item.kind === "progress") {
      while (!this.#closed) {
        const i = this.#items.findIndex(x => x.kind === "progress" && x.operationId === item.operationId);
        if (i < 0) break;
        const oldSize = this.#size(this.#items[i]);
        if (this.#bytes - oldSize + size <= this.maxBytes) {
          this.#items[i] = item;
          this.#bytes += size - oldSize;
          this.#waitReader?.();
          return;
        }
        await this.#waitForSpace();
      }
    }
    if (item.kind === "text") {
      const last = this.#items.at(-1);
      if (last?.kind === "text" && last.messageId === item.messageId) {
        const merged = { ...last, text: last.text + item.text };
        const delta = this.#size(merged) - this.#size(last);
        if (this.#bytes + delta <= this.maxBytes) {
          this.#items[this.#items.length - 1] = merged;
          this.#bytes += delta;
          this.#waitReader?.();
          return;
        }
      }
    }
    while ((this.#items.length >= this.maxItems || this.#bytes + size > this.maxBytes) && !this.#closed) {
      await this.#waitForSpace();
    }
    if (this.#closed) throw new Error("stream closed");
    this.#items.push(item);
    this.#bytes += size;
    this.#waitReader?.();
  }

  close(): void {
    this.#closed = true;
    this.#waitReader?.();
    this.#notifyWriters();
  }

  async *[Symbol.asyncIterator](): AsyncIterator<StreamItem> {
    if (this.#iteratorClaimed) throw new Error("BoundedStream supports exactly one consumer");
    this.#iteratorClaimed = true; // 多 producer / 单 consumer；每个订阅者各建一个实例
    while (!this.#closed || this.#items.length) {
      if (!this.#items.length) {
        await new Promise<void>(resolve => { this.#waitReader = resolve; });
        continue;
      }
      const next = this.#items.shift()!;
      this.#bytes -= this.#size(next);
      this.#notifyWriters();
      yield next;
    }
  }
}
```

这个示例使用 MPSC（多 producer、单 consumer），同时限制 item 数与 UTF-8 字节数；writer 使用 waiter 队列，不会互相覆盖。生产实现还需处理 AbortSignal、异常关闭、text 分块上限与 waiter 清理。跨进程 IPC 推荐 `MessagePort`/流式协议，给每个 renderer 独立订阅缓冲；renderer 消失不能阻塞 Run 本身。

## 5. 取消是端到端协议

关闭我们自己的 UI SSE/WebSocket 只说明订阅断开，不表示用户要取消 Run；Runtime API 必须有独立的 `cancel(runId, reason, expectedSeq)`。不要把这条产品语义机械套到 MCP：在 2026-07-28 Streamable HTTP 中，一个 MCP 请求对应自己的响应流，abort/timeout 关闭该请求流就是该请求的协议取消信号；持久长任务另用 Tasks 扩展的 `tasks/cancel`。stdio 和旧协议的取消由对应 SDK 兼容层处理。

```ts
class CancellationScope {
  readonly controller = new AbortController();
  #children = new Set<AbortController>();

  child(): AbortSignal {
    const c = new AbortController();
    if (this.controller.signal.aborted) c.abort(this.controller.signal.reason);
    else this.controller.signal.addEventListener("abort", () => c.abort(this.controller.signal.reason), { once: true });
    this.#children.add(c);
    return c.signal;
  }

  cancel(reason = "user_cancelled") {
    this.controller.abort(reason);
    for (const c of this.#children) c.abort(reason);
  }
}
```

取消路径：持久化 `cancel_requested` → 状态进入 cancelling → abort 模型请求/工具/子流程 → 等待清理或 grace deadline → 写 `run_cancelled`。若工具已进入不可中断的提交区，记录 `cancellation_pending`，完成后根据事件序列决定展示“已完成但取消来晚”或执行补偿。

不要把所有 `AbortError` 计为系统故障；区分 user、deadline、shutdown、parent_cancel。进程优雅退出时先停止领取新 Run，再取消或 checkpoint 在途任务，最后 flush 事件与 trace。

## 6. 重试与恢复：从最后一个已提交事实继续

恢复从最后一个已提交事实开始，而不是重新发送最后一条用户消息：

1. 获取 Run lease，防止两个 worker 同时恢复；
2. 读取最新兼容 snapshot 与其后事件；
3. 重放 reducer 得到状态，验证 snapshot checksum/headSeq；
4. 扫描未完成 operation：有 `started` 无 terminal 的模型/工具调用；
5. 对每项按幂等与查询能力决定 resume、confirm、retry 或人工介入；
6. 重新发布当前状态和缺失事件，继续状态机。

Checkpoint 至少包括 `stateVersion/headSeq/currentStep/pendingOperations/contextManifestRef/artifactRefs`。不要序列化 `AbortController`、socket、闭包或 SDK client；这些是恢复后重新创建的进程资源。

模型文字流中断时有三种策略：

- 丢弃未完成 attempt，保留 partial 并以新 attempt 重新生成；输出可能不同；
- 若 provider 支持稳定的响应/续传 ID，使用它恢复；
- 要求模型基于已提交 partial 续写，但需处理重复前缀，且不保证语义等价。

无论哪种都要形成显式事件，不能把两个 attempt 的 token 伪装成一次连续生成。

## 7. 重连与客户端状态

客户端保存每个 Run 的 `lastAppliedSeq`，收到事件时：

```ts
function applyClientEvent(state: UIState, e: RunEvent): UIState {
  if (e.seq <= state.lastAppliedSeq) return state; // duplicate
  if (e.seq !== state.lastAppliedSeq + 1) {
    return { ...state, connection: "resync_required" };
  }
  return project(state, e); // 纯投影函数
}
```

发现 gap 时停止直接应用实时流，按 afterSeq 补历史。服务端对已淘汰的细粒度 delta 可返回 snapshot + 新 baseSeq，但必须明确 `reset` 语义，客户端不能把 snapshot 追加到旧文本后。

多窗口同时查看同一 Run 时，它们各自维护游标；批准/取消命令使用 `clientRequestId` 去重，并通过 expected state/version 防止双击和过期 UI 操作。

### 7.1 分清 deadline、idle timeout 与心跳

整次 Run 的 deadline、一次模型请求的 timeout、流多久没有字节的 idle timeout、工具 heartbeat timeout 含义不同。模型长时间思考可能没有文本 delta，但控制连接仍正常；因此不能仅凭“几秒没字”判死。Adapter 应把 provider heartbeat 与内容事件分开，Runtime 依据各层策略产生明确超时事件。

客户端心跳只判断订阅健康，不续租业务执行权限；worker heartbeat 才用于 Run lease。机器休眠后单调时钟可能跳变，恢复时以持久化 deadline 重新计算，不沿用内存定时器。网络恢复要加指数退避和随机抖动，防大量客户端同时重连；服务端返回重试提示时优先尊重，但仍受本地最大等待上限约束。

## 8. 失败模式

| 失败模式 | 表现 | 修正 |
| --- | --- | --- |
| socket 断开即取消 Run | 切后台任务意外停止 | 订阅与执行生命周期分离 |
| 无限缓存 delta | renderer 慢导致主进程 OOM | bounded queue + 分类合并/落盘重放 |
| 每 token 持久化 | 数据库写放大 | 时间/大小批量 chunk |
| completed 先于最后内容 | UI 刷新后答案截断 | 同事务/严格事件顺序提交 |
| 重启重放用户消息 | 重复工具和费用 | snapshot + event replay + operation receipt |
| 所有中断都自动重试 | 不可逆副作用重复 | unknown outcome 查询与人工介入 |

## 9. 测试、练习与验收

故障注入比普通 happy path 更重要：随机在每个事件提交前后 kill worker；让 UI 每秒只消费一条而模型每秒生产百条；在 tool 成功、receipt 上报前断网；让 SSE 重复、乱序和缺失。

**练习**：实现一个支持 `Last-Event-ID` 的 SSE `/events` 端点。在“历史补齐到实时订阅”的交界处注入新事件，验证不会漏；让客户端重复连接十次，确认最终投影与一次连接完全一致。

**验收点**：

- [ ] 连接断开、显式取消、deadline、进程 shutdown 有不同语义和指标。
- [ ] 事件有单 Run seq，客户端能去重、发现 gap、从 afterSeq 恢复。
- [ ] 慢消费者不会形成无限内存，关键事件不可丢。
- [ ] completed 只在最终内容和 usage 持久化后出现。
- [ ] 恢复能识别 started-without-terminal，并按幂等能力分类处理。

## 延伸阅读

- [OpenAI Agents SDK：Running agents（streaming / AbortSignal）](https://developers.openai.com/api/docs/guides/agents/running-agents)
- [MCP 2026-07-28 发布说明（stateless、Tasks、subscriptions）](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [MCP TypeScript SDK：2026-07-28 取消与 subscriptions/listen](https://ts.sdk.modelcontextprotocol.io/v2/migration/support-2026-07-28)
- [WHATWG Streams Standard](https://streams.spec.whatwg.org/)
- [Node.js：Backpressuring in Streams](https://nodejs.org/en/learn/modules/backpressuring-in-streams)
