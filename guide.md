# Agent Runtime 阅读指南与学习路径

按四条路线组织 Agent 学习。根据自己的任务选一条，再用参考项目把知识写成代码和证据。

## 推荐顺序

### 从 Agent 基础开始

先看 [架构能力地图](docs/00-roadmap/capability-map.md)，再读 [LLM 系统原理](docs/01-foundations/llm-systems.md)、[模型接口](docs/01-foundations/model-io.md) 和 [检索、上下文与评测基础](docs/01-foundations/retrieval-evaluation.md)。接着用 README 中的 [Hello-Agents 对照表](README.md#hello-agents)跑一遍 ReAct、Plan-and-Solve 和 Reflection 的最小例子。

### 从 Runtime 深入

从 [Runtime 总体架构](docs/03-runtime/overview.md)开始，依次阅读 Agent Loop、执行契约、持久化、Tool System、Context、流式恢复和可观测性。重点看模型、进程、网络或工具失败后，系统如何回到可解释的状态。

### 从桌面端深入

从 [AI 桌面客户端](docs/02-desktop/overview.md)进入，重点看进程模型、Typed IPC、Coding Agent Host、安全和更新。练习要覆盖 Renderer 零信任、本地工作区授权、Sidecar 生命周期、凭据存储和补丁回滚。

### 从 SDK 与平台深入

先建立 [统一 SDK 的稳定边界](docs/04-sdk/overview.md)，再读 Provider Adapter、Framework 集成、任务控制平面、模型网关、执行环境和能力供应链。这里要把供应商 API 留在 Adapter 内，把租户、预算、权限、审计和发布门禁放在平台边界内。

## 两套外部路线

- 刚入门：按 [Agent-Learning-Hub Stage 0–2](README.md#agent-learning-hub)做一个计算器或研究助手，再回到本地 Runtime 章节补失败处理。
- 已经会写 Demo：按 Hub Stage 3–8 研究一个 Harness，完成 Skill、浏览器操作、评测和发布；同时用 [Hello-Agents 第四章至第十二章](README.md#hello-agents)补上经典范式、框架、协议和评测。
- 想做完整项目：从 Hub Project Ladder 选择一级，使用[参考项目](docs/07-practice/reference-project.md)作为代码底座，不必每个阶段重新搭脚手架。

外部项目提供例子和横向视角；本地章节把这些例子放进状态、权限、可靠性和可观测性约束中。

## 每章留下可检查的结果

开始前用 [架构能力地图](docs/00-roadmap/capability-map.md)记录基线。每完成一个主题，把结果放进同一套 [参考项目](docs/07-practice/reference-project.md)，至少留下以下一种证据：

- 一张标明信任边界、状态和失败路径的架构图；
- 一段可运行代码及其契约测试或故障注入结果；
- 一份 ADR，写清约束、备选方案和退出条件；
- 一组 Trace、Eval 或运行指标，证明设计在异常条件下仍成立。

第 6 周和第 12 周使用 [架构自测与验收](docs/00-roadmap/assessment.md)复测。比较设计与故障处理质量，不比较记忆数量。

## 资料边界

- **生产决策**记录边界、约束、失败模式和观测方法。
- **示意代码**表达稳定接口或设计模式；涉及外部 SDK 时，以项目锁定版本的官方类型为准。
- **练习与验收**给出可复现的完成条件，避免把“读过”当成“掌握”。
- **厂商和外部事实**附一手来源，并注明适用版本或核验日期。推导出的增强设计标为参考方案。
