- [首页](README.md)
- [Agent Runtime 阅读指南](guide.md)

- **00 · Agent Runtime 进阶路线**
  - [Agent Runtime 架构能力地图](docs/00-roadmap/capability-map.md)
  - [12 周学习计划](docs/00-roadmap/12-week-plan.md)
  - [30/60/90 天架构落地](docs/00-roadmap/first-90-days.md)
  - [Agent Runtime 架构自测与验收](docs/00-roadmap/assessment.md)

- **01 · 必备底层知识**
  - [LLM 系统原理](docs/01-foundations/llm-systems.md)
  - [模型接口、流式与结构化输出](docs/01-foundations/model-io.md)
  - [分布式系统语义](docs/01-foundations/distributed-systems.md)
  - [检索、上下文与评测基础](docs/01-foundations/retrieval-evaluation.md)

- **02 · AI 桌面客户端**
  - [总体架构与技术选型](docs/02-desktop/overview.md)
  - [进程模型与 Typed IPC](docs/02-desktop/process-ipc.md)
  - [Coding Agent 本地主机](docs/02-desktop/coding-agent-host.md)
  - [安全、权限与密钥](docs/02-desktop/security.md)
  - [Codex / Claude Provider 适配](docs/02-desktop/provider-adapters.md)
  - [打包、签名、更新与恢复](docs/02-desktop/distribution.md)

- **03 · Agent Runtime 核心**
  - [Runtime 总体架构](docs/03-runtime/overview.md)
  - [执行契约、规划与验证](docs/03-runtime/execution-contract-verification.md)
  - [Agent Loop 与状态机](docs/03-runtime/agent-loop.md)
  - [Durable Workflow](docs/03-runtime/workflow-engine.md)
  - [SQLite 持久化内核](docs/03-runtime/persistence-sqlite.md)
  - [Context Engineering](docs/03-runtime/context-engineering.md)
  - [Tool System](docs/03-runtime/tool-system.md)
  - [多 Agent 编排](docs/03-runtime/multi-agent-orchestration.md)
  - [Memory 与 State](docs/03-runtime/memory-state.md)
  - [流式、取消与恢复](docs/03-runtime/streaming-recovery.md)
  - [可观测性与 Evals](docs/03-runtime/observability-evals.md)

- **04 · 统一 AI SDK**
  - [SDK 分层与稳定边界](docs/04-sdk/overview.md)
  - [契约、事件与生命周期](docs/04-sdk/contracts-events.md)
  - [Provider 抽象与能力协商](docs/04-sdk/provider-abstraction.md)
  - [Transport、插件与多端](docs/04-sdk/transport-extensions.md)
  - [兼容性与契约测试](docs/04-sdk/compatibility-testing.md)

- **05 · Framework 与协议**
  - [主流 Framework 全景](docs/05-frameworks/landscape.md)
  - [Build vs Buy 决策](docs/05-frameworks/build-vs-buy.md)
  - [MCP 与 A2A](docs/05-frameworks/mcp-a2a.md)
  - [Framework 集成边界](docs/05-frameworks/framework-integration.md)
  - [Framework Adapter 实战](docs/05-frameworks/hands-on-adapters.md)

- **06 · AI 平台**
  - [平台总体架构](docs/06-platform/overview.md)
  - [任务控制平面](docs/06-platform/task-control-plane.md)
  - [控制面与数据面](docs/06-platform/control-data-plane.md)
  - [模型网关](docs/06-platform/model-gateway.md)
  - [企业级 AI Gateway 技术架构方案](docs/06-platform/enterprise-ai-gateway.md)
  - [执行环境生命周期](docs/06-platform/execution-environment-lifecycle.md)
  - [Tool / Skill / Agent 能力供应链](docs/06-platform/capability-supply-chain.md)
  - [多租户与治理](docs/06-platform/multitenancy-governance.md)
  - [可靠性、容量与成本](docs/06-platform/reliability-cost.md)
  - [安全威胁模型](docs/06-platform/security-threat-model.md)
  - [交付与持续演进](docs/06-platform/delivery-evolution.md)

- **07 · 架构与实战**
  - [端到端参考架构](docs/07-practice/reference-architecture.md)
  - [参考项目](docs/07-practice/reference-project.md)
  - [可运行 Runtime Kernel](labs/runtime-kernel/README.md)
  - [架构 Katas](docs/07-practice/architecture-katas.md)
  - [生产检查清单](docs/07-practice/production-checklists.md)
  - [ADR 模板与示例](docs/07-practice/adr-template.md)

- **08 · 附录**
  - [Agent Runtime 架构覆盖矩阵](docs/08-appendix/architecture-source-gap-analysis.md)
  - [核心术语表](docs/08-appendix/glossary.md)
  - [架构速查表](docs/08-appendix/cheatsheet.md)
  - [官方资料索引](docs/08-appendix/reading-list.md)
