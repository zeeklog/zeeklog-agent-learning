# 交付、灰度与平台演进

## 1. 分阶段推进

| 阶段 | 目标 | 关键交付物 | 进入下一阶段门槛 |
|---|---|---|---|
| 0 基线（2~4 周） | 建立术语、风险与测量 | canonical contracts、模型/工具网关薄层、trace、黄金数据集 | 一个 read-only 场景可复现、可计费、可审计 |
| 1 受控试点（4~8 周） | 端到端跑通 | 单 Agent、白名单工具、人工确认、runbook、SLO | 失败注入/红队通过，业务指标有改善 |
| 2 自助平台（2~4 月） | 多团队复用 | SDK、模板、registry、eval CI、release bundle、租户配额 | 两个以上团队独立发布且支持量可控 |
| 3 规模化（4~9 月） | 多 cell/多模型/长任务 | durable runtime、policy、成本路由、灰度、灾备 | SLO/成本/安全预算持续达标 |
| 4 联邦生态 | 域自治、中央治理 | MCP/A2A 目录、域平台、统一证据与协议 | owner/下线/兼容流程成熟，无影子接入 |

不要第一天建设“万能 Agent 平台”。先选高频、可测且不可逆风险较低的工作流，再按 read-only → 可撤销写 → 高风险写逐级提升自治。

## 2. Platform as a Product

平台团队的客户是业务研发。产品指标包括 time-to-first-safe-agent、发布 lead time、复用率、升级成功率、支持工单、单位成功成本和安全事件，而非“提供了多少组件”。提供 paved road：

- TypeScript/其他目标语言统一 SDK、示例和本地 fake runtime；
- agent/tool 模板、MCP server 安全模板、默认 OTel、预算和策略；
- CLI/Portal 查看 release diff、eval、trace、成本和回滚；
- 明确的 escape hatch：业务可用专有框架特性，但仍经过身份、Tool Gateway、审计与 SLO 边界；
- 版本兼容表、迁移窗口、office hours、runbook 和 owner 目录。

推荐组织：平台团队拥有 Runtime/SDK/gateway；安全团队拥有控制目标和策略基线；领域团队拥有 Agent、工具、数据与业务 eval；SRE/FinOps 共建 SLO、容量和成本。高风险用例设业务 risk owner，不能把责任都推给平台。

## 3. AI 变更流水线

```text
PR: code + prompt + workflow + tool schema + policy refs
 ├─ lint/type/schema/SBOM/secret scan
 ├─ unit + adapter contract + deterministic workflow tests
 ├─ offline eval：质量/安全/成本/回归/公平性
 ├─ sandbox integration + fault injection + red-team subset
 ├─ compile/sign ReleaseBundle + approvals
 ├─ shadow → 1% canary → 10% → 50% → 100%
 └─ continuous eval / SLO / cost / rollback / retire
```

prompt、模型、retrieval、tool schema 和策略都是代码级变更。每次 release 记录完整 digest、数据集版本、评测器版本与阈值；避免“模型自动升级但 release 未变”的不可复现状态。

```ts
interface PromotionGate {
  releaseDigest: string;
  offline: { taskPass: number; attackSuccess: number; costP95: number };
  online: { errorRate: number; p95Ms: number; policyDenies: number; humanOverride: number };
  thresholds: { minPass: number; maxAttack: number; maxCostRegression: number };
}

const promotable = (g: PromotionGate) =>
  g.offline.taskPass >= g.thresholds.minPass &&
  g.offline.attackSuccess <= g.thresholds.maxAttack &&
  g.offline.costP95 <= baseline.costP95 * (1 + g.thresholds.maxCostRegression) &&
  g.online.errorRate <= errorBudget.remainingRate;
```

阈值按任务风险分层；LLM-as-judge 不能单独做高风险门禁，需规则、参考答案、执行结果和人工抽检组合。评测数据防污染，生产失败经脱敏审批后进入回归集。

## 4. 灰度与回滚

Shadow 模式调用新 release 但禁止副作用，只比较输出/计划；写场景可使用仿真工具或 dry-run。Canary 按 tenant/user/run 一致性哈希，避免同一对话切版本。观测不只看 5xx：task success、工具选择/参数、policy deny、人工覆盖、投诉、P95/P99、token、单位成功成本、攻击成功率都可触发暂停。

回滚对象是完整 release bundle，不是只改 prompt。对已运行长任务，定义 continue-on-old、safe-point migrate 或 cancel/compensate，绝不让新 workflow 解释旧 checkpoint。数据库/事件 schema 使用 expand-contract，至少 N/N-1 双读。kill switch 可独立禁用 tool/model/release，演练而非只写文档。

## 5. 框架与协议持续演进

每季度维护 technology radar，分为 Adopt/Trial/Assess/Hold，跟踪官方 release、兼容策略、CVE、license、维护活跃度和 benchmark。MCP、A2A 和框架版本精确固定，并通过 adapter contract tests 升级；每次只改变一个主要变量。对 AutoGen、Semantic Kernel 等既有资产，先统一身份、工具和遥测边界，再逐个 workflow 迁移到目标框架，避免一次性重写全部系统。

技术债有预算：adapter N-1 支持期、废弃通知、使用方清单和自动迁移工具。过期 agent/tool 必须有 owner 和自动下线日；无人认领资产默认隔离，不无限在线。

## 6. 失败模式

- **Demo 驱动平台**：没有 SLO、owner、成本和退出条件。试点 charter 先写成功/停止标准。
- **只做离线 eval**：真实权限、延迟、用户分布不同。shadow/canary + 持续抽样。
- **只回滚 prompt**：tool/schema/模型不一致。原子 release bundle。
- **中央团队成为瓶颈**：每个 prompt 都需平台审批。中央定义风险基线，领域团队在 guardrail 内自治。
- **指标被优化游戏化**：judge 分数上涨但人工覆盖/投诉增加。组合业务结果、执行证据与人工采样。
- **没有下线机制**：旧 tool/Agent 长期带凭据。TTL、owner heartbeat、usage-based retirement。

## 7. 练习与验收

关联 platform engineering、Team Topologies、GitOps、progressive delivery、feature flags、SLSA、MLOps/LLMOps、ADRs 与组织变革。先读[Build vs Buy](../05-frameworks/build-vs-buy.md)和[控制面/数据面](control-data-plane.md)。

练习：为一个真实内部助手制定 90 天路线图；创建 50 条质量集、20 条攻击集、10 个故障注入；设计 shadow/canary、回滚、kill switch 演练和 RACI。

验收：新团队一周内按 paved road 上线 read-only Agent；每个 release 可复现、可审计、15 分钟内回滚；canary 能自动发现预置质量/成本回归；季度内完成一次 provider 或 framework 替换演练；平台指标能证明业务价值而非组件数量。

## 官方资料

- [NIST AI RMF Playbook](https://www.nist.gov/itl/ai-risk-management-framework/nist-ai-rmf-playbook)
- [OpenFeature Specification](https://openfeature.dev/specification/)
- [Argo Rollouts](https://argo-rollouts.readthedocs.io/)
- [SLSA Specification](https://slsa.dev/spec/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)
- [Microsoft AutoGen 迁移到 Agent Framework](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/)
