# Runtime Kernel 可运行实验

这个实验实现 Agent Runtime 的最小纵向链路，覆盖模型流式输出、幂等 Tool、追加式 Event Store、乐观并发和取消。它用于验证执行语义，不应直接当作生产框架：

```text
RunCreated → Model stream → ToolRequested → 幂等 Tool → Model final → RunCompleted
       ↘ append-only Event Store / optimistic concurrency / cancellation ↗
```

## 运行

在知识库根目录执行：

```bash
npm run lab
npm run test:lab
npm run typecheck:lab
```

## 文件导读

| 文件 | 观察重点 |
| --- | --- |
| `src/contracts.ts` | Provider、Tool、Event Store 与领域事件的稳定端口 |
| `src/reducer.ts` | 纯函数如何从事件重建 `RunState` 并决定下一动作 |
| `src/event-store.ts` | append-only、`expectedSeq` 乐观并发 |
| `src/fake-provider.ts` | AsyncIterable streaming、AbortSignal 与可重复 fixture |
| `src/tool-gateway.ts` | 同一进程内，稳定幂等键如何复用已缓存的工具结果 |
| `src/runtime.ts` | I/O effect 与状态决策分离、终态与取消 |
| `test/runtime.test.ts` | 正常链路、并发冲突、取消和 crash window 注入 |

## 有意保留的生产缺口

实验中的 Event Store 和 Idempotency Store 都在内存中。`crash window` 测试只重建 `AgentRuntime` 并复用同一个 Tool Gateway，证明 Runtime 会复用稳定调用键；它**没有**模拟操作系统进程死亡，也没有证明 durable exactly-once。真实副作用与本地 ledger 无法放进同一事务时，进程可能在副作用完成后、receipt 落库前退出。生产环境必须替换为 SQLite/PostgreSQL 等持久化实现，并加入：

- 跨进程 lease + fencing token，不能只依赖进程内 `inflight` Map；
- schema version、snapshot、transactional outbox、artifact store；
- Tool 参数 JSON Schema、Policy/Approval、沙箱、审计与结果去敏；
- Provider retry/backoff、usage、capability、deadline、未知终态处理；
- OpenTelemetry spans、Eval 与租户隔离。

练习：先让现有测试全部通过，再把 `InMemoryEventStore` 与 Tool receipt store 一并替换成 SQLite。对于外部写操作，把同一 `idempotencyKey` 透传给支持幂等的下游；若下游不支持，则保存 operation reference，并在 `started/unknown` 后先查询 reconcile、禁止盲重试。最后在 `tool effect 已完成、tool.succeeded 事件未落库` 的位置注入进程级 kill，验证重启后的结果是“下游复用原结果”或“进入 unknown 待核对”，而不是声称仅靠本地 Map 实现 exactly-once。
