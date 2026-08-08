# 附录 A：术语总表

本笔记用到的全部术语，按领域分组。凡是某个术语在 LWS 语境下的含义与通用 Kubernetes 含义不同的，两者都给出。

!!! info "溯源信息"
    对照 `kubernetes-sigs/lws` 的 `32a9c37`（2026-08-06）、v0.9.0 核实。

---

## LeaderWorkerSet 核心

| 术语 | 含义 |
| :--- | :--- |
| **LWS** | LeaderWorkerSet。同时也是 CRD 短名（`kubectl get lws`）和 SSA field manager 的名字 |
| **组（Group）** | 一个 leader Pod 加 `size - 1` 个 worker Pod，作为单一复制单元管理。API 文档里也叫「replica」，README 里叫「super pod」 |
| **组索引（group index）** | 标识一个组的序号 `0..replicas-1`，来自 leader StatefulSet 的序号。label 为 `leaderworkerset.sigs.k8s.io/group-index` |
| **worker 索引** | 组内序号：leader 为 `0`，worker 为 `1..size-1`。label 为 `leaderworkerset.sigs.k8s.io/worker-index`；环境变量为 `LWS_WORKER_INDEX` |
| **组 key（group key）** | `sha1("<namespace>/<leaderPodName>")`——40 字符十六进制摘要，组内所有 Pod 共享，用作独占放置的亲和性 key。组重建时会改变 |
| **leader Pod** | `worker-index=0` 的那个 Pod。拥有 worker StatefulSet、每组 Service 和 Volcano PodGroup。它的名字**就是**组名 |
| **leader StatefulSet** | 每个 LWS 唯一的一个 StatefulSet，其副本即 leader Pod。selector：`name=<lws>, worker-index=0` |
| **worker StatefulSet** | 每组一个，以其 leader Pod 命名，由 leader **Pod** 拥有。`ordinals.start = 1`，`replicas = size - 1` |
| **子组（Subgroup）** | 把一个组切成等大的单元，每个单元可占据自己的拓扑域。KEP-115、KEP-257 |
| **`size`** | `leaderWorkerTemplate.size`——每组 Pod 总数，含 leader。可变（KEP-552） |
| **`replicas`** | `spec.replicas`——**组**的数量，不是 Pod 数量。scale 子资源的目标 |
| **burst / surge 副本** | `maxSurge` 滚动期间创建的、超出 `spec.replicas` 的组。计入 `status.replicas`，但被 condition 算术排除 |
| **revision key** | `leaderworkerset.sigs.k8s.io/template-revision-hash` 的值。滚动路径中「已更新」的**唯一**定义 |
| **Partition** | 低于该组索引的组都不更新。既是控制器内部变量，自 KEP-511 起也是用户可设字段 |
| **rolling step（滚动步长）** | `maxUnavailable + 已物化的 surge`。partition 每步下降多少 |
| **独占拓扑** | annotation 驱动的放置：一个拓扑域独占一个组。实现为 leader 上的 Pod 亲和性加 worker 上的 `nodeSelector` |

## DisaggregatedSet

| 术语 | 含义 |
| :--- | :--- |
| **DS** | DisaggregatedSet。管理 N 个版本耦合的 LeaderWorkerSet |
| **Role** | DisaggregatedSet 的一个具名层级——典型为 `prefill` 和 `decode`。每个 DS 2–10 个 |
| **Slice** | 整个 role 拓扑的一份独立副本，独立滚动。映射到放置域的持久身份。KEP-846 |
| **Revision（DS）** | 仅对 role 名和 `LeaderWorkerTemplate` 做 SHA-256，取 8 个十六进制字符。其余一切都在哈希之外 |
| **`UpdateStep`** | planner 的输出：按 role 索引给出的绝对目标副本数（`Past`、`New`）。是绝对值而非增量，因此重复施加是幂等的 |
| **`totalSteps`** | 各 role 的 `ceil(maxReplicas / batchSize)` 取最大值。齐步走的节拍器 |
| **批次大小（batch size）** | `maxSurge` 非零时取它，否则取 `max(1, maxUnavailable)` |
| **Orphan prevention** | 「在另一个 role 非零时，任何 role 都不得归零，除非所有 role 能一次全部归零」这条规则。KEP-766 Property 3 |
| **Force drain** | 当 surge 预算卡住扩容时，偏离轨迹地把旧副本排空到刚好够容纳下一次扩容的那一步 |
| **`initial-replicas`** | 记录各 role 在滚动开始时副本数的 annotation——排空插值的分母 |
| **`dsrs`** | `DisaggregatedSetRoleScaler`，为 `scaling.mode: External` 创建的 per-role `/scale` 目标。KEP-849 |
| **Static / External 伸缩** | `RoleScalingMode`。Static 用内联的 `spec.replicas`；External 委托给一个 `dsrs` |
| **no-shrink 守卫** | 滚动中途防止 HPA 写入更小值时缩小 External role 新 revision 集群的规则 |
| **`-prv` Service** | 每 `(slice, revision, role)` 一个的 headless Service。使 revision 感知路由成为可能 |

## Kubernetes

| 术语 | 含义 |
| :--- | :--- |
| **CRD** | CustomResourceDefinition——注册 `LeaderWorkerSet` 等类型的 schema |
| **CEL** | Common Expression Language。用于 DS 的 all-or-nothing replicas 规则。在结构化默认值填充**之后**运行 |
| **ControllerRevision** | LWS 用来快照 `leaderWorkerTemplate` + `networkConfig` 的内置类型。KEP-238 |
| **SSA** | Server-Side Apply。LWS 用 field manager `lws` 和 `Force: true` apply leader StatefulSet |
| **field manager** | `metadata.managedFields` 里记录「哪个执行者拥有哪个字段」的归属信息 |
| **apply configuration** | 用来构造 SSA 补丁的生成式 builder 类型（`appsapplyv1.StatefulSet(...)`） |
| **owner reference** | 驱动垃圾回收的链接。在 LWS 里 worker StatefulSet 的 owner 是 leader **Pod** |
| **foreground 传播** | 保证被依赖对象先于 owner 消失的删除语义。组重建时删除 leader 用的就是它 |
| **envtest** | 跑真实 apiserver 和 etcd、但没有 kubelet 和 webhook 的测试脚手架 |
| **Ginkgo / Gomega** | `test/integration` 和 `test/e2e` 用的 BDD 测试框架和匹配器库 |
| **kind** | Kubernetes in Docker。e2e 的底座 |
| **Prow** | Kubernetes 的 CI 系统。LWS 的重型 presubmit 跑在那里，配置在仓库之外 |
| **KAL** | Kube API Linter（`sigs.k8s.io/kube-api-linter`）。强制 API 约定；只对 `api/leaderworkerset/*` 运行 |
| **KEP** | Kubernetes Enhancement Proposal。`keps/<编号>-<标题>/{README.md,kep.yaml}` |
| **PRR** | Production Readiness Review。`kep.yaml` 里的 `feature-gates`、`disable-supported`、`metrics` 字段 |
| **`StatefulSetStartOrdinal`** | 让 StatefulSet 序号从非零值开始的上游特性。LWS 的 worker 序号依赖它 |
| **`MaxUnavailableStatefulSet`** | 让 StatefulSet 支持 `maxUnavailable > 1` 的上游特性。Kubernetes 1.35 起 beta 且默认开启 |
| **`HPAScaleToZero`** | 让 HPA 能伸缩到零、从零起的上游 gate。它默认不开，正是 DS scaler 播种为 1 的原因 |
| **Downward API** | 把 Pod 的 label 和 annotation 以文件或环境变量形式暴露给容器的机制 |

## 推理与并行

| 术语 | 含义 |
| :--- | :--- |
| **TP** | 张量并行。把每个权重矩阵跨 rank 切开；每个 Transformer 层两次 `all-reduce`。对带宽极其敏感 |
| **PP** | 流水并行。把层切成顺序 stage，边界上点对点传递激活值。对跨机链路容忍度高 |
| **EP** | 专家并行。把 MoE 专家跨 rank 切开；每个 MoE 层一次 `all-to-all` token 分发。吃带宽且突发 |
| **DP** | 数据并行。互相独立的副本，副本间无通信。`spec.replicas` 数的就是它 |
| **Prefill** | 处理整个 prompt、构建 KV cache 的阶段。算力受限 |
| **Decode** | 一次生成一个 token 的阶段。显存带宽受限 |
| **P/D 分离** | 把 prefill 和 decode 跑在独立池里，中间传 KV cache。DisaggregatedSet 存在的理由 |
| **KV cache** | 缓存的注意力 key 和 value，避免解码步骤间重复计算 |
| **TTFT** | 首 token 时间。prefill 阶段的 SLO |
| **ITL / TPOT** | token 间延迟 / 每个输出 token 的时间。decode 阶段的 SLO |
| **Continuous batching** | 每一步都往运行中的批次里加减序列。它把 prefill 和 decode 交织，造成队头阻塞 |
| **Rendezvous** | 各 rank 互相发现、组成集合通信组的启动握手 |
| **World size** | 参与的 rank 总数。`LWS_GROUP_SIZE` 映射到它 |
| **NVLink / NVSwitch** | NVIDIA 的机内 GPU 互联，数百 GB/s。正是这条边界让多机变难 |
| **RDMA / RoCE / EFA** | 用于跨机 GPU 通信的远程直接内存访问 fabric |
| **HBM** | 高带宽内存——加速器的封装内内存 |
| **MoE** | 专家混合。把每个 token 路由到一部分专家子网络 |

## 生态

| 项目 | 与 LWS 的关系 |
| :--- | :--- |
| **JobSet** | SIG-Apps 中面向多机**训练**的姊妹 API。共享设计基因，不是竞品 |
| **Kueue** | 配额、准入、拓扑感知调度、DWS flex-start。LWS 靠打 label 来集成 |
| **Volcano** | 提供 `PodGroup` gang 调度的批调度器。唯一被实现的 `SchedulerProvider` |
| **TAS** | Kueue 的 Topology Aware Scheduling。按层级放置，由 Pod 模板 annotation 驱动 |
| **DWS** | Dynamic Workload Scheduler。Google Cloud 的容量供给，含 flex-start |
| **HPA** | HorizontalPodAutoscaler。通过 scale 子资源驱动 `spec.replicas`，selector 只选 leader |
| **KEDA** | 事件驱动自动伸缩器。另一个 `/scale` 写入方，且支持缩到零 |
| **llm-d** | CNCF sandbox 服务栈；两个 API 共同设计时的参考消费方 |
| **Gateway API Inference Extension** | 模型感知与 prefix-cache 感知的路由。把 LWS Pod 当后端 |
| **vLLM / SGLang / TensorRT-LLM / llama.cpp** | 有上游 LWS 示例的推理引擎（[模块 9](09_inference_engine_integration.md)） |
| **NVIDIA Dynamo / llmaz / OME / AXLearn / Kubeflow Trainer** | 上游列出的集成方 |
| **Ray** | vLLM 多机所用的分布式运行时。leader 是 head node |
| **cert-manager** | LWS 内置证书轮换的可选替代 |
| **Prometheus Operator** | 消费 `config/components/prometheus` 里附带的 `ServiceMonitor` |

## 文件与标识符

| 标识符 | 是什么 |
| :--- | :--- |
| `leaderworkerset.x-k8s.io/v1` | LWS 的 API group 与版本 |
| `disaggregatedset.x-k8s.io/v1` | DS 的 API group 与版本 |
| `config.lws.x-k8s.io/v1alpha1` | 控制器的 `Configuration` 文件格式——**不是**集群资源 |
| `leaderworkerset.sigs.k8s.io/*` | label 和 annotation 的域名。**与 API group 不同** |
| `b8b2488c.x-k8s.io` | 默认的 leader election 资源名 |
| `lws-system` | 默认安装命名空间 |
| `lws-ca` / `lws` | 内置 CA 的名字与组织 |
| `rollingUpdateParameters()` | 计算 `(partition, replicas)` 的函数——[模块 6](06_rollout_and_revisions.md) |
| `ComputeNextStep()` | DS planner 的唯一入口——[模块 8](08_disaggregatedset.md) |
| `SetExclusiveAffinities()` | 注入组级与子组级放置亲和性——[模块 7](07_scheduling_placement_and_networking.md) |
| `handleRestartPolicy()` | 组重建的决策——[模块 5](05_pod_controller_and_failure_handling.md) |
| `AddLWSVariables()` | 注入三个 `LWS_*` 环境变量——[模块 3](03_group_lifecycle_and_identity.md) |
| `GetParentNameAndOrdinal()` | 从 Pod 名解析 `<parent>-<ordinal>`。含义随调用方而异 |
