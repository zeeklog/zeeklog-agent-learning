# Memory 与 State：事实、派生知识和工作集的生命周期

执行状态、会话历史、工作记忆和长期记忆有不同的真相级别与生命周期。Memory 治理要覆盖写入、召回、冲突和删除，尤其要阻止错误事实被长期放大。

## 1. 四种状态分开

| 层 | 示例 | 真相级别 | 生命周期 |
| --- | --- | --- | --- |
| Run State | 当前 step、pending tool、attempt | 执行真相 | 一次 Run，可快照恢复 |
| Event History | 用户输入、审批、工具 receipt | 不可变事实 | 按审计策略长期保留 |
| Working Memory | 当前目标、约束、已打开文件 | 派生工作集 | 跨若干 turn，可压缩 |
| Long-term Memory | 用户偏好、项目约定、经验 | 可修订知识 | 跨会话，需来源/TTL/删除 |

Run State 决定“下一步能否执行”，不能由向量召回结果替代。长期 Memory 可能过期、冲突或被删除，因此只能作为有来源的候选 Context。把所有聊天都 embedding 进一个库，看似有记忆，实际会让模型反复召回猜测、旧秘密和已撤销指令。

## 2. 记忆类型与适用召回

- **Episodic（情节）**：发生过的事件，如“上次迁移因索引锁超时失败”；适合按项目、时间、相似任务召回。
- **Semantic（语义）**：稳定事实，如“项目使用 pnpm workspace”；需要证据和有效期。
- **Procedural（程序）**：做事方法，如“发布前运行哪些检查”；适合记录为版本化 Skill/Workflow，而不是自由文本。
- **Preference（偏好）**：用户明确选择，如“默认用中文解释”；需限定作用域和可撤销。

“模型自己总结出的猜测”不应直接升级为稳定 Semantic Memory。建议区分 `asserted_by_user`、`observed_from_source`、`inferred_by_model`，并给予不同置信度和写入门槛。

## 3. Memory Record 设计

```ts
type MemoryKind = "episodic" | "semantic" | "procedural" | "preference";
type Provenance = "user" | "tool" | "document" | "model_inference";

interface MemoryRecord {
  id: string;
  tenantId: string;
  subjectId: string;         // user/project/team，不能只靠自然语言描述作用域
  kind: MemoryKind;
  statement: string;
  normalizedKey?: string;    // 如 project.package_manager
  value?: unknown;
  provenance: Provenance;
  evidence: Array<{ eventId?: string; uri?: string; contentHash?: string }>;
  confidence: number;
  validFrom: string;
  validUntil?: string;
  supersedes?: string;
  status: "active" | "superseded" | "retracted";
  sensitivity: "public" | "internal" | "confidential" | "secret";
  embeddingVersion?: string;
  createdAt: string;
}
```

原始 statement 便于语义检索，`normalizedKey/value` 便于确定性冲突处理。例如新记忆 `package_manager=pnpm` 可以明确 supersede 旧的 `npm`，而不是让两个 embedding 都保持 active。

## 4. 写入管线：不是每句话都值得记

Memory Write 应是独立策略流程：

1. 从事件中提取 candidate，保留 evidence pointer；
2. 按作用域、敏感级别、租户策略判断是否允许长期保存；
3. 规范化实体/key，检测重复和冲突；
4. 按来源设置信心与 TTL；
5. 对高影响事实请求用户确认，或只写为低置信候选；
6. 事务写 record + audit event，异步生成 embedding；
7. 后台执行过期、重嵌入和删除任务。

```ts
interface Candidate {
  kind: MemoryKind;
  statement: string;
  key?: string;
  value?: unknown;
  provenance: Provenance;
  evidence: MemoryRecord["evidence"];
  sensitivity: MemoryRecord["sensitivity"];
}

function writePolicy(c: Candidate): { action: "reject" | "confirm" | "store"; ttlDays?: number } {
  if (c.sensitivity === "secret") return { action: "reject" };
  if (c.provenance === "model_inference") return { action: "confirm" };
  if (c.kind === "episodic") return { action: "store", ttlDays: 90 };
  if (c.kind === "preference" && c.provenance !== "user") return { action: "confirm" };
  return { action: "store", ttlDays: 365 };
}
```

工具读到的 access token、密码和私钥永不因为“相关”而写入 Memory；最多写“某凭据已配置”的非秘密元数据。桌面端本地库需要系统 Keychain/DPAPI 保护密钥，不能把加密密钥与数据库放在同一明文配置中。

## 5. 召回：过滤、检索、重排、验证

正确顺序是：**权限/作用域过滤 → lexical/vector 候选 → 时间与置信衰减 → 去重/冲突 → rerank → 预算装箱**。先全库向量搜索再过滤，可能通过日志、缓存和 timing 泄漏跨租户信息。

```ts
interface RecallQuery {
  tenantId: string;
  subjectIds: string[];
  text: string;
  now: Date;
  limit: number;
}

function rankMemory(m: MemoryRecord, semantic: number, now: Date): number {
  if (m.status !== "active") return -Infinity;
  if (m.validFrom && new Date(m.validFrom) > now) return -Infinity;
  if (m.validUntil && new Date(m.validUntil) <= now) return -Infinity;
  const ageDays = (now.getTime() - new Date(m.createdAt).getTime()) / 86_400_000;
  const decay = Math.exp(-ageDays / (m.kind === "semantic" ? 365 : 45));
  const sourceWeight: Record<Provenance, number> = {
    user: 1, tool: .9, document: .8, model_inference: .45
  };
  return .55 * semantic + .20 * m.confidence + .15 * decay + .10 * sourceWeight[m.provenance];
}

async function recall(q: RecallQuery, store: MemoryStore): Promise<MemoryRecord[]> {
  const candidates = await store.searchWithinScope({
    tenantId: q.tenantId, subjectIds: q.subjectIds, text: q.text, limit: q.limit * 5
  }); // 存储层先做租户与 subject 过滤
  return candidates
    .map(x => ({ ...x, score: rankMemory(x.record, x.semanticScore, q.now) }))
    .filter(x => Number.isFinite(x.score))
    .sort((a, b) => b.score - a.score)
    .slice(0, q.limit)
    .map(x => x.record);
}

declare interface MemoryStore {
  searchWithinScope(q: Omit<RecallQuery, "now"> & { limit: number }):
    Promise<Array<{ record: MemoryRecord; semanticScore: number }>>;
}
```

召回结果进入 Context 时带标签：“可能过期的历史记忆，必要时验证”，而不是伪装成 system instruction。对路径、版本、人员权限等高波动事实，应调用工具确认当前值。

## 6. 冲突、修订与时间语义

不要原地覆盖 Memory。新事实创建新 record，并用 `supersedes` 连接旧记录；旧记录标为 superseded，审计仍可见。若证据不足以判定新旧，保留冲突组并在上下文明确呈现：“来源 A 认为 X，来源 B 认为 Y”。

有效时间与记录时间不同：用户今天说“从下周起使用 pnpm”，`createdAt=今天`，`validFrom=下周`。缺少双时间语义会让未来规则提前生效，也无法回答“当时系统为什么这样做”。

用户更正“我不使用英文回复了”应撤销旧偏好，而不是新增一条相反向量。Normalized key 让这一过程可确定执行。

## 7. State Snapshot 与 Event History

状态快照是性能优化，可从历史重建，不应比事件更权威。快照需记录 `runId/headSeq/reducerVersion/checksum/state`；加载时验证 checksum 和 reducerVersion，不匹配就从最近兼容快照 + 后续事件重放。

大对象如代码 diff、截图、日志不要塞进状态 JSON。保存到内容寻址 Artifact Store，状态只留 `uri/hash/mime/size`。这样事件复制、重放与同步都保持轻量。

本地/云同步时，Run Event 通常要求单写者顺序；偏好或笔记可使用版本向量/CRDT，但安全策略和审批不可用“最后写入者获胜”随意合并。

## 8. 删除、保留与合规

Memory 必须支持：按用户/项目导出、单条纠正、作用域删除、TTL 到期、租户销毁。删除向量库条目还不够，还要处理原文、embedding、检索缓存、备份保留和派生摘要。Event History 若因审计必须保留，应通过加密擦除、字段级脱敏或合法保留策略处理，不能承诺实际上做不到的即时物理删除。

记录 `deletion_tombstone` 防止离线设备同步时把已删 Memory 重新上传。Embedding 也可能泄露敏感特征，应继承源数据分类和访问控制。

### 8.1 记忆质量也需要观测与评测

记录每次召回的候选、最终入选、证据是否被再次验证，以及该记忆是否影响了答案或工具决策。离线评测至少覆盖 precision@k、过期事实召回率、冲突识别率、跨租户泄漏为零和删除后不可召回；线上可以观察用户纠正率、被召回后立即否定率，但不能把“模型引用了记忆”当成正确。

建立反馈循环时要防自我强化：模型依据旧 Memory 生成回答，再把该回答作为新证据写回，会让错误置信度越来越高。派生内容必须追溯到独立原始证据，同源重复不增加置信度；没有新证据的模型复述不允许创建新的 Semantic Memory。

## 9. 失败模式

| 失败模式 | 后果 | 修正 |
| --- | --- | --- |
| 自动记住所有对话 | 错误、秘密、噪声长期放大 | 写入策略、确认、TTL、敏感过滤 |
| 向量库作为事实真相 | 相似不等于正确，难做冲突 | 结构化 key/value + provenance/evidence |
| 新事实覆盖旧行 | 无法解释历史行为 | append + supersedes/retract |
| 先检索再权限过滤 | 跨租户侧信道泄漏 | 存储层 scope filter |
| 召回内容获得系统权限 | 旧偏好或注入提权 | Memory 作为带标签 data，策略另存 |
| 只删主表 | embedding/缓存/离线副本复活 | 删除编排 + tombstone + 保留审计 |

## 10. 测试、练习与验收

测试应固定时间，验证过期边界、来源权重、冲突组、撤销和跨租户隔离。构造同一 normalized key 的三条互相矛盾记忆，确保只有明确 supersede 后才隐藏旧值。

**练习**：实现 `project.package_manager` 的写入与召回。README 声称 npm、`packageManager` 字段表明 pnpm、用户刚刚确认 pnpm。给出 evidence、confidence、冲突解决和 README 后续更新时的行为。

**验收点**：

- [ ] Run State、Event、Working Memory、Long-term Memory 使用不同存储与生命周期。
- [ ] 每条长期 Memory 有作用域、来源、证据、置信度、有效期和敏感级别。
- [ ] 召回先做权限过滤，并能解释得分与排除原因。
- [ ] 更正使用 supersede/retract，不破坏审计历史。
- [ ] 删除覆盖原文、embedding、缓存、离线同步与备份策略。

## 延伸阅读

- [OpenAI Agents SDK：Results and state](https://developers.openai.com/api/docs/guides/agents/results)
- [Temporal：Event History](https://docs.temporal.io/workflow-execution/event)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
