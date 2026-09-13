# 一手资料清单：按问题阅读

官方文档说明产品用法，规范说明实现要求，论文解释设计原因。带版本的条目会注明版本或访问日期。

## 1. 三遍读法

### 第一遍：建立地图（20% 时间）

先读目录、术语、architecture/overview、版本/兼容和安全章节，写下一页摘要：

```markdown
对象：Run / Thread / Task / Tool / Resource
身份：由谁生成，生命周期多长
状态：终态、顺序、恢复句柄
边界：谁信任谁，授权在哪里
版本：wire/version/capability 如何协商
```

### 第二遍：沿关键路径（50% 时间）

沿“请求 Tool → 审批 → 执行 → timeout → 恢复”这条路径阅读。取最小 schema/代码，改写成自己的 canonical contract，并记录术语映射。

### 第三遍：逆向找失败（30% 时间）

专门查 cancellation、retry、error、security、limits、deprecation、unknown field、long-running 和 stream reconnect。每读一项，产出一个 contract test 或故障注入案例。

### 资料笔记模板

```yaml
source: "URL"
kind: "normative-spec | official-doc | source-code | paper"
verified_at: "YYYY-MM-DD"
version_or_commit: "exact version/commit, not latest"
applies_to: ["provider-adapter", "tool", "desktop"]
normative_claims:
  - "MUST/SHOULD or precise current behavior"
local_mapping:
  external_session: "providerSessionId, not domain runId"
tests_to_add:
  - "cancel during stream"
open_questions:
  - "resume retention window is not specified"
```

## 2. 推荐 6 周主线

| 周 | 主题 | 必读 | 必须产出 |
| --- | --- | --- | --- |
| 1 | 执行模型 | 本页 §3、§4 | Run/Turn/Attempt/Tool canonical types |
| 2 | 协议与 Tool | §5、§6 | MCP/A2A 映射、capability grant |
| 3 | 可靠状态 | §7 | event/reducer、effect crash matrix |
| 4 | 可观测/安全 | §8、§9 | span tree、threat model、DLP |
| 5 | Desktop | §10 | renderer→host→sandbox 数据流与更新模型 |
| 6 | Eval/运营 | §11、§12 | eval gate、SLO、upgrade runbook |

每天 60～90 分钟：40 分钟读原文，20 分钟做映射，20 分钟写测试。只阅读而不产出合约或测试，很难判断是否真正掌握。

### 2.1 Datawhale 两套路线

- [Hello-Agents](https://github.com/datawhalechina/hello-agents)：按“基础 → 经典范式 → 框架与自研 → 记忆/上下文/协议/评测 → 综合案例”学习。对应这里的章节映射见 [README](../../README.md#hello-agents)。
- [Agent-Learning-Hub](https://github.com/datawhalechina/Agent-Learning-Hub)：按 Stage 0～8 和 Project Ladder 学习，重点覆盖 Agent Harness、Skills、浏览器 Agent、评测安全与发布。对应这里的映射见 [README](../../README.md#agent-learning-hub)。

两套路线都要求完成可运行项目。本地练习优先复用 [参考项目](../07-practice/reference-project.md)，把外部示例转换成自己的事件、权限和验证合同。

## 3. OpenAI：Codex 与 Agents SDK

> 产品 API 会变化。实现前以官方页面或仓库为准，博客只用来寻找关键词。

### 3.1 必读

1. [Codex SDK](https://learn.chatgpt.com/docs/codex-sdk)
   - **重点**：TypeScript/Python 的 thread start/continue/resume、进程/运行时前提、sandbox preset。
   - **读法**：把 provider thread ID 映射为 opaque adapter state；不要让它替代平台 `runId`。
   - **产出**：Codex adapter contract，至少测 resume、cancel、sandbox、错误映射。
   - **页面记录**：2026-08-30 页面同时给出 TypeScript 与 Python 用法；实现时以锁定依赖和实际响应为准。

2. [Codex App Server](https://developers.openai.com/codex/app-server)
   - **重点**：富客户端的认证、Thread/Turn/Item、审批与事件协议；区分 stdio、Unix socket 与仍处实验状态的远程 WebSocket。
   - **读法**：App Server 是 Codex 集成面，不是企业 Task Control Plane；另行设计租户、预算、队列、fencing、审计保留和 DeliveryBundle。
   - **产出**：App Server event → Runtime event 的兼容层与未知事件测试。

3. [OpenAI Agents SDK TypeScript](https://openai.github.io/openai-agents-js/)
   - **重点**：Agent、tool、handoff/agent-as-tool、guardrail、session、tracing、sandbox。
   - **读法**：先读 Overview，再顺序读 [Agents](https://openai.github.io/openai-agents-js/guides/agents/)、[Running agents](https://openai.github.io/openai-agents-js/guides/running-agents/)、[Tools](https://openai.github.io/openai-agents-js/guides/tools/)、[Results](https://openai.github.io/openai-agents-js/guides/results/)。
   - **产出**：Framework primitive → 平台 primitive 映射；标出哪些能力只是 SDK convenience、哪些需 Runtime 持久化。

4. [OpenAI Agents SDK JavaScript 源码](https://github.com/openai/openai-agents-js)
   - **重点**：公开类型、stream event union、model interface、版本/changelog。
   - **读法**：文档理解概念，源码/类型确认精确行为；把依赖锁到 commit/tag 并做 contract fixture。

### 3.2 第二层

- [Responses API：Streaming](https://platform.openai.com/docs/guides/streaming-responses)：读事件终态、增量、错误；产出 canonical stream mapper。
- [Codex non-interactive mode](https://developers.openai.com/codex/non-interactive-mode)：读 `codex exec`、JSONL、sandbox/approval 配置；产出受监督子进程 adapter，而不是用 stdout 猜状态。
- [Agent approvals & security](https://developers.openai.com/codex/agent-approvals-security)：读不同运行环境的沙箱、审批与网络边界；不要把当前默认配置写成不可变承诺。
- [Function calling](https://platform.openai.com/docs/guides/function-calling)：读 schema、call/result 关联；产出 Tool 参数验证测试。
- [Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)：读“结构合法”和“业务事实正确”的边界。
- [Background mode](https://developers.openai.com/api/docs/guides/background)：读异步 Response 的 polling/terminal/retention；产出 Provider attempt adapter，不能代替业务 Task。
- [Webhooks](https://developers.openai.com/api/docs/guides/webhooks)：读签名、快速 ACK、重试与重复事件；产出 inbox 去重、outbox/DLQ 和 reconcile 测试。
- [Compaction](https://developers.openai.com/api/docs/guides/compaction)：读 opaque compaction item 与延续语义；产出 Context 策略，禁止用它替代 Runtime state/evidence。
- [Production best practices](https://platform.openai.com/docs/guides/production-best-practices)：把速率、成本、密钥、监控建议映射到生产清单。

## 4. Anthropic：Claude Agent SDK

1. [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)
   - **重点**：SDK 自带 loop/context/tool 的边界、TypeScript/Python 包、认证方式。
   - **产出**：Claude adapter 的启动/消息/结果/权限映射。

2. [How the agent loop works](https://code.claude.com/docs/en/agent-sdk/agent-loop)
   - **重点**：主 loop、subagent 上下文、Tool result 如何继续。
   - **产出**：Provider attempt 与平台 Run/Step 的层级图。

3. [Permissions](https://code.claude.com/docs/en/agent-sdk/permissions)
   - **重点**：allowed/disallowed Tool、permission callback、模式。
   - **读法**：将其视为 adapter 层控制之一，不替代平台 resource capability 和 sandbox。
   - **产出**：参数篡改、deny、approval expiry contract test。

4. [Streaming output](https://code.claude.com/docs/en/agent-sdk/streaming-output)
   - **重点**：message/event 分类、partial、result、errors。
   - **产出**：显式 terminal、取消、慢消费者 fixture。

5. [Hosting the Agent SDK](https://code.claude.com/docs/en/agent-sdk/hosting)
   - **重点**：长进程、容器/sandbox、资源和部署模式。
   - **产出**：本地/云 hosting ADR，不把“容器”当完整隔离答案。

6. [Claude Code features in the SDK](https://code.claude.com/docs/en/agent-sdk/claude-code-features)
   - **重点**：CLAUDE.md、skills、subagents、hooks、MCP 的不同作用。
   - **产出**：instruction/tool/hook/policy 的边界表。

## 5. Tool 与 Agent 互操作协议

### 5.1 MCP

1. [MCP Specification — latest](https://modelcontextprotocol.io/specification/latest)
   - **性质**：规范入口；`latest` 会跳转到日期版本。
   - **版本**：2026-08-30 访问时跳转到 `2026-07-28`；实现时记录实际版本，不只写 `latest`。
   - **第一遍**：[Architecture（2026-07-28）](https://modelcontextprotocol.io/specification/2026-07-28/architecture)、[Versioning（2026-07-28）](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning)。
   - **版本差异**：2026-07-28 版本把版本/身份/capability 放在每个请求的 `_meta`，并以 `server/discover` 做可选预发现；`initialize` 属于旧版兼容路径。采用其他版本时，以该版本规范为准。
   - **第二遍**：Base Protocol/Transport、Resources/Prompts/Tools、Cancellation/Progress/Error。
   - **第三遍**：Authorization 与 Security；特别验证 Tool description/annotation 不能被 host 当可信授权。
   - **产出**：Host/Client/Server、capability negotiation、Tool approval 威胁模型。

2. [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
   - **重点**：schema 与 transport 的实际实现；tag/commit 要与规范版本对应。
   - **产出**：协议 golden message，覆盖现代 `server/discover`、每请求 version/capability metadata、`tools/list`、`tools/call`、cancel/error；仅在支持旧规范时另测 `initialize` fallback。

### 5.2 A2A

1. [A2A Protocol Specification — latest](https://a2a-protocol.org/latest/specification/)
   - **重点**：Task/Message/Artifact、stream/push、idempotency、版本、Agent Card、认证授权。
   - **读法**：先读 “Life of a Task”，再画 A2A Task → 平台 Run 的映射；不要直接等同。
   - **产出**：Task terminal/interrupt/input-required 状态映射、恢复与鉴权测试。

2. [A2A and MCP](https://a2a-protocol.org/latest/topics/a2a-and-mcp/)
   - **重点**：Agent 协作与 Tool/context 连接的职责差异。
   - **产出**：说明什么时候用 MCP、什么时候用 A2A、什么时候只需内部 API。

### 5.3 底层规范

- [JSON-RPC 2.0](https://www.jsonrpc.org/specification)：request/notification/id/error；产出 notification 无响应和 ID 关联测试。
- [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12/json-schema-core)：dialect、vocabulary、reference；再读 validation spec。产出 Tool schema 安全限制。
- [RFC 8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785)：参数 digest/签名前 canonicalization。
- [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457)：HTTP 错误 envelope 的参考；不要把 HTTP code 当完整 Runtime error。
- [RFC 9110 HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)：安全/幂等 method、retry/缓存语义。
- [CloudEvents Specification](https://github.com/cloudevents/spec)：事件 envelope/transport binding 的参考；领域顺序/恢复仍由平台定义。

## 6. Wire、身份与安全协议

- [W3C Trace Context](https://www.w3.org/TR/trace-context/)：`traceparent`/`tracestate`、传播与安全；产出跨 IPC/queue/provider span correlation。
- [OAuth 2.0 Security Best Current Practice, RFC 9700](https://www.rfc-editor.org/rfc/rfc9700)：OAuth 威胁与当前推荐。
- [OAuth 2.0 for Native Apps, RFC 8252](https://www.rfc-editor.org/rfc/rfc8252)：桌面/native 登录与外部 user-agent。
- [PKCE, RFC 7636](https://www.rfc-editor.org/rfc/rfc7636)：native 授权码拦截防护。
- [JWT BCP, RFC 8725](https://www.rfc-editor.org/rfc/rfc8725)：算法混淆、audience/issuer 等。
- [UUID, RFC 9562](https://www.rfc-editor.org/rfc/rfc9562)：ID 格式；不要在 ID 中编码 tenant/PII。
- [BCP 14: RFC 2119 + RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)：准确理解规范中的 MUST/SHOULD/MAY。

读法：每个认证/授权协议画“主体—客户端—授权服务器—资源服务器—token audience/scope”，再做 token 重放/错 audience/过期/撤销测试。

## 7. 分布式状态与可靠执行

### 7.1 必读论文/规范

1. Lamport, [Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)
   - **读法**：聚焦 happens-before 与“墙上时钟不能提供业务顺序”。
   - **产出**：为什么事件用 Run 内 `seq`，trace 用因果 link。

2. Saltzer, Reed, Clark, [End-to-End Arguments in System Design](https://web.mit.edu/Saltzer/www/publications/endtoend/endtoend.pdf)
   - **产出**：为什么传输 exactly-once 不能自动保证付款 effect exactly-once。

3. Gray, [The Transaction Concept: Virtues and Limitations](https://jimgray.azurewebsites.net/papers/thetransactionconcept.pdf)
   - **产出**：事务边界、外部 effect 和恢复责任。

4. Garcia-Molina & Salem, [Sagas](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)
   - **产出**：长事务拆分与补偿；说明补偿不是 rollback。

5. Ongaro & Ousterhout, [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)
   - **读法**：理解 leader/term/log safety；不要因此自己实现共识。
   - **产出**：lease 为什么还需要 fencing/一致存储。

### 7.2 工程实现资料

- [Temporal Workflow Execution](https://docs.temporal.io/workflow-execution)：Workflow history/replay、Activity、retry、timer；产出“确定代码 vs Activity”边界。
- [PostgreSQL Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)：读 write skew/serialization retry；产出 event append CAS 事务测试。
- [SQLite WAL](https://www.sqlite.org/wal.html)：桌面本地持久化的并发/恢复限制；产出 crash/磁盘满测试。

阅读这类引擎文档时，不要把实现私有 history 直接暴露为产品事件协议。

## 8. 可观测性

1. [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/)
   - **顺序**：Overview → Trace API/SDK → Context propagation → Metrics → Logs → Versioning。
   - **产出**：`agent.run/model.generate/tool.invoke` span tree、采样和敏感字段规范。

2. [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
   - **重点**：稳定级别、schema、attribute cardinality、error 记录。
   - **注意**：GenAI semantic conventions 可能处于演进状态；固定所用版本并准备 mapping。

3. [W3C Trace Context](https://www.w3.org/TR/trace-context/)
   - **产出**：跨 Desktop IPC、HTTP、queue 的 propagator；明确 trace ID 不用于授权。

4. [Prometheus Metric and Label Naming](https://prometheus.io/docs/practices/naming/)
   - **产出**：拒绝 runId/userId label 的 review rule。

## 9. AI 安全、零信任与供应链

1. [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/)
   - **读法**：每项映射到资产/边界/控制/测试；不要把它当完整 threat model。
   - **产出**：prompt injection、insecure output、excessive agency、supply chain 红队集。

2. [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
   - **重点**：Govern/Map/Measure/Manage；产出风险登记、owner、证据和复核节奏。

3. [NIST SP 800-207 Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
   - **产出**：设备/用户/workload 身份、策略决策点与执行点；避免“内网可信”。

4. [SLSA Specification](https://slsa.dev/spec/)
   - **产出**：Desktop/插件/Runtime 构建 provenance、依赖与发布门禁。

5. [Sigstore Documentation](https://docs.sigstore.dev/)
   - **产出**：artifact/插件签名验证、信任根与撤销设计。

6. [CNCF TAG Security — Secure Software Factory](https://tag-security.cncf.io/)
   - **读法**：按供应链威胁选择相关项目文档，不盲目套全套工具。

## 10. Windows/macOS Desktop 一手资料

### 10.1 Electron/Tauri

- [Electron Process Model](https://www.electronjs.org/docs/latest/tutorial/process-model)：renderer/preload/main 职责；产出类型化 IPC 边界。
- [Electron Security Checklist](https://www.electronjs.org/docs/latest/tutorial/security)：逐条映射 `contextIsolation`、navigation、permission、IPC；产出自动/人工测试。
- [Electron Context Isolation](https://www.electronjs.org/docs/latest/tutorial/context-isolation)：不要暴露通用 `send`。
- [Electron Updates](https://www.electronjs.org/docs/latest/tutorial/updates)：更新源、签名、回滚/分批仍需产品补充。
- [Tauri Security](https://v2.tauri.app/security/)：若选 Tauri，读 capability/permission/scope，而不是用“Rust 更安全”代替威胁模型。

### 10.2 macOS

- [Apple App Sandbox](https://developer.apple.com/documentation/security/app-sandbox)：entitlement、容器与用户选择文件。
- [Hardened Runtime](https://developer.apple.com/documentation/security/hardened-runtime)：签名能力与运行限制。
- [Notarizing macOS software before distribution](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)：签名/notarization 发布链。
- [Keychain Services](https://developer.apple.com/documentation/security/keychain-services)：凭据存储；仍需 access control/rotation。

### 10.3 Windows

- [AppContainer isolation](https://learn.microsoft.com/en-us/windows/win32/secauthz/appcontainer-isolation)：进程/token/capability 边界。
- [Windows Sandbox](https://learn.microsoft.com/en-us/windows/security/application-security/application-isolation/windows-sandbox/)：理解它与产品内受限 worker 的差异。
- [MSIX documentation](https://learn.microsoft.com/en-us/windows/msix/)：打包、签名、更新、企业分发。
- [Windows Credential Locker](https://learn.microsoft.com/en-us/windows/apps/develop/security/credential-locker)：本机 credential reference。

### 10.4 运行时基础

- [Node.js AbortController](https://nodejs.org/api/globals.html#class-abortcontroller)：取消传播。
- [Node.js Streams](https://nodejs.org/api/stream.html)：背压与 pipeline。
- [Node.js Child Process](https://nodejs.org/api/child_process.html)：stdio、signal、shell 风险；子进程不等于 sandbox。

## 11. Eval 与质量

- [OpenAI Evals design guide](https://platform.openai.com/docs/guides/evals)：数据集、grader、run；产出按风险分层的发布 gate。
- [OpenAI Evaluation best practices](https://platform.openai.com/docs/guides/evaluation-best-practices)：读 task-specific、迭代、人工校准。
- [Anthropic Evaluation Tool docs](https://docs.anthropic.com/en/docs/test-and-evaluate/eval-tool)：比较不同 Provider 的 eval 结构，保留平台 canonical result。
- [HELM paper](https://arxiv.org/abs/2211.09110)：读多指标/场景/透明性，不直接把公开 benchmark 当产品 eval。
- [Model Cards paper](https://arxiv.org/abs/1810.03993)：产出内部 model/provider capability 与限制卡。

阅读要求：每个质量指标都要定义 sample unit、grader、阈值、重复次数、置信区间、阻断/告警行为。模型裁判必须有人工校准集。

## 12. 架构与运营书目

以下书籍不是实时 API 事实，适合构建长期思维；购买/借阅正版：

- Martin Kleppmann, *Designing Data-Intensive Applications*：先读可靠性、数据模型、复制、事务、流处理；映射 event store/stream。
- Michael Nygard, *Release It!*：稳定模式、生产就绪、故障放大；每章产出一个 chaos case。
- Burns, Oppenheimer, Brewer, *Site Reliability Engineering*：SLI/SLO、事故、发布；产出 error budget policy。
- Ross Anderson, *Security Engineering*：安全边界、激励、真实攻击；用资产/对手/控制而非 feature checklist。
- Sam Newman, *Building Microservices*：边界、演进、迁移；不要因此默认拆微服务。

## 13. 如何识别“该信哪一层资料”

| 问题 | 首选来源 | 不足时 |
| --- | --- | --- |
| 当前 SDK 方法/参数 | 官方 API docs + 源码/tag | contract test |
| 协议 MUST/兼容 | 日期固定的规范 | 官方 schema/reference implementation |
| 安全推荐 | BCP/NIST/OWASP + SDK 官方安全页 | threat model/红队 |
| 性能/限制 | 官方 limit + 自己 benchmark | 生产 telemetry |
| 架构取舍 | 论文/书 + 本地约束 | ADR/spike |
| 模型质量 | 自己的版本化 eval | 官方 model card 仅做背景 |
| 价格/可用区域 | 官方实时页面/账户控制台 | 不从旧笔记推断 |

二手文章可以帮助发现关键词，但 API、限制、定价和权限语义应以一手资料为准。

## 14. 阅读验收

- [ ] 每类资料至少有一份“外部术语 → 本地领域术语”映射。
- [ ] Codex/Claude adapter 各有 cancel/resume/error/tool contract test。
- [ ] MCP 固定规范版本，并覆盖 initialization/tool/security 测试。
- [ ] 阅读至少三篇可靠性论文，并能用于一个 ADR，而不是背结论。
- [ ] Desktop 产出 renderer→host→sandbox 威胁模型。
- [ ] OTel 产出 span/metric/redaction 规范，domain event 明确分离。
- [ ] 为会变化的事实保留一手来源、实际版本或访问日期，并在 Contract Test 或实验记录中注明适用范围。
