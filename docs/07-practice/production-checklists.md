# 生产检查清单：Agent Runtime 全生命周期门禁

> 每项绑定 **Owner、Evidence、Due date、Exception expiry**。没有证据就不算完成；`P0` 项不能以“后续优化”豁免。

## 1. 清单数据结构

建议把门禁纳入仓库，由 CI 校验：

```yaml
service: agent-runtime
release: 2026.09.0
items:
  - id: EVT-001
    priority: P0
    requirement: "每个 Run 恰好一个终态"
    owner: runtime-team
    status: passed              # pending | passed | exception
    evidence:
      - "ci://tests/reducer-terminal-property"
      - "db://constraints/run_terminal_unique"
    reviewed_at: "2026-08-30"
  - id: OPS-014
    priority: P1
    requirement: "Provider 全故障演练"
    status: exception
    exception:
      approver: sre-director
      reason: "预生产配额尚未开通"
      expires_at: "2026-09-15"
      compensating_control: "禁用自动切流，仅手动降级"
```

状态不是自由文本。Exception 必须有到期、风险所有者和补偿控制；到期自动使发布门禁失败。

## 2. Phase A：业务、风险与 SLO

### A1. 业务边界

- [ ] `P0` 写清 Agent **允许做什么、禁止做什么、谁承担最终责任**。
- [ ] `P0` 对每个 Tool 标注 read / reversible-write / irreversible-write。
- [ ] `P0` 定义 Human-in-the-loop 的触发条件，不能只写“必要时审批”。
- [ ] 明确 Run/Thread/用户操作的生命周期与最长持续时间。
- [ ] 明确支持与不支持的平台、OS 版本、网络环境和数据区域。
- [ ] 明确供应商故障时：拒绝、排队、只读、切换还是降级。
- [ ] 对模型输出标注“建议/提取/决策/事实”的法律与业务地位。

**证据**：产品边界文档、Tool 风险登记表、数据流图、异常 UX 原型。

### A2. SLI/SLO 与错误预算

- [ ] `P0` 可用性以“Run 被接受并最终可达终态”定义，不只看 HTTP 200。
- [ ] `P0` 可靠性包含重复 effect、丢事件、错误授权等正确性指标。
- [ ] 定义接受延迟、首增量、完成时长的 p50/p95/p99。
- [ ] 定义 Provider/Tool/Runtime 各自的可用性和依赖预算。
- [ ] 安全、租户隔离、支付重复等指标无错误预算。
- [ ] 长等待 Run 单独统计，不用完成时长污染交互式 SLO。
- [ ] SLO 有查询、dashboard、alert 和演练数据，不是 PPT 数字。

```yaml
slo:
  name: recoverable-run-success
  objective: 0.999
  window: 30d
  good: "terminal in [completed,cancelled] OR retryable failure recovered"
  total: "accepted runs excluding documented user validation errors"
  correctness_guard:
    duplicate_irreversible_effects: 0
    cross_tenant_access: 0
```

## 3. Phase B：架构与状态语义

### B1. 边界与所有权

- [ ] `P0` 组件图标出 renderer/host/sandbox/cloud/provider/MCP 信任边界。
- [ ] `P0` 每类状态有唯一 source of truth：event、artifact、policy、credential、telemetry。
- [ ] 控制平面与数据平面分离，策略以版本化快照进入 Run。
- [ ] Runtime core 不依赖 Desktop framework、Provider SDK、数据库实现。
- [ ] 本地/云 Activity 的所有权、lease、heartbeat、失联语义明确。
- [ ] `P0` lease 配套单调 fencing token，并在 event/artifact/checkpoint 的接收端校验旧执行者。
- [ ] 同步 RPC 只用于短操作；长操作返回 durable handle/event cursor。
- [ ] Runtime Task、Provider Response、Thread、Turn、Attempt 使用独立 ID/状态，不以外部 ID 充当业务主键。

### B2. 状态机与事件

- [ ] `P0` Run/Turn/Tool/Approval 状态机有 transition table。
- [ ] `P0` phase 与 outcome 分开；`BLOCKED` 可恢复，`CANCELING` 不冒充已取消，`outcome_unknown` 是一等语义。
- [ ] `P0` terminal 唯一，由 reducer 和存储约束双重保证。
- [ ] `P0` event 有 `tenantId/runId/eventId/seq/type/schemaVersion/time`。
- [ ] `seq` 在单 Run 内单调且 gap 可检测；不按时间戳排序。
- [ ] command 与 event 分离；外部 I/O 不在 reducer 内。
- [ ] 历史 event 不原地改写；修正用新 event/投影。
- [ ] snapshot 可丢弃重建，包含 `throughSeq` 与 reducer version。
- [ ] delta 有 completed snapshot 收口，不能成为唯一事实。
- [ ] Workflow definition/version 固定到 Run，旧 Run 有兼容执行路径。

### B3. 并发、幂等与恢复

- [ ] `P0` 外部副作用前先持久化意图。
- [ ] `P0` 每个 effect 有稳定 idempotency key、查询或补偿策略。
- [ ] timeout 被视为“结果未知”，不等于失败且未执行。
- [ ] Run 只有一个 active lease；追加事件使用 expectedSeq/CAS。
- [ ] 重试有 retryable 分类、deadline、退避+jitter、最大次数和预算。
- [ ] 取消传播到队列/Provider/Tool；不可取消 effect 有后续处理。
- [ ] 恢复算法能处理 requested 无 completed、effect 已做事件未写。
- [ ] 定时器/审批等待是 durable state，不依赖进程 `setTimeout`。
- [ ] 等待审批前 checkpoint/park，可释放 Worker 和非必要 Secret；批准后恢复时重新 admission/authorize。
- [ ] 客户端幂等键绑定 canonical request digest；同 key 不同内容返回明确冲突。

## 4. Phase C：合约与统一 SDK

### C1. Wire contract

- [ ] `P0` JSON Schema/Protobuf 是 source of truth，类型由 schema 生成。
- [ ] `P0` CI 检测 breaking change：删字段、改语义、收窄枚举、改 required。
- [ ] 客户端对 unknown event/enum 安全降级，不获得额外权限。
- [ ] cursor 明确 inclusive/exclusive；断线补拉、snapshot fallback 已定义。
- [ ] SSE cursor 有 retention watermark；早于水位时显式返回过期/snapshot 协议，不静默跳事件。
- [ ] 事件/错误/Tool 的版本策略相互独立。
- [ ] 大内容用 content-addressed artifact ref，不塞事件。
- [ ] 输入限制字节、深度、数组长度，防解析资源耗尽。

### C2. SDK 行为

- [ ] `P0` local/remote transport 运行相同 contract suite。
- [ ] SDK 公开稳定错误码、retry hint、request/run/trace ID。
- [ ] SDK 的 timeout 是调用等待上限，不擅自取消服务端 Run。
- [ ] `AbortSignal`/cancellation token 有明确传播和竞态语义。
- [ ] capability negotiation 区分 required/preferred；降级可观察。
- [ ] Provider-specific metadata 有 namespace/escape hatch，不污染核心类型。
- [ ] 多语言 golden vectors 与 wire fixtures 可共享。
- [ ] N/N-1/最旧扩展版本的兼容矩阵和支持窗口已发布。

## 5. Phase D：安全、隐私与合规

### D1. 身份与授权

- [ ] `P0` 用户、设备、workload、tenant 身份分开验证。
- [ ] `P0` 所有数据查询/缓存/事件 key 包含 tenant；有数据库级隔离。
- [ ] `P0` Tool 授权绑定主体、run/call、tool version、args digest、资源、期限、nonce。
- [ ] 发起人与审批人分离的场景由策略引擎强制。
- [ ] 授权在实际执行点重验，不能只在 UI/API gateway 检查。
- [ ] 审批绑定 canonical args、tool/resource/policy version、环境 generation、期限与次数；审批不扩大基础授权。
- [ ] 管理员撤销、kill switch、离线策略和传播时限明确。

### D2. Desktop 与 Sandbox

- [ ] `P0` renderer 无 API key、任意 IPC、任意文件/命令能力。
- [ ] `P0` IPC channel allowlist + schema + caller/window/session 绑定。
- [ ] workspace 路径 realpath 校验，覆盖 symlink/TOCTOU/大小写差异。
- [ ] sandbox 有 OS 用户/namespace、只读根、资源、进程数、时间限制。
- [ ] egress 默认拒绝，按域/端口/Tool capability 放行。
- [ ] secret 由 broker 注入，Tool 看不到长期主凭据。
- [ ] 插件/MCP server 身份、签名、版本、来源、撤销可审计。
- [ ] Tool/Skill/Agent registry 保存内容 digest、provenance、SBOM/依赖、兼容性和紧急撤销状态；“已注册/已签名”不等于可执行信任。
- [ ] 自动更新包签名、provenance、SBOM、降级保护已验证。
- [ ] 执行环境记录不可变镜像/工具/策略/源码指纹；Setup 与 Agent 身份、网络和 Secret 分离。
- [ ] warm pool 有跨租户 canary 与清理证明；清理失败的实例进入隔离而非回池。

### D3. Prompt injection 与 Tool 安全

- [ ] `P0` 用户/网页/仓库/MCP/Tool 输出均视为不可信数据。
- [ ] `P0` 内容不能改变授权；system prompt 不是安全边界。
- [ ] Tool schema 默认 `additionalProperties: false`，服务端重新校验。
- [ ] Tool 描述/annotation 不作为风险判断的唯一依据。
- [ ] 高风险审批 UI 展示规范参数、真实 destination、数据摘要，不复述模型理由。
- [ ] 外发前 DLP/secret scan，使用 canary secret 红队。
- [ ] Tool output 带 source/integrity/sensitivity，进入 context 保留边界。
- [ ] 恶意 Tool search 排名、schema drift、Server 替换均有测试。

### D4. 数据与隐私

- [ ] `P0` 数据清单覆盖 prompt、response、context、memory、artifact、trace、audit、backup。
- [ ] 每类数据有用途、法源/同意、区域、加密、访问、保留、删除。
- [ ] Provider 路由遵守租户的数据使用/保留/区域政策。
- [ ] 默认 telemetry 不含正文、secret、绝对路径、用户名。
- [ ] 日志脱敏发生在写出进程前；禁止依赖下游再清洗。
- [ ] 加密密钥按 tenant/region 隔离，轮换与吊销演练通过。
- [ ] 删除覆盖主存、索引、artifact、cache、backup 到期流程。
- [ ] 删除 SLA 准确区分在线删除、密钥销毁、备份过期和 legal hold，不承诺不可实现的瞬时物理删除。
- [ ] Evals/标注数据复用有单独政策，不能默认拿生产 prompt 训练。

## 6. Phase E：实现与测试

### E1. 工程质量

- [ ] 所有外部调用有 deadline，所有队列/buffer 有界。
- [ ] Clock、ID、Provider、Tool、Store 可注入，测试可确定性运行。
- [ ] Adapter 之外不出现 Provider SDK 类型。
- [ ] config 有 schema、默认值、合法范围、secret reference。
- [ ] feature flag 有 owner、expiry、kill semantics，不成为永久分支。
- [ ] 依赖锁定、漏洞扫描、license、SBOM、构建 provenance 自动化。
- [ ] 数据库 migration 有 forward/backward compatibility 和回滚/roll-forward。

### E2. 测试组合

- [ ] `P0` reducer property test：gap、重复、乱序、terminal、非法转换。
- [ ] `P0` effect crash-point test：副作用前/后重启不重复。
- [ ] Provider/Tool/Transport adapter contract suite。
- [ ] schema golden vectors：各语言 decode/encode 一致。
- [ ] 多租户隔离、IDOR、权限提升、安全默认值测试。
- [ ] path traversal、symlink、shell injection、资源耗尽测试。
- [ ] 断网、429、5xx、慢流、半开连接、迟到响应测试。
- [ ] 取消 vs 完成、审批 vs 撤销、双 worker 等竞态测试。
- [ ] context 超窗、压缩失败、检索污染、恶意文档测试。
- [ ] 性能 load/soak：峰值、持续、突发、单热租户、慢客户端。
- [ ] chaos：store/queue/provider/OTel/disk/clock skew/区域故障。

### E3. Evals

- [ ] 任务数据集按场景/风险/语言/规模分层，有版本和来源。
- [ ] 同时测成功率、事实性、Tool 正确性、安全、延迟、token、费用。
- [ ] 模型裁判有校准集和人工抽检，不作为唯一发布门禁。
- [ ] TaskSpec/Plan DAG 有 schema 与语义校验；Verifier 独立于执行者，最终 claim 可映射到 evidence hash 与环境指纹。
- [ ] Structured Output 先检查 completed/refusal/incomplete/filter，再做 schema 与业务语义验证。
- [ ] 非确定性用重复样本和置信区间，不比较单次截图。
- [ ] 副作用 eval 使用 stub/sandbox，不触发真实邮件/付款。
- [ ] regression threshold、blocking/warning、owner 清楚。
- [ ] 生产漂移监控与离线 eval 可关联，但不泄露正文。

## 7. Phase F：Provider、Tool 与成本

### F1. Provider

- [ ] 能力表覆盖 streaming、Tool、并行、结构化输出、context、区域。
- [ ] 错误映射保留 provider code 供诊断，但 UI 只见安全稳定错误。
- [ ] 路由输入包含 capability/compliance/health/cost/latency，不由 prompt 决定。
- [ ] fallback 不是静默：记录 degraded event 和实际 provider/model。
- [ ] 速率配额按 tenant/tier 隔离，有全局熔断和 half-open 探测。
- [ ] 请求/响应 usage 缺失和最终校正已处理。
- [ ] SDK/API deprecation、model retirement 有 owner 与迁移窗。

### F2. Tool

- [ ] registry 中 schema version 与 implementation version 分离。
- [ ] Tool 风险、effect 类型、幂等/查询/补偿能力已登记。
- [ ] 参数 canonicalization 与 digest 算法跨语言一致。
- [ ] 输出大小、媒体类型、敏感度、artifact 化规则明确。
- [ ] MCP/remote Tool 的身份、OAuth scope、token audience 正确。
- [ ] 长任务提供 handle/progress/cancel/query，避免无限同步请求。
- [ ] Tool 版本升级有 contract fixtures 与旧 Run 兼容策略。

### F3. 成本

- [ ] Run 接受前有预算预留；每步更新已用/预计费用。
- [ ] 预算单独预留 verifier、checkpoint、撤权、通知和清理额度，业务执行不可消耗 finish reserve。
- [ ] 达预算时有 stop/ask/route cheaper 的显式产品行为。
- [ ] pricing 版本化，不把动态单价硬编码在 reducer。
- [ ] cache 命中、重试、废弃结果、推理 token 均正确归属。
- [ ] 日/周预算、异常增长、单租户/单 Tool 成本告警。
- [ ] 成本优化没有破坏质量/安全门禁，有 A/B 证据。

## 8. Phase G：可观测与运营

### G1. Telemetry

- [ ] `P0` trace 可从 SDK→Runtime→Provider/Tool 传播，异步用 span link。
- [ ] `P0` domain event store 与 trace backend 明确分离。
- [ ] span 包含 operation、status、attempt、provider/tool/version、duration。
- [ ] metric 标签低基数；runId/userId 不作为 label。
- [ ] error code、取消、用户拒绝分别统计。
- [ ] 采样保留错误/慢/高风险 Run，正常按策略采样。
- [ ] Collector 不可用时有界降级，不阻塞业务或占满磁盘。
- [ ] dashboard 能从 SLI 直接算 SLO 和 burn rate。

```yaml
# Prometheus 风格示意
alert: AgentRuntimeFastBurn
expr: |
  (
    sum(rate(agent_runs_total{result="bad"}[5m]))
    /
    sum(rate(agent_runs_total[5m]))
  ) > (14.4 * 0.001)
for: 5m
labels:
  severity: page
annotations:
  runbook: "runbooks/runtime-fast-burn.md"
```

### G2. Runbook

每个关键告警必须有：

- [ ] 用户影响和判定阈值；
- [ ] 五分钟内可执行的止血：限流、关闭 Tool、固定 Provider、只读模式；
- [ ] 查询命令/dashboard，不要求值班人先读源码；
- [ ] 数据正确性检查，尤其重复/unknown effect；
- [ ] webhook 目标来自租户预注册配置；入站验签/防重放，出站使用 outbox、重试、DLQ 和回执对账；
- [ ] 升级路径、沟通模板、恢复验证；
- [ ] 回滚/回切条件与观察窗口。

## 9. Phase H：发布、Desktop 分发与回滚

### H1. 发布前

- [ ] `P0` 所有 P0 门禁 passed，无未到期例外也需安全负责人确认。
- [ ] release artifact 可复现、签名、hash、SBOM、provenance 可验证。
- [ ] migration 在生产量级副本演练；旧 binary 与新 schema 可共存。
- [ ] feature flag 默认安全，紧急关闭路径不依赖同一故障组件。
- [ ] canary cohort 包含内部、不同 OS、不同租户等级和真实网络。
- [ ] 变更日志明确协议/策略/模型/Tool 的行为变化。
- [ ] 支持、SRE、安全、隐私和产品 sign-off 有记录。

### H2. 渐进发布

```text
dev → deterministic integration → staging shadow
→ internal 1% → tenant canary 5% → 25% → 50% → 100%
```

每阶段比较：SLO、错误分类、重复 effect、审批率、质量 eval、单位 Run 成本、crash-free session。自动回滚阈值必须在发布前设定，不能看到数据后再挑。

### H3. Desktop 专项

- [ ] Windows/macOS 安装、更新、卸载、代理、证书、受限账户测试。
- [ ] code signing/notarization、更新 manifest 签名、anti-downgrade。
- [ ] crash loop 自动回到最后已知良好版本/禁用新 flag。
- [ ] N/N-1 与云端协议兼容；最小支持版本可远程阻断。
- [ ] 本地 DB migration 可恢复；升级中断不损坏事件/凭据。
- [ ] 卸载/退出登录按政策清理 keychain、cache、artifact、log。

### H4. 回滚

- [ ] binary 回滚与 workflow/event/schema 回滚分开设计。
- [ ] 已由新版本产生的 event，旧版本能忽略或有兼容 reader。
- [ ] 不尝试回滚已发生外部 effect；以补偿/对账处理。
- [ ] Provider/model 回退要重新检查 capability 和质量阈值。
- [ ] 回滚后运行完整性查询：终态、悬空 Tool、outbox、lease。

## 10. Phase I：事故响应

### I1. 首 15 分钟

- [ ] 宣告 incident commander、scribe、technical lead。
- [ ] 判断是可用性、数据完整性、安全/隐私还是供应商事件。
- [ ] 保留证据但不复制敏感正文到聊天工具。
- [ ] 启动最小风险止血：暂停高风险 Tool 优先于暂停只读查询。
- [ ] 检查是否存在重复/unknown effect、跨租户、secret 泄漏。
- [ ] 通知依赖方与用户，区分已知事实和推测。

### I2. 恢复验证

- [ ] event seq/terminal/lease 完整性查询通过；
- [ ] outbox 与 effect ledger 对账，无未知副作用未处理；
- [ ] 失联旧 Worker 的 fencing probe 通过，环境进程/Secret/egress/工作卷均完成清理对账；
- [ ] Provider/Tool 探测通过，熔断逐步 half-open；
- [ ] backlog 以公平/预算方式排空，不形成重试风暴；
- [ ] SLO 恢复并保持观察窗口；
- [ ] 临时 flag/权限/日志提升有到期。

### I3. 复盘

- [ ] 时间线来自事件/trace/audit，不靠记忆；
- [ ] 找系统条件而非个人失误；
- [ ] 行动项覆盖预防、检测、缓解、恢复；
- [ ] 每项有 owner/date/test；关键项进入本清单；
- [ ] 检查同类风险是否横向存在于其他 Provider/Tool/端。

## 11. Phase J：升级、弃用与下线

### J1. 依赖/Provider/框架升级

- [ ] 阅读官方 changelog、migration、security advisory；记录核验日期。
- [ ] 锁定版本并在兼容矩阵中试跑，而非直接追 `latest`。
- [ ] contract、golden trace、eval、cost、soak 全套对比。
- [ ] 历史 Run 固定旧 definition/adapter 或有明确迁移。
- [ ] canary 与 kill switch 已验证；回滚不依赖已删除的包。
- [ ] 升级结束删除临时双写/兼容层有明确日期。

### J2. API/事件/Tool 弃用

- [ ] 先测使用量，再宣布弃用，不靠猜测。
- [ ] 给替代方案、codemod/adapter、时间线、owner。
- [ ] 客户端收到可观测 deprecation warning，不包含敏感内容。
- [ ] 到期前分 cohort 阻断并提供支持。
- [ ] 服务端保留历史 event decoder 到数据保留期/迁移完成。

### J3. 下线与数据处置

- [ ] 停止新 Run，等待/迁移/终止已有长 Run。
- [ ] 撤销 Provider/Tool OAuth、设备证书、service account、signing key。
- [ ] 排空 outbox、timer、notification、effect unknown 队列。
- [ ] 按政策删除主存、artifact、索引、cache；记录 backup 到期。
- [ ] 生成 deletion proof，记录无法立即删除的 backup/legal-hold 例外和最终到期条件。
- [ ] 导出必要审计与法律留存，验证不可再由普通产品路径访问。
- [ ] 删除 alert/dashboard/runbook/域名/队列前确认无调用。
- [ ] 最终成本、风险、经验进入平台 ADR/标准。

## 12. Go/No-Go 汇总门

| Gate | Go 的最低证据 | No-Go 示例 |
| --- | --- | --- |
| Correctness | replay/property/crash-point 全绿 | timeout 后可能重复付款 |
| Security | threat model、授权/隔离红队全绿 | renderer 持有 API key |
| Privacy | 数据清单、保留/删除/DLP 验证 | prompt 默认进入 trace |
| Reliability | SLO dashboard、chaos、DR 演练 | 事件库失败仍先做 Tool |
| Compatibility | N/N-1、unknown、schema diff | 旧客户端遇新 enum 崩溃 |
| Quality | 分层 eval 达门槛 | 只看人工 demo |
| Operations | page 有 runbook，回滚演练 | 只能由作者手工修复 |
| Supply chain | 签名、SBOM、provenance | 更新包来源不可验证 |

发布负责人只有在所有 P0 gate 为 Go 时签字。P1 例外不能掩盖 P0 风险。

## 13. 章节验收

- [ ] 把清单复制到一个真实项目，至少为 20 项填上证据链接。
- [ ] 任选一个 Tool，完成从风险登记、授权、失败、审计到下线的全生命周期。
- [ ] 任选一个 Provider 升级，走完 contract+eval+canary+rollback 计划。
- [ ] 做一次“Event Store 失败 + Tool timeout”的桌面推演并更新 runbook。
- [ ] 能解释为什么通过功能测试、类型检查仍不足以证明生产就绪。

关联：[参考项目](reference-project.md)、[端到端架构](reference-architecture.md)、[一手资料索引](../08-appendix/reading-list.md)。
