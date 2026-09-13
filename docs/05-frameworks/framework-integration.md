# 框架集成：把 SDK 差异留在 Adapter

## 1. 集成原则

框架集成要保护领域层的稳定语义。领域层只认识任务、事件、工具意图、审批与 Artifact；adapter 处理框架特有的 message、stream、checkpoint 与异常。专有能力通过显式 capability 暴露，缺失时明确拒绝。

```text
Domain Agent / Workflow
        │  internal contracts
Runtime Port ── Policy/Identity/Budget/Telemetry
        │
Adapter: OpenAI | LangGraph | MAF | ADK
        │
Framework SDK + Model Provider + MCP/A2A
```

## 2. 内部契约设计

```ts
type Capability = "stream" | "durable_resume" | "handoff" | "tool_approval";
type RuntimeEvent =
  | { type: "model.started"; stepId: string; model: string }
  | { type: "tool.proposed"; stepId: string; callId: string; name: string; args: unknown }
  | { type: "approval.required"; callId: string; risk: string }
  | { type: "artifact.created"; artifactId: string; mediaType: string }
  | { type: "step.failed"; stepId: string; code: string; retryable: boolean };

interface FrameworkAdapter {
  readonly capabilities: ReadonlySet<Capability>;
  start(command: StartRun, signal: AbortSignal): AsyncIterable<RuntimeEvent>;
  resume(command: ResumeRun, signal: AbortSignal): AsyncIterable<RuntimeEvent>;
  cancel(runId: string): Promise<void>;
}
```

契约中不要放 `LangGraphCheckpoint`、`OpenAIResponseItem` 等 vendor type。原始 payload 可加密保存在诊断仓，业务事件只保存稳定字段和 `rawRef`。事件 schema 独立版本化；读端支持 N/N-1，写端仅写当前版。

## 3. 责任边界

| 能力 | 框架可提供 | 企业 Runtime 必须掌握 |
|---|---|---|
| Agent loop/图表达 | 是 | 最大轮次、deadline、取消传播 |
| session/checkpoint | 可用作实现 | 事实源、恢复协议、数据保留、迁移 |
| tool schema/call | 是 | registry、身份、策略、审批、幂等 |
| trace | 开发体验 | OTel、脱敏、审计、SLO、成本账本 |
| guardrail | 辅助 | fail-open/closed 策略与不可绕过门禁 |
| model retry | 常内置 | 全链路 retry budget、熔断、降级 |

例如 OpenAI Agents SDK 的 input guardrail 若并行执行可减少延迟，但模型可能已经花 token 或开始工具调用；高风险场景应在进入 SDK 前做同步策略门禁。LangGraph interrupt 能暂停图，但“谁有权批准、批准什么参数、多久过期”仍属于内部审批服务。

## 4. Adapter 的映射步骤

1. **输入规范化**：canonical message → provider content；附件转 ArtifactRef，校验租户 ACL。
2. **能力协商**：workflow 声明 requirements；adapter 缺能力就拒绝部署，不得假装支持。
3. **事件归一**：每个 model/tool/handoff/guardrail 生成稳定事件并传递 `traceparent`。
4. **副作用截获**：所有工具请求先进入统一 Tool Gateway；框架不得持有业务系统长期凭据。
5. **checkpoint 协调**：先持久化意图/outbox，再执行副作用；完成事件与恢复指针可重放。
6. **错误分类**：provider 429、schema error、policy deny、user cancel 不得都变成 `Error` 字符串。

```ts
async function* guardedRun(adapter: FrameworkAdapter, cmd: StartRun, signal: AbortSignal) {
  requireCaps(adapter, cmd.requiredCapabilities);
  for await (const e of adapter.start(cmd, signal)) {
    const safe = redactAndValidate(e);
    await eventStore.append(cmd.runId, safe); // expectedVersion 防并发写
    if (safe.type === "tool.proposed") {
      yield await toolGateway.authorizeOrRequestApproval(cmd.principal, safe);
    } else yield safe;
  }
}
```

## 5. Contract Test 与升级门禁

每个 adapter 共享一套黑盒用例：流式顺序、取消、超时、tool schema、重复事件、handoff、人工审批、crash/restart、trace 关联、敏感字段脱敏。再增加框架专属 golden tests。依赖升级用 canary workflow 跑固定数据集，比较事件序列而不只比较最终文本。

| 测试 | 不变量 |
|---|---|
| 进程在工具成功后崩溃 | 重启不重复副作用，最终事件唯一 |
| provider 流重复/乱序 | 由 sequence 去重，Artifact 完整 |
| 用户取消 | 模型、工具、子 Agent 均收到取消；run 终态稳定 |
| SDK breaking change | adapter 编译/契约测试失败，领域层不变 |
| trace 含 PII | 默认不记录正文；受权诊断才可短期解密 |

版本策略：固定精确 SDK/协议版本；维护升级日历和 CVE 响应；一次只升级一个 adapter；保留 N-1 镜像和回滚；checkpoint 迁移先 shadow-read，再切写。

## 6. 失败模式

- **万能 facade**：把所有特性压成 `chat()`，丢掉恢复、审批与取消语义。
- **漏网工具**：Agent-as-tool、handoff 或远程 MCP 绕过统一 Tool Gateway。所有执行边统一截获。
- **双重重试**：SDK、HTTP client、队列同时重试，形成指数风暴。全链路共享 retry budget。
- **无法取消**：只停 UI stream，后台仍执行昂贵或破坏性操作。AbortSignal 必须贯穿。
- **原始 trace 泛滥**：prompt/tool result 进入日志造成 PII 与 secret 泄露。默认元数据化、按字段白名单。

## 7. 练习与验收

关联 hexagonal architecture、Anti-Corruption Layer、event sourcing、transactional outbox、OpenTelemetry、consumer-driven contract testing，以及[控制面/数据面](../06-platform/control-data-plane.md)。

练习：实现一个 fake adapter 与一个真实框架 adapter，跑“模型 → 读工具 → 写工具审批 → 完成”的共享测试。模拟流中断、重复 tool call、SDK 字段变更与进程重启。

验收：领域层零 vendor import；替换 adapter 不改 workflow 业务代码；共享 contract tests 全绿；故障恢复无重复写；capability 缺失在发布期而非运行中暴露；升级可在 15 分钟内回滚。

## 官方资料

- [OpenAI Agents SDK：官方 API quickstart](https://developers.openai.com/api/docs/quickstart)
- [LangGraph persistence](https://langchain-ai.github.io/langgraph/concepts/persistence/)
- [Microsoft Agent Framework concepts](https://learn.microsoft.com/en-us/agent-framework/user-guide/)
- [Google ADK workflows](https://adk-labs.github.io/adk-docs/workflows/)
- [OpenTelemetry 规范](https://opentelemetry.io/docs/specs/)
