# Tool System：Schema、权限、沙箱与幂等副作用

Tool System 把模型提议转换为受控执行。JSON Schema 只描述接口，真正的边界来自授权、资源范围、审批、幂等和副作用一致性。

## 1. Tool 是协议面，Capability 才是安全面

一个 Tool 通常包含名称、描述、输入/输出 schema 和执行函数。它告诉模型“能做什么”，但不应单独决定“本次能不能做”。企业 Runtime 需要额外的 Capability/Policy：

```ts
type Risk = "read" | "write" | "execute" | "external_side_effect";

interface ToolDefinition<I = unknown, O = unknown> {
  name: string;
  version: string;
  description: string;
  inputSchema: object;
  outputSchema: object;
  risk: Risk;
  timeoutMs: number;
  concurrencyKey?: (input: I) => string;
  execute(input: I, ctx: ToolExecutionContext): Promise<O>;
}

interface ToolExecutionContext {
  runId: string;
  callId: string;
  principal: { tenantId: string; userId: string };
  capability: { resources: string[]; expiresAt: string };
  idempotencyKey: string;
  retryPolicy: { version: string; retryKnownFailure: boolean };
  signal: AbortSignal;
}
```

同一个 `read_file` 定义，在 Run A 可能只获准读取 `/repo/src/**`，在 Run B 可能完全不可见。Tool Registry 管定义；Policy Decision Point 根据主体、Run、资源、风险和组织策略裁决；Executor 只接受已签发的短期执行票据。模型不参与权限判断。

## 2. 从模型输出到执行的七道门

1. **解析**：完整聚合流式 tool call，拒绝重复 callId、无效 JSON 和未知工具；
2. **Schema 校验**：输入必须匹配注册版本，默认拒绝额外字段；
3. **规范化**：路径 canonicalize、URL 解析、枚举归一，得到稳定参数；
4. **策略裁决**：检查 principal、workspace、数据分类、网络域、风险和额度；
5. **人工批准**：高风险操作展示完整规范化参数、影响范围和理由；
6. **隔离执行**：超时、取消、沙箱、输出上限、并发限制；
7. **结果校验与审计**：校验 output schema，脱敏，保存 receipt，再作为 observation 回给模型。

审批必须发生在规范化之后，并绑定 `tool@version + canonicalArgsHash + capabilityScope + expiry`。否则模型可让用户批准相对路径 `./report`，执行时工作目录变化后写到另一位置。

## 3. Schema 是运行时合约

TypeScript 类型会在运行时消失；工具边界必须做 JSON Schema/Zod/Ajv 校验。MCP Tool 的 `inputSchema` 与可选 `outputSchema` 都是协议的一部分，客户端不能因为服务端“声称只读”就信任 annotations。

```ts
import Ajv, { JSONSchemaType } from "ajv";

type WriteFileInput = { path: string; content: string; expectedSha256?: string };
const schema: JSONSchemaType<WriteFileInput> = {
  type: "object",
  additionalProperties: false,
  properties: {
    path: { type: "string", minLength: 1, maxLength: 4096 },
    content: { type: "string", maxLength: 1_000_000 },
    expectedSha256: { type: "string", nullable: true, pattern: "^[a-f0-9]{64}$" }
  },
  required: ["path", "content"]
};

const validate = new Ajv({ allErrors: true }).compile(schema);
export function parseWriteFile(value: unknown): WriteFileInput {
  if (!validate(value)) {
    throw new ToolInputError(validate.errors?.map(e => `${e.instancePath} ${e.message}`).join("; "));
  }
  return value;
}

class ToolInputError extends Error {}
```

描述也属于合约：写明单位、格式、前置条件、是否产生副作用和错误语义。避免“处理文件”这类模糊工具；让工具小而正交，但也不要拆到每次任务需要几十次往返。Schema 变更采用新版本或兼容扩展；正在恢复的 Run 固定使用当时的工具版本与 schema hash。

## 4. 路径、网络与命令的真实边界

### 文件系统

只做字符串 `startsWith(workspace)` 会被 `..`、符号链接、大小写和 Junction 绕过。应解析真实路径并验证其位于授权 root；创建新文件时验证最近存在父目录的 realpath，写入后再次核验。Windows 还要考虑盘符、UNC、保留设备名；macOS 要考虑符号链接与大小写不敏感卷。

### 网络

URL Tool 需限制 scheme/domain/port，解析 DNS 后阻断 private、loopback、link-local 和云 metadata 地址，并对每次重定向重新校验，防 SSRF/DNS rebinding。认证令牌由凭据代理按目标域注入，不进入模型参数。

### Shell

优先提供结构化领域工具，而不是万能 shell。必须执行命令时，避免 `shell: true` 拼接用户字符串；用 argv 数组、固定 executable、受限 cwd/env、资源限额和沙箱。展示批准信息时不能截掉尾部参数。

本地 MCP Server 与客户端同权限运行时风险极高。首装需展示精确启动命令并明确批准；优先 stdio 或受保护 IPC。若用本地 HTTP，绑定 loopback、校验 Origin 并认证。

## 5. Policy：Allow/Deny/Require Approval

```ts
type Decision =
  | { effect: "deny"; code: string; userMessage: string }
  | { effect: "approval"; reason: string; expiresInMs: number }
  | { effect: "allow"; capabilityToken: string };

interface PolicyInput {
  principal: { tenantId: string; userId: string; roles: string[] };
  runId: string;
  tool: { name: string; version: string; risk: Risk };
  args: unknown;
  normalizedResources: string[];
  dataClassifications: string[];
}

function decide(p: PolicyInput): Decision {
  if (p.normalizedResources.some(r => r.includes("/.ssh/"))) {
    return { effect: "deny", code: "SENSITIVE_PATH", userMessage: "禁止访问凭据目录" };
  }
  if (p.tool.risk === "external_side_effect" || p.tool.risk === "execute") {
    return { effect: "approval", reason: "操作将影响外部系统或执行代码", expiresInMs: 60_000 };
  }
  return { effect: "allow", capabilityToken: signShortLivedCapability(p) };
}

declare function signShortLivedCapability(input: PolicyInput): string;
```

Policy 需版本化，决策输入与结果进入审计事件。Capability Token 至少绑定 principal、runId、callId、tool/version、resources、args hash、expiry 与 nonce，Executor 重新验签；不能把 UI 上一个 `approved=true` 布尔值当授权。

## 6. 幂等、并发控制与结果收据

工具调用的幂等键应由 Runtime 生成，如 `tenant/run/step/call/attempt-policy`，而非相信模型给出的字段。重复同一逻辑调用返回原 receipt；“重试生成一次新的内容”才创建新 attempt。

```ts
interface Receipt<O> {
  key: string;
  status: "started" | "succeeded" | "failed" | "unknown";
  output?: O;
  externalRef?: string;
  argsHash: string;
}

async function executeOnce<I, O>(
  tool: ToolDefinition<I, O>, input: I, ctx: ToolExecutionContext, receipts: ReceiptStore
): Promise<O> {
  const argsHash = stableHash(input);
  const old = await receipts.get<O>(ctx.idempotencyKey);
  if (old && old.argsHash !== argsHash) {
    throw new IdempotencyConflictError(ctx.idempotencyKey);
  }
  if (old?.status === "succeeded") return old.output as O;
  if (old?.status === "started" || old?.status === "unknown") {
    throw new IndeterminateExecutionError(ctx.idempotencyKey); // 先 reconcile，禁止 begin/execute
  }

  if (old?.status === "failed") {
    if (!ctx.retryPolicy.retryKnownFailure) throw new PriorToolFailureError(ctx.idempotencyKey);
    // ledger 在事务中校验旧状态、参数摘要和 policy 版本，再开始新 attempt。
    await receipts.retryKnownFailure(ctx.idempotencyKey, argsHash, ctx.retryPolicy.version);
  } else {
    await receipts.begin({ key: ctx.idempotencyKey, argsHash });
  }
  try {
    const output = await withTimeout(tool.execute(input, ctx), tool.timeoutMs, ctx.signal);
    await validateOutput(tool.outputSchema, output);
    await receipts.succeed(ctx.idempotencyKey, output);
    return output;
  } catch (error) {
    const classified = classifyToolError(error);
    // timeout、进程丢失、连接中断只说明“没有拿到结果”，不说明副作用没发生。
    if (classified.outcome === "unknown") {
      await receipts.unknown(ctx.idempotencyKey, classified);
    } else {
      await receipts.fail(ctx.idempotencyKey, classified);
    }
    throw error;
  }
}

declare const stableHash: (x: unknown) => string;
declare const withTimeout: <T>(p: Promise<T>, ms: number, signal: AbortSignal) => Promise<T>;
declare const validateOutput: (schema: object, value: unknown) => Promise<void>;
type ClassifiedToolError = { code: string; outcome: "definitely_failed" | "unknown" };
declare const classifyToolError: (e: unknown) => ClassifiedToolError;
declare interface ReceiptStore {
  get<O>(key: string): Promise<Receipt<O> | undefined>;
  begin(x: Pick<Receipt<never>, "key" | "argsHash">): Promise<void>;
  succeed<O>(key: string, output: O): Promise<void>;
  fail(key: string, error: unknown): Promise<void>;
  unknown(key: string, error: unknown): Promise<void>;
  retryKnownFailure(key: string, argsHash: string, policyVersion: string): Promise<void>;
}
class IndeterminateExecutionError extends Error {}
class IdempotencyConflictError extends Error {}
class PriorToolFailureError extends Error {}
```

`started` 后崩溃形成 **unknown outcome**，不能自动假设失败。应先用 externalRef 或业务查询确认；无法确认的不可逆操作转人工处理。针对同一文件/账户的写操作使用 `concurrencyKey` 串行化，并结合版本号/ETag 做乐观并发。

## 7. 结果不是一段任意字符串

建议统一 ToolResult：`status/data/content/artifacts/error/metrics`。大输出保存为 artifact，模型仅拿摘要和 URI；二进制不转成巨型 base64 prompt。错误分类为：

- `INVALID_INPUT`：可让模型修正参数；
- `PERMISSION_DENIED`：不得换写法绕过，应提示用户；
- `TRANSIENT`：按 policy 重试；
- `BUSINESS_REJECTED`：业务条件不满足；
- `UNKNOWN_OUTCOME`：必须查询确认或人工介入。

不要把内部堆栈、数据库语句或凭据回给模型。审计记录可包含 error fingerprint，敏感正文独立加密保存。

## 8. MCP 适配层

MCP 提供发现、`tools/list`、`tools/call`、结构化结果和 transport，但不替代你的 Policy。远端 server 返回的 annotations、schema 和资源链接均按不可信输入处理：限制 tool 数量/描述长度、校验 schema 复杂度；在 2026-07-28 协议中，通过客户端主动打开的 `subscriptions/listen` 接收 `list_changed`，任何目录变化都重新走注册审核。

Protocol Error 表示 JSON-RPC/未知方法等协议问题；Tool Execution Error 是模型可能修正的业务执行问题，两者不应统一成“工具失败”。需要按协议版本处理取消：2026-07-28 Streamable HTTP 为每个请求建立响应流，客户端 abort/timeout 关闭该请求流就是取消信号，不再 POST `notifications/cancelled`；stdio 与兼容旧版由 SDK 映射旧取消通知。长任务使用 `io.modelcontextprotocol/tasks` 的 `tasks/get/update/cancel`，变更订阅则使用 `subscriptions/listen`，不能继续套用旧 session event-ID 恢复方案。

### 8.1 工具发现、命名与供应链

工具越多，选择错误率和 Context 成本越高。按租户策略先过滤 Registry，再通过命名空间、任务意图或轻量 tool search 暴露小集合；最终执行仍按注册 ID 查找，绝不能让模型提供任意模块路径。名称稳定且表达领域动作，破坏性升级发布新版本，旧 Run 固定旧版本。

本地插件/MCP 包属于软件供应链：记录来源、签名、包哈希、启动命令和依赖锁；自动更新后重新评估权限，不能沿用旧版本的批准。Tool 描述变化也可能改变模型行为，应纳入 Eval 与灰度。远端工具列表突然变化时先隔离新能力，对正在运行的 Run 保持 tool-set snapshot，避免同一历史恢复后出现不同工具。

## 9. 失败模式

| 失败模式 | 风险 | 防线 |
| --- | --- | --- |
| TypeScript 类型代替运行时校验 | 原型污染、越界参数 | JSON Schema 严格校验、拒绝额外字段 |
| 批准前后参数不一致 | 审批绕过 | canonical args hash 绑定票据 |
| 信任 MCP annotations | 恶意工具伪装只读 | 本地 risk 分类和策略覆盖 |
| `startsWith` 检查路径 | `..`/symlink 越权 | realpath + root containment + 写后复核 |
| 网络工具任意 URL | SSRF、凭据外泄 | egress allowlist、IP/redirect/DNS 防护 |
| 超时后直接重试写操作 | 重复副作用 | receipt + 幂等键 + unknown outcome 查询 |

## 10. 测试、练习与验收

测试不仅跑 happy path，还应 fuzz JSON 参数，覆盖 `../`、符号链接、UNC、重定向到 `169.254.169.254`、重复 callId、超大输出、取消竞态、receipt 已 started 等场景。Policy 用表驱动测试固定组织规则。

**练习**：实现 `publish_release` 工具：仅允许指定 GitHub 仓库，必须展示 tag、commit SHA、release notes hash 并审批；网络超时后能查询 tag/release 是否已创建；重复调用不创建第二个 Release。

**验收点**：

- [ ] 输入和输出均有运行时 schema 校验，工具版本与 schema hash 可追踪。
- [ ] Tool Registry、Policy、Approval、Executor 四层职责独立。
- [ ] 文件、网络、shell 均有真实资源边界，而非 Prompt 约束。
- [ ] 每个写/外部副作用有幂等键、receipt 和 unknown outcome 策略。
- [ ] MCP Server 被视为不可信能力提供方，不能自行授予权限。

## 延伸阅读

- [MCP 2026-07-28 发布与迁移概要](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [MCP 最新规范](https://modelcontextprotocol.io/specification/)
- [MCP：Security Best Practices](https://modelcontextprotocol.io/specification/draft/basic/security_best_practices)
- [JSON Schema 2020-12](https://json-schema.org/draft/2020-12)
- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
