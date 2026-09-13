# MCP 与 A2A：工具协议、Agent 互操作与信任边界

> 截至 2026-08，MCP 规范版本为 `2026-07-28`，A2A 发布版本为 `1.0.0`。旧教程中的 MCP `initialize`、`Mcp-Session-Id`、HTTP+SSE 常驻会话和 A2A 0.x 字段需要按对应版本重新核对。

## 1. 两个协议解决不同问题

| 对比 | MCP | A2A |
|---|---|---|
| 关系 | Host/Client 调用 Server 暴露的 tool、resource、prompt | 一个自治 Agent 委托另一个可能不透明的 Agent |
| 基本单位 | `tools/list`、`tools/call`、resource/prompt | AgentCard、Message、Task、Artifact、状态更新 |
| 控制权 | 调用方掌握 Agent loop，Server 提供能力 | 被调用 Agent 自己规划与执行，调用方看任务契约 |
| 长任务 | `io.modelcontextprotocol/tasks` 扩展 | 原生 async-first Task、stream/poll/push 模式 |
| 典型场景 | 查 CRM、改工单、读仓库、执行内部 API | “采购 Agent”委托“供应商 Agent”完成报价流程 |

不要把每个 REST API 包成 A2A Agent，也不要把远端自治 Agent 伪装成一个无状态 MCP 工具。选择依据是：调用方是否应控制执行计划，以及对方是否具有独立身份、状态和 SLO。

## 2. MCP 2026-07-28 的架构含义

新核心是无状态 request/response：每次请求在 `_meta` 携带协议版本、客户端身份/能力；可选 `server/discover`，不再强制握手。HTTP 请求还带 `Mcp-Method`、`Mcp-Name`，网关无需解析 body 即可路由、限流和鉴权。`tools/list` 等返回 `ttlMs/cacheScope`；需要中途补充输入时，MRTR 返回 `input_required`，客户端带 `inputResponses` 重试原调用。

```text
Desktop Host ── OAuth/OIDC ── MCP Gateway ── policy ── MCP Server
      │                           │                │
      ├─ consent / approval       ├─ tool catalog ├─ domain API
      ├─ delegated identity       ├─ quota/audit  └─ explicit state handle
      └─ prompt-injection guard   └─ schema pin
```

无传输会话不等于业务无状态。需要状态时，让工具显式返回 `handle`，下一调用显式传入；这样状态可审计、可授权、可迁移。

## 3. Tool Registry 与调用门禁

工具描述、annotation（read-only、destructive、idempotent 等）只是 Server 自声明的**不可信元数据**，不能直接成为授权事实。企业 registry 应保存 owner、schema hash、数据分级、允许租户、网络出口、所需 scope、审批级别、版本与 provenance。

```ts
type Principal = { tenantId: string; userId: string; agentId: string; scopes: string[] };
type ToolRecord = { name: string; schemaHash: string; risk: "read"|"write"|"destructive"; scopes: string[] };

async function invoke(p: Principal, t: ToolRecord, args: unknown) {
  assert(validateSchema(t.schemaHash, args));
  assert(t.scopes.every(s => p.scopes.includes(s)));
  const decision = await policy.decide({ principal: p, tool: t.name, risk: t.risk, argsDigest: sha256(args) });
  if (decision.effect !== "allow") throw new Error("policy_denied");
  if (decision.approvalRequired) return approvals.create({ ...decision, args });
  return sandboxedCall(t, args, { idempotencyKey: decision.callId, deadlineMs: 10_000 });
}
```

HTTP MCP 使用 OAuth 体系时必须校验 issuer、resource/audience、PKCE 等；禁止 token passthrough。桌面 stdio server 的凭据来自受控环境/密钥代理，不应让模型读取环境变量或原始 refresh token。执行身份采用 task-scoped、短时、最小权限 token，而非用户永久令牌。

## 4. A2A 的任务契约

A2A 1.0 由三层构成：protobuf 规范数据模型；`Send Message/Get/List/Cancel Task/Get Agent Card` 等抽象操作；JSON-RPC、gRPC、HTTP/REST 等 binding。公开发现位置为 `/.well-known/agent-card.json`；AgentCard 可声明版本、skills、endpoint、安全方案并用 JWS 签名。Message 用于输入/澄清/状态沟通，最终结果应进入 Artifact。

```ts
interface DelegationEnvelope {
  contextId: string; taskId: string; tenantId: string;
  callerAgent: string; targetAgent: string;
  goal: string; inputArtifactRefs: string[];
  constraints: { deadline: string; maxCostUsd: number; dataZone: string };
  authzContext: { subjectTokenRef: string; scopes: string[] };
  traceparent: string; nonce: string;
}
```

接收端必须把 caller 身份映射成本地权限，不能因“AgentCard 声称可信”跳过认证。Artifact URI 需防 SSRF，内容需 malware scan、类型/大小限制和 DLP；外部 Agent 返回的自然语言与工具结果均是不可信输入。

## 5. 生产失败模式

- **混淆身份与能力发现**：发现了 AgentCard/MCP tool 不等于获得调用授权。
- **token 透传**：上游 bearer token 被下游任意 Server 重放。使用 token exchange、audience binding 和短时凭据。
- **目录投毒**：恶意 Server 修改工具描述或伪造 AgentCard。registry allowlist、签名、schema hash 与 owner 审批缺一不可。
- **版本漂移**：客户端自动回退导致安全字段悄悄丢失。固定 major/minor，兼容测试，不满足能力则 fail closed。
- **重试副作用**：MRTR/网络重试重复下单。写工具必须接收幂等键并返回可查询 operation id。
- **跨 Agent 级联**：一个被注入的 Artifact 污染共享记忆。标注 provenance/trust level，跨边界重新校验，不传播隐式权限。

## 6. 练习与验收

关联 OAuth 2.1/OIDC、workload identity、schema registry、SSRF、零信任、幂等、事件驱动长任务和[安全威胁模型](../06-platform/security-threat-model.md)。

练习：实现“读日历”MCP tool 和“差旅预订”A2A Agent。注入伪造 tool annotation、过期 token、错误 audience、重复请求、恶意 Artifact URL、旧协议版本与审批超时。

验收：未授权调用 100% fail closed；重复写不产生第二个订单；所有调用能由 `traceId + principal + schemaHash + policyDecisionId` 审计；Server 被撤销后 60 秒内不可再发现/调用；兼容矩阵覆盖当前版和一个明确支持的旧版。

## 官方资料

- [MCP 2026-07-28 规范发布说明](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [MCP 规范](https://modelcontextprotocol.io/specification/)
- [MCP 安全最佳实践](https://modelcontextprotocol.io/specification/draft/basic/security_best_practices)
- [A2A 1.0 规范](https://a2a-protocol.org/latest/specification)
- [A2A 项目仓库](https://github.com/a2aproject/A2A)
