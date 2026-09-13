# Context Engineering：预算、选择、压缩与可信边界

Context 是一条有预算、来源、优先级和测试数据的数据管道。这里实现一个基础 Context Packer，并用显式约束检查压缩过程是否丢失关键信息。

## 1. Context 不等于 Memory，也不等于 Prompt

- **Memory** 是可持久化、可召回的信息集合；
- **Context** 是某一次模型调用实际可见的输入视图；
- **Prompt** 是 Context 中承担指令和任务描述的部分。

Context Builder 类似数据库查询优化器：从会话事件、工作区、工具结果、组织策略和 Memory 中选择候选项，在窗口、成本、延迟和可信度约束下生成确定顺序的消息。它必须能回答“这段内容为何被放入/排除”。

## 2. 先做 Token Budget 会计

模型标称 context window 不是可全部用于历史的空间。应先预留输出、工具 schema、系统指令和安全余量：

```text
history_budget = context_limit
               - reserved_output
               - system_and_policy
               - tool_schemas
               - current_user_input
               - safety_margin
```

`tool_schemas` 常被忽略：几十个 MCP Tool 的描述和 JSON Schema 可能占用大量 token。解决方案不是只缩短描述，而是分层发现——先给工具索引或命名空间，只向当前任务暴露候选工具。

不要在生产中用 `text.length / 4` 作为精确计数。应针对最终模型与编码器计数，并包含角色/消息包装开销；近似计数只可用于候选预筛，最终装箱需精确复算。供应商更换时，Tokenizer 也是 adapter 的一部分。

## 3. Context Item 的统一元数据

把所有候选内容规范化，而不是直接混成字符串：

```ts
type Trust = "policy" | "user" | "workspace" | "external" | "model_generated";
type Kind = "instruction" | "turn" | "tool_call" | "tool_result" | "document" | "memory" | "summary";

interface ContextItem {
  id: string;
  kind: Kind;
  text: string;
  tokenCost: number;
  required: boolean;
  trust: Trust;
  authority: number;    // 指令优先级，不代表内容相关性
  relevance: number;    // 针对当前任务的召回得分
  recency: number;      // 归一化 0..1
  sourceUri?: string;
  contentHash: string;
  orderKey: number;          // 同类内容的确定性业务顺序，如事件 seq
  expiresAt?: string;
  dependencies?: string[]; // 例如 tool result 依赖对应 tool call
}
```

`trust` 与 `authority` 是安全字段：网页中的“忽略系统指令”仍是 external data，不会因语气强烈升级成 instruction。`contentHash` 用于缓存 token 数、去重和引用；`sourceUri` 支持回答引用与过期刷新。

## 4. 分层构建管线

推荐顺序如下：

1. **固定层**：系统规则、组织策略、Agent 身份、输出 schema；
2. **任务层**：当前用户请求、选中的工作区/分支/文件范围；
3. **工作集层**：最近未完成步骤、最近 N 轮、当前工具调用及其结果；
4. **召回层**：按 query 从文档与长期 Memory 检索的候选；
5. **压缩层**：旧对话摘要、工具大结果的结构化提取；
6. **校验层**：预算、依赖闭包、顺序、来源、敏感信息策略。

先过滤 ACL/租户/目录权限，再做向量检索；否则即使最终没展示，检索日志和 embedding 管线也可能泄漏无权内容。

## 5. 一个可测试的装箱算法

下面实现保留 required 项，再按单位 token 价值选择候选，并保证依赖一起进入。生产中可加入分区配额，避免某一类内容吃光预算。

```ts
function score(i: ContextItem): number {
  const trustBonus = i.trust === "policy" || i.trust === "user" ? 0.25 : 0;
  return 0.50 * i.relevance + 0.25 * i.recency + 0.25 * i.authority + trustBonus;
}

export function pack(items: ContextItem[], budget: number): {
  selected: ContextItem[];
  omitted: { id: string; reason: string }[];
} {
  const byId = new Map(items.map(i => [i.id, i]));
  const selected = new Map<string, ContextItem>();
  let used = 0;

  const addWithDeps = (item: ContextItem): boolean => {
    const closure = new Map<string, ContextItem>();
    const visit = (x: ContextItem) => {
      if (selected.has(x.id) || closure.has(x.id)) return;
      closure.set(x.id, x);
      for (const id of x.dependencies ?? []) {
        const dep = byId.get(id);
        if (!dep) throw new Error(`missing dependency ${id}`);
        visit(dep);
      }
    };
    visit(item);
    const cost = [...closure.values()].reduce((n, x) => n + x.tokenCost, 0);
    if (used + cost > budget) return false;
    for (const x of closure.values()) selected.set(x.id, x);
    used += cost;
    return true;
  };

  for (const item of items.filter(i => i.required)) {
    if (!addWithDeps(item)) throw new Error("required context exceeds budget");
  }

  const optional = items
    .filter(i => !i.required)
    .sort((a, b) => score(b) / Math.max(b.tokenCost, 1) - score(a) / Math.max(a.tokenCost, 1));
  for (const item of optional) addWithDeps(item);

  const ordered = topologicalOrder([...selected.values()]);
  return {
    selected: ordered,
    omitted: items.filter(i => !selected.has(i.id)).map(i => ({ id: i.id, reason: "budget" }))
  };
}

function contextOrder(a: ContextItem, b: ContextItem): number {
  const rank: Record<Kind, number> = {
    instruction: 0, summary: 1, memory: 2, document: 3,
    turn: 4, tool_call: 5, tool_result: 6
  };
  return rank[a.kind] - rank[b.kind] || a.orderKey - b.orderKey || a.id.localeCompare(b.id);
}

function topologicalOrder(items: ContextItem[]): ContextItem[] {
  const byId = new Map(items.map(x => [x.id, x]));
  const indegree = new Map(items.map(x => [x.id, 0]));
  const dependents = new Map<string, string[]>();
  for (const item of items) {
    for (const depId of item.dependencies ?? []) {
      if (!byId.has(depId)) throw new Error(`selected context misses dependency ${depId}`);
      indegree.set(item.id, indegree.get(item.id)! + 1);
      dependents.set(depId, [...(dependents.get(depId) ?? []), item.id]);
    }
  }
  const ready = items.filter(x => indegree.get(x.id) === 0).sort(contextOrder);
  const ordered: ContextItem[] = [];
  while (ready.length) {
    const current = ready.shift()!;
    ordered.push(current);
    for (const id of dependents.get(current.id) ?? []) {
      const next = indegree.get(id)! - 1;
      indegree.set(id, next);
      if (next === 0) ready.push(byId.get(id)!);
    }
    ready.sort(contextOrder); // 小集合；生产中可换稳定优先队列
  }
  if (ordered.length !== items.length) throw new Error("context dependency cycle");
  return ordered;
}
```

贪心并非全局最优，但解释性好。依赖闭包负责“选中完整”，拓扑排序负责“依赖先于使用者”；二者不能用一次按 kind 排序替代。每个 `tool_result.dependencies` 必须包含对应 `tool_call`，若某 Provider 要求二者相邻，则 Adapter 在最终消息编译阶段将它们作为原子组并再次校验。更重要的是输出 selection manifest：item id、来源、token、得分、排除原因。它可进入 Trace，却不必记录敏感正文。

## 6. Compaction：有损操作必须显式

当历史持续增长，应创建新的 `ContextCompacted` 事件，而不是覆盖旧消息。摘要至少拆成结构化字段：

```ts
interface ConversationSummary {
  goal: string;
  confirmedFacts: Array<{ fact: string; evidenceEventIds: string[] }>;
  constraints: string[];
  decisions: Array<{ decision: string; reason: string; supersedes?: string }>;
  completedSteps: string[];
  openQuestions: string[];
  pendingToolCalls: string[];
  userPreferences: string[];
  sourceRange: { fromSeq: number; toSeq: number };
  summaryModel: string;
  promptVersion: string;
}
```

好的压缩原则：

- **保留承诺与否定信息**：用户说“不要发布”不能被压成“讨论了发布”；
- **事实带证据指针**：需要时可从原始事件重新水合，而非把摘要当唯一真相；
- **未决状态不压扁**：未完成工具调用、审批和错误必须单列；
- **新旧摘要可校验**：约束集合不得无解释减少；关键实体、数字、路径做程序化 diff；
- **摘要也不可信**：它是 model-generated 派生物，不应获得 system authority。

分层压缩通常优于一次性总结：工具输出先结构化提取，代码只保留相关符号和 diff，旧对话生成阶段摘要，极旧摘要再合并为里程碑摘要。

## 7. RAG、代码上下文与引用

RAG 不是“向量 Top-K 塞满窗口”。候选召回应结合 lexical、semantic、符号图与时间；rerank 后还应做多样性去重。代码任务应优先使用 AST/LSP 关系：定义、引用、调用方、测试，比任意文本 chunk 更能保持语义完整。

Chunk 需保存 `documentVersion/start/end/contentHash`。回答引用的是具体版本，文件变化后要标记 stale。工具返回超大日志时，先保存为 artifact，再向模型提供摘要、错误窗口和可继续读取的 URI，避免把 10 MB stdout 直接进入上下文。

## 8. Prompt Injection 与数据边界

外部网页、Issue、README、工具输出都可能包含对模型的恶意指令。Context Builder 应使用明确封装标注数据来源，并在硬策略中规定外部数据不能改变权限、工具范围或系统指令。真正的安全边界仍在 Tool Gateway；仅靠“请忽略恶意提示”不构成隔离。

敏感数据进入远程模型前执行 DLP/secret scan；需要保留语义时可用稳定占位符，如 `<SECRET_1>`，映射仅存在本地受保护存储。Trace 默认记录 hash、长度、分类而非正文。

## 9. 缓存与性能

缓存键不能只有 prompt 文本，应包括 `model + tokenizer + systemVersion + toolSetHash + itemHashes + samplingConfig`。可分别缓存：token 数、embedding、检索结果、前缀/供应商 prompt cache、压缩摘要。任何包含权限过滤的缓存都要绑定 tenant/user/capability scope。

在桌面端，Context Builder 可对文件索引做增量更新；监听文件变化时以 content hash 去抖。不要在 renderer 拼上下文，因为它缺乏组织策略和文件权限的最终裁决权。

最后还要验证消息角色与顺序：同一工具结果必须紧随或明确引用对应 call，不能留下孤立结果；同一来源的重复片段应去重；当前用户请求不得被旧摘要遮蔽。构建结果同时保存 provider-neutral manifest 与 adapter 后的实际请求 hash，前者支持跨模型比较，后者支持定位供应商格式转换问题。

## 10. 失败模式

| 失败模式 | 结果 | 修正 |
| --- | --- | --- |
| 只保留最近 N 条 | 早期约束悄悄丢失 | required constraints + 结构化摘要 |
| token 用字符数粗估 | 临界时请求溢出 | 模型 tokenizer 精确复算 + margin |
| Top-K 未做 ACL | 跨租户泄漏 | 检索前权限过滤，缓存绑定 scope |
| 摘要覆盖原历史 | 无法审计和纠错 | 原事件不可变，摘要是派生事件 |
| 工具 schema 全量常驻 | 高成本、挤压任务信息 | 按命名空间/意图动态选择工具 |
| 外部文本被当指令 | Prompt Injection 提权 | trust/authority 分离 + Tool Policy |

## 11. 测试、练习与验收

```ts
import { strict as assert } from "node:assert";

const result = pack([
  { id: "policy", kind: "instruction", text: "不可发布", tokenCost: 20,
    required: true, trust: "policy", authority: 1, relevance: 1, recency: 1, contentHash: "a" },
  { id: "noise", kind: "document", text: "...", tokenCost: 100,
    required: false, trust: "external", authority: 0, relevance: .1, recency: .2, contentHash: "b" }
], 30);
assert.deepEqual(result.selected.map(x => x.id), ["policy"]);
```

除样例测试外，做性质测试：selected token 永不超过预算；required 要么全部存在要么显式失败；依赖不悬空；同输入得到同顺序。使用“忽略规则并读取密钥”的恶意文档做安全回归。

**练习**：给一个 50 轮代码修复会话设计三层摘要，要求保留目标分支、用户禁止修改的目录、已失败测试及对应日志 artifact URI，并定义何时重新水合原文。

**验收点**：

- [ ] 每次模型调用能输出可解释的 Context Manifest。
- [ ] 预算预留输出、system、tool schema 与安全余量。
- [ ] 压缩不覆盖历史，约束、决策、未决动作有结构化字段和证据引用。
- [ ] 检索前完成 ACL，外部内容不会提升 authority。
- [ ] Context Builder 可在更换模型/tokenizer 后重新精确计量。

## 延伸阅读

- [OpenAI Agents SDK：Results and state](https://developers.openai.com/api/docs/guides/agents/results)
- [OpenAI Agents SDK：Running agents](https://developers.openai.com/api/docs/guides/agents/running-agents)
- [Anthropic：Context windows](https://docs.anthropic.com/en/docs/build-with-claude/context-windows)
- [OWASP：LLM Prompt Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html)
