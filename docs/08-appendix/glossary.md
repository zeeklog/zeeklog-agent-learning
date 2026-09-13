# 精确术语表：先统一语言，再讨论架构

> AI 平台最昂贵的误会往往来自同词不同义。以下术语作为 Runtime 内部的规范语言；接入外部 SDK 时可以保留原名，但必须映射到明确的领域概念。

## 1. 执行主体与平台

| 术语 | 精确定义 | 不等于 | 判断问题 |
| --- | --- | --- | --- |
| **Model（模型）** | 给定输入产生概率性输出的推理能力，可能支持 Tool/结构化/多模态 | Agent、Provider、Runtime | 它自己是否持久状态、执行 Tool、处理恢复？通常否 |
| **Provider（供应商）** | 托管/交付模型 API 的组织或运行环境 | Model | 同一模型可由不同 Provider 提供；同一 Provider 有多模型 |
| **Agent** | 被赋予目标、指令、模型和能力，并通过 loop 决定下一动作的执行主体 | 一次模型调用、Workflow | 是否会基于观察继续选择动作？ |
| **Agent Loop** | observe→decide→act→observe，直到完成/等待/预算耗尽 | Durable Workflow | loop 可在单进程内；未必能跨天恢复 |
| **Runtime** | 为 Task/Run 提供状态机、上下文、Tool、持久化、恢复、权限、验证和观测的运行平台 | Agent Framework | 换框架后仍应保留 Runtime 的产品契约 |
| **Framework** | 帮助定义/执行 Agent、graph、handoff 的库或引擎 | 企业平台 | 通常不拥有租户、审计、桌面权限、长期兼容 |
| **Orchestrator** | 协调多个步骤/Agent/Activity 的组件 | Agent 本身 | 谁决定先后、并发、等待和恢复？ |
| **Workflow** | 版本化的长时业务编排，强调确定步骤、等待、重试和恢复 | prompt 中的计划 | 能否在进程重启后从历史继续？ |
| **Activity** | Workflow 调用的非确定性/外部 I/O 单元，可重试且需幂等 | Workflow reducer | Provider/Tool/本地设备操作通常是 Activity |
| **Platform** | 供多产品复用的 Runtime、SDK、治理、运营和控制面集合 | 单个服务 | 是否提供组织级能力、SLO、生命周期和自助接入？ |

**易错例**：“我们用了 Agents SDK，所以已经有企业 Runtime。”SDK 可能提供 agent loop 与 trace，但租户策略、事件真相、桌面 sandbox、升级兼容仍需平台设计。

## 2. 身份与执行层级

| 术语 | 生命周期/身份 | 与相邻概念的差异 |
| --- | --- | --- |
| **Conversation / Thread** | 用户可见的长期交互容器，可包含多个 Task/Run/Turn | 不代表正在执行；可以跨设备、跨模型 |
| **Session** | 有上下文的连续使用/连接区间；不同产品含义差异大 | 不要直接把第三方 `session_id` 当领域 Thread |
| **Task** | 平台向调用方作出的一项耐久业务执行承诺；可排队、阻塞、恢复、验证和交付 | HTTP 请求、Provider Response、对话 Thread |
| **Run** | 推进同一 Task 的一次有身份执行尝试；失败后可由新 Run/Attempt 接手 | 业务 Task 本身；外部 SDK 同名对象需适配 |
| **Turn** | 一次用户输入到 Agent 交还控制/等待输入的交互单元 | 一个 Run 可含多个 Turn；SDK 定义可能不同 |
| **Step** | Workflow/Agent 内一个逻辑推进单元 | 是否可重试取决于定义；不能替代 Attempt |
| **Attempt** | 同一逻辑 operation 的一次实际尝试 | 重试产生新 attempt，但沿用 operation/idempotency 身份 |
| **外部协议 Task** | A2A、队列或厂商协议定义的 Task | 必须映射为 `a2aTaskId/providerTaskId` 等命名空间，不能覆盖平台 `taskId` |
| **Job** | 通常指调度系统的一次后台工作 | 未必有对话/Agent 语义 |
| **Call** | 一次 Provider/Tool/RPC 调用 | timeout 后 call 的 effect 可能未知 |
| **Handoff** | 把后续控制权交给另一个 Agent | 与“Agent as Tool”不同：后者调用后控制权返回调用者 |

推荐 ID：

```ts
type ExecutionIdentity = {
  tenantId: string;       // 永不从 runId 推断
  threadId?: string;      // 用户长期会话；CI/webhook Task 可没有
  taskId: string;         // 平台业务承诺
  runId: string;          // 一次可替换执行尝试
  turnId?: string;        // 一次交互
  operationId: string;    // 逻辑操作
  attempt: number;        // 物理尝试
  callId?: string;        // 模型/Tool call
  traceId: string;        // 诊断关联，不做授权
};
```

## 3. 状态、事实与意图

| 术语 | 定义 | 关键性质 | 常见误用 |
| --- | --- | --- | --- |
| **Command** | 希望系统做某事的意图 | 可拒绝/失败/重试；命名祈使式 | 把 `SendEmail` 当已发送事实 |
| **Domain Event** | 领域中已经发生的不可改写事实 | 过去式、有序、可重放 | 把 debug log 当事件 |
| **State** | 截至某序号由历史归约出的当前视图 | 可由 event 重建（若采用 event log） | 只有可变 status，无因果 |
| **Reducer** | `(state,event)→state` 的确定性纯函数 | 无 I/O、时钟、随机数 | 在 reducer 里调用 Provider |
| **Projection** | 为 UI、查询、审计等构建的派生视图 | 可按用途重建 | 把 projection 当唯一真相 |
| **Snapshot** | 某序号状态的性能缓存 | 应可丢弃；带 throughSeq/version/digest | 与 event 等价，覆盖历史 |
| **Checkpoint** | 可恢复位置/状态的保存点，含义依引擎 | 可能包含外部 SDK 私有状态 | 未版本化就长期保存 |
| **Outbox** | 与业务事务同提交、随后可靠发布的待发送记录 | 解决 DB commit 与消息发布原子性缺口 | 误称消息队列本身 |
| **Effect Ledger** | 记录外部副作用 intent/started/confirmed/unknown 的账本 | 需配合下游幂等/查询/补偿 | 单独宣称 exactly-once |
| **Artifact** | 大体积/二进制/不可变输出，以 ref 引用 | 内容寻址、分类、独立保留 | 把 MB 级 Tool 输出塞事件 |
| **RequestEnvelope** | 入口身份、来源事件、租户、输入引用、callback ref、幂等键等的统一可信信封 | 让客户端自报 tenant/权限/任意 URL | 只校验 JSON，不校验来源与摘要 |
| **TaskSpec** | 目标、范围、约束、预算、交付与验收条件的版本化执行契约 | Prompt、Plan | 让 Planner 改写用户授权范围 |
| **Plan** | 完成 TaskSpec 的版本化步骤 DAG 和当前假设 | 授权票据、已发生事实 | 原地改数组，使旧审批继续有效 |
| **Acceptance Criterion** | 对完成条件的可判定规则，关联固定 verifier | “模型认为完成” | 只有“质量好”等主观口号 |
| **Evidence** | 对特定 subject/version 的命令回执、状态快照、Artifact 或评审证明 | 普通日志、Agent 陈述 | 没有环境和来源仍复用旧结果 |
| **DeliveryBundle** | 结果、Artifact、Claim-Evidence、验证状态、限制、usage 和未知项的不可变交付 | 最后一段自然语言 | 隐藏 partial/unverified/unknown |

事件命名：`approval.resolved`、`tool.execution.completed`。命令命名：`ResolveApproval`、`ExecuteTool`。`run.failed` 表示事实，不是“请失败”。

## 4. Prompt、Context、Memory 与检索

| 术语 | 定义 | 不等于 |
| --- | --- | --- |
| **Prompt** | 发送给模型的指令与输入的序列/结构 | UI 文本；实际 prompt 还含 system/tool/context |
| **System instruction** | 最高产品指令层之一，具体优先级由 API 定义 | 安全授权边界 |
| **Context** | 本次模型调用可见的全部输入 | 长期 Memory；数据库所有内容 |
| **Context window** | 模型可处理 token 的容量限制 | 应把窗口塞满的目标 |
| **Context engineering** | 选择、排序、压缩、引用、隔离来源和预算 | 只写“更好的 prompt” |
| **Working memory** | 当前 Run/Turn 中短期可变状态 | 长期事实库 |
| **Long-term memory** | 跨 Run 保留并可召回的信息 | 聊天历史无限拼接 |
| **Semantic memory** | 事实/概念类长期知识 | 用户事件时间线 |
| **Episodic memory** | 经历、事件、时间线 | 向量索引本身 |
| **Procedural memory** | 技能、规则、工作方法 | Tool implementation |
| **RAG** | 检索外部材料并注入上下文的模式 | Memory；RAG 可用于知识但不自动学习 |
| **Embedding** | 把输入映射到向量表示 | 相似度就是真实相关性 |
| **Vector store** | 按向量近邻检索的数据结构/服务 | 权威数据源、权限系统 |
| **Compaction** | 在保留任务语义的前提下降低上下文体积 | 删除审计事件 |
| **Prompt cache** | Provider/应用复用相同前缀计算/响应的机制 | 语义 Memory；命中不代表新鲜 |

**边界例子**：Context Builder 可以从 Memory/RAG 取候选，但必须在注入前做租户授权、版本/新鲜度、来源和 token 预算。检索命中不能绕过权限。

## 5. Tool、Capability、Policy 与 Sandbox

| 术语 | 精确定义 | 区分要点 |
| --- | --- | --- |
| **Tool** | 模型可请求调用的结构化操作合约 | 描述“怎么请求”，未自动授予执行权 |
| **Function call** | 模型输出的结构化调用建议 | 不是已执行；参数仍需校验/授权 |
| **Capability** | 主体在范围、动作、条件、期限下可使用的能力 | 比 Tool 名更细，如只读 `repo/src/**` |
| **Permission** | 政策授予主体对资源执行动作的权利 | 可以是长期 RBAC，也可生成短期 capability |
| **Approval** | 人/系统对一项确切拟议动作的决策事实 | 必须绑定参数/版本/资源；不等于永久 permission |
| **Policy** | 可评估的组织规则，输入身份/资源/动作/上下文 | 与 prompt 指令分离 |
| **Guardrail** | 运行前/中/后的验证或限制 | 只是控制之一，不能替代授权/sandbox |
| **Sandbox** | 通过 OS/虚拟化/资源/网络边界限制代码影响范围 | “在子进程/容器里”不自动等于安全 |
| **Credential broker** | 按 capability 兑换短期、窄 scope 凭据 | 不把长期 secret 交给模型/Tool |
| **Capability snapshot** | Task 固定的 Tool/Skill/Agent Release digest、策略与信任根集合 | 运行中解析 `latest` |
| **Execution ticket** | 绑定主体、release/args/resource/policy/approval/fence 的短期签名执行票据 | `approved: true` 布尔值 |
| **Compensation** | 对已发生 effect 执行语义上的逆操作 | 不是数据库 rollback；可能失败且需审计 |
| **DLP** | 识别/阻止敏感数据外发的控制 | 不保证理解所有业务秘密 |

授权检查推荐表达：

```ts
canExecute({
  subject: "user:42",
  tenant: "t1",
  action: "workspace.patch",
  resource: "repo:7/path:src/a.ts",
  argumentsDigest: "sha256:...",
  toolVersion: "2.1.0",
  runId: "r9",
  expiresAt: "2026-08-30T10:00:00Z"
});
```

## 6. MCP、A2A 与插件

| 术语 | MCP 中的规范角色/对象 | 容易混淆 |
| --- | --- | --- |
| **MCP Host** | 发起并管理一个或多个 MCP Client 的 LLM 应用 | 不是远程 Server |
| **MCP Client** | Host 内面向一个 MCP Server 通信的客户端实例/连接器 | 一个 Host 可有多个 Client；现代协议可无会话，不能从名称假设持久 session |
| **MCP Server** | 提供 Resources/Prompts/Tools 的程序/服务 | Server 的描述与输出仍不可信 |
| **Resource** | Server 暴露的上下文/数据 | 不等于 Tool；通常是“读取内容” |
| **Prompt** | Server 暴露的模板化消息/工作流入口 | 不等于 host system instruction |
| **MCP Tool** | Server 提供、模型可请求执行的函数 | 不自动可信或自动获批 |
| **Elicitation** | Server 请求用户/客户端补充信息 | 不等于 Tool approval，仍需产品策略 |
| **A2A Agent Card** | Agent 的发现/能力/接口描述 | 不等于 MCP Tool schema |
| **A2A Task** | Agent-to-agent 长时工作单元，具有状态/消息/artifact | 不等于本项目内部 Run，需映射 |
| **Plugin** | 可打包扩展技能、工具、MCP、UI 等的分发单元 | 协议不是插件，插件也未必含代码 |
| **Skill** | 可发现、按需加载的任务知识/工作流说明 | 不等于 Tool；Skill 指导怎么做，Tool 提供能做什么 |

**MCP vs A2A**：MCP 重点是应用连接上下文与 Tool；A2A 重点是独立 Agent 之间的发现、消息和长任务协作。两者可组合，不是替代关系。

## 7. 可靠性与分布式语义

| 术语 | 定义 | 必须记住 |
| --- | --- | --- |
| **At-most-once** | 最多执行/投递一次，可能丢 | 适合可丢的 UI delta，不适合关键 effect |
| **At-least-once** | 可能重复但最终投递/尝试 | 消费者必须幂等/去重 |
| **Exactly-once delivery** | 传输层宣称每条只交付一次 | 不等于业务 effect exactly-once |
| **Exactly-once effect** | 外部世界的业务效果恰好一次 | 需要端到端协议/唯一约束；通常只能按特定 Tool 保证 |
| **Idempotency** | 相同逻辑请求重复执行，最终效果等价一次 | “不报错”不够；key 范围和期限要定义 |
| **Deduplication** | 识别重复 ID 并抑制重复处理 | 去重窗口过期后仍可能重复 |
| **Retry** | 对一次失败/未知 operation 发起新 attempt | 不改变逻辑 idempotency key |
| **Replay** | 重新应用历史 event/输入以重建状态或比较决策 | 不应自动重做外部 effect |
| **Resume** | 从 durable handle/cursor/checkpoint 继续 | 是否同一 Provider session 取决于能力 |
| **Recovery** | 故障后恢复一致状态并继续/处置 | 包含 reconcile，不只是重启 |
| **Reconciliation** | 对比内部 intent 与外部事实，补齐 confirmed/unknown | 对 timeout 副作用尤其关键 |
| **Lease** | 有期限的推进权，需要续约 | 过期后旧 worker 必须被 fencing |
| **Fencing token** | 单调 epoch，外部资源拒绝旧 owner 写入 | 仅 lease 时间判断会受暂停/时钟影响 |
| **Finish reserve** | Admission 时专门保留给验证、checkpoint、撤权、清理和交付的预算 | 可被探索步骤耗尽的普通余额 |
| **Optimistic concurrency** | 以 expected version/seq 提交，冲突则重读 | 不是“最后写赢” |
| **Backpressure** | 消费能力不足时反向限制生产 | 无界 buffer 只是推迟崩溃 |
| **Circuit breaker** | 依赖异常时快速失败，随后半开探测 | 不等于 retry；避免放大故障 |
| **Bulkhead** | 隔离租户/依赖/工作负载资源池 | 防 noisy neighbor |

## 8. 时间、取消与错误

| 术语 | 定义 | 区分 |
| --- | --- | --- |
| **Timeout** | 某次等待超过时长 | 不证明远端未执行 |
| **Deadline** | 操作必须结束的绝对/传播截止时间 | 比每层各自 timeout 更能控制总预算 |
| **Cancellation** | 请求停止不再需要的工作 | 合作式；不能撤销已提交 effect |
| **Abort** | 具体语言/API 中中断本地等待/操作的机制 | AbortSignal 不是分布式事务 |
| **Failure** | 操作没有按合约完成 | 分 retryable/permanent/unknown |
| **Outcome unknown** | 已发起副作用但当前证据无法确认其成功或失败 | 普通失败、可安全盲重试 |
| **Error code** | 稳定机器分类 | message 是安全人类描述；stack/provider body 是诊断 |
| **Partial result** | 未完成但可消费的中间产物 | 不能冒充 completed |
| **Degraded** | 以明确较弱语义继续 | 必须可见且不突破安全/合规 |
| **Poison event/message** | 重复处理都失败的数据 | quarantine，不应无限重试阻塞分区 |

## 9. 观测、审计与评估

| 术语 | 回答的问题 | 性质 |
| --- | --- | --- |
| **Trace** | 一次请求/Run 的调用链发生了什么、慢在哪里 | 可采样、诊断用途 |
| **Span** | Trace 中一个有开始/结束的 operation | 有 parent/link、attribute、status |
| **Log** | 某时点的非结构化/结构化诊断记录 | 可能丢、无完整顺序，不做恢复真相 |
| **Metric** | 一段时间的数值聚合趋势 | label 必须低基数 |
| **Domain Event** | 业务事实是什么 | 完整、有序、用于恢复/投影 |
| **Audit Record** | 谁以什么身份/授权做了什么 | 防篡改、最小披露、合规保留 |
| **Eval** | 系统在代表性任务上好不好 | 需要数据集、grader、版本、统计 |
| **Online monitoring** | 生产是否漂移/故障 | 不等于离线质量评估 |
| **Golden trace** | 固定输入对应的规范事件/调用序列 fixture | 验证协议，不要求模型自然语言逐字相同 |
| **SLI** | 实际测量指标 | 如 recoverable success ratio |
| **SLO** | 对 SLI 的内部目标 | 应有窗口和错误预算 |
| **SLA** | 对外合同承诺与补偿 | 不等于内部 SLO |
| **Error budget** | SLO 允许的坏事件额度 | 安全/隔离不应以预算容忍 |

## 10. 安全与密码学

| 术语 | 定义 | 不等于 |
| --- | --- | --- |
| **Authentication** | 证明“你是谁/哪个 workload” | Authorization |
| **Authorization** | 判断主体能否对资源执行动作 | 登录成功 |
| **RBAC** | 基于角色授予权限 | 参数/资源细粒度 capability |
| **ABAC** | 基于主体/资源/环境属性的策略 | 人工 Approval |
| **Tenant** | 数据、策略、配额、计费的隔离单位 | 用户/组织名字符串 |
| **Workload identity** | 服务/进程的可验证身份 | 长期 API key |
| **Encryption** | 用密钥保护机密性 | 完整性/真实性自动成立 |
| **Hash/Digest** | 单向内容摘要，用于完整性/寻址 | 签名；任何人都可计算 hash |
| **MAC** | 共享密钥验证完整性/真实性 | 公钥签名 |
| **Digital signature** | 私钥签名、公钥验证来源/完整性 | 内容加密 |
| **Nonce** | 一次性随机值，阻止重放/关联 | idempotency key |
| **Secret** | 可用于获得权限的敏感值 | 普通 confidential data；保护要求更高 |
| **PII** | 可识别/关联个人的信息 | 只有姓名邮箱；路径/设备 ID 也可能属于 |
| **Threat model** | 资产、主体、边界、攻击路径和控制的系统分析 | 漏洞扫描报告 |

## 11. API、SDK、协议与兼容

| 术语 | 定义 | 边界 |
| --- | --- | --- |
| **API** | 一个组件对调用方公开的可用操作/合约 | 可是进程内，也可跨网络 |
| **SDK** | 帮调用方使用 API/协议的语言库与工具 | 服务端最终策略权威不在 SDK |
| **Protocol** | 跨组件通信的 wire 语法与语义 | TypeScript interface 不是协议规范 |
| **Transport** | 承载消息的机制，如 in-process/SSE/WS/gRPC | 不应改变 Run 语义 |
| **Adapter** | 把外部/具体接口映射到 canonical port | 不应泄漏供应商类型 |
| **Anti-corruption layer** | 防外部模型污染领域模型的适配边界 | 简单 re-export |
| **Backward compatible** | 新 producer/service 不破坏旧 consumer/client | 新增 required 字段通常不是 |
| **Forward compatible** | 旧 consumer 能安全面对新数据/producer | 需要 unknown 处理 |
| **Semantic Versioning** | 以 major/minor/patch 表达公共 API 兼容承诺 | 不能代替 wire negotiation/迁移 |
| **Capability negotiation** | 双方显式协商可用特性/版本 | 根据版本号猜功能 |
| **Deprecation** | 仍可用但计划移除的阶段 | 立即删除 |

## 12. 平面与部署

| 术语 | 定义 | 例子 |
| --- | --- | --- |
| **Control Plane** | 配置、策略、路由、身份、额度、版本管理 | 发布签名 policy snapshot |
| **Data/Execution Plane** | 实际处理 Run、模型/Tool/数据 | 本地 sandbox、云 worker |
| **Local-first** | 权威数据/核心执行优先在设备 | 不等于永远不上云 |
| **Cloud-first** | 权威执行/状态优先在云 | 本地资源需受控代理 |
| **Hybrid** | 云持久编排 + 本地/云 Activity | 要定义断线与双平面协议 |
| **Offline-capable** | 部分能力离线可用并有明确同步语义 | 不等于所有在线任务排队后自动重放 |
| **Multi-tenant** | 一套平台隔离服务多个 tenant | 在表里加 tenantId 仍不足 |
| **Cell architecture** | 把租户/流量分进相对独立故障域 | 不等于只做数据库分片 |
| **Environment fingerprint** | 镜像、源码、依赖、工具、策略、OS 等已声明执行输入的规范摘要 | 完全确定性证明；仍可能有未受控输入 |
| **Deletion proof** | 记录在线删除、索引/缓存失效、密钥处理、备份到期与例外的证据 | 声称所有物理副本瞬间消失 |

## 13. 使用规则与验收

团队应把这些词写进 schema/ADR，而不是只在口头中使用。若外部 SDK 把 `session` 称为可恢复执行，adapter 应显式映射：

```ts
type ProviderSessionMapping = {
  providerSessionId: string; // opaque，仅 adapter 使用
  runId: string;             // 平台执行身份
  resumable: boolean;        // 实际能力，不从名字推断
  expiresAt?: string;
};
```

**验收**：

- [ ] 能区分 Thread/Task/Run/Turn/Step/Attempt，并给出 ID 关系。
- [ ] 能区分 Command/Event/State/Snapshot/Trace/Audit。
- [ ] 能解释 Tool 不等于 Capability，Approval 不等于 Permission。
- [ ] 能解释 timeout、cancel、failure、effect unknown 的差异。
- [ ] 能解释 idempotency 与 exactly-once effect 的边界。
- [ ] 能准确描述 MCP Host/Client/Server 与 A2A Task。
- [ ] 团队 ADR 和接口中不再使用无定义的 `session/task/context`。
