# 12 周学习计划：以可交付成果驱动

## 计划原则

每周建议投入 12～16 小时，比例保持在：原理 25%、编码 45%、失败实验 20%、ADR/复盘 10%。如果时间只有一半，减少阅读面，不删除代码、测试和失败注入。

贯穿式项目统一使用 TypeScript：一个 `runtime-core` 包、一个 Electron 壳、两个 Provider Adapter、一个本地 SQLite Event Store、三种 Tool 和一套 Trace/Eval。详细规格见 [参考项目](../../docs/07-practice/reference-project.md)。

## Phase 1：建立正确的执行模型（第 1～3 周）

### 第 1 周：模型 I/O 与统一事件

**学习**：token、context window、采样、streaming、tool call、structured output、usage、finish reason、错误分类。

**实现**：定义不依赖 Provider 的事件联合类型；写一个 Fake Provider，用异步迭代器依次发送文本增量、工具参数增量、usage 和 completed。

```ts
type RuntimeEvent =
  | { type: 'text.delta'; runId: string; text: string }
  | { type: 'tool.requested'; runId: string; callId: string; name: string; input: unknown }
  | { type: 'usage.reported'; runId: string; inputTokens: number; outputTokens: number }
  | { type: 'run.failed'; runId: string; error: RuntimeError }
  | { type: 'run.completed'; runId: string };
```

**故障实验**：中途断流、重复 delta、未知 finish reason、无效 JSON、用户取消。

**产物**：`provider-contract.md`、事件类型、Fake Provider、至少 12 个 Contract Tests。

### 第 2 周：Agent Loop 与状态机

**学习**：先读[执行契约、规划与验证](../../docs/03-runtime/execution-contract-verification.md)，再学习 Observe/Decide/Act、TaskSpec、AcceptanceCriteria、Plan DAG、终止条件、step budget、循环检测、确定性边界、human-in-the-loop。

**实现**：Reducer 驱动状态机；每个 transition 产生事件，外部副作用由 effect runner 执行。

**故障实验**：模型连续请求同一工具、工具永久失败、审批超时、进程在工具返回后但事件写入前崩溃。

**产物**：TaskSpec、版本化 Plan、状态图、纯 reducer、effect executor、Verifier、Claim-Evidence Map 与回放测试。

### 第 3 周：Context Engineering

**学习**：消息规范化、token 预算、优先级、裁剪、摘要、RAG、cache、context poisoning。

**实现**：`ContextAssembler` 接收 system policy、task、recent turns、tool results、retrieval chunks，并输出带 provenance 和 token estimate 的 context plan。

**故障实验**：超长工具输出、检索文档含恶意指令、摘要遗漏关键约束、不同模型 token 估算偏差。

**产物**：Context 策略、30 条 golden cases、质量/成本曲线。

## Phase 2：把 Demo 变成 Runtime（第 4～6 周）

### 第 4 周：Tool Runtime

**学习**：JSON Schema、capability、policy、approval binding + reauthorize、idempotency、timeout、sandbox、audit、result redaction，以及[能力供应链](../../docs/06-platform/capability-supply-chain.md)的 digest/signature/SBOM/revocation。

**实现**：只读文件、写文件、HTTP 三个工具；写操作必须审批；所有工具带风险等级和幂等键。

**故障实验**：路径穿越、SSRF、符号链接逃逸、重复写入、巨量输出、取消后继续运行。

**产物**：Tool Manifest、Policy Engine、审批协议、安全测试。

### 第 5 周：Durable Workflow 与 Event Store

**学习**：event sourcing、snapshot、at-least-once、retry/backoff、compensation、workflow versioning、lease 与 fencing 的区别。逐段实现 [SQLite 持久化内核](../../docs/03-runtime/persistence-sqlite.md) 中的 schema、CAS append、outbox/inbox、备份与恢复协议。

**实现**：SQLite event store；每个 Run 从事件恢复；定时器和审批可跨进程重启。

**故障实验**：在每个持久化边界随机 kill 进程 100 次，确认不会丢状态或重复副作用。

**产物**：事件 schema、migration、恢复矩阵、chaos test 报告。

### 第 6 周：Streaming、取消与背压

**学习**：AsyncIterable、AbortSignal、bounded queue、consumer backpressure、partial artifact、resume token。

**实现**：从 UI 到 Provider/Tool 的取消传播；慢 UI 不拖垮 Runtime；文本 delta 可合并但关键事件不可丢。

**故障实验**：每秒 500 个 delta、UI 卡顿 5 秒、Provider 无视取消、断网后重连。

**产物**：流控策略、内存曲线、P95 首 token/完成延迟指标。

## Phase 3：桌面端与统一 SDK（第 7～9 周）

### 第 7 周：AI Desktop 信任边界

**学习**：Electron Main/Renderer/preload、context isolation、sandbox、typed IPC、Sidecar、OS credential store，并完成 [Coding Agent 本地主机](../../docs/02-desktop/coding-agent-host.md) 的 PTY、worktree、patch transaction 与跨平台权限模型。

**实现**：Renderer 只能访问窄接口；Runtime 运行于 utility process 或 sidecar；文件访问经过 workspace root 校验。

**故障实验**：XSS 调用敏感 IPC、伪造 channel、Sidecar 崩溃、窗口关闭时仍有 Run。

**产物**：Threat Model、IPC allowlist、进程监督器、恢复 UX。

### 第 8 周：Codex/Claude Adapter

**学习**：各 Provider 的 session/thread、event、permission、tool、sandbox 等差异；区分官方稳定能力与示意适配层。

**实现**：两个 Adapter 通过同一 Contract Test；使用 capability negotiation 处理“不支持恢复/工具审批/结构化输出”等差异。

**故障实验**：Provider 升级字段、未知事件、rate limit、认证过期、部分 capability 缺失。

**产物**：差异矩阵、Adapter、兼容测试、降级策略。

**加练**：按 [Framework Adapter 实战](../../docs/05-frameworks/hands-on-adapters.md) 用同一套 contract suite 接入 OpenAI Agents SDK 与 LangGraph，验证 checkpoint claim、审批绑定和跨进程恢复。

### 第 9 周：Unified SDK 与多端

**学习**：ports/adapters、facade、transport、serialization、schema evolution、plugin lifecycle。

**实现**：同一 `AgentClient` 通过 in-process、WebSocket 两种 transport 工作；Electron 与浏览器 UI 共用 domain types。

**故障实验**：客户端/服务端版本错配、断线重连、事件乱序、未知扩展字段。

**产物**：SDK 包结构、API Extractor 报告、版本策略、Web 示例。

## Phase 4：平台化与生产验收（第 10～12 周）

### 第 10 周：Observability 与 Eval

**实现**：Run/Step/Model/Tool Span；token、cost、queue、error 指标；基于事件日志的离线回放；50 条 Eval 数据集。

**发布门禁**：任务成功率不下降超过 1%；高风险工具误调用为 0；P95 成本不增超过 10%；恢复成功率 100%。

### 第 11 周：控制面、多租户与安全

**实现**：按[任务控制平面](../../docs/06-platform/task-control-plane.md)完成可信 RequestEnvelope、digest 防重、budget reserve、公平调度与 lease fencing；按[执行环境生命周期](../../docs/06-platform/execution-environment-lifecycle.md)完成环境指纹、Setup/Agent 分权和清理证明；加入 tenant-scoped policy/model/tool catalog、短期令牌、审计查询、租户级配额和 kill switch；按 [企业模型网关](../../docs/06-platform/model-gateway.md) 实作 region/feature/policy 硬过滤、路由决策快照、quota reservation 与安全 fallback。

**演练**：跨租户 ID 猜测、旧 Worker 恢复写入、审批后参数替换、恶意 MCP Server、Prompt Injection、密钥轮换、环境清理失败和策略回滚。

### 第 12 周：架构评审与发布

**完成**：使用 [Agent Runtime 架构覆盖矩阵](../../docs/08-appendix/architecture-source-gap-analysis.md)做 20 分钟架构讲解；演示受限[多 Agent 编排](../../docs/03-runtime/multi-agent-orchestration.md)、10 分钟事故处置、5 个 ADR、运行手册、SLO、成本模型、未来两个季度演进路线。

请找同事扮演安全、SRE、产品和 SDK 使用者进行质询。如果系统只能在你讲解时成立、无法从代码与观测中自行证明，它还没有达到平台标准。

## 每周复盘模板

```md
# Week N Review
- 本周可演示能力：
- 新增稳定契约：
- 失败注入与结果：
- 最大的不确定性：
- 新增/修改 ADR：
- 下周要删除或简化的复杂度：
- 指标：tests / replay success / P95 latency / cost / eval score
```

## 通过标准

12 周结束时，不以代码行数判断，而检查：

- Provider 或 Framework 可被替换，核心 Runtime 不重写。
- 任意 Run 都能用事件和 Trace 解释。
- 崩溃后恢复不会悄悄重复外部副作用。
- 工具权限默认拒绝，审批与审计链完整。
- SDK 在至少两个 transport、两个 UI 环境工作。
- 一次变更必须经过 Contract Test、Replay、Eval 和 Canary 证据。

下一章：[Agent Runtime 30/60/90 天架构落地 →](../../docs/00-roadmap/first-90-days.md)
