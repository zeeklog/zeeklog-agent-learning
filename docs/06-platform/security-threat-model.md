# 安全威胁模型、Prompt Injection 与红队

## 1. 安全目标与资产

Agent 的危险不止“说错话”，而是被诱导后能读取私有数据、调用工具、执行代码和影响其他 Agent。需要保护：用户/租户数据、系统提示与策略、凭据、记忆/向量库、工具与业务系统、模型供应链、Artifact、审计证据、预算和可用性。

```text
[Untrusted user/web/email/document]
              │ content ≠ instruction
      [Agent Runtime / Model] ── [Memory/RAG]
          │ intent                    │ poisoning boundary
      [Policy + Tool Gateway] ── [Sandbox/MCP/A2A]
          │ delegated identity          │ external trust boundary
      [Enterprise systems]          [Internet/other agents]
```

每条边标注身份、数据分级、协议、允许动作、校验、超时、网络出口和审计。最危险组合是：**可访问私密数据 + 会摄入不可信内容 + 存在可外传/写入的工具**；不要指望单个 injection classifier 化解结构性权限问题。

## 2. 用 OWASP Agentic Top 10 做检查表

2026 版包含：ASI01 Agent Goal Hijack、ASI02 Tool Misuse & Exploitation、ASI03 Identity & Privilege Abuse、ASI04 Agentic Supply Chain Vulnerabilities、ASI05 Unexpected Code Execution、ASI06 Memory & Context Poisoning、ASI07 Insecure Inter-Agent Communication、ASI08 Cascading Failures、ASI09 Human-Agent Trust Exploitation、ASI10 Rogue Agents。

| 攻击面 | 代表攻击 | 主要控制 |
|---|---|---|
| Goal/Prompt | 网页隐藏指令要求上传机密 | 数据/指令标记、最小工具集、egress deny、敏感动作审批 |
| Tool/Identity | 参数注入、越权、token 重放 | schema + 语义校验、task token、audience、PDP、幂等 |
| Memory/RAG | 恶意文档写入长期记忆 | provenance、信任等级、写入门禁、TTL、隔离索引 |
| Code/Supply chain | 恶意 MCP/skill/依赖、RCE | allowlist、签名/SBOM、沙箱、只读挂载、网络策略 |
| Multi-agent | 伪造 AgentCard、污染 Artifact、级联错误 | mTLS/JWS、重新鉴权、内容扫描、深度/扇出/预算上限 |
| Human trust | 欺骗性审批、输出伪造证据 | 展示真实 diff/资源/身份，二次确认，高风险双人审批 |

## 3. Prompt Injection 的正确认识

直接 injection 来自用户；间接 injection 藏在网页、邮件、Issue、代码、PDF、tool result 或另一个 Agent 的 Artifact 中。模型无法可靠地仅靠提示词区分“内容中的命令”，因此采用纵深防御：

1. 摄入时保留来源、作者、签名、时间和 trust label，清除主动内容但不宣称已“净化语义”。
2. 上下文模板显式隔离 policy、user intent、retrieved content；系统指令短而稳定。
3. 每次 tool call 在模型之外做参数 schema、资源授权、DLP、风险分类和目标域校验。
4. 将读与写、检索与外传分开；默认禁止任意 URL、任意 shell、任意文件系统和跨租户访问。
5. 高风险动作向人展示标准化事实：真实 endpoint、资源 ID、完整 diff、费用和不可逆性，而非让模型生成审批文案。
6. 限制 step、fan-out、token、时间、美元与数据量；异常时 fail closed 并保留证据。

```ts
async function secureToolCall(ctx: TrustedRun, intent: ToolIntent) {
  const tool = registry.getPinned(ctx.releaseId, intent.name);
  assert(jsonSchemaValidate(tool.schema, intent.args));
  const facts = resolveCanonicalResources(intent.args); // 不信任显示名/URL
  const destinations = extractActualDestinations(facts); // 规范化 host/IP，展开 redirect 仍需逐跳复核
  assert(destinations.every(host =>
    tool.allowedEgress.includes(host) && networkPolicy.allows(tool.id, host)
  )); // 有实际目的地但 allowlist 为空时必然拒绝
  const decision = await pdp.decide({ ctx, tool, facts, dataLabels: ctx.inputLabels });
  if (!decision.allow) throw new SecurityError(decision.reason);
  if (decision.approval) return approval.request(renderNonLlmDiff(facts, decision));
  const idempotencyKey = makeToolIdempotencyKey({
    tenantId: ctx.tenantId, runId: ctx.runId, stepId: ctx.stepId,
    callId: intent.callId, argsHash: canonicalHash(facts)
  });
  return sandbox.execute(tool, facts, { tokenRef: mintScopedToken(decision), idempotencyKey });
}
```

网络沙箱还要在 DNS 解析后校验目标 IP、连接时 pin 解析结果，并对每次重定向重新执行域名/IP 策略，防止 DNS rebinding 与允许域跳转到内网地址。`callId` 由模型/Provider 提供，只是关联标识；幂等键必须由可信 Runtime 加上 tenant/run/step 与规范参数摘要生成。

## 4. 沙箱、密钥与供应链

代码执行优先 disposable VM/microVM 或受限容器：非 root、只读 rootfs、临时工作区、CPU/内存/PID/磁盘/时限、syscall 限制、默认无网络、域名/IP 双重出口控制、禁止宿主 socket 与云 metadata。Windows/macOS 本地 Agent 还需明确 TCC/UAC、keychain、文件书签和进程继承边界。

模型和沙箱不应看到长期 secret。使用 vault/credential broker，在工具调用获批后签发 resource/audience/task-scoped 短凭据；日志只留 token reference。MCP server、skill、prompt 模板、模型与依赖都进入 AI-BOM/SBOM，固定版本、签名、来源、owner、CVE 和撤销状态；运行期下载同样受监控。

## 5. 红队与安全测试

建立可重复 corpus，而不是一次聊天：

- 直接/间接/编码/多语言 injection；工具返回中嵌套指令；跨轮延迟触发。
- 数据外泄：URL query、DNS、邮件、Artifact、错误日志、模型上下文和 side channel。
- 身份：伪造 tenant、confused deputy、token audience/issuer、审批重放、撤销竞态。
- 工具：路径穿越、SSRF、shell 参数、schema 边界、重复写、TOCTOU。
- Agent：伪造 AgentCard、恶意 Artifact、循环 handoff、级联 hallucination、共享记忆投毒。
- 资源：超长输入、tool bomb、fan-out、沙箱 fork/disk bomb、预算耗尽。

对每例记录 precondition、attack、expected control、actual result、trace 与修复 owner。评测指标包括攻击成功率、误拦率、越权副作用数、泄露字节、检测时间和证据完整度；高风险 escape 是发布阻断项。

## 6. 事故响应

预置 kill switch：禁用 agent release/tool/MCP server/model、撤销 token/证书、冻结记忆写入和外网出口。保留不可篡改事件、release digest、策略版本、tool args hash、Artifact lineage；隔离受影响 cell/租户。恢复前轮换凭据、清理被投毒记忆/索引、回放受影响 run、补偿副作用并把攻击加入回归集。

## 7. 失败模式

- **只加分类器**：分类器会漏报且可被绕过；权限与 egress 必须结构性收紧。
- **HITL 万能论**：审批 UI 可被冗长/欺骗文本操纵。展示非 LLM 生成的规范事实。
- **Server 自报安全属性即可信**：tool annotation/AgentCard 是输入；registry 和策略才是事实源。
- **沙箱有容器就安全**：宿主挂载、网络、metadata、内核漏洞仍可突破。按威胁等级选隔离。
- **安全日志保存全部内容**：反而形成高价值泄露库。最小化、脱敏和分权访问。

## 8. 练习与验收

关联 STRIDE、attack tree、零信任、OAuth/OIDC、SSRF、sandbox escape、SLSA/SBOM、DLP、MITRE ATLAS、NIST AI RMF。配合[MCP/A2A](../05-frameworks/mcp-a2a.md)学习。

练习：对“可读邮件、检索内部文档、发送消息”的 Agent 完成 DFD + STRIDE，并实现 30 条 injection/exfiltration 自动测试和一次 tabletop incident。

验收：测试中私密数据不可到达未授权 egress；写工具均经外部 PDP；撤权/kill switch 60 秒生效；沙箱无法访问宿主凭据/metadata；任一告警可追溯到 release、principal、policy 和具体副作用。

## 官方资料

- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
- [OWASP LLM/GenAI Security Project](https://genai.owasp.org/)
- [MITRE ATLAS](https://atlas.mitre.org/)
- [NIST AI 600-1：Generative AI Profile](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf)
- [SLSA Specification](https://slsa.dev/spec/)
