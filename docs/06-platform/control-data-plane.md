# 控制面与数据面：让治理不阻塞运行

## 1. 分离原则

**控制面**管理期望状态：Agent 定义、prompt/tool/model policy、租户、评测、发布、撤销。**数据面**执行已发布状态：接收流量、运行 Agent、调用模型/工具、持久化 checkpoint、输出事件。两者需要不同的一致性、可用性和变更节奏，不能只在图上分成两个盒子。

| 属性 | 控制面 | 数据面 |
|---|---|---|
| 优化目标 | 正确发布、可审计、强治理 | 低延迟、高可用、隔离、背压 |
| 一致性 | 关键写强一致；发布单调 | 基于 immutable snapshot 最终一致 |
| 故障策略 | 暂停新发布，旧 release 继续 | 熔断/降级/排队；不能临时绕策略 |
| 数据 | desired state、版本、策略、评测 | run、step、event、artifact、usage |
| 扩缩容 | 配置/发布吞吐 | 请求、token、工具并发和队列深度 |

## 2. 发布而非在线拼装

控制面将可变源编译为 `ReleaseBundle`：解析引用、固定版本、校验工具 schema、执行策略静态检查、跑 eval gate、签名并分发。数据面只加载验证过签名的 bundle。

```ts
interface ReleaseBundle {
  releaseId: string; generation: number; digest: string;
  compiledWorkflow: unknown; prompt: string;
  toolGrants: { id: string; schemaHash: string; risk: string }[];
  modelPolicy: unknown; runtimeLimits: unknown;
  policyBundleDigest: string; createdAt: string; signature: string;
}

function admit(bundle: ReleaseBundle, trustedKey: CryptoKey) {
  // digest 与 signature 都是完整性元数据，不参与被哈希 payload，避免自引用。
  const payload = canonicalize(withoutDigestAndSignature(bundle));
  const actualDigest = sha256(payload);
  assert(timingSafeEqual(actualDigest, bundle.digest));
  assert(verifySignature(actualDigest, bundle.signature, trustedKey));
  assert(bundle.generation >= localMinimumGeneration(bundle.releaseId));
}
```

防止 rollback attack：撤销或紧急最小 generation 单独由高可用安全通道分发；即使普通控制面不可用，数据面也不能重新接受已撤销 release。

## 3. Reconciliation Loop

控制面采用声明式 desired/observed state，而不是发布 API 中串行调用十个下游。reconciler 必须幂等、可重入、有 generation fence。

```ts
async function reconcile(agentId: string) {
  const desired = await store.getDesired(agentId);
  const observed = await store.getObserved(agentId);
  if (observed.generation >= desired.generation) return;

  const bundle = await compileAndEvaluate(desired);
  await objectStore.putIfAbsent(bundle.digest, bundle);
  await distributor.stage(bundle, desired.targets);
  await rollout.canary({ releaseId: bundle.releaseId, percent: 1 });
  await store.compareAndSetObserved(agentId, observed.version, {
    generation: desired.generation, digest: bundle.digest, phase: "canary"
  });
}
```

状态机可采用 `Draft → Validated → Staged → Canary → Active → Retired/Revoked`。每次跃迁记录 actor、理由、评测结果、diff 和审批；禁止直接修改 Active 对象。

## 4. 数据面分区与关键路径

请求路径只依赖本地/区域内 bundle cache、身份验证、公钥、策略 PDP cache、队列和数据存储。Portal、评测 UI、Git provider、控制面主库不应在 token 生成关键路径。建议按地域/数据区部署 cell：每个 cell 包含 runtime、model/tool gateway、queue、checkpoint、artifact 与 telemetry buffer；单 cell 故障限制爆炸半径。

```text
Global Control Plane ── signed bundles ─┬─ APAC Cell A
                 └── revocation feed    ├─ APAC Cell B
                                       └─ EU Cell A
客户端 → locality router → cell；run 不跨 cell 随意漂移
```

长任务用 home-cell + durable queue；流连接中断不代表任务取消。跨区域灾备必须符合数据驻留：可复制元数据而不复制正文，或使用同区双 cell。

## 5. 配置与策略传播

- Release 配置是不可变快照，允许分钟级传播；运行绑定具体 digest。
- kill switch、凭据吊销、工具禁用是安全快路径，目标秒级传播并 fail closed。
- feature flag 用于体验灰度，不可用来绕过强制安全策略。
- PDP 决策可短缓存，但 key 必须包含 tenant、principal、resource、action、policyVersion 与风险参数摘要。
- 所有 cache 都需要最大陈旧时间；超时后高风险写操作拒绝，低风险读可按明确策略降级。

## 6. 失败模式

- **控制面同步依赖**：配置库抖动导致所有对话失败。用 bundle 与本地缓存。
- **双写发布**：registry 显示 Active，但 bundle 未分发。用状态机/reconciler，不用分布式事务假象。
- **旧实例复活**：滚动发布时加载旧策略。generation fence + 签名 + 最小版本。
- **撤销跟普通配置同 SLA**：攻击期间传播几分钟。独立 revocation channel 与本地 deny cache。
- **跨 cell 状态漂移**：重连路由到另一 cell 找不到 checkpoint。home-cell token 或全局目录只存定位元数据。
- **Canary 只看 5xx**：质量、越权、token 成本恶化未触发回滚。把 eval、安全和成本指标纳入 rollout analysis。

## 7. 练习与验收

关联 Kubernetes controller/operator、GitOps、CRDT/最终一致、cell architecture、PKI、策略即代码、CQRS。结合[交付与演进](delivery-evolution.md)和[可靠性与成本](reliability-cost.md)学习。

练习：实现内存版 desired/observed store、幂等 reconciler 和带签名 bundle cache；注入控制面断网、分发重复、乱序 generation、签名错误与撤销消息延迟。

验收：控制面停机 30 分钟，既有 release 仍满足 SLO；新发布被安全暂停；被撤销工具 60 秒内全 cell 禁用；旧 generation 永不覆盖新版本；所有状态跃迁可重放审计。

## 官方资料

- [Kubernetes Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Kubernetes API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
- [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/)
- [OpenFeature specification](https://openfeature.dev/specification/)
- [SLSA supply-chain levels](https://slsa.dev/spec/)
