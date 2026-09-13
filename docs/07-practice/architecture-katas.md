# 架构 Katas：用约束训练 Agent Runtime 设计能力

> 每题都给出业务目标、不可违背的约束、故障事件和评分规则。可用 60～90 分钟独立设计，30 分钟评审，再对照“强方案信号”。

## 0. 通用作答模板

每题至少交付：

1. 一张容器/组件图，标出信任边界、状态归属、同步/异步边；
2. 一张关键时序图，包含正常、超时、取消、恢复；
3. 5～10 条不变量；
4. 一个核心协议/接口/事件 schema 代码片段；
5. SLO、容量估算、成本上限；
6. 威胁模型前五项；
7. 测试、故障注入、发布与回滚方案；
8. 两个 ADR：选了什么、拒绝什么、何时重审。

**通用一票否决**：密钥进入 renderer/prompt/log；未授权执行高风险 Tool；多租户数据无 tenant 隔离；宣称任意副作用 exactly-once 却没有协议依据；只有 happy path。

---

## Kata 1：离线优先的跨平台代码助手

### 场景

为 8,000 名研发提供 Windows/macOS 桌面助手。用户可离线阅读/搜索仓库，联网时使用两个外部模型。企业要求源代码默认不离开设备，部分团队可由管理员放宽。

### 硬约束

- 70% 仓库 2 GB 以内，最大 20 GB；设备可能只有 8 GB RAM；
- 客户端每天可能离线 8 小时，强制升级窗口为 30 天；
- renderer 被视为低权限；本地插件和仓库内容不可信；
- 在线首 token p95 < 3 秒，离线搜索 p95 < 500 ms；
- 禁止把源代码、绝对路径和用户名写入云 Trace；
- 必须支持撤销某插件和某模型，不等待客户端升级。

### 设计任务

- 划分 renderer、host、indexer、sandbox、云控制平面；
- 定义本地索引、会话事件和策略快照的数据模型；
- 说明“本地检索结果是否能发给 Provider”的策略执行点；
- 设计离线队列与重新联网后的冲突/过期处理；
- 给出 Windows/macOS sandbox 差异的统一抽象。

### 故障卡

评审时随机抽两张：索引中途磁盘满；插件被撤销但设备离线；Provider 已收请求后断网；本地 DB 损坏；升级后的事件 schema 无法读取旧 Run。

### 评分（100）

| 维度 | 分值 | 满分证据 |
| --- | ---: | --- |
| 边界与权限 | 25 | renderer 无 Node 权限，workspace capability、插件隔离、撤销检查 |
| 离线/同步语义 | 20 | 策略版本、过期、冲突和恢复状态明确 |
| 数据最小化 | 15 | 分类、DLP、trace redaction、绝对路径替换 |
| 性能容量 | 15 | 增量索引、内存/磁盘预算和背压估算 |
| 兼容升级 | 15 | N/N-1、事件迁移、kill switch、回滚 |
| 验证证据 | 10 | 断网、磁盘满、恶意插件故障注入 |

**强方案信号**：云端发布签名策略快照，本地执行点验证；索引分片且可重建，不把索引当事实真相；远程撤销有短 TTL 与“离线保守拒绝”规则。

**常见陷阱**：把 Electron 主进程当“安全沙箱”；恢复联网后自动重放所有旧 Tool。

---

## Kata 2：高风险财务 Agent 的审批与幂等

### 场景

Agent 读取发票、在 ERP 创建付款草稿，并在两人审批后提交银行。读取是低风险，创建草稿是中风险，真实付款不可逆。

### 硬约束

- 单笔最高 500 万；付款提交必须双人复核，审批人不能是发起人；
- 银行 API 有 idempotency key，但 timeout 后最多 24 小时才能查询；
- 工作流可能等待审批 7 天，期间汇率、供应商账户、策略可能变化；
- 任何模型输出都不能直接成为金额/账户的最终事实；
- 审计保留 7 年，prompt 正文只保留 30 天；
- RTO 1 小时，RPO 0（付款意图/结果）。

### 设计任务

- 设计 draft、validation、approval、commit、reconcile 状态机；
- 明确审批绑定的 canonical payload、digest、策略版本与过期；
- 处理付款 API 在“已提交但响应丢失”的情形；
- 区分业务事件、审计记录和敏感 artifact；
- 设计人工处置队列与禁止自动重试的错误类别。

### 必须给出的协议片段

```ts
type PaymentAuthorization = {
  paymentId: string;
  payloadDigest: string;
  policyVersion: string;
  approvers: [{ subject: string; at: string }, { subject: string; at: string }];
  validUntil: string;
  nonce: string;
  signature: string;
};
```

### 评分（100）

| 维度 | 分值 | 满分证据 |
| --- | ---: | --- |
| 状态/恢复 | 25 | intent 先落盘、unknown effect、reconcile、唯一终态 |
| 授权正确性 | 25 | 双人分离、digest、防重放、执行前重新验证 |
| 财务事实校验 | 15 | ERP/银行主数据校验，不信模型提取 |
| 审计与保留 | 15 | 7 年不可篡改事实与 30 天正文分离 |
| DR | 10 | 跨区事件复制、RPO 0 的实际机制 |
| 测试 | 10 | timeout/重复/账户变更/审批过期场景 |

**强方案信号**：timeout 后进入 `commit_unknown`，按银行 idempotency key 对账，绝不生成新 key 重付；付款 payload 任一字段变化都使审批失效。

**常见陷阱**：“失败重试三次”；把审批记录存在聊天消息里。

---

## Kata 3：十万并发 Run 的多租户 Runtime

### 场景

平台向企业内部 120 个产品提供 Agent Runtime。峰值 2,000 新 Run/s，活跃 Run 100,000；租户规模相差四个数量级。模型成本是主要支出。

### 硬约束

- 单租户不能耗尽全局 Provider 配额；Free/Standard/Critical 三档；
- Provider A/B 有不同区域、速率和 Tool 能力；
- 每个 Run 最多 50 步、15 分钟、2 美元；Critical 可申请例外；
- 事件必须按 Run 有序，跨 Run 不要求全序；
- 单区域故障时 Critical RTO < 10 分钟；
- Metric label 不得使用 runId/userId 等无界维度。

### 设计任务

- 设计接入限流、公平队列、worker shard、lease、event partition；
- 给出 Provider 路由输入和熔断/探活/降级策略；
- 设计 cost reservation、实时 usage、最终结算；
- 计算事件写入与 Provider 并发的大致量级；
- 说明 noisy neighbor、热租户和热点 Run 如何处理。

### 容量起始数据

- 平均 Run 活跃 50 秒；
- 平均每 Run 8 次模型调用，每次 1.2 秒；
- 关键事件平均 35 条/Run，UI delta 合并后另 20 条；
- 平均 artifact 40 KB，保留 30 天。

### 评分（100）

| 维度 | 分值 | 满分证据 |
| --- | ---: | --- |
| 容量模型 | 20 | 算出并发、写 QPS、存储量，列假设与峰值系数 |
| 公平与背压 | 20 | 分层令牌桶、WFQ/DRR、有界队列、admission |
| 状态一致性 | 20 | run shard、lease/CAS、单 Run 顺序、幂等 |
| 路由/成本 | 20 | capability+region+health+budget，预留/结算 |
| 容灾 | 10 | Critical 的数据复制、流量切换、演练 |
| 观测 | 10 | 低基数 SLO、tenant tier 维度、抽样 trace |

**强方案信号**：把“接受 Run”与“马上运行”分开；按 tenant+tier 公平调度；成本预算作为执行状态而非事后报表。

**常见陷阱**：为保证顺序建立全局单队列；Metric 使用 runId 标签。

---

## Kata 4：统一 SDK 的跨端与协议演进

### 场景

同一 AI 能力将用于 Desktop、Web、Browser Extension、VS Code/JetBrains。Desktop 可本地执行 Tool，Web 只能调用云 Tool；各客户端发布节奏不同。

### 硬约束

- Desktop 支持 N/N-1，Browser Extension 可能落后 90 天；
- 事件流支持 WebSocket、SSE 和进程内 transport；
- 客户端断线 24 小时后仍需恢复；
- Provider 新增并行 Tool call，但旧客户端只理解串行；
- JavaScript SDK 之外，半年后要支持 Java；
- 不能用 TypeScript 类型充当 wire contract。

### 设计任务

- 设计 capability negotiation、event envelope、cursor、error model；
- 明确哪些语义在 SDK、transport、Runtime；
- 制定 breaking/non-breaking schema 规则；
- 处理 unknown event/enum、snapshot fallback、delta 合并；
- 给出 provider-specific escape hatch，且不污染核心合约。

### 评分（100）

| 维度 | 分值 | 满分证据 |
| --- | ---: | --- |
| Wire contract | 25 | JSON Schema/Protobuf、版本、ID、seq、unknown 规则 |
| 恢复与流控 | 20 | cursor、gap、snapshot、有界 buffer、背压 |
| Capability | 20 | required/preferred、显式降级、旧端安全行为 |
| 跨语言 | 15 | schema 生成、golden vectors、Java 可实现 |
| 演进流程 | 10 | diff CI、弃用窗口、兼容矩阵 |
| 测试 | 10 | transport contract、乱序/重复/新事件测试 |

**强方案信号**：事件 schema 独立发布；客户端对未知事件保留 seq 且默认不授予能力；provider metadata 是 namespaced opaque data。

**常见陷阱**：WebSocket 消息格式就是 TypeScript interface；用 URL 版本解决一切。

---

## Kata 5：从 Agent Framework A 迁移到 B

### 场景

现有产品深度依赖某 Framework 的 message、memory、tool decorator 和 trace。新框架在持久 Workflow、评估与 Provider 支持上更好，要求六个月渐进迁移且不中断历史 Run。

### 硬约束

- 每天 30 万 Run，历史中有最长等待 60 天的工作流；
- 两框架的 Tool call ID、消息 role、handoff 和 checkpoint 语义不同；
- 质量下降超过 2%、成本增加超过 10% 必须回滚；
- 一部分 workflow 无法双跑，因为 Tool 有真实副作用；
- 团队 15 人，不能停功能开发六个月。

### 设计任务

- 识别应由平台拥有的 canonical contract；
- 设计 anti-corruption layer 与旧 Run version pinning；
- 制定 shadow、replay、canary、切流、回滚阶段；
- 对不可双跑副作用设计“决策对比”或 stub；
- 定义质量、可靠性、成本比较及统计门槛。

### 评分（100）

| 维度 | 分值 | 满分证据 |
| --- | ---: | --- |
| 解耦边界 | 25 | 领域 Run/Event/Tool/State 不依赖框架类 |
| 历史兼容 | 20 | definition pin、旧 worker、迁移/终止策略 |
| 迁移阶段 | 20 | shadow→canary→cohort→default，有 kill switch |
| 评估科学性 | 15 | 固定数据集、置信区间、分层指标、成本 |
| 副作用安全 | 10 | stub/recorded result/decision-only comparison |
| 组织执行 | 10 | strangler、owner、双写期限、删除旧代码条件 |

**强方案信号**：迁移的是 adapter，不是历史事件；先提取 canonical contract 再换引擎；旧 Run 由旧版本 worker 排空。

**常见陷阱**：一次性重写；对 60 天等待 Run 强行重放新逻辑。

---

## Kata 6：MCP 生态与提示注入防线

### 场景

允许员工安装内部和第三方 MCP Server。Server 可提供 resources、prompts、tools；一个恶意 resource 尝试诱导模型读取 SSH key 并调用“上传日志” Tool。

### 硬约束

- 第三方 Server 元数据、Tool 描述和输出全部不可信；
- 本地 stdio Server 与远程 HTTP Server 都要支持；
- 管理员可按租户 allow/deny，用户可按 workspace 授权；
- 工具列表可能有 5,000 个，不能全部放入模型上下文；
- 必须可撤销、可审计、可归因到 Server 包和版本；
- 低风险读取尽量少打断用户，高风险外发必须明确批准。

### 设计任务

- 设计 discovery→verification→registration→selection→invocation 生命周期；
- 定义 Server 身份、包签名、Tool schema/version、capability grant；
- 设计 Tool search，防止恶意描述抢占；
- 说明 resource/tool output 进入 context 时的来源标记；
- 构造 approval UI，展示真实 destination、数据摘要和参数。

### 评分（100）

| 维度 | 分值 | 满分证据 |
| --- | ---: | --- |
| 零信任模型 | 25 | 元数据/输出不可信，host 做独立 policy |
| 能力授权 | 25 | server/tool/resource/args/expiry 绑定，撤销 |
| 注入防护 | 15 | 来源隔离、DLP、不可由文本提升权限 |
| 规模 | 15 | 分层索引、Tool search、候选验证，不塞 5,000 schema |
| UX/审计 | 10 | 风险分级、可理解批准、完整 attribution |
| 测试 | 10 | 恶意描述、schema 漂移、server 替换、重放 |

**强方案信号**：Tool search 结果仍需 policy filter；审批展示从已验证参数生成，不复述模型解释；Server 更新改变 digest 后旧授权失效。

**常见陷阱**：相信 `readOnlyHint` 等自报 annotation；“系统 prompt 写了不要泄密”。

---

## Kata 7：取消、恢复与 Human-in-the-loop 竞态

### 场景

一个研究 Agent 并行调用 8 个工具，随后等待用户选择候选结果。用户可能在任意时刻取消，客户端可能同时断线，Tool 也可能在取消后返回。

### 硬约束

- 取消请求 1 秒内反馈“正在取消”，10 秒内停止可中断工作；
- 已提交的外部任务不一定可取消；
- Run 只能有一个终态，但取消后返回的真实副作用不能丢失；
- 用户选择等待可持续 14 天；
- 重连时 UI 不重复展示 delta 或审批；
- 同一用户可能从两台设备同时操作。

### 设计任务

- 定义 cancelling、cancelled、completed、effect_unknown 的竞态规则；
- 设计结构化并发：父 Run 与子任务的 cancellation scope；
- 处理双设备审批/取消的 CAS；
- 定义迟到结果的业务事件与诊断事件；
- 设计 14 天 timer、lease、通知和过期。

### 评分（100）

| 维度 | 分值 | 满分证据 |
| --- | ---: | --- |
| 状态机 | 30 | 原子终态、迟到 effect、等待/过期路径明确 |
| 取消传播 | 20 | AbortSignal/deadline、child scope、不可取消分类 |
| 并发控制 | 20 | expectedVersion/CAS、幂等 command、双设备 |
| 恢复/UI | 15 | seq/cursor/snapshot、delta 聚合 |
| 长等待运营 | 10 | durable timer、提醒、策略更新处理 |
| 测试 | 5 | 模型检查或系统化竞态枚举 |

**强方案信号**：`cancel.requested` 是事实但不是立即终态；不可取消 Tool 的 effect 完成仍被记录并触发补偿/告警；终态用数据库唯一约束/CAS。

**常见陷阱**：AbortController 当分布式事务；取消后直接删除 Run。

---

## Kata 8：灾难恢复与区域/Provider 故障

### 场景

关键企业 Agent 跨新加坡、法兰克福部署。数据驻留禁止跨区域复制 prompt/artifact，但控制元数据可跨区。某区域 Event Store 损坏，同时主 Provider 大面积故障。

### 硬约束

- Critical Run RTO 15 分钟、状态 RPO < 1 分钟；
- restricted artifact 必须留在原区域；
- Provider 切换可能改变 Tool call 行为和输出质量；
- 30% Run 正等待本地设备 Tool；
- 恢复期间不得重复发送邮件/付款等外部 effect；
- 每季度必须演练，演练不能真的触发生产副作用。

### 设计任务

- 分类哪些状态可跨区、哪些只能区域内备份；
- 设计 event/snapshot/outbox/effect ledger 的备份与恢复顺序；
- Provider failover 的 capability 与质量门禁；
- 制定进入/退出灾备模式、Run fencing、DNS/路由切换；
- 设计“演练凭证”和无副作用 replay。

### 评分（100）

| 维度 | 分值 | 满分证据 |
| --- | ---: | --- |
| 数据/驻留 | 20 | metadata/artifact 分类、区域密钥、恢复边界 |
| 一致性与 fencing | 25 | epoch/lease，旧区恢复后不能双推进 |
| effect 安全 | 20 | ledger/outbox/query，replay 不重做 |
| Provider 故障 | 15 | capability gate、degraded event、质量 canary |
| Runbook | 10 | 检测→宣告→切换→验证→回切 |
| 演练 | 10 | 合成租户、stub Tool、可量化 RTO/RPO |

**强方案信号**：先 fence 旧 writer 再提升新 writer；恢复顺序以事件/effect 真相优先，不以缓存优先；Provider 切换是产品语义变化并可见。

**常见陷阱**：把多可用区当跨区域 DR；只恢复数据库，不处理 outbox 与外部 effect。

---

## 9. 评审主持方法

### 9.1 90 分钟流程

| 时间 | 活动 |
| ---: | --- |
| 0～10 分钟 | 复述目标、SLO、合规与最大未知 |
| 10～30 分钟 | 画边界、状态归属和关键路径 |
| 30～50 分钟 | 协议、状态机、错误/恢复 |
| 50～65 分钟 | 容量、成本、可观测 |
| 65～75 分钟 | 抽取两张故障卡现场推演 |
| 75～85 分钟 | 演进、发布、回滚 |
| 85～90 分钟 | 写下 ADR 与待验证假设 |

主持人不断追问四句话：“失败发生在哪一侧？”“事实写到哪里了？”“谁有权限？”“拿什么测试证明？”

### 9.2 能力分级

| 得分 | 判定 |
| ---: | --- |
| < 60 | 能列技术，但语义/边界/失败处理不完整 |
| 60～74 | 可做模块设计，仍需指导生产可靠性 |
| 75～89 | 可主导系统设计并权衡演进 |
| 90～100 | 能把技术、风险、运营、组织证据闭环 |

分数不是为了“标准答案”，而是暴露遗漏。一个方案可以选不同组件，只要约束、语义和证据自洽。

## 10. 自己出题的模板

```markdown
## Kata：名称
### 业务结果
### 工作负载与 SLO
### 数据分类/合规
### 不可改变的约束
### 当前系统与团队能力
### 三张故障卡
### 必交付物
### 评分：边界 / 状态 / 安全 / 容量 / 演进 / 证据
### 一票否决
```

## 验收

- [ ] 完成至少 6 题，每题都在故障卡后修改过设计。
- [ ] 每题都有状态机、信任边界、容量估算、发布/回滚，不只画组件。
- [ ] 对每个自动重试能回答“为什么安全”。
- [ ] 对每个高风险 Tool 能回答“谁授予什么资源、到何时”。
- [ ] 能用 [ADR 模板](adr-template.md) 固化争议，并用 [生产检查清单](production-checklists.md) 找缺口。
