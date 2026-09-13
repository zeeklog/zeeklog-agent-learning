# AI 平台总体架构

## 1. 平台要解决什么

AI 平台让团队以受治理、可恢复、可观测、成本可归属的方式交付 Agent，并让 Windows/macOS、Web、浏览器扩展和 IDE 共用能力。平台提供执行合同和默认路径，不替业务决定所有实现。

平台目标：多模型可替换、工具统一接入、长任务可恢复、用户身份可委托、策略不可绕过、租户/数据隔离、端到端 SLO、离线/在线评测和可逆发布。非目标：替业务写 prompt、用 LLM 取代确定性规则、把所有数据集中到一个向量库。

## 2. 参考架构

```text
 Windows / macOS / Web / IDE SDK
          │ OIDC + device/workload proof
 Edge/API Gateway ── session/stream/cancel
          │
 ┌────────┴────────────── Data Plane ───────────────────┐
 │ Task Service ── Agent Runtime ── Workflow/Queue/Fence│
│      │                 │                             │
 │ Model Gateway      Environment/Tool ─ MCP/A2A/Sandbox│
│      │                 │                             │
 │ Context/RAG ─ Verifier/Delivery ─ Event/Artifact/Trace│
 └─────────────── policy enforcement points ────────────┘
          ▲ signed immutable release
 ┌────────┴───────────── Control Plane ─────────────────┐
 │ Agent/Prompt/Tool Registry │ Policy │ Eval │ Release │
 │ Tenant/Identity/Quota      │ Cost   │ Audit│ Portal  │
 └───────────────────────────────────────────────────────┘
```

| 组件 | 稳定责任 | 不应承担 |
|---|---|---|
| Agent Runtime | run/step/event、预算、取消、恢复、HITL | 业务数据真相 |
| Task Service | 请求防重、准入、队列、公平调度、lease/fencing、Task API | 把 Provider Response 当业务状态 |
| Model Gateway | provider adapter、路由、配额、降级、使用量 | 偷改 prompt 语义 |
| Tool Gateway | registry、鉴权、审批、幂等、网络策略 | 信任模型生成参数 |
| Environment Broker | 指纹、隔离、短期身份、工作卷、清理证明 | 把“容器”名字当完整安全边界 |
| Verifier/Delivery | 按 TaskSpec 产证据、diff gate、结果可信度 | 复述主 Agent 的完成声明 |
| Context 服务 | ACL-aware 检索、上下文预算、provenance | 把“记忆”当永久正确事实 |
| Control Plane | 版本、策略、评测、发布、租户治理 | 位于每个 token 的同步关键路径 |
| Telemetry | 跨端 trace、指标、成本与审计关联 | 默认保存全部 prompt/PII |

## 3. 一次请求如何流动

1. 客户端取得短时用户 token，提交 `agentReleaseId`、输入和 `idempotencyKey`；Task Service 将 key 绑定 canonical request digest，生成 `taskId/traceId` 并做准入与预算预留。
2. Runtime 读取已签名、不可变的 release snapshot；不能在运行中拼接“最新 prompt + 最新 tools”。
3. Context 服务按 `tenant/user/purpose` 执行 ACL-aware 检索，输出带来源与信任级别的片段。
4. Model Gateway 根据质量等级、区域、预算和健康度选模型，记录输入/缓存/输出/推理 token。
5. 工具意图进入 Tool Gateway；重新做 schema、身份、策略、审批和幂等检查。模型永远不是授权主体。
6. 每一步先写事件/意图，再执行外部副作用；Worker 推进携带 fencing token，流式结果只是一种视图，最终 Artifact 有独立生命周期。
7. `WAITING_APPROVAL/BLOCKED` 是耐久可恢复态而非终态；候选结果经过独立 Verifier 后生成带 Evidence 的 DeliveryBundle。账单、SLO、审计和评测用同一 correlation IDs 关联。

## 4. 稳定资源模型

```ts
interface AgentRelease {
  id: string; tenantId: string; version: string; digest: string;
  promptRef: { id: string; version: string };
  workflowRef: { id: string; version: string };
  toolGrants: { toolId: string; schemaHash: string; scopes: string[] }[];
  modelPolicyRef: string; guardrailPolicyRef: string;
  budgets: { turns: number; tokens: number; usd: number; wallMs: number };
  evalGate: { dataset: string; minPassRate: number };
}

interface RunContext {
  tenantId: string; userId: string; agentReleaseId: string;
  taskId: string; runId: string; attempt: number;
  traceId: string; dataZone: string; purpose: string;
}
```

`AgentRelease` 必须内容寻址或带 digest，发布后不可变；修改任何 prompt、tool schema、模型策略都产生新版本。`RunContext` 从已验证身份和服务端配置构造，不能从用户 prompt 或模型输出生成。

## 5. 关键架构权衡

- **同步 vs 异步**：首 token 可同步流式；超过 30~60 秒、含审批或外部副作用的任务进入耐久队列，并允许客户端重连。
- **中央 vs 联邦**：身份、策略、审计、协议和成本字段中央统一；prompt、workflow、领域工具由业务域拥有。
- **共享 vs 隔离**：无状态网关可共享；执行沙箱、向量索引、密钥和高敏队列按风险分级隔离。
- **平台透明路由 vs 可解释性**：路由可动态，但每次 run 必须固定实际模型/版本，且输出能追溯路由理由。
- **日志完整 vs 隐私**：默认记录摘要、hash、token 与决策；正文按数据分级、授权和短期保留处理。

## 6. 失败模式

- **平台巨石**：所有能力同步串联，任一配置服务故障阻断推理。用 release snapshot、本地缓存和降级策略隔离控制面。
- **统一接口丢语义**：将流式、tool call、reasoning、Artifact 都压成字符串。内部事件必须保留结构化语义。
- **策略只在入口**：handoff、MCP 或重试绕过门禁。每个副作用边界重新鉴权。
- **客户端持有供应商 key**：无法撤权、审计和成本归属。客户端只持平台短时 token。
- **在线效果不可复现**：prompt/tool/model 漂移。每个 run 绑定完整 release digest。

## 7. 练习与验收

继续阅读[任务控制平面](task-control-plane.md)、[控制面/数据面](control-data-plane.md)、[执行环境生命周期](execution-environment-lifecycle.md)、[能力供应链](capability-supply-chain.md)、[多租户治理](multitenancy-governance.md)和[可靠性与成本](reliability-cost.md)。关联 DDD、事件溯源、零信任、工作流引擎、API gateway、OTel 与平台工程。

练习：为“研发助手”画 C4 Container 图与一次写工具的 sequence diagram，定义上述资源的 JSON Schema，并列出每条边的身份、数据分级、超时、重试和 owner。

验收：架构图能回答“谁授权、谁持久化、谁计费、谁恢复、谁撤权”；任一 provider/MCP server/控制面短暂故障都有确定行为；从 `runId` 可在 10 分钟内完成一次事故审计。

## 官方资料

- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [OpenTelemetry GenAI Semantic Conventions](https://github.com/open-telemetry/semantic-conventions-genai)
- [Kubernetes 控制面概念](https://kubernetes.io/docs/concepts/overview/components/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)
- [MCP 规范](https://modelcontextprotocol.io/specification/)
