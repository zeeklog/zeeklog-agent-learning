# Observability 与 Evals：信号、指标与质量验证

Event、Log、Metric、Trace 与 Eval 各自回答不同问题。本章定义 Agent Run 的 Span 和指标，再用可复现数据集评估 Prompt、模型、工具及 Runtime 变更。

## 1. 五类信号回答不同问题

- **Event History**：运行恢复与审计的事实来源；要求完整、有序、可重放。
- **Log**：离散诊断文本；适合错误细节，不适合作为流程状态真相。
- **Metric**：低基数聚合趋势；回答成功率、延迟、成本、容量是否异常。
- **Trace**：一次 Run 跨模型、工具、服务的因果路径；回答慢/错在哪里。
- **Eval**：在定义的数据集和评分标准上比较质量；回答版本是否更好。

Trace 采样或导出失败不能影响 Run 正确性；Event Store 则属于执行关键路径。反过来，不要用 Event History 承载所有 token delta 与 debug stack，让恢复存储变成日志仓库。

## 2. Trace 结构与关联 ID

推荐层次：

```text
run / workflow
├── context.build
│   ├── memory.recall
│   └── document.retrieve
├── agent.turn #1
│   ├── model.generate attempt=1
│   ├── policy.evaluate
│   └── tool.execute read_file
├── approval.wait
└── agent.turn #2
    └── model.generate attempt=1
```

每个 span 至少包含 trace/span/parent、runId、tenant（可哈希）、agent/workflow/version、model/provider、tool/version、attempt、status/error class、start/end。模型 span 记录 input/output/cached token 和首 token 延迟；Context span 记录 budget/selected/omitted 数量和 manifest hash；Tool span 记录 queue/execute latency、risk、approval、sandbox 与 receipt 状态。

`runId` 是业务关联键，`traceId` 是观测键，二者不要合并。重试、异步 Activity 或跨设备恢复可能产生多个 Trace，但都关联同一 Run；用 Span Link 连接异步生产/消费，而不是伪造跨数小时的单一父子调用栈。

## 3. OpenTelemetry 包装示例

```ts
import { context, propagation, SpanStatusCode, trace } from "@opentelemetry/api";

const tracer = trace.getTracer("enterprise-ai-runtime", "1.0.0");

export async function tracedTool<I, O>(input: {
  runId: string;
  toolName: string;
  toolVersion: string;
  risk: string;
  execute: () => Promise<O>;
}): Promise<O> {
  return tracer.startActiveSpan(`tool ${input.toolName}`, {
    attributes: {
      "ai.run.id": input.runId,
      "ai.tool.name": input.toolName,
      "ai.tool.version": input.toolVersion,
      "ai.tool.risk": input.risk
    }
  }, async span => {
    try {
      const result = await input.execute();
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.recordException(error as Error);
      span.setAttribute("error.type", classifyError(error));
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw error;
    } finally {
      span.end();
    }
  });
}

export function injectTraceHeaders(): Record<string, string> {
  const carrier: Record<string, string> = {};
  propagation.inject(context.active(), carrier);
  return carrier;
}

declare const classifyError: (e: unknown) => string;
```

优先对齐 OpenTelemetry GenAI/MCP 语义约定，但约定仍会演进；平台内部字段先放稳定命名空间，集中 adapter 映射，不要把实验字段散落业务代码。OpenAI Agents SDK 等框架有自带 tracing 时，可实现 processor/exporter 接入统一后端，避免同一次模型调用生成两套互不关联 Span。

## 4. Agent Runtime 的关键指标

### 可靠性与延迟

- Run success/cancel/failure rate，按稳定 error class 分组；
- end-to-end p50/p95/p99、time-to-first-token、turn latency；
- tool queue/execute latency、retry/timeout/unknown-outcome rate；
- recovery count、resume latency、stuck run age、lease conflict；
- approval wait 与 deny rate（等待时间通常不计入系统处理 SLO）。

### 成本与效率

- input/output/cache token、供应商费用估算与最终账单差异；
- 每成功 Run 成本、每类任务平均 turn/tool 次数；
- context utilization、被压缩/遗漏 token、tool schema 占比；
- 重试浪费 token、重复工具调用率、cache hit rate。

### 容量与体验

- 并发 Run、模型/Activity 队列深度、worker saturation；
- streaming buffer 水位、事件落后 seq、断连/重连率；
- 用户重新提问/人工接管/撤销率等谨慎定义的 outcome signal。

Metric 标签禁止使用 runId/userId/prompt/tool args 等高基数字段；这些放 Trace/Log。错误 message 也不要直接做 label，应映射稳定 `error_class`。

## 5. 隐私、采样与脱敏

Prompt、模型输出、工具参数可能包含源码、个人信息和密钥。建议定义采集级别：

- Level 0：只记录长度、hash、分类、token、版本；生产默认；
- Level 1：记录脱敏摘要/结构字段；经租户策略允许；
- Level 2：记录加密正文；仅诊断会话、短 TTL、严格 RBAC 与审计。

先脱敏再入 exporter，不能指望观测后端二次处理。Head sampling 可能漏掉稍后失败的 Trace；关键 Run 可全采，普通成功 Run 低比例采样，或使用 tail sampling 按 error/latency/成本保留。无论是否采 Trace，核心 Metrics 与安全 Audit Event 不采样。

OpenAI Agents SDK 的 tracing 可包含模型与工具输入输出，接入时显式配置敏感数据策略；浏览器与服务器运行时的默认行为也可能不同，不应靠默认值满足合规。

## 6. Eval 数据模型

Eval Case 不只是一组 prompt/expected string：

```ts
interface EvalCase {
  id: string;
  datasetVersion: string;
  task: { input: string; fixtures: Record<string, unknown> };
  expected: {
    mustUseTools?: string[];
    forbiddenTools?: string[];
    assertions: string[];
    maxCostUsd?: number;
    maxTurns?: number;
  };
  tags: string[];
  sensitivity: string;
}

interface EvalResult {
  caseId: string;
  config: {
    datasetVersion: string;
    model: string; promptVersion: string; toolSetHash: string;
    workflowVersion: string; runtimeCommit: string;
    judge: { model: string; promptVersion: string; rubricVersion: string };
  };
  scores: Record<string, number>;
  outcome: "pass" | "fail" | "error";
  traceId: string;
  costUsd: number;
  latencyMs: number;
}
```

数据集应覆盖常规、边界、安全对抗和历史事故回归；按 tenant 数据构建时先脱敏并取得合法用途。生产失败经人工归因后可纳入 regression case，但不要把用户敏感原文直接复制到公共测试集。

## 7. 评分器组合

从高确定性到低确定性：

1. **程序化断言**：JSON schema、测试通过、文件 diff、是否调用禁用 Tool；最可信；
2. **结果/环境验证**：在沙箱执行产物、查询最终系统状态；
3. **规则/相似度**：关键词、引用覆盖率；只适合特定维度；
4. **LLM-as-judge**：正确性、完整性、风格；必须有 rubric、示例、盲测版本并校准人类一致性；
5. **人工评审**：高成本，用于金标、模糊样例和 judge 校准。

一个总分会隐藏安全退化。应保留维度，并设置硬门槛：`unsafe_tool_call=0`、`data_leak=0`；质量与成本再做权衡。Judge 模型、prompt 和温度也要版本化，评分时避免把被评输出中的 prompt injection 当 judge 指令。

```ts
async function evaluate(c: EvalCase, run: (c: EvalCase) => Promise<RunArtifact>): Promise<EvalResult> {
  const started = performance.now();
  const artifact = await run(c);
  const schema = scoreSchema(artifact.finalOutput);
  const tools = scoreToolPolicy(artifact.events, c.expected);
  const outcome = await verifySandboxOutcome(artifact, c);
  const assertions = scoreAssertions(artifact, c.expected.assertions);
  const judged = await rubricJudge({ case: c, output: artifact.finalOutput });
  const scores = { schema, tools, outcome, assertions, quality: judged.score };
  const overCost = c.expected.maxCostUsd !== undefined && artifact.costUsd > c.expected.maxCostUsd;
  const overTurns = c.expected.maxTurns !== undefined && artifact.turns > c.expected.maxTurns;
  const hardFail = schema < 1 || tools < 1 || outcome < 1 || assertions < 1 || overCost || overTurns;
  return {
    caseId: c.id,
    config: { ...artifact.config, datasetVersion: c.datasetVersion, judge: judged.version },
    scores,
    outcome: hardFail ? "fail" : "pass",
    traceId: artifact.traceId,
    costUsd: artifact.costUsd,
    latencyMs: performance.now() - started
  };
}

declare interface RunArtifact {
  finalOutput: unknown; events: unknown[]; traceId: string; costUsd: number;
  turns: number;
  config: Omit<EvalResult["config"], "datasetVersion" | "judge">;
}
declare const scoreSchema: (x: unknown) => number;
declare const scoreToolPolicy: (e: unknown[], x: EvalCase["expected"]) => number;
declare const scoreAssertions: (a: RunArtifact, assertions: string[]) => number;
declare const verifySandboxOutcome: (a: RunArtifact, c: EvalCase) => Promise<number>;
declare const rubricJudge: (x: { case: EvalCase; output: unknown }) => Promise<{
  score: number;
  version: EvalResult["config"]["judge"];
}>;
```

## 8. Offline、Replay、Shadow 与 Online

- **Offline eval**：固定 fixtures，适合 PR/发布门禁；可比较 prompt/model/runtime 组合。
- **Trace replay**：重用历史输入和工具结果以低成本测试决策，但会隐藏真实工具延迟与环境变化；注明哪些节点被 stub。
- **Shadow**：生产输入复制给候选版本但禁止副作用，比较决策；涉及隐私与双倍模型成本。
- **Canary/A-B**：少量真实流量，必须有 kill switch；用户/租户稳定分桶，避免同一会话版本漂移。
- **Online eval**：采样真实结果做安全/质量评分；绝不能让不可信 judge 自动执行补救副作用。

模型有随机性，单次 pass/fail 容易误判。关键 case 可多次运行，比较置信区间、win rate 与成本分布。发布门槛使用与 baseline 的差异，而非永远固定的绝对分数。

## 9. 可运营的 SLO 与告警

示例 SLO：“排除用户审批等待后，过去 28 天交互式 Run 99% 在 120 秒内完成或给出可恢复失败；高风险未批准工具调用为 0。”错误预算消耗过快时自动减缓发布/回滚模型路由。

告警应对应动作：provider 429 激增触发限流/切换；unknown outcome 触发人工队列；stuck run age 触发恢复 worker；Eval 安全维度回归阻断发布。不要为每个单次模型错误半夜叫醒值班人员。

事故复盘要能从告警指标下钻到代表性 Trace，再关联不可变 Event History；三者通过 runId 和 operationId 对齐。若只能看到“模型失败率升高”却无法分辨供应商、Context 版本、工具策略或特定任务标签，就无法安全回滚。每项发布变更都写 deployment/config marker，才能把质量与延迟拐点归因到具体版本。

## 10. 失败模式

| 失败模式 | 后果 | 修正 |
| --- | --- | --- |
| Trace 当恢复真相 | 采样/导出失败后任务不可恢复 | Event Store 与 Trace 分离 |
| 全量记录 prompt/tool 参数 | 源码与密钥泄漏 | 分级采集、源端脱敏、RBAC/TTL |
| runId 作为 metric label | 时序库基数爆炸 | Metric 低基数，明细放 Trace |
| 只测最终文本相似度 | 工具越权仍可能高分 | 轨迹、策略、环境结果多维评分 |
| 只跑平均分 | 安全长尾被掩盖 | 硬门槛 + 分标签分位数 |
| Judge 未版本化/校准 | 分数漂移不可比较 | judge config 固化，与人类金标对齐 |

## 11. 练习与验收

**练习**：为“代码修复 Agent”设计 30 个 Eval Case：10 个功能、5 个上下文不足、5 个危险命令、5 个取消/恢复、5 个成本边界。给出至少 3 个程序化评分器和 1 个 rubric judge，并定义 PR 阶段和 canary 阶段门槛。

**验收点**：

- [ ] 能从 Run Trace 定位 Context、模型、Policy、Tool、恢复各阶段耗时和错误。
- [ ] 指标标签受控，成功率、成本、恢复、背压和 unknown outcome 均可监控。
- [ ] 敏感正文默认不进入 Trace，调试采集有授权、TTL 和审计。
- [ ] Eval Result 固化 model/prompt/tool/workflow/runtime/judge 版本。
- [ ] 安全与副作用正确性是硬门槛，质量、成本、延迟分维度比较。

## 延伸阅读

- [OpenAI Agents SDK：Integrations and observability](https://developers.openai.com/api/docs/guides/agents/integrations-observability)
- [OpenTelemetry GenAI Semantic Conventions 官方仓库](https://github.com/open-telemetry/semantic-conventions-genai)
- [OpenTelemetry：Trace 规范](https://opentelemetry.io/docs/specs/otel/trace/)
- [OpenAI：Evaluate agent workflows](https://developers.openai.com/api/docs/guides/agent-evals)
- [Google SRE Book：Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
