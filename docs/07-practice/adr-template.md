# ADR 模板：让架构决策可验证、可撤销

> ADR（Architecture Decision Record）记录“在当时约束下为什么这样选”。它应让后来者判断条件是否变化，以及决定继续、迁移还是撤销。

## 1. 何时必须写 ADR

满足任一条件就应写：

- 改变 Run/Event/Tool/SDK wire contract；
- 引入或更换 Provider、Agent Framework、Workflow/Event Store；
- 改变信任边界、sandbox、凭据、数据区域/保留；
- 做出难以逆转或跨三个以上团队的选择；
- 接受一项 P0/P1 风险或生产检查清单例外；
- 选择一致性、可用性、成本、延迟之间的重要权衡；
- 决定如何兼容历史 Run、旧客户端或迁移数据。

不必为“变量命名”“普通库升级”“容易撤销的局部实现”写 ADR，除非它们改变上述契约。

## 2. 文件命名与状态

```text
docs/adr/
├─ 0000-template.md
├─ 0001-event-log-source-of-truth.md
└─ 0002-tool-capability-grants.md
```

状态只用：

- `proposed`：待评审；
- `accepted`：已决定，可能尚未完全实施；
- `superseded by ADR-xxxx`：由新决策替代，旧文件不删除；
- `deprecated`：不再建议用于新场景，但历史仍存在；
- `rejected`：评估后不采用；
- `reversed by ADR-xxxx`：回滚/撤销。

## 3. 可复制模板

````markdown
---
id: ADR-XXXX
title: 用一句有方向性的陈述命名
status: proposed
date: YYYY-MM-DD
owners: [team-or-person]
reviewers: [runtime, security, sre, data, client]
decision_scope: [runtime, sdk, desktop, platform]
review_by: YYYY-MM-DD
supersedes: []
superseded_by: null
---

# ADR-XXXX：标题

## 1. 决策摘要

我们决定 **做什么**，适用于 **什么范围**，从 **何时** 生效。
一句话说明不做什么。

## 2. 背景与问题

- 用户/业务结果：
- 当前行为和痛点：
- 为什么现在必须决定：
- 未决定会发生什么：

不要提前写方案。先写可观察问题。

## 3. 决策驱动因素

按优先级列出：

1. Correctness / security 不变量
2. SLO、RTO/RPO
3. 数据驻留、隐私、审计
4. 工作负载/容量
5. 兼容与演进
6. 团队/时间/成本

## 4. 约束与假设

| 项 | 类型 | 值/描述 | 如何验证 | 失效影响 |
| --- | --- | --- | --- | --- |
| 峰值 RPS | 假设 |  | load test | 重审容量 |
| 客户端兼容 | 约束 | N/N-1 | contract CI | 禁止 breaking |

## 5. 不变量

- 例如：任何 Tool 副作用前必须有 durable intent。
- 例如：历史 Run 必须使用固定 workflow version 恢复。

## 6. 选项

### Option A：名称

**机制**：不是宣传语，说明状态、调用、失败如何工作。

**优点**：

- ...

**代价/风险**：

- ...

**失败模式**：

- ...

**验证 spike**：

- 用什么原型/数据消除未知。

### Option B：名称

同上。

## 7. 比较矩阵

| 标准（权重） | A | B | C | 证据 |
| --- | ---: | ---: | ---: | --- |
| 恢复正确性（30） |  |  |  | chaos result |
| 运维复杂度（15） |  |  |  | runbook drill |

评分只是辅助；P0 不变量不允许被总分抵消。

## 8. 决策

选择 Option X，因为……

### 适用范围

- 包含：
- 不包含：
- 历史 Run：
- 新 Run：

### Guardrails

- 最大值、权限、版本、kill switch、默认拒绝等。

## 9. 后果

### 正面

- ...

### 负面/接受的债务

- 风险、owner、到期和补偿控制。

### 新增运营责任

- dashboard、alert、runbook、on-call、容量。

## 10. 安全、隐私与合规

- 新信任边界：
- 数据分类/区域/保留：
- 授权与审计：
- 威胁及缓解：

## 11. 合约与数据演进

- Wire/schema 变化：
- N/N-1 行为：
- migration/backfill：
- 历史 decoder：

## 12. 实施计划

1. Spike
2. Contract/adapter
3. Shadow
4. Canary
5. Default
6. 删除旧路径

每步写 owner、退出条件和证据。

## 13. 验证与可观测

- 单元/contract/property/chaos/security/eval：
- SLI/SLO：
- 成本：
- 证明决策正确的 dashboard/query：

## 14. 发布、回滚与灾备

- 渐进发布 cohort：
- 自动停止条件：
- 回滚边界：
- 已发生 effect 的处理：
- DR 影响：

## 15. 重审触发器

- 日期：
- 量化触发：RPS、延迟、成本、故障、法规、Provider deprecation。

## 16. 未解决问题

| 问题 | Owner | 截止 | 阻塞什么 |
| --- | --- | --- | --- |

## 17. 参考与证据

- 官方规范、benchmark、PR、实验、incident；标核验日期和版本。
````

## 4. 示例：Event Log 为 Run 真相，Snapshot 加速

---

### ADR-0001：以 append-only Event Log 作为 Run 的恢复真相

**状态**：accepted<br>
**日期**：2026-08-30<br>
**Owner**：Runtime Platform<br>
**Reviewers**：Desktop、SRE、Security、Data<br>
**Review by**：2027-02-28

#### 4.1 决策摘要

新 Run 使用 append-only domain event log 作为恢复真相，每 50 个事件或进入等待状态时生成可丢弃 snapshot。UI、审计和通知从 outbox 消费事件投影。Trace 仅用于诊断，不能用于恢复。旧版只存消息快照的 Run 不自动伪造历史事件，由 legacy worker 排空。

#### 4.2 背景

当前 Runtime 只保存最终 message 和一个可变 `run.status`。进程在 Tool 成功、状态更新前崩溃时，不知道 effect 是否发生；UI 流和服务日志含义不同；事故后无法回答“审批后执行的确切参数”。产品即将支持跨天审批和 Desktop 断线恢复，当前模型不足。

#### 4.3 驱动因素

1. 高风险 Tool 不得因恢复重复执行；
2. Run 需要跨进程、跨 14 天等待恢复；
3. 审计需要不可改写的批准/执行事实；
4. 峰值 2,000 新 Run/s，每 Run 55 个持久事件；
5. 客户端支持 N/N-1，历史最长保留 1 年；
6. 团队必须在 8 周内迁移，不能重写所有 Provider adapter。

#### 4.4 约束与假设

| 项 | 类型 | 值 | 验证 | 失效影响 |
| --- | --- | --- | --- | --- |
| 事件写峰值 | 假设 | 110k/s，峰值系数后 165k/s | replay load test | 分片/批写重审 |
| 事件大小 | 假设 | p95 < 4 KB | staging histogram | 大 payload artifact 化 |
| Run 内顺序 | 约束 | 严格 | unique/CAS test | 无法恢复 |
| 跨 Run 顺序 | 非目标 | 不保证 | API 文档 | 消费者不得假设 |
| RPO | 约束 | Tool intent/result 为 0 | DR 演练 | 禁止发布 |

#### 4.5 不变量

- `(tenant_id, run_id, seq)` 唯一，append 使用 `expectedSeq`；
- 一个 Run 恰好一个 terminal event；
- 任何外部 effect 前先追加 `tool.execution.requested`；
- event payload 不存 secret/大正文，只存受控 artifact ref；
- snapshot 的 `throughSeq` 必须对应已提交事件；
- workflow/reducer/schema version 随事件或 snapshot 固化；
- outbox 与 event append 同一数据库事务；
- telemetry 丢失不能影响事件提交。

#### 4.6 评估选项

**A. 继续可变 Run/Message 快照**

- 优点：简单、写入少、现有代码最少；
- 缺点：失去因果历史，副作用窗口不可确认，无法可靠重建新投影；
- 失败：两个 worker 最后写入覆盖，审批参数无法证明。

**B. Event Log + Snapshot（选择）**

- 优点：恢复/审计共同事实；CAS 防双推进；可建立新投影；
- 代价：schema 演进、事件量、reducer 确定性、运维复杂；
- 失败：坏事件/坏 reducer 影响重放，需要版本固定和 quarantine。

**C. 只使用 Workflow 引擎 history**

- 优点：durable timer/activity/retry 成熟；
- 缺点：产品协议绑定引擎 history；UI/SDK/审计难稳定；迁移供应商困难；
- 用法：可作为执行实现，但仍向领域 event contract 投影。

| 标准（权重） | A | B | C |
| --- | ---: | ---: | ---: |
| 恢复正确性（30） | 8 | 28 | 25 |
| 审计/投影（20） | 5 | 19 | 12 |
| 引擎可替换（15） | 14 | 13 | 4 |
| 运维复杂度（15，分高更简单） | 14 | 8 | 9 |
| 交付风险（10） | 9 | 6 | 5 |
| 成本（10，分高更省） | 9 | 6 | 7 |
| **总分** | **59** | **80** | **62** |

#### 4.7 数据与写入协议

```sql
CREATE TABLE run_event (
  tenant_id      TEXT NOT NULL,
  run_id         TEXT NOT NULL,
  seq            BIGINT NOT NULL,
  event_id       TEXT NOT NULL,
  event_type     TEXT NOT NULL,
  schema_version INTEGER NOT NULL,
  payload_json   JSONB NOT NULL,
  occurred_at    TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (tenant_id, run_id, seq),
  UNIQUE (tenant_id, event_id)
);

CREATE TABLE run_head (
  tenant_id TEXT NOT NULL,
  run_id    TEXT NOT NULL,
  head_seq  BIGINT NOT NULL,
  terminal  BOOLEAN NOT NULL DEFAULT FALSE,
  PRIMARY KEY (tenant_id, run_id)
);
```

追加伪代码：

```ts
async function append(runId: string, expected: number, drafts: EventDraft[]) {
  return db.transaction(async tx => {
    const head = await tx.runHead.lock(runId);
    if (head.headSeq !== expected || head.terminal) throw conflict();

    const events = assignSeq(drafts, expected + 1);
    await tx.runEvent.insert(events);
    await tx.runHead.update(runId, {
      headSeq: events.at(-1)!.seq,
      terminal: events.some(isTerminal)
    });
    await tx.outbox.insert(events.map(toOutbox));
    return events;
  });
}
```

数据库约束还需阻止第二个 terminal event，可用 append transaction 中的 locked head + terminal flag；定期完整性任务检测 gap、多个 terminal、悬空 requested。

#### 4.8 Snapshot

Snapshot 不是事实真相：

```ts
type RunSnapshot = {
  runId: string;
  throughSeq: number;
  reducerVersion: string;
  workflowVersion: string;
  state: RunState;
  stateDigest: string;
};
```

加载时校验版本/digest，从 `throughSeq + 1` 重放。无法读取旧 snapshot 时丢弃并从事件重建；若旧 event decoder 缺失则将 Run quarantine，不能猜测。

#### 4.9 安全与隐私

- 事件按租户分区并启用数据库 RLS；
- prompt/file/tool output 存加密 artifact，事件只存 ref、digest、分类；
- audit 投影只保留必要主体/动作/授权，不复制正文；
- support 人员默认只能看元数据，break-glass 访问有时限与审计；
- 删除 artifact 后，事件中的 digest 是否仍为个人数据由隐私评估决定。

#### 4.10 实施

1. 定义 v1 schema、reducer 与 golden trace；
2. 新 Run 双写旧 snapshot + event，event shadow replay；
3. 连续两周比较 terminal/status/tool effect 投影；
4. 内部租户以 event 恢复，旧 snapshot 继续比对；
5. 5%→25%→100% 新 Run 切换；
6. legacy Run 固定旧 worker，最长 60 天排空；
7. 停旧写入；保留只读迁移工具到数据保留结束。

#### 4.11 验证

- property test：合法历史重放，重复/gap/非法 terminal；
- crash test：requested 前后、effect 前后、completed 前后；
- load：165k event/s 峰值 30 分钟，p99 append < 100 ms；
- DR：主区丢失后恢复 head/event/outbox 一致；
- privacy：canary secret 不进入 event/trace；
- N/N-1：旧客户端安全忽略新事件并继续 cursor。

#### 4.12 发布与回滚

切换通过 feature flag 按新 Run 生效。回滚只把**新 Run**重新交给旧入口；已经写 v1 event 的 Run 继续由 v1 worker 推进，避免旧实现不理解新历史。若 event store 不可用，停止接受需要副作用的新 Run，不降级为“先执行后补日志”。

#### 4.13 负面后果

- 需要长期维护 event decoder/reducer 版本；
- 写入、索引和备份成本上升；
- 开发者必须学习 command/event/effect 分离；
- 临时保留双写路径，Owner 为 Runtime Migration，最晚 2026-12-31 删除。

#### 4.14 重审触发器

- 事件成本超过平台成本 20%；
- p99 append 连续一周超 100 ms；
- 单 Run 事件 p99 超 10,000；
- 更换 Workflow/Event Store；
- 新法规要求不同审计/删除语义；
- 2027-02-28 定期重审。

---

## 5. ADR 评审清单

- [ ] 标题和摘要表达了真实方向，不是“讨论 X”。
- [ ] 背景没有偷渡方案；约束、假设可验证。
- [ ] P0 不变量没有被加权总分抵消。
- [ ] 每个选项都说明机制和失败模式，不只列品牌优缺点。
- [ ] 明确历史 Run、旧客户端和数据迁移。
- [ ] 明确安全/隐私、观测、运营责任。
- [ ] 有渐进发布、自动停止、回滚边界。
- [ ] 有重审日期和量化触发器。
- [ ] 被替代 ADR 保留且互相链接。

关联：[参考架构](reference-architecture.md)、[生产清单](production-checklists.md)、[一手资料索引](../08-appendix/reading-list.md)。
