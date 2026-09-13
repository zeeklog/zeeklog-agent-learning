# 可靠性、可观测性、容量与成本

## 1. 先定义“成功”

HTTP 200 只能证明入口返回，不能证明 Agent 正确完成。可靠性设计要沿用户旅程定义 SLI，并同时约束结果质量、完成时延、副作用一致性、恢复能力与单位任务成本。

| 旅程 | SLI 示例 | SLO 示例（需按业务校准） |
|---|---|---|
| 交互问答 | 首 token、完整响应、答案通过 eval | 99.5% 可用；P95 首 token < 2s |
| 读工具任务 | 正确工具/参数、最终事实正确 | 99% 在 15s 内正确完成 |
| 写工具任务 | 正确审批、单一业务效果/未知结果可调解 | 99.9% 无重复业务效果；100% 有审计 |
| 长任务 | 接受、进度可查、恢复、deadline | 99% 在目标时间完成或明确失败 |

质量、延迟、成本是联合约束。建议 `cost_per_success = 总成本 / 通过业务验收的任务数`，而不是只看每百万 token 单价。错误预算同时覆盖系统错误和经抽样/评测确认的错误结果；高风险越权属于零容忍事件，不能被普通错误预算平均。

## 2. 可观测数据模型

```text
Trace(run)
 ├─ agent/turn(model request, tokens, cache, route)
 ├─ retrieval(query, corpus, ACL digest, top-k)
 ├─ tool(call, policy decision, approval, attempt)
 ├─ handoff/child-run(linked trace)
 └─ artifact/evaluation
```

统一字段：`tenant.id`、`agent.id/version`、`run.id`、`step.id/attempt`、`workflow.name/version`、`model.provider/request/response model`、`tool.name/version`、token、cache、cost、policy decision、finish reason、error class。OpenTelemetry GenAI 语义约定截至当前仍含 Development 状态且已迁到独立仓库，应固定 schema URL/版本并保留内部稳定字段。

prompt、system instructions、tool args/results 可能含 PII/secret。默认只记录长度、hash、分类、token 与受控摘要；内容采集必须 opt-in、脱敏、加密、短保留并记录谁解密访问。

## 3. 全链路可靠性模式

每层只做自己能判断的重试，所有层共享 retry budget。429/暂态 5xx 可指数退避 + jitter；policy deny、schema invalid、预算超限不可重试；超时后不确定的写操作先查询 operation 状态，不能盲目重放。

```ts
async function retry<T>(op: (attempt: number) => Promise<T>, budget: RetryBudget): Promise<T> {
  if (budget.maxAttempts < 1 || Date.now() >= budget.deadline) {
    throw new RetryBudgetExhaustedError("retry budget expired before first attempt");
  }
  let last: unknown = new RetryBudgetExhaustedError("retry budget exhausted");
  for (let i = 0; i < budget.maxAttempts && Date.now() < budget.deadline; i++) {
    try { return await op(i); }
    catch (e) {
      last = e;
      if (!isTransient(e) || !budget.consume()) throw e;
      const waitMs = fullJitter(Math.min(8_000, 250 * 2 ** i), retryAfterMs(e));
      if (Date.now() + waitMs >= budget.deadline) break;
      await delay(waitMs);
    }
  }
  throw last;
}
```

- **幂等**：`idempotencyKey = tenant/run/step/effect`，服务端持久化结果；消息使用 inbox 去重。
- **背压**：入口 admission、每租户队列、公平调度；队列 age 超阈值时拒绝新低优先级任务，而非无限堆积。
- **熔断/舱壁**：provider、model、tool、tenant 各自连接池和熔断器，避免一个慢工具耗尽全部 worker。
- **降级**：只在业务允许且经 eval 验证时切小模型、关非关键工具或返回异步；绝不通过跳过安全策略降级。
- **取消**：取消信号贯穿 model stream、tool、child agent 和 sandbox；已提交外部副作用进入补偿/状态查询，而非伪装取消。

## 4. 容量规划

区分请求并发、模型 TPM/RPM、长连接数、工具并发、队列 worker、沙箱 CPU/内存和存储写入。用真实 trace 建 workload model：输入/输出 token 分布、turns/run、tools/run、P50/P95 服务时间、峰谷与租户集中度。

Little 定律 `L = λW`：若峰值 20 run/s、平均占用 12s，理论在途约 240；再乘突发/故障余量。模型容量还受 token 约束：`required_TPM = runs_per_min × P95_tokens_per_run`。采用 P95 而非平均数可避免长上下文压垮配额。压测必须包含 streaming 慢消费者、429、provider latency 翻倍、工具超时和单租户洪峰。

## 5. 成本账本与优化顺序

```ts
const runCost =
  inputTokens * inputRate + cachedReadTokens * cacheReadRate +
  cacheWriteTokens * cacheWriteRate + outputTokens * outputRate +
  reasoningTokens * reasoningRate + toolFees + sandboxSeconds * sandboxRate +
  storageGbDays * storageRate + egressGb * egressRate;
```

账本保存 provider invoice dimensions 和内部维度（tenant/agent/release/run/route），费率带生效时间。先优化无效工作，再换便宜模型：去掉重复上下文/工具、限制循环、并行独立读取、缓存稳定 prompt prefix、压缩/检索上下文、早停、批处理离线任务；之后用 eval 验证小模型路由。缓存 key 必须含 prompt/tool/model/ACL 版本，缓存命中也要计入质量回归。

预算可分层：组织月度、租户日度、Agent release、单 run。软阈值告警/降级，硬阈值在下一个安全边界停止；已批准的写事务不要因预算恰好耗尽停在不可恢复中间态。

## 6. 失败模式

- **只监控均值**：P99 工具尾延迟和长上下文被掩盖。使用 histogram、分租户/模型/工具切片。
- **多层无限重试**：流量放大。共享 retry budget、尊重 `Retry-After`。
- **fallback 无评测**：小模型便宜却提高失败重试和人工介入，单位成功成本更高。
- **trace 采样丢失败**：尾部采样保留 error、policy deny、高成本和高风险写 run。
- **成本无归属**：月底只能看到 provider 总账。usage event 必须与 run/tenant/release 关联并对账。
- **取消等于断开 socket**：后台继续烧钱/写数据。任务状态与传输状态分离。

## 7. 练习与验收

关联 SRE error budget、排队论、分布式幂等、backpressure、circuit breaker、tail-based sampling、FinOps。结合[控制面/数据面](control-data-plane.md)理解 cell 容量。

练习：生成三种 workload（短问答、工具任务、长任务），构建 dashboard 与成本账本；故障注入 429、10% 慢工具、重复投递、OTel collector 中断和 provider 全挂。

验收：能按 run 对账到 provider 用量误差 <1%；重复副作用为 0；遥测中断不阻塞数据面且可缓冲；过载时高优先级 SLO 保持目标；降级前后以同一 eval 证明质量下限。

## 官方资料

- [Google SRE Workbook：Implementing SLOs](https://sre.google/workbook/implementing-slos/)
- [OpenTelemetry GenAI Semantic Conventions](https://github.com/open-telemetry/semantic-conventions-genai)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [FinOps Framework](https://www.finops.org/framework/)
- [OpenCost specification](https://www.opencost.io/docs/specification)
