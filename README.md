# Zeeklog Agent Learning

面向工程师的 Agent 学习资料：从 LLM 与分布式系统基础开始，经过 Agent Loop、工具、检索、记忆、协议和多 Agent，最后落到桌面端、可恢复 Runtime、统一 SDK 与平台治理。

本仓库是从极客日志文档中抽取的独立知识库，只保留 AI 学习内容。它把 [datawhalechina/hello-agents](https://github.com/datawhalechina/hello-agents) 与 [datawhalechina/Agent-Learning-Hub](https://github.com/datawhalechina/Agent-Learning-Hub) 接入同一条学习路线，提供章节映射、练习建议和工程验收标准。

## 仓库边界

这里包含：

- Agent 原理、Runtime、SDK、Framework、桌面端和平台治理方面的架构笔记；
- 学习路线、Project Ladder 对照、练习、ADR 模板和生产检查清单；
- 外部项目的公开链接、章节映射和版本记录。

这里不包含：

- 极客日志网站的业务代码、Next.js/Prisma 配置、数据库、部署脚本或环境变量；
- 用户数据、生产凭据、内部地址、构建产物和截图；
- `hello-agents` 的正文、图片或示例代码副本。

## 先从这里开始

如果刚接触 Agent，按下面顺序阅读：

1. [架构能力地图](docs/00-roadmap/capability-map.md)，确认自己的基础和缺口。
2. [LLM 系统原理](docs/01-foundations/llm-systems.md)与[模型接口、流式与结构化输出](docs/01-foundations/model-io.md)。
3. [Runtime 总体架构](docs/03-runtime/overview.md)，建立 Run、Task、Event、Tool 的边界。
4. [执行契约、规划与验证](docs/03-runtime/execution-contract-verification.md)，再实现一个最小 Agent Loop。
5. [贯穿式参考项目](docs/07-practice/reference-project.md)，把每个练习留下代码、测试或故障证据。

已经会调用模型 API 的读者，可以直接从下方的 [Agent-Learning-Hub Stage 2](#agent-learning-hub) 或 [Hello-Agents 第二部分](#hello-agents)开始，然后回到 Runtime 和平台章节补生产语义。

## 本地章节地图

| 模块 | 解决的问题 | 入口 |
| --- | --- | --- |
| 00 · 路线 | 先修项、12 周计划、能力测评 | [路线与验收](docs/00-roadmap/capability-map.md) |
| 01 · 基础 | LLM 接口、结构化输出、分布式系统、检索与评测 | [基础知识](docs/01-foundations/llm-systems.md) |
| 02 · 桌面端 | Electron/本地主机、IPC、凭据、沙箱、更新 | [桌面端架构](docs/02-desktop/overview.md) |
| 03 · Runtime | Agent Loop、Workflow、事件、工具、上下文、恢复 | [Runtime 总览](docs/03-runtime/overview.md) |
| 04 · SDK | Provider、事件协议、Transport、兼容性测试 | [统一 SDK](docs/04-sdk/overview.md) |
| 05 · Framework | 框架选型、Adapter、MCP、A2A | [Framework 与协议](docs/05-frameworks/landscape.md) |
| 06 · 平台 | 控制面、模型网关、环境、供应链、多租户、成本 | [平台架构](docs/06-platform/overview.md) |
| 07 · 实战 | 参考架构、可运行内核、Kata、生产清单、ADR | [参考项目](docs/07-practice/reference-project.md) |
| 08 · 附录 | 术语、速查表、覆盖矩阵和一手资料 | [资料索引](docs/08-appendix/reading-list.md) |

## Hello-Agents

[datawhalechina/hello-agents](https://github.com/datawhalechina/hello-agents) 是一套从零开始构建智能体的中文教程，按“基础 → 经典范式 → 框架与自研 → 高级主题 → 综合案例”展开。下面的表把它的 16 章接到本地章节；左侧链接始终指向上游原文。

| Hello-Agents 内容 | 在本仓库的对应阅读 | 学习产物 |
| --- | --- | --- |
| 前言、第一章至第三章：智能体概念、发展史、LLM 基础 | [LLM 系统原理](docs/01-foundations/llm-systems.md)、[架构能力地图](docs/00-roadmap/capability-map.md) | 画出 chatbot、workflow、agent、multi-agent 的边界 |
| 第四章：ReAct、Plan-and-Solve、Reflection | [执行契约、规划与验证](docs/03-runtime/execution-contract-verification.md)、[Agent Loop](docs/03-runtime/agent-loop.md) | 一个有步数、超时、终止条件和工具调用的最小 Loop |
| 第五章：Coze、Dify、n8n 等低代码平台 | [Build vs Buy](docs/05-frameworks/build-vs-buy.md) | 用决策矩阵说明何时用工作流，何时引入 Agent |
| 第六章：AutoGen、AgentScope、LangGraph 等框架 | [Framework 全景](docs/05-frameworks/landscape.md)、[Adapter 实战](docs/05-frameworks/hands-on-adapters.md) | 用同一组 Contract Test 对比两个框架 |
| 第七章：从零构建 HelloAgents 框架 | [统一 SDK](docs/04-sdk/overview.md)、[可运行 Runtime Kernel](labs/runtime-kernel/README.md) | 把模型、Tool、Event、Approval 放进稳定端口 |
| 第八章：记忆与检索 | [检索、上下文与评测基础](docs/01-foundations/retrieval-evaluation.md)、[Memory 与 State](docs/03-runtime/memory-state.md) | 带来源引用的研究助手，区分短期上下文和长期记忆 |
| 第九章：上下文工程 | [Context Engineering](docs/03-runtime/context-engineering.md) | 记录上下文预算、裁剪规则、压缩和恢复测试 |
| 第十章：MCP、A2A、ANP 等通信协议 | [MCP 与 A2A](docs/05-frameworks/mcp-a2a.md)、[Transport 与扩展](docs/04-sdk/transport-extensions.md) | 协议版本、能力协商、取消、错误和权限映射表 |
| 第十一章：Agentic-RL | [模型接口](docs/01-foundations/model-io.md)、[评测与观测](docs/03-runtime/observability-evals.md) | 把训练指标与 Agent 运行指标分开；训练主题作为外部补充阅读 |
| 第十二章：性能评估 | [可观测性与 Evals](docs/03-runtime/observability-evals.md)、[兼容性测试](docs/04-sdk/compatibility-testing.md) | 固定数据集、Trace、失败分类和回归门槛 |
| 第十三章至第十五章：旅行助手、深度研究、赛博小镇 | [端到端参考架构](docs/07-practice/reference-architecture.md)、[多 Agent 编排](docs/03-runtime/multi-agent-orchestration.md) | 先写任务边界和证据链，再决定是否拆分 Agent |
| 第十六章：毕业设计 | [贯穿式参考项目](docs/07-practice/reference-project.md)、[生产检查清单](docs/07-practice/production-checklists.md) | 一个可运行、可观测、可回滚的完整项目 |

建议读法：先读上游章节获得直观例子，再回到本地对应章节补状态、权限、失败处理和验收。上游教程中的示例代码按其仓库版本运行，不把示例接口当成长期稳定的公共契约。

<a id="agent-learning-hub"></a>

## Agent-Learning-Hub：Stage 0–8

[datawhalechina/Agent-Learning-Hub](https://github.com/datawhalechina/Agent-Learning-Hub) 是一份持续更新的学习清单，重点放在可运行项目、Agent Harness、Skills、协议、浏览器 Agent、评测安全和发布。下面保留它的学习顺序，并给出本地落点。

| Hub 阶段 | 要掌握的内容 | 本地落点 |
| --- | --- | --- |
| Stage 0 · Understand What an Agent Is | 区分 chatbot、workflow、agent；知道何时不该使用 Agent | [能力地图](docs/00-roadmap/capability-map.md)、[Build vs Buy](docs/05-frameworks/build-vs-buy.md) |
| Stage 1 · Build a Minimal Agent Loop | 对话、结构化输出、Function Calling、步数与超时 | [模型接口](docs/01-foundations/model-io.md)、[Agent Loop](docs/03-runtime/agent-loop.md) |
| Stage 2 · Tool Use, RAG and Memory | 工具失败、检索、引用、短期和长期记忆 | [检索基础](docs/01-foundations/retrieval-evaluation.md)、[Tool System](docs/03-runtime/tool-system.md)、[Memory](docs/03-runtime/memory-state.md) |
| Stage 3 · Study a Modern Agent Harness | 工具注册、权限、Session、上下文压缩、Trace | [Coding Agent Host](docs/02-desktop/coding-agent-host.md)、[Runtime 总览](docs/03-runtime/overview.md) |
| Stage 4 · Multi-Agent Is Coordination | Planner、Executor、Reviewer、停止条件、循环和上下文膨胀 | [多 Agent 编排](docs/03-runtime/multi-agent-orchestration.md) |
| Stage 5 · Skills, Protocols and Capability Packaging | Skill 与 Tool 的区别，MCP/A2A/ACP，能力包的版本和撤销 | [能力供应链](docs/06-platform/capability-supply-chain.md)、[MCP 与 A2A](docs/05-frameworks/mcp-a2a.md) |
| Stage 6 · Browser and Computer-Use Agents | 页面观察、动作边界、失败恢复、截图与审计 | [桌面安全](docs/02-desktop/security.md)、[Context Engineering](docs/03-runtime/context-engineering.md) |
| Stage 7 · Evaluation, Observability and Safety | 固定测试集、Trace、成本、延迟、Prompt Injection 和人工确认 | [可观测性与 Evals](docs/03-runtime/observability-evals.md)、[安全威胁模型](docs/06-platform/security-threat-model.md) |
| Stage 8 · Ship a Real Agent | 明确用户和成功标准，加入日志、重试、权限、部署和 README | [参考架构](docs/07-practice/reference-architecture.md)、[生产清单](docs/07-practice/production-checklists.md) |

### Project Ladder 对照

Hub 的 Project Ladder 适合把“读懂”变成一组小项目。每一级都应留下运行说明、示例输入输出和失败记录。

| Level | 项目 | 本地练习建议 |
| --- | --- | --- |
| 1 | Calculator Agent | 结构化输出、Tool Schema、最小调用循环 |
| 2 | Web Research Agent | 检索、筛选、引用和报告生成 |
| 3 | PDF QA Agent | 分块、Embedding、召回、引用校验 |
| 4 | Coding Review Agent | 读取 Diff、权限边界、风险排序 |
| 5 | Browser Agent | 公开网页操作、定位失败和动作日志 |
| 6 | Claude Code-like Nano Agent | Shell、文件编辑、Session、权限和上下文压缩 |
| 7 | OpenClaw-like Gateway | Channel、路由、Session、Memory、Heartbeat、Delivery |
| 8 | Reusable Skill Pack | `SKILL.md`、脚本、模板、触发条件和 Smoke Test |
| 9 | Multi-Agent Writer | Planner、Writer、Reviewer 的输入输出合同 |
| 10 | Personal Agent | 长期记忆、消息入口、Skills 和本地优先边界 |
| 11 | Production Harness | Eval、Trace、Runner、CI、回放、回滚 |

## 贯穿式学习方法

每个主题至少留下一个能检查的结果：架构图、可运行代码、Contract Test、故障注入、Trace、Eval、ADR 或运行手册。推荐始终沿着同一条链路练习：

```text
请求接入 → TaskSpec/Plan → 权限与隔离 → Provider 流式输出
→ Tool 审批与执行 → Event Store → 验证与证据交付 → 恢复与对账
```

不要先堆更多 Agent。先确认单 Agent、普通 Workflow 或脚本是否已经足够；确实需要拆分时，再为每个子 Agent 定义输入输出、授权范围、预算和停止条件。

## 仓库结构

```text
README.md                         入口、路线映射和范围说明
guide.md                          推荐阅读路径与学习方法
_sidebar.md                       可选的 Docsify 导航
docs/00-roadmap                  路线、能力地图和阶段验收
docs/01-foundations              LLM、模型接口、分布式系统、检索
docs/02-desktop                  桌面端、IPC、安全和 Provider
docs/03-runtime                  Loop、Workflow、事件、工具、恢复
docs/04-sdk                      SDK、事件协议、Transport、兼容性
docs/05-frameworks               Framework、Adapter、MCP、A2A
docs/06-platform                 平台治理、供应链、租户、成本
docs/07-practice                 参考架构、Kata、ADR、生产清单
docs/08-appendix                 术语、速查表、来源索引
labs/runtime-kernel               最小 Runtime Kernel 的学习入口
```

## 如何使用

GitHub 可以直接渲染所有 Markdown 文件，不需要安装依赖或启动服务。建议先读 `README.md` 和 `guide.md`，再按章节完成练习。`_sidebar.md` 仅用于需要 Docsify 导航的阅读环境；它不是网站运行时或业务代码。

每次提交学习成果时，尽量同时记录：适用的模型或 SDK 版本、实验输入、失败情况、测试结果和外部资料核验日期。模型、协议、价格和厂商 API 都可能变化，不能把“当前可用”写成永久承诺。

## 来源与许可

- 本仓库的架构笔记、章节映射、练习和验收内容由 Zeeklog 整理，具体版权与许可边界见 [NOTICE.md](NOTICE.md) 和 [LICENSE](LICENSE)。
- [Hello-Agents](https://github.com/datawhalechina/hello-agents) 与 [Agent-Learning-Hub](https://github.com/datawhalechina/Agent-Learning-Hub) 是独立的上游项目；本仓库只提供公开链接、路线映射和学习索引，没有复制其正文、图片或代码。
- 上游参考版本和各自许可证已记录在 [NOTICE.md](NOTICE.md)。上游项目会继续更新，阅读时应以对应仓库当前的许可证、版本和文档为准。

## 参与贡献

请先阅读 [CONTRIBUTING.md](CONTRIBUTING.md)。贡献应聚焦 AI 学习内容，并为易变的厂商事实附一手来源、核验日期和适用范围；不要提交凭据、用户数据、生产配置或与本知识库无关的项目文件。
