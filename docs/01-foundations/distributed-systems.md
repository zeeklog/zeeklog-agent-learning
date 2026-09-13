# 分布式系统语义：Agent Runtime 的隐藏主干

## 1. 为什么“本地桌面应用”也是分布式系统

AI Desktop 至少包含 Renderer、Main、Runtime Sidecar、远程 Provider、Tool/MCP Server 和平台控制面。这些组件会独立失败，并面对网络延迟、消息重复与版本错配；即使都在同一台电脑，跨进程 IPC 也不能提供业务意义上的 exactly-once。

Runtime 必须回答：

- 消息可能丢失或重复时，哪个状态是 source of truth？
- 外部副作用已发生但响应丢失时，重试会怎样？
- 两个 worker 同时恢复一个 Run 时，谁拥有执行权？
- 时钟不一致时，timeout、lease 和排序如何定义？
- 新版本代码如何恢复旧版本 Workflow？

## 2. 交付语义：at-most-once、at-least-once、effectively-once

| 语义 | 行为 | 适用场景 | 代价/风险 |
| --- | --- | --- | --- |
| at-most-once | 最多执行一次，丢失不补 | UI delta、可丢 telemetry | 结果可能缺失 |
| at-least-once | 未确认就重投 | 事件消费、可幂等任务 | 必须处理重复 |
| effectively-once | at-least-once + 幂等/去重，让业务效果近似一次 | 写文件、创建工单 | 依赖幂等键、事务或状态核对 |

不要声称“exactly-once Tool execution”，除非副作用和 Runtime 事件能处于同一原子事务——跨 SaaS、Shell、文件系统时通常做不到。更诚实的状态是 `outcome_unknown`，由状态核对、补偿或人工处理。

## 3. Event Sourcing：保存事实，不保存“当前感觉”

事件日志是 append-only 事实：

```ts
type RunEvent =
  | { seq: number; type: 'run.created'; runId: string; workflowVersion: string }
  | { seq: number; type: 'step.started'; stepId: string; attempt: number }
  | { seq: number; type: 'tool.dispatched'; callId: string; idempotencyKey: string }
  | { seq: number; type: 'tool.succeeded'; callId: string; resultRef: string }
  | { seq: number; type: 'tool.outcome_unknown'; callId: string; reason: string }
  | { seq: number; type: 'run.completed'; outputRef: string }
  | { seq: number; type: 'run.cancelled'; reason: string };

type RunState = {
  status: 'running' | 'waiting' | 'completed' | 'cancelled' | 'failed';
  nextSeq: number;
  activeStep?: string;
  toolCalls: Record<string, 'dispatched' | 'succeeded' | 'unknown'>;
};

function reduce(state: RunState, event: RunEvent): RunState {
  if (event.seq !== state.nextSeq) throw new Error('event sequence gap');
  // 每个 case 只做纯状态转换，不调用网络、文件或时钟。
  // ...
  return { ...state, nextSeq: state.nextSeq + 1 };
}
```

Reducer 必须纯且确定，这样可从同一事件重建状态、复现 Bug、运行新旧 reducer 对比。Snapshot 是性能优化，不是事实源；要保存 `lastEventSeq` 和 schema/version，损坏时可从事件重建。

## 4. 原子性边界与 Transactional Outbox

常见 Bug：先更新数据库为 `tool.completed`，再发布 UI 事件，发布失败；或先发消息，再写数据库，进程崩溃后重复。

如果数据库和消息 outbox 可处于同一事务：

```sql
BEGIN;
INSERT INTO run_events(run_id, seq, type, payload)
VALUES (?, ?, 'tool.succeeded', ?);

INSERT INTO outbox(event_id, topic, payload, status)
VALUES (?, 'runtime.events', ?, 'pending');
COMMIT;
```

独立 publisher 重复发送 pending outbox；consumer 通过 `event_id` 去重。这样保证“状态提交后，事件最终可见”，但仍是 at-least-once。

桌面本地 SQLite 同样适用：UI 重连时以 Event Store cursor 补齐，不把内存 EventEmitter 当 source of truth。

## 5. Tool 幂等与未知结果

### 幂等键

幂等键必须由业务意图确定，不能每次重试重新生成：

```ts
import { createHash } from 'node:crypto';
import canonicalize from 'canonicalize'; // RFC 8785/JCS 实现；固定版本并跑官方 test vectors

function toolIdempotencyKey(runId: string, logicalStepId: string, input: unknown) {
  const canonicalInput = canonicalize(input);
  if (canonicalInput === undefined) throw new TypeError('input is not canonicalizable JSON');
  return createHash('sha256')
    .update(`${runId}\0${logicalStepId}\0${canonicalInput}`)
    .digest('hex');
}
```

不能用 `JSON.stringify(input, Object.keys(input).sort())` 充当 canonical JSON：array replacer 会让嵌套对象丢字段，产生哈希碰撞。实际实现需遵循 RFC 8785 一类跨语言规范，递归规范对象键、数字与 Unicode，并用官方 test vectors 验证 TypeScript/Rust/Python 输出一致。Hash 用于标识，不替代审计中的去敏输入摘要。

### 执行状态机

```text
prepared -> dispatched -> succeeded
                     \-> failed_known
                     \-> outcome_unknown
```

- `prepared`：事件已落库，尚未跨越副作用边界，可安全执行。
- `dispatched`：已发出请求。超时不能证明未执行。
- `succeeded/failed_known`：得到确定结果。
- `outcome_unknown`：连接断开或进程崩溃，必须 query-by-idempotency-key、检查真实状态或人工调解。

## 6. Lease、Fencing Token 与并发恢复

仅用 `locked = true` 会产生永久锁和脑裂。使用有期限 lease，并为每次所有权发递增 fencing token：

```ts
interface Lease {
  runId: string;
  ownerId: string;
  expiresAt: number;
  fencingToken: bigint;
}

// 所有写操作带 fencingToken；存储层拒绝小于当前 token 的旧 owner。
async function appendWithFence(event: RunEvent, token: bigint) {
  // UPDATE ... WHERE current_fencing_token <= token
}
```

旧 worker 即使暂停后恢复，也不能覆盖新 owner 的进度。Lease 到期用单调时钟判断本进程等待；跨节点到期事实依赖存储服务器时间或数据库条件更新，避免客户端墙钟漂移。

## 7. Retry、Backoff、Deadline 与 Budget

重试是放大器：它可能放大流量、成本和副作用。每层都重试会形成指数级调用。应由最了解语义的一层决定：

```ts
type RetryPolicy = {
  maxAttempts: number;
  baseDelayMs: number;
  maxDelayMs: number;
  retryableCodes: ReadonlySet<string>;
};

function fullJitter(attempt: number, policy: RetryPolicy) {
  const cap = Math.min(policy.maxDelayMs, policy.baseDelayMs * 2 ** attempt);
  return Math.floor(Math.random() * cap);
}
```

必须同时检查：Run deadline、Step attempt budget、Provider cost budget、用户取消、操作是否幂等。收到服务端 `retry-after` 时尊重它，但不能超过 deadline。

## 8. Workflow Versioning

长任务可能在代码发布前开始、发布后恢复。直接用新代码解释旧状态会破坏确定性。可选策略：

1. **版本钉住**：Run 保存 workflow version，旧 worker/定义保留到任务结束。
2. **显式迁移**：对暂停状态编写 `v3 -> v4` migration，并保留审计事件。
3. **兼容分支**：reducer 根据 version marker 采用不同路径，之后逐步清理。

禁止在重放路径读取当前时间、随机数或网络。把非确定结果记录为事件：`ClockRead`、`RandomChosen`、`ModelResponded`。

## 9. CAP、SAGA 等知识如何实际关联

- **CAP**：控制面短暂不可用时，数据面是停止执行（偏一致）还是使用最后已知策略（偏可用）？高风险工具通常宁可 fail closed。
- **SAGA**：多系统写操作无法分布式事务时，以步骤 + 补偿组织，但补偿也可能失败且未必能完全逆转。
- **CQRS**：写入 append-only events，读取用物化视图；UI 和审计查询无需扫描全部事件。
- **Actor Model**：每个 Run 串行处理 mailbox 可简化并发，但持久化、跨进程迁移和背压仍需设计。
- **CRDT**：适合部分离线协作状态，不适合拿来解决有副作用 Tool 的业务顺序。

## 10. 故障注入清单

在以下边界随机终止进程：事件写前、事件写后/副作用前、副作用请求发出后、结果收到后/事件写前、outbox 发布前后、snapshot 写入中。每次恢复检查：

- Run 状态可以从事件重建。
- 不产生未授权的新动作。
- 可幂等动作最多一个业务效果。
- 未知结果被显式标记，不能伪装成功或失败。
- seq、trace、audit 中没有无法解释的间隙。

## 11. 练习与验收

1. 用 SQLite 实现 Event Store：`(run_id, seq)` 唯一约束、乐观并发、snapshot、outbox。
2. 为“创建工单”设计支持幂等键与状态查询的 Tool Adapter。
3. 启动两个 worker 竞争同一 Run，证明 fencing token 阻止旧 owner 写入。
4. 在 100 次随机 kill 测试中证明恢复不丢状态，并统计 `outcome_unknown`。

验收：你不再用“失败就重试”“消息只消费一次”描述可靠性，而能明确每个边界的事实、交付语义和调解路径。

## 延伸阅读

- [Martin Kleppmann：Designing Data-Intensive Applications](https://dataintensive.net/)
- [Temporal 文档：Durable Execution](https://docs.temporal.io/temporal)
- [OpenTelemetry：Trace 规范](https://opentelemetry.io/docs/specs/otel/trace/)

下一章：[检索、上下文与评测基础 →](../../docs/01-foundations/retrieval-evaluation.md)
