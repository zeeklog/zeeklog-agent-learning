# Agent Runtime 架构自测与验收

## 验证方法

Agent Runtime 架构能力要通过**解释、设计、实现、故障处理**四类工程证据验证。评分不仅检查 happy path，也检查失败语义、测试方法和治理边界。每项 0～4 分：

- 0：无法解释；1：能复述概念；2：能实现 happy path；3：能处理失败并给出测试；4：能平台化、量化和治理演进。

## 1. 架构设计演练（45 分钟）

### 演练场景

设计一个 Windows/macOS 企业 AI 客户端：用户选择一个代码工作区，Agent 可检索、编辑文件、运行测试、请求人工批准后执行外部命令；任务最长 2 小时；应用或电脑重启后能继续；支持两个模型 Provider；管理员可以禁用工具并审计每次操作。

### 验收维度

| 维度 | 0～2 分 | 3 分 | 4 分 |
| --- | --- | --- | --- |
| 边界 | 罗列组件 | 画出进程/网络/信任边界 | 同时说明为什么放在该边界与替代方案 |
| 状态 | 只有消息历史 | Run/Step/Tool 有状态机 | 说明持久化点、版本、恢复与并发 |
| 控制面 | 只有 FIFO 队列 | admission、配额、lease | digest 防重、公平调度、fencing 与收尾预算 |
| 验证 | 复述模型结论 | 测试与 diff | 独立 Verifier、环境指纹、Claim-Evidence、部分/未知语义 |
| 工具 | 普通函数调用 | schema/timeout/approval/audit | 幂等、沙箱、结果裁剪、策略与撤销 |
| SDK | 单一 Provider 接口 | adapter + normalized events | capability negotiation、扩展与兼容测试 |
| 可靠性 | “失败就重试” | 分类重试、取消、回退 | 半完成副作用、补偿、SLO、chaos test |
| 安全 | 登录与加密 | least privilege、密钥、IPC allowlist | prompt injection、MCP、供应链、审计完整性 |
| 观测 | 日志 | run trace + metrics | replay + eval + rollout gate + cost attribution |

基础线：28/36；生产级进阶线：32/36，且安全、状态、控制面、验证、工具五项不得低于 3。

## 2. 实现练习：可恢复 Tool Step（120 分钟）

实现以下接口：

```ts
interface DurableStepStore {
  claim(runId: string, stepId: string, leaseMs: number): Promise<
    | { status: 'acquired'; fencingToken: number }
    | { status: 'busy' | 'done' }
  >;
  append(expectedSeq: number, fencingToken: number, event: StepEvent): Promise<void>;
  load(runId: string): Promise<readonly StepEvent[]>;
}

interface ToolExecutor {
  execute(request: ToolRequest, signal: AbortSignal): Promise<ToolResult>;
}
```

要求：

- 同一 `idempotencyKey` 并发请求只产生一次副作用。
- 进程可在 execute 前、执行中、执行后、completed event 前任意崩溃。
- 超时与用户取消可区分；未知结果不得自动重试写操作。
- 测试至少覆盖重复投递、lease 过期、取消、崩溃恢复。
- 证明 lease 过期的旧执行者不能追加事件或提交 artifact；相同幂等键但不同请求摘要必须冲突。

验收重点是副作用语义。通用 exactly-once 无法凭空获得，系统必须通过幂等端点、状态核对或人工调解处理不确定结果；ORM 和代码风格不影响这条结论。

## 3. 代码审查练习

找出以下实现的风险：

```ts
ipcMain.handle('agent:run-tool', async (_event, payload) => {
  const tool = tools[payload.name];
  return tool(payload.args);
});
```

至少应指出：来源窗口未验证、通用 channel 暴露全部工具、输入无 schema、无 tenant/user/workspace context、无权限策略/审批、无超时取消、无幂等、无审计、返回值可能泄密或过大、错误信息可能暴露本地路径、Renderer XSS 可直接越权。进一步应提出 Renderer → preload 窄 API → Main policy → Sidecar executor 的修复结构。

## 4. 事故响应演练

### 事故

发布后出现少量用户文件被同一 Agent 修改两次。Trace 显示 Provider 只请求一次工具，但 Runtime 在重连后记录两次 `tool.completed`；有一部分日志缺失。

### 参考排查路径

1. 先停用写工具或切只读，保护用户数据。
2. 按 `runId/callId/idempotencyKey` 查询事件、工具审计和文件变更记录。
3. 区分重复消费、ACK 丢失、Event Store 事务边界、客户端重放、工具自身非幂等。
4. 不以“Provider 只请求一次”排除 Runtime，因为执行与记录之间存在不确定窗口。
5. 修复包含唯一约束/幂等、outbox 或状态核对、缺失日志补强、回放测试和受影响数据修复方案。

## 5. 综合项目验收

参考项目需完成以下演示：

- 两个 Provider/Fake Provider 在相同 Contract Tests 下工作。
- 一个 Run 中包含文本、工具审批、写文件、测试和最终 Artifact。
- 在 5 个指定时间点 kill 进程，重启后恢复且不重复写入。
- 用户取消在 1 秒内停止新 Step，并让 Provider/Tool 收到 signal。
- Prompt Injection 样本无法越过 Tool Policy。
- Trace 可解释 token、latency、tool、error、cost；Eval 对改动前后给出结果。
- 切换 SDK transport 后 UI 业务代码无需修改。
- 审批绑定参数/diff/policy 摘要，篡改后必须重新审批；Verifier 能阻止无证据的“已完成”。
- 模拟 webhook 重复、SSE cursor 过期、环境清理失败和外部效果未知，系统均有明确恢复/对账路径。

## 6. 复测节奏

- 学习开始：记录基线，不追求高分。
- 第 6 周：重点复测 Runtime/Tool/Recovery。
- 第 12 周：完成架构设计、实现和事故响应演练。
- 首次接入真实系统约束后：重新验证一次，修正学习项目中的理想化假设。

下一模块：[LLM 系统原理 →](../../docs/01-foundations/llm-systems.md)
