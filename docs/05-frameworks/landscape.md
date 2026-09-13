# 主流 Agent Framework 全景与选型基线

> 基线日期：2026-08-30。Agent 框架变化很快，版本、稳定级别和迁移建议必须在每次架构评审时重新核对。框架不是 Runtime；它通常只解决模型循环、编排表达或集成体验中的一部分。

## 1. 先建立分层语言

不要把 `Agent`、`Workflow`、`Runtime`、`Platform` 混为一谈：

```text
产品/业务 Agent：提示词、工具、领域策略、输出契约
编排框架：节点/边、handoff、group chat、条件路由
企业 Runtime：状态机、恢复、审批、配额、幂等、策略、审计
平台控制面：目录、版本、发布、租户、模型/工具治理、评测
基础设施：队列、数据库、沙箱、模型供应商、遥测系统
```

框架可以替换，Runtime 的 `Run/Step/Event/Artifact/Approval` 语义应由企业自己掌握。否则一次 SDK 大版本升级会同时污染业务模型、持久化格式和运维体系。

## 2. 2026-08 官方现状

| 方案 | 当前定位 | 强项 | 生产边界与适合场景 |
|---|---|---|---|
| LangGraph | Python/JS 的低层、状态化编排 Runtime | typed state、checkpoint、interrupt/resume、流式与长任务 | 需要显式图、可恢复执行；团队愿意管理图状态与部署。不是模型/工具平台全家桶 |
| OpenAI Agents SDK | Python/TypeScript 的轻量 Agent SDK | agent loop、function tool、handoff/agent-as-tool、guardrail、session、tracing、MCP | OpenAI/Responses 生态、代码优先的中等复杂编排；企业策略、跨供应商语义和耐久队列仍应外置 |
| Microsoft Agent Framework | AutoGen 与 Semantic Kernel 团队的新一代 Python/.NET 框架 | agents、显式 workflow、checkpoint/resume、HITL、middleware、hosting | Microsoft/Azure/.NET 企业栈。新项目优先评估它；AutoGen/SK 项目按迁移指南渐进迁移 |
| AutoGen | 事件驱动多 Agent 研究与工程框架 | AgentChat/Core、消息驱动协作、分布式 runtime、研究模式 | 既有系统和多 Agent 实验仍有价值；官方已鼓励迁移到 Microsoft Agent Framework，不宜作为全新企业基线 |
| Semantic Kernel | 多语言 AI 集成内核与 Agent/Process 能力 | Kernel、plugin/filter、连接器、.NET/Java/Python 企业集成 | 既有 SK 应用、插件生态和渐进改造；其 Agent 能力正由 Microsoft Agent Framework 继承演进 |
| Google ADK 2.x | Python/TS/Go/Java 的 Agent 与 Workflow 开发套件 | template、graph、dynamic、collaborative workflow，评测与 Google Cloud 部署 | Google Cloud/Gemini/Vertex 体系或多语言团队；虽可接多模型，托管体验明显偏 Google 生态 |

CrewAI、LlamaIndex Workflows、Strands 等方案也可纳入评估。架构选型看同一组生产指标下的替换实验，不看框架清单背得多长。默认先用单 Agent 配合确定性代码；只有隔离上下文、并行能力或独立权限边界确有收益时才拆多 Agent。

## 3. 一组可移植的最小语义

```ts
type RunStatus = "queued" | "running" | "waiting" | "succeeded" | "failed";

interface AgentPort {
  invoke(input: {
    runId: string;
    messages: unknown[];
    allowedTools: string[];
    budget: { maxTurns: number; maxTokens: number; deadlineMs: number };
  }): AsyncIterable<
    | { type: "delta"; text: string }
    | { type: "tool_requested"; callId: string; name: string; args: unknown }
    | { type: "handoff_requested"; target: string; payload: unknown }
    | { type: "completed"; output: unknown }
  >;
}
```

业务代码只依赖 `AgentPort`；LangGraph state、OpenAI `Agent/run`、MAF workflow 或 ADK event 都在 adapter 中转换为内部事件。事件必须附带 `tenantId/runId/stepId/attempt/schemaVersion`，并在持久化前脱敏。

## 4. 生产权衡

| 决策问题 | 偏轻量 SDK | 偏显式工作流/图 | 偏自研 Runtime |
|---|---|---|---|
| 任务少于 3 步、无需恢复 | ✓ | 过重 | 不应自研 |
| 长任务、断点、人工审批 | 需外置耐久层 | ✓ | 有特殊合规时 |
| 多语言/多模型强可移植 | adapter 后可用 | adapter 后可用 | 核心语义自持 |
| 动态多 Agent 研究 | handoff 足够起步 | ✓ | 先证实价值 |
| 强隔离、细粒度策略、审计 | 平台补齐 | 平台补齐 | 必须由 Runtime/网关兜底 |

评估时做同一批 30~100 条真实任务，比较成功率、P95、平均/尾部 token、工具误调用、恢复成功率、人工介入率、trace 完整度和升级破坏面。不要用“Hello World 能跑”替代生产 PoC。

## 5. 常见失败模式

- **框架即架构**：直接把 SDK 对话对象序列化为数据库事实，升级后无法回放。修复：内部事件模型 + schema migration。
- **多 Agent 迷信**：角色越拆越多，token、延迟和错误传播同步增加。修复：证明上下文隔离/并行/权限边界至少一项收益。
- **隐藏循环**：框架默认无限或高轮次执行。修复：turn、token、wall-clock、工具调用和成本五重预算。
- **状态双写**：框架 checkpoint 与业务库分别写成功。修复：outbox/inbox、幂等键和明确的事实源。
- **观测锁定**：只依赖框架私有 trace。修复：导出 OpenTelemetry 并保留内部事件日志。

## 6. 关联知识

先学习[控制面与数据面](../06-platform/control-data-plane.md)、[可靠性与成本](../06-platform/reliability-cost.md)，再看[框架集成边界](framework-integration.md)。框架选型还依赖状态机、事件溯源、分布式幂等、OAuth/OIDC、沙箱和评测工程。

## 7. 练习与验收

用同一“工单分类 → 查 CMDB → 生成变更计划 → 人工批准”任务分别实现轻量 Agent loop 和显式图：

1. 注入模型 429、工具超时、进程重启和重复消息；两版都能从稳定点恢复。
2. 输出统一 `RunEvent`，trace 能重建每次工具调用与审批。
3. 预算超限在一次工具副作用前停止；副作用调用具有 idempotency key。
4. 写一页 ADR：为什么选/不选某框架，并给出退出成本。

验收标准：30 条回归集成功率差异可解释，恢复测试 100% 不产生重复副作用，框架替换不修改领域层接口。

## 官方延伸阅读

- [LangGraph overview](https://langchain-ai.github.io/langgraph/index.html)
- [OpenAI API quickstart：Agents SDK 示例](https://developers.openai.com/api/docs/quickstart)
- [Microsoft Agent Framework 文档](https://learn.microsoft.com/en-us/agent-framework/)
- [AutoGen → Microsoft Agent Framework 迁移](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/)
- [Semantic Kernel 文档](https://learn.microsoft.com/en-us/semantic-kernel/)
- [Google ADK 文档](https://adk-labs.github.io/adk-docs/)
