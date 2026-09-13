# Build vs Buy：框架、托管平台与自研的决策方法

## 1. 先定义选型问题

企业选型通常有四条路线：直接调用模型 API、自托管开源框架、使用云厂商托管 Agent 服务、自研 Runtime。它们可以组合使用。常见做法是：**购买模型与基础托管能力，采用开源编排表达，自研必要的企业策略与可移植边界**。

```text
低差异化：模型推理、向量库、基础队列             倾向 Buy
中差异化：编排 DSL、连接器、评测 UI              Adopt + Adapter
高差异化：身份委托、审批、策略、审计、领域状态     Build
危险自研：通用模型客户端、又一套图引擎、又一套 OAuth  Avoid
```

## 2. 加权决策矩阵

先让安全、平台、业务、财务共同确定权重，再评分 1~5。禁止先选产品再调整权重。

| 维度 | 建议权重 | 必问证据 |
|---|---:|---|
| 功能适配与可恢复执行 | 18% | crash 后从哪个边界恢复？副作用会否重复？ |
| 安全/合规/数据边界 | 18% | token 是否透传？训练/保留策略？区域与密钥归属？ |
| 可观测与可评测 | 12% | 是否能导出原始事件、OTel、token/成本和评测数据？ |
| 可移植与退出成本 | 12% | 模型、状态、工具协议、prompt 是否可导出？ |
| 团队技能与开发速度 | 10% | 真实功能从零到灰度需多久，而非 demo 多快？ |
| 性能、容量与 SLO | 10% | 并发、限流、冷启动、长任务、灾备数据？ |
| 三年 TCO | 12% | 牌照、推理、平台人力、迁移、值班、机会成本 |
| 生态成熟度 | 8% | 发布节奏、兼容政策、issue 响应、依赖风险 |

总分只是讨论起点。任何红线（无法满足数据驻留、不能审计高风险工具、无退出路径）都应直接淘汰，而不是被平均分掩盖。

## 3. 计算真实 TCO

```ts
type Candidate = {
  license: number; inference: number; infra: number;
  engineeringFte: number; opsFte: number;
  migrationProbability: number; migrationCost: number;
  outageHours: number; outageCostPerHour: number;
};

const threeYearTco = (x: Candidate, loadedFteCost: number) =>
  3 * (x.license + x.inference + x.infra) +
  3 * (x.engineeringFte + x.opsFte) * loadedFteCost +
  x.migrationProbability * x.migrationCost +
  x.outageHours * x.outageCostPerHour;
```

还要单列“不确定性区间”：模型调用量可能是基线的 0.5~4 倍；多 Agent 会放大 token 与并发；托管服务的出站费、私网、日志保留和高级安全常不在标价中。做 P50/P90 情景，而非单点数字。

## 4. 何时 Build，何时 Buy

### 值得自研的最小内核

- 稳定的 `Run/Step/Event/Artifact/Approval` 领域模型和版本迁移；
- 统一身份委托、策略决策点（PDP）、工具网关和副作用审批；
- 多供应商模型路由、预算、降级与使用量账本；
- 与业务 SLO 对齐的事件日志、评测和审计导出；
- 框架 adapter 及 contract test。

### 通常不要自研

- 通用 OAuth 库、遥测采集器、消息队列、密钥系统；
- 没有业务差异化的向量数据库、图编辑器或“万能 Agent 框架”；
- 自己解析各厂商流式协议而不依赖官方 SDK；
- 以 prompt 代替成熟工作流/BPM 的确定性流程。

托管平台适合：上线时间极短、云栈单一、数据边界明确、团队不想值守 Runtime。开源自托管适合：需深度扩展、部署边界复杂、具备平台运维能力。自研适合：策略/审计/隔离构成核心竞争力，且有至少 3~5 人长期 owner，而不是一次性项目组。

## 5. 可逆性设计

| 锁定面 | 反锁定控制 |
|---|---|
| 模型消息格式 | 内部 canonical message + provider adapter |
| 框架 checkpoint | 内部事件日志；checkpoint 视为派生缓存 |
| 私有工具协议 | MCP/JSON Schema 边界 + contract tests |
| 私有 trace | OTel 导出 + 自有 correlation IDs |
| 托管记忆/RAG | 可导出原文、ACL、embedding 版本与索引配置 |
| 平台 prompt | Git 版本、schema、eval 与发布清单 |

迁移演练比迁移文档更可信：每季度挑一个低风险 workflow，在备用 provider/framework 跑回归集，记录达到等价质量所需的代码改动和人日。

## 6. PoC 应如何设计

用 2~3 个候选，在 2~4 周内完成同一“黄金路径 + 失败路径”：高并发读工具、需审批的写工具、长任务恢复、RAG ACL、流式取消、模型降级。数据必须含歧义输入、越权请求、prompt injection、429/5xx、重复投递和 schema 演进。

决策记录示例：

```ts
interface DecisionEvidence {
  commit: string; datasetVersion: string; frameworkVersion: string;
  passRate: number; p95Ms: number; costPerSuccess: number;
  duplicateSideEffects: number; recoveryRate: number;
  securityFindings: { severity: string; control: string }[];
}
```

## 7. 失败模式

- **星数驱动选型**：社区热度不等于兼容承诺和运维成熟度。
- **PoC 只测快乐路径**：无法暴露 checkpoint、幂等和限流问题。
- **沉没成本绑架**：已有半年投入不代表未来三年应继续；比较边际成本。
- **过度抽象**：为所有框架求最低公分母，反而失去强项。只统一稳定语义，专有能力经 capability flag 暴露。
- **影子平台**：各业务自行接模型/工具，最后无法统一撤权、审计和成本归属。

## 8. 练习与验收

关联：[框架全景](landscape.md)、[多租户治理](../06-platform/multitenancy-governance.md)、[可靠性与成本](../06-platform/reliability-cost.md)。

练习：为所在企业写一份 ADR，选择两个框架和一个托管方案；给出权重、原始证据、三年 P50/P90 TCO、红线、退出计划和 owner。

验收：评审者能从仓库复现评分；至少包含 10 个故障注入用例；备用方案能在一天内运行同一回归集；任何“必须自研”项都能映射到明确的企业差异化或合规控制。

## 官方资料

- [Microsoft Agent Framework 概览](https://learn.microsoft.com/en-us/agent-framework/overview/)
- [Google ADK：Agents 与 Workflows](https://adk-labs.github.io/adk-docs/agents/)
- [LangGraph durable execution](https://langchain-ai.github.io/langgraph/concepts/durable_execution/)
- [OpenAI API 数据使用策略](https://developers.openai.com/api/docs/guides/your-data)
- [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework)
