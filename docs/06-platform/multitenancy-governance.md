# 多租户、身份与治理

## 1. 租户隔离不是一个 `tenant_id`

多租户 Agent Runtime 必须同时隔离身份、计算、网络、数据、密钥、检索、缓存、队列、遥测和成本。共享程度应由数据敏感度、监管约束与故障爆炸半径共同决定。

| 等级 | 典型场景 | 计算/数据隔离 | 代价 |
|---|---|---|---|
| L1 逻辑隔离 | 内部低敏助手 | 共享服务；行级策略、租户 cache key | 低，但配置错误爆炸半径大 |
| L2 cell/namespace | 多事业部、中敏数据 | 独立队列/索引/密钥，计算按 cell 隔离 | 中，常见默认 |
| L3 专用环境 | 强监管/外部大客户 | 独立账号/VPC/数据库/KMS/部署 | 高，运维与容量碎片化 |
| L4 air-gapped/on-prem | 最高敏感或断网 | 本地模型/工具、离线升级与审计 | 最高，功能和运维受限 |

同一企业也可按 workload 而非客户统一分级：公开知识问答用 L1，代码执行和生产变更用 L2/L3。

## 2. 身份链与授权

区分 `human identity`、客户端设备、平台 workload、Agent release、工具服务和外部 Agent。Agent 不是用户本人；它是在具体 task/purpose/budget 下受限委托的 workload。

```text
User OIDC token
   └─ token exchange → task-scoped delegation token
       sub=user, act=agent-release, aud=tool-gateway
       tenant, purpose, scopes, runId, exp≤15m
           └─ Tool Gateway → resource-specific short token
```

禁止共享服务账号和 token passthrough。认证说明“是谁/谁代表谁”，授权还必须评估 tenant、resource、action、purpose、数据分级、设备态、工具风险、审批状态和时间。

```ts
interface TrustedContext {
  tenantId: string; subject: string; actorAgent: string;
  runId: string; purpose: string; scopes: string[]; dataZone: string;
}

async function authorize(ctx: TrustedContext, req: ToolIntent) {
  if (req.tenantId !== ctx.tenantId) return { allow: false, reason: "cross_tenant" };
  return pdp.check({
    subject: ctx.subject, actor: ctx.actorAgent, action: req.tool,
    resource: req.resourceRef, purpose: ctx.purpose,
    argsDigest: sha256(req.args), policyVersion: ACTIVE_POLICY
  });
}
```

`TrustedContext` 只能由网关从已验证 token 和服务端路由生成，不能接受模型输出的 `tenantId/admin=true`。

## 3. 数据隔离与生命周期

- 关系库：所有表显式 tenant key；数据库 RLS/独立 schema 作为第二道控制，DAO contract test 检查漏条件。
- 向量检索：检索前强制 ACL filter；不能先全局 top-k 再应用层过滤。embedding、chunk、原文 ACL 与版本绑定。
- cache：key 包含 tenant、user/ACL digest、model/prompt/schema 版本；高敏内容不进共享语义缓存。
- Artifact：对象路径和 KMS key 带租户；预签名 URL 短时、绑定用途；下载再次鉴权。
- trace/eval：正文默认不入 trace；采样、脱敏、保留期与数据区按租户策略。
- 删除：定义 conversation、memory、Artifact、backup、eval corpus 的级联 tombstone 与可证明删除 SLA。

记忆写入需保存 provenance、owner、confidence、TTL 和敏感级别；用户可查看/更正/删除。跨租户“学习”只能使用经批准、去标识化的聚合数据，不能把生产 prompt 自动加入公共训练集。

## 4. 配额与 noisy neighbor

建立分层限额：租户月预算 → Agent 日预算 → 用户/设备速率 → run 的 turn/token/tool/wall-clock → 单工具并发。调度器使用 weighted fair queue；不要让一个长上下文批任务占满在线模型配额。

| 资源 | 必看信号 | 控制 |
|---|---|---|
| 模型 | RPM/TPM、并发、token/成功任务 | admission、模型降级、预算停止 |
| 工具 | 并发、P95、错误、写 QPS | semaphore、熔断、每租户池 |
| 沙箱 | CPU/内存/磁盘/网络 | cgroup/VM、wall timeout、egress policy |
| 队列 | age、depth、tenant share | WFQ、deadline、优先级与丢弃策略 |
| 存储 | 向量/Artifact/trace bytes | TTL、归档、hard quota |

## 5. 治理闭环

每个 Agent release 要有 owner、业务目的、风险等级、数据清单、允许工具、评测集、SLO、预算、保留期、应急联系人和下线日期。风险分级决定控制：低风险读助手可自动发布；能发送邮件、改生产或处理敏感数据的 Agent 需要安全评审、人工审批点、红队和更短凭据。

审计日志至少记录 actor/subject、release digest、policy decision、tool/args digest、审批人、实际资源、结果、时间和 traceId；审计存储 append-only、分权访问，不能与普通调试日志混用。

## 6. 失败模式

- **应用层漏 tenant 条件**：单次查询即可泄露。RLS/分库 + contract tests + canary data。
- **共享语义缓存串租户**：相同问题返回另一客户内容。ACL digest 入 key，高敏禁用。
- **Agent 继承用户全部权限**：prompt injection 立即变成横向移动。task-scoped token 与 resource audience。
- **只限请求数**：超长上下文和多 Agent 绕过配额。按 token、工具、时间、美元复合计量。
- **删除只删聊天 UI**：checkpoint、Artifact、trace、备份仍保留。维护数据 lineage 和 deletion workflow。

## 7. 练习与验收

关联零信任、OAuth token exchange、SPIFFE workload identity、RBAC/ABAC/ReBAC、KMS envelope encryption、RLS、数据驻留、隐私工程与 FinOps。

练习：构造两个租户各三种角色，针对 RAG、Artifact、工具和 trace 写 50 条授权矩阵；用恶意 prompt 尝试修改 tenant、复用 URL、越过 quota 和读取缓存。

验收：跨租户 fuzz/property test 零泄露；凭据撤销 60 秒生效；成本可归属至 tenant/agent/release；删除请求覆盖所有派生数据并输出证明；noisy-neighbor 压测下其他租户 P95 增幅低于约定阈值。

## 官方资料

- [NIST Zero Trust Architecture SP 800-207](https://csrc.nist.gov/publications/detail/sp/800-207/final)
- [Kubernetes Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/)
- [SPIFFE specifications](https://spiffe.io/docs/latest/spiffe-about/overview/)
- [OAuth 2.0 Token Exchange RFC 8693](https://www.rfc-editor.org/rfc/rfc8693)
- [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/)
