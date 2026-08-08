# 附录 C：一手资料阅读清单

带注解的资料来源，使本笔记里的任何论断都能被独立复核。条目按「对准备提上游 PR 的人有多大用」排序，不按字母序。

!!! info "溯源信息"
    链接截至 2026-08 有效。LWS 相关引用指向 `kubernetes-sigs/lws` 的 `32a9c37`（2026-08-06）、v0.9.0。

---

## 第一部分：真相来源

仓库是唯一的权威来源，而本笔记存在的意义是让它可读——不是取代它。

| 来源 | 为什么要读 |
| :--- | :--- |
| [`kubernetes-sigs/lws`](https://github.com/kubernetes-sigs/lws) | 仓库本身。引用时请固定 commit；`main` 每周都在动 |
| `api/leaderworkerset/v1/leaderworkerset_types.go` | 459 行定义了整个核心 API，外加每一个 label 和 annotation 常量。**先读这个。** [模块 2](02_api_surface_anatomy.md) |
| `pkg/controllers/leaderworkerset_controller.go` | 十二步 reconcile 和滚动更新算术。项目里最密的文件。[模块 4](04_lws_reconciler_internals.md) 与 [6](06_rollout_and_revisions.md) |
| `pkg/controllers/pod_controller.go` | 组创建、重启策略、拓扑钉扎。[模块 3](03_group_lifecycle_and_identity.md) 与 [5](05_pod_controller_and_failure_handling.md) |
| `pkg/controllers/disaggregatedset/planner.go` | N 维滚动的纯函数、无 client 实现。项目里文档写得最好的算法。[模块 8](08_disaggregatedset.md) |
| `pkg/webhooks/pod_webhook.go` | 身份注入：环境变量、group key、子组算术、亲和性、TPU。[模块 3](03_group_lifecycle_and_identity.md) |
| [文档站](https://lws.sigs.k8s.io/docs/) | 面向用户的文档。整体准确，具体例外见[附录 B](appendix_pr_opportunities.md) |
| [Releases](https://github.com/kubernetes-sigs/lws/releases) | 每个版本的 `manifests.yaml` 和打包好的 Helm chart |

---

## 第二部分：LWS 的 KEP 语料

全部在仓库的 `keps/` 下。按「能解释多少东西」排序。

| KEP | 状态 | 为什么值得读 |
| :--- | :--- | :--- |
| **766 — DisaggregatedSet** | Alpha，已实现 | 语料里论证最强的一份。它的 **Alternatives** 章节——尤其是关于 `partition` 为何被否的替代方案 4——解释了整个 planner 的存在理由。就算你永远不碰 DisaggregatedSet 也该读 |
| **849 — DisaggregatedSet HPA** | Alpha，已实现 | 一份「坑」清单，每一条都是真实踩过的 bug，带缓解和推理。CEL 与默认值填充的先后陷阱是一条广泛适用的 Kubernetes API 教训 |
| **820 — Fail-Fast Restart Budget** | **Provisional，未实现** | 它的「Why No `Failed` Before」一节是难得的历史背景。注意其中已过时的 `publishNotReadyAddresses` 前提（[附录 B](appendix_pr_opportunities.md) K5） |
| **238 — ControllerRevision** | Stable | LWS 为什么需要 revision 历史，以及一个 revision 里到底装了什么 |
| **407 — Gang Scheduling** | Alpha | **对照代码读**——落差很大（[模块 7](07_scheduling_placement_and_networking.md) §3.1） |
| **511 — Partition Update** | Implementable | condition 语义和截断逻辑的改动才是微妙之处，而 KEP 用 before/after 代码把它讲清楚了 |
| **846 — Slices** | Alpha | 「一个 slice 会轮换多个 revision，而不是反过来」这句话让命名方案变得显然 |
| **848 — Placement Policy** | Alpha | `NotIn` 会匹配缺失 key 这个微妙点，以及一节关于「只有硬约束」的诚实 Risks |
| **115 / 257 — 子组** | Stable | 都很短。257 解释了 `LeaderExcluded` 为什么存在 |
| **135 — Startup Policy** | Stable | 很短。`LeaderReady` 为什么存在、代价是什么 |
| **173 — 每组一个 Headless Service** | Stable | `UniquePerReplica` 的 DNS 规模论据 |
| **552 — 可调整的 worker** | Implementable | 读 Implementation History，别读正文——正文描述的是一个从未被实现的 API |
| **622 — VolumeClaimTemplates** | Implementable | 很短；共用模板这个设计决策就是全部内容 |
| **NNNN — 模板** | — | 必填章节和 `kep.yaml` schema。自己写之前先读 |

---

## 第三部分：上游 Kubernetes 依赖

LWS 的行为有一部分是继承来的。这些是它依赖的上游特性。

| 特性 | 相关性 |
| :--- | :--- |
| [StatefulSet 序号（`StatefulSetStartOrdinal`）](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#start-ordinal) | worker StatefulSet 设 `ordinals.start = 1`。没有它 LWS 会无限递归——这就是 1.26 硬下限的由来（[模块 3](03_group_lifecycle_and_identity.md) §1.3） |
| [StatefulSet 的 `maxUnavailable`](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#maximum-unavailable-pods) | 1.35 起 beta 且默认开。没有它，`maxUnavailable > 1` 会静默表现为 1（[模块 6](06_rollout_and_revisions.md) §9） |
| [Server-Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/) | field manager 与冲突解决。解释了为什么手改 StatefulSet 会被改回去（[模块 4](04_lws_reconciler_internals.md) §4） |
| [Pod 亲和性与反亲和性](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity) | 尤其是 `NotIn` **也会匹配缺失该 key 的对象**——这是 LWS 和 DisaggregatedSet 里两处微妙守卫的来源 |
| [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | per-pod 指标的平均公式，正是「selector 必须每组只匹配一个 Pod」的原因（[模块 2](02_api_surface_anatomy.md) §4） |
| [CRD 结构化 schema 与 CEL 校验](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation-rules) | 默认值填充在 CEL **之前**运行——KEP-849 的那个陷阱 |
| [`kubernetes/kubernetes#64023`](https://github.com/kubernetes/kubernetes/issues/64023) | LWS 名字长度失败背后的 57 字符 StatefulSet 名字上限 |
| [`kubernetes/kubernetes#135017`](https://github.com/kubernetes/kubernetes/issues/135017) | 序列化漂移；被 `SetMatchesRevision` 引用（[模块 4](04_lws_reconciler_internals.md) §5.3） |
| [API 约定](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) | KAL 机械强制的那部分，以及 reviewer 手工强制的那部分 |

---

## 第四部分：姊妹项目与相邻项目

| 项目 | 为什么重要 |
| :--- | :--- |
| [JobSet](https://github.com/kubernetes-sigs/jobset) | 训练侧的对应物。共享设计基因；对比两者能看清 LWS 哪些决策是服务场景特有的 |
| [Kueue](https://kueue.sigs.k8s.io/) | 配额与准入。LWS 靠打 label 集成——完全不涉及 LWS 代码 |
| [Kueue 拓扑感知调度](https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/) | [模块 7](07_scheduling_placement_and_networking.md) §4 里的 `podset-required-topology` 和 `podset-group-name` annotation |
| [Volcano](https://volcano.sh/) | `PodGroup` API 及其 `minMember` / `minResources` 语义——正是这两个字段让 `LeaderReady` 下的 gang 调度正确工作 |
| [llm-d](https://github.com/llm-d/llm-d) | 两个 API 共同设计时的 CNCF sandbox 参考消费方。好几个 KEP 引用了它的需求 |
| [Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/) | 模型感知路由。消费 LWS Pod；DisaggregatedSet 的 revision 感知路由是它要解决的问题 |
| [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) | manager 装配、watch、predicate、client 缓存。它的文档能解释 `cmd/main.go` 的一半 |
| [kube-api-linter](https://github.com/kubernetes-sigs/kube-api-linter) | 拦住 API PR 的那些规则，以及每条规则的理由 |

---

## 第五部分：推理引擎

引擎参数变化很快。**照抄命令之前务必查当前文档**——上游好几个 LWS 示例固定的版本已明显落后。

| 引擎 | 读它的哪部分 |
| :--- | :--- |
| [vLLM 分布式服务](https://docs.vllm.ai/en/latest/serving/distributed_serving.html) | Ray 的 head/worker 模型，以及「TP 在 Pod 内 / PP 跨 Pod」这个选择 |
| [vLLM 分离式 prefill](https://docs.vllm.ai/en/latest/features/disagg_prefill.html) | 从引擎侧看 DisaggregatedSet 到底在编排什么 |
| [SGLang 多节点](https://docs.sglang.ai/) | `--dist-init-addr`、`--nnodes`、`--node-rank`——到 `LWS_*` 契约最干净的映射 |
| [TensorRT-LLM](https://nvidia.github.io/TensorRT-LLM/) | MPI launcher 模型，以及 leader 为什么必须轮询 worker 就绪 |
| [llama.cpp RPC](https://github.com/ggml-org/llama.cpp) | 唯一纯 CPU、可在 kind 上跑的示例；把 LWS 双 Service 模型讲得最清楚的一个 |
| [Ray](https://docs.ray.io/) | vLLM 依赖的 head/worker rendezvous |
| [NVIDIA NIM 多节点](https://docs.nvidia.com/nim/) | 一个推荐使用 LWS 的生产部署 |

---

## 第六部分：演讲

来自上游 adoption 页面。对那些从未进入 KEP 的设计意图很有用。

| 演讲 | 讲者 |
| :--- | :--- |
| KubeCon NA 2024 —— "Distributed Multi-Node Model Inference Using the LeaderWorkerSet API" | @ahg-g、@liurupeng |
| KubeCon EU 2025 —— 分布式训练的内存数据缓存 | @akshaychitneni、@bigsur0 |
| KubeCon EU 2025 —— 闪电演讲 | @kerthcet |
| KubeCon HK 2025 —— 《不止于模型分片》（中文） | @panpan0000、@nicole-lihui |
| KubeCon HK 2025 | @kerthcet |
| KubeCon JP 2025 | @yankay |

2024 KubeCon NA 那场是最初的设计发布，也是「这个 API 为什么长这样」现有最好的表述。

---

## 第七部分：配套笔记

| 来源 | 关系 |
| :--- | :--- |
| [GKE 上 LLM 服务的机密计算](https://marinette101.github.io/llm-cc-in-k8s/) | 本笔记的姊妹篇。其模块 6 把 LWS 用作机密信任边界内部的编排层——硬件 TEE、GPU 远程证明、经证明的权重交付，而 LWS 原封不动地负责多机组。见[模块 9](09_inference_engine_integration.md) §5.4 |

---

## 第八部分：如何核实本笔记里的论断

这里每一条机制论断都可追溯。流程：

```bash
git clone https://github.com/kubernetes-sigs/lws && cd lws
git checkout 32a9c37          # 本笔记描述的那个 commit
```

然后按你要查的内容：

| 论断类型 | 如何核实 |
| :--- | :--- |
| 某个 API 字段或常量 | `grep -n "<名字>" api/leaderworkerset/v1/leaderworkerset_types.go` |
| 某条校验规则 | 读 `pkg/webhooks/leaderworkerset_webhook.go`，然后在活集群上试着违反它 |
| 滚动算术 | 读 `rollingUpdateParameters` 和 `rollingUpdatePartition`，再把[模块 6](06_rollout_and_revisions.md) §6 的推演走一遍 |
| DisaggregatedSet 的论断 | planner 是纯函数——读 `planner.go` 和它 38 KB 的表驱动测试 |
| 「X 没有实现」 | `grep -rn "<标识符>" api/ pkg/ config/ charts/`。四处皆无就是证据 |
| 某处文档缺陷 | 把 `site/content/en/docs/...` 与它所描述的代码路径对照 |

如果本笔记里的某条论断经不住这样的检查，那是笔记错了——代码不会错。欢迎对本笔记提 issue，同时想一想那处差异是不是也是一个上游 PR 机会。
