# 附录 B：PR 机会清单

写作本笔记过程中浮现出的每一个具体的上游贡献机会，附文件路径、证据和难度评级。每一条都是通过读代码而不是读 issue tracker 发现的——所以动手之前**先确认是否已有 issue 或 PR**，并且凡是超出「琐碎」层级的都先开 issue。

!!! info "溯源信息"
    全部发现对照 `kubernetes-sigs/lws` 的 `32a9c37`（2026-08-06）、v0.9.0 核实。行号可能漂移，但给出的 grep 不会。

**难度图例**

| | 含义 |
| :--- | :--- |
| 🟢 | 一个下午。不需要设计讨论。证据能塞进 PR 描述里 |
| 🟡 | 一两天。可能需要一个小测试，或与 reviewer 就方案聊一下 |
| 🟠 | 一周以上。要先开 issue，可能还要 KEP |
| 🔴 | 写代码之前必须先有 KEP 和设计讨论 |

---

## 第 1 层 —— 文档

项目里「用户影响 / 投入」比值最高的一批。每一条都是「照着文档做会得到错误结果」的地方。

| # | 发现 | 位置 | 证据 | ⬤ |
| :--- | :--- | :--- | :--- | :---: |
| D1 | **写了一个不存在的 `SubGroupType: LeaderOnly`**，而且描述的语义正好相反。真实取值是 `LeaderExcluded`，含义是 leader **不在任何**子组 | `site/content/en/docs/concepts/_index.md` | `grep -n "LeaderExcluded\|LeaderOnly" api/leaderworkerset/v1/leaderworkerset_types.go`。见[模块 3](03_group_lifecycle_and_identity.md) §6.2 | 🟢 |
| D2 | **两页在 `MaxUnavailableStatefulSet` 上自相矛盾**：`rollout-strategy` 说 1.35 起 beta 且默认开；`installation` 说「自 v1.24 起仍是 alpha」 | `concepts/rollout-strategy/_index.md` 对 `installation/_index.md` | 两页都随 v0.9.0 发布。见[模块 6](06_rollout_and_revisions.md) §9 | 🟢 |
| D3 | **参考表里少了四个 key**：`subgroup-policy-type`、`experimental-recreate-group-after-start`、`TPU_PROCESS_ADDRESSES`、`TPU_PROCESS_PORT` | `reference/labels-annotations-and-environment-variables.md` | 常量在 `leaderworkerset_types.go:90,98` 和 `pkg/utils/accelerators/tpu.go` | 🟢 |
| D4 | **说要禁用 `internalCertManager`**；真实字段是 `internalCertManagement.enable`。开头那句话语法也是坏的，kustomize 步骤还不如 `hack/e2e-test.sh` 实际做的完整 | `manage/cert_manager.md` | `api/config/v1alpha1/configuration_types.go`。见[模块 10](10_operations_and_troubleshooting.md) §3 | 🟢 |
| D5 | **排障页里没有 Helm CRD 删除风险**——已知最具破坏性的失效模式，却只记在安装页和 chart README 里 | `troubleshooting/_index.md` | 上游 issue #880。见[模块 10](10_operations_and_troubleshooting.md) §2.1 | 🟢 |
| D6 | **复制粘贴的 port-forward**：TensorRT-LLM 页面上写着 `svc/vllm-leader`，而那里根本没有这个 Service。`kubectl get pods` 的输出也与正文矛盾 | `examples/tensorrt-llm.md` | 见[模块 9](09_inference_engine_integration.md) §2.3 | 🟢 |
| D7 | **TAS 示例照抄跑不通**：拓扑层级声明为 `cloud.google.com/gce-topology-block`，而 LWS annotation 引用的是 `cloud.provider.com/topology-block`。另外镜像固定在 `vllm/vllm-openai:v0.8.5`，`tabpane` 只有一个 tab | `examples/tas.md` | 见[模块 7](07_scheduling_placement_and_networking.md) §4 | 🟢 |
| D8 | **已废弃的 `Default` 重启策略取值全站无文档。** 用户既不知道它存在，也不知道它等于 `None` 而不是「默认那个」 | `concepts/failure-handling/_index.md` | `leaderworkerset_types.go:346`。顺手修掉「distributed influence」这个错字 | 🟢 |
| D9 | **实验性 annotation 的文档取值有误导性。** 页面写 `: true`，测试 wrapper 设 `"enable"`；而代码是**存在性检查**，完全不看值 | `concepts/failure-handling/_index.md` | `pkg/controllers/pod_controller.go:220`。见[模块 5](05_pod_controller_and_failure_handling.md) §4.1 | 🟢 |
| D10 | **陈旧镜像**：`nginx:1.14.2`，而站点其余部分用的是 `nginxinc/nginx-unprivileged:1.27` | `examples/hpa.md` | | 🟢 |
| D11 | **`kubectl get endpoints` 已废弃**（应用 EndpointSlice）；另有「llamap.cpp」错字 | `examples/llamacpp.md` | | 🟢 |
| D12 | **两页都是 `weight: 5`**，导致它们的排序未定义 | `examples/hpa.md`、`examples/tas.md` | | 🟢 |
| D13 | **目录名里带空格**：`site/content/en/docs/contribution guidelines/` | | | 🟢 |
| D14 | **社区联系方式互相矛盾**：`CONTRIBUTING.md` 写 `#sig-apps` / `kubernetes-sig-apps`；站点的贡献指南写 Slack `C071WA7R9LY` / `wg-serving` | | 见[模块 11](11_contributor_workflow.md) §7.3 | 🟢 |
| D15 | **排障页自相矛盾**：原因写「< 1.27」，修法写「≥ 1.26」 | `troubleshooting/_index.md` | 见[模块 10](10_operations_and_troubleshooting.md) §5.1 | 🟢 |
| D16 | **四个已发布的特性没有概念文档**：`subdomainPolicy`、用户可设的 `partition`、可调整的 `size`、gang 调度 | `site/content/en/docs/concepts/` | 模块 [3](03_group_lifecycle_and_identity.md)、[6](06_rollout_and_revisions.md)、[7](07_scheduling_placement_and_networking.md) | 🟡 |
| D17 | **overview 的特性列表漏了 gang 调度**，而 `README.md` 是列了的。两份列表已经分岔 | `overview/_index.md` | | 🟢 |
| D18 | **一段残留的尾部章节**（「Install with Helm chart —— 请参考 release 页面」），与上面完整的 Helm 章节重复 | `installation/_index.md` | | 🟢 |
| D19 | **pre-v0.9.0 的 DS 升级片段只 apply 了两个 DS CRD 中的一个**，漏了 chart 附带的 `disaggregatedsetrolescalers` | `installation/_index.md` | `charts/lws/crds/` 里有三个文件 | 🟢 |

---

## 第 2 层 —— 项目元数据与治理

| # | 发现 | 位置 | 证据 | ⬤ |
| :--- | :--- | :--- | :--- | :---: |
| G1 | **`CONTRIBUTING.md` 完全是未修改的样板**，连遗留的 HTML 注释都在。完全没提 `make test`、`make verify`、envtest、wrapper、KEP，也没提 GNU sed 的要求 | `CONTRIBUTING.md` | 见[模块 11](11_contributor_workflow.md) §7.4。**可以说是仓库里杠杆率最高的贡献者 PR** | 🟡 |
| G2 | **`RELEASE.md` 是模板文本而且是明确错误的**——仍然写着「The Kubernetes **Template Project**」，描述的 `git tag -s` 流程与 `make artifacts` / `make prepare-release-branch` / `cloudbuild.yaml` / `hack/cherry_pick_pull.sh` 对不上 | `RELEASE.md` | | 🟡 |
| G3 | **`SECURITY_CONTACTS` 里列着一位荣休 approver**——`liurupeng` 在 `OWNERS` 里是 `emeritus_approvers` | `SECURITY_CONTACTS` | | 🟢 |
| G4 | **`hack/genref/config.yaml` 指向 Kubernetes 1.28 的 API 参考**，而项目构建用的是 `k8s.io/api v0.36.3`。生成的参考文档里每个交叉链接都指向 1.28 | `hack/genref/config.yaml` | 一行改动 | 🟢 |

---

## 第 3 层 —— KEP 语料清理

每一条都是自足的文档修复，而且都是把某份 KEP 读细的好借口。

| # | 发现 | 位置 | ⬤ |
| :--- | :--- | :--- | :---: |
| K1 | **目录名是 `115`，`kep-number` 却写 `127`**，且 `creation-date: yyyy-mm-dd` 从未填写 | `keps/115-Subgroup-support/kep.yaml` | 🟢 |
| K2 | **标题和编号是从 KEP-238 复制过来的** | `keps/257-Subgroup-leader-only/kep.yaml` | 🟢 |
| K3 | **描述了一个从未被实现的 `ResizePolicy` API 字段。** Implementation History 记录了转向（`2025-08-05: Implementation revised to avoid additions to the API surface`），但正文从未更新。它也是唯一一份既无 `latest-milestone`、又无 `stage`、也无 `milestone` 块的内容型 KEP | `keps/552-worker-resizing/` | 🟢 |
| K4 | **描述的 `PodGroupPerReplica` feature gate、三个 provider 接口、PodGroup 命名方案，与实际落地的全都不同。** `grep -rn "PodGroupPerReplica"` 只命中 `kep.yaml:35` | `keps/407-gang-scheduling/` | 🟡 |
| K5 | **`publishNotReadyAddresses` 的前提已过时。** LWS 自己的 headless Service 硬编码为 `true`，还有测试断言，因此一个默认 `false` 的开关是回归风险。另外 Implementation History 的日期早于 `kep.yaml` 里的 `creation-date` | `keps/820-distributed-preflight-check/` | 🟢 |
| K6 | **KEP-766 说 `RolloutStrategy` 不会传播到子 LWS；而 `lws_manager.go` 会传播**（虽然惰性）。文本或代码总得改一个 | `keps/766-DisaggregatedSet/` | 🟢 |
| K7 | **KEP-849 说 `status.selector`「只在 scaler 创建时写一次」**，而实现每次 reconcile 都会原样重写 | `keps/849-DisaggregatedSet-HPA/` | 🟢 |
| K8 | **好几份 `status: implementable` 的 KEP 里还留着 `<test>: <link to test coverage>` 占位符**，而 135/173/511/552/622 缺少 `stage:` / `milestone:` 块 | `keps/` | 🟢 |
| K9 | **`spec.slices` 在代码里有 `Maximum=100`，KEP 的 API 块里却没有上限** | `keps/846-disaggregatedset-slices/` | 🟢 |

!!! tip "K5 是与维护者建立联系的最佳开场"
    它只需要两次 grep，能改进一份在议的提案，而且是评论而不是 PR——低风险的自我介绍方式。

---

## 第 4 层 —— 代码：边界清晰的修复

| # | 发现 | 位置 | 证据 | ⬤ |
| :--- | :--- | :--- | :--- | :---: |
| C1 | **LWS 校验器里潜伏的空指针解引用。** `generalValidate` 在下面两行的 `!= nil` 守卫**之前**就读了 `RollingUpdateConfiguration.MaxUnavailable`。defaulter 在跑时不可达，但被绕过的 mutating webhook 会让校验器 panic 而不是拒绝 | `pkg/webhooks/leaderworkerset_webhook.go` | 见[模块 2](02_api_surface_anatomy.md) §5.3 | 🟢 |
| C2 | **`topologyValueFromPod` 用 `IgnoreNotFound` 吞掉了不存在的 Node**，返回 `("", nil)` 并设出一个**匹配不到任何东西的空 node selector 值**——一个无法调度、又没有清晰报错的 worker StatefulSet | `pkg/controllers/pod_controller.go` | 见[模块 5](05_pod_controller_and_failure_handling.md) §6 | 🟢 |
| C3 | **拓扑报错信息插值的是空的值而不是 key**：`"node does not have topology label: %s"` 完全没告诉你缺的是哪个 label | `pkg/controllers/pod_controller.go` | 与 C2 同一函数 | 🟢 |
| C4 | **硬编码的 envtest `1.28.3`**，而 `ENVTEST_K8S_VERSION = 1.36.0`。通过 `make test-integration` 无害（KUBEBUILDER_ASSETS 胜出），但文档里写明的直接运行路径是坏的 | `test/integration/controllers/suite_test.go` | 见[模块 11](11_contributor_workflow.md) §3.2 | 🟢 |
| C5 | **一个陈旧的 Ginkgo entry 名字**引用了 `subdomain policy LeadersSharedWorkersDedicated`，那是 KEP-173 里被否掉、从未发布的取值。真实取值是 `UniquePerReplica` | `test/integration/controllers/leaderworkerset_test.go` | | 🟢 |
| C6 | **DisaggregatedSet 子系统里的死代码**：`NumRequiredRoles`（零引用）、`ComputeInitialReplicaState`（只有测试用）、自由函数 `SetInitialReplicas`（无引用）、`ServiceManager.scheme`（存了从不用）、`sortByNewestTimestamp` 的 `roleNames` 参数、`ValidateUpdate` 的 `oldDisagg` 参数 | `pkg/utils/disaggregatedset/`、`pkg/controllers/disaggregatedset/` | 见[模块 8](08_disaggregatedset.md) §10.1 | 🟢 |
| C7 | **一个形同虚设的 field index。** `lwsOwnerKey` 被注册了，却从未在任何 `client.MatchingFields` 查询里使用——纯粹的 informer 缓存开销。要么接上，要么删掉 | `pkg/controllers/leaderworkerset_controller.go` | `grep -rn "lwsOwnerKey"` 只找到定义和注册。见[模块 4](04_lws_reconciler_internals.md) §1.4 | 🟢 |
| C8 | **`SetupIndexes` 的错误只记日志、不致命**，导致启动期失败之后 manager 以半初始化状态继续运行 | `cmd/main.go` | | 🟢 |
| C9 | **过宽的 RBAC**：Pod 控制器持有对 `nodes` 的 `update` 和 `patch`，而核心控制面里没有任何代码路径用到 | `pkg/controllers/pod_controller.go:67` | 见[模块 5](05_pod_controller_and_failure_handling.md) §5 | 🟢 |
| C10 | **`hashRevision` 里潜伏的 bug**：`deepHashObject` 在写入前调用 `hasher.Reset()`，所以若 `Data.Raw` 和 `Data.Object` 同时有值，`Raw` 的字节会被静默丢弃。今天不可达 | `pkg/utils/revision/revision_utils.go` | 见[模块 4](04_lws_reconciler_internals.md) §5.2 | 🟢 |
| C11 | **DS planner 里的浮点算术**——`int(float64(a) * float64(b) / float64(c))`。整数运算完全等价且不受浮点边界影响 | `pkg/controllers/disaggregatedset/planner.go` | | 🟡 |
| C12 | **Pod validating webhook 是空实现**——`validate()` 返回 `(nil, nil)`。要么删掉，要么给它安排用途 | `pkg/webhooks/pod_webhook.go` | | 🟡 |
| C13 | **一段误导性注释**：`setNodeSelectorForWorkerPods` 里描述了「后续 apply 逻辑会自动更新它」的行为，而只创建的代码并不实现它 | `pkg/controllers/pod_controller.go` | 见[模块 5](05_pod_controller_and_failure_handling.md) §2 | 🟢 |
| C14 | **`initial-replicas` 的写失败被吞掉**——只 `log.Error`，静默地把该 role 的排空基线降级为 `spec.Replicas`，产生错误的排空轨迹却没有任何错误浮出 | `pkg/controllers/disaggregatedset/executor.go` | 见[模块 8](08_disaggregatedset.md) §4.6 | 🟡 |
| C15 | **`extractRollingUpdateConfig` 不对称**：`unavail > 0` 时两个字段都设；只有 `surge > 0` 时只设一个；两者都为 0 时静默退回 `MaxSurge=1` | `pkg/controllers/disaggregatedset/executor.go` | | 🟡 |

---

## 第 5 层 —— 代码：特性与校验缺口

| # | 发现 | 位置 | 理由 | ⬤ |
| :--- | :--- | :--- | :--- | :---: |
| F1 | **`DisaggregatedSet.status` 从不被写。** API 声明了 `RoleStatuses` 和 `Conditions`，CRD 有 status 子资源，RBAC 也授予了，上游文档还叫用户去看它——但没有任何代码写它 | `pkg/controllers/disaggregatedset/` | **项目里最好的贡献机会。** 所有输入每次 reconcile 都已经取到了。见[模块 8](08_disaggregatedset.md) §10 | 🟠 |
| F2 | **`DisaggregatedSet` 上没有 printcolumn**，所以 `kubectl get disaggregatedset` 只显示 NAME 和 AGE。天然与 F1 配对 | `api/disaggregatedset/v1/disaggregatedset_types.go` | | 🟢 |
| F3 | **没有任何 LWS 专有或 DS 专有的 Prometheus 指标。** KEP-766、KEP-846、KEP-849 都把指标列为 Beta 标准 | | 有 KEP 背书、动机充分。见[模块 10](10_operations_and_troubleshooting.md) §4 | 🟠 |
| F4 | **DS role 上的非模板 spec 变更被静默丢弃。** `ComputeRevision` 只哈希 role 名 + `LeaderWorkerTemplate`，因此改 `networkConfig`、`startupPolicy`、`rolloutStrategy` 或 role `metadata` 会得到同一个 revision，既不创建新 LWS 也不 patch 已有的 | `pkg/utils/disaggregatedset/utils.go` | **DS 子系统里对用户最尖锐的可见 bug。** 先开 issue——修法里含有设计选择 | 🟠 |
| F5 | **DS 的 Service 只创建、从不修复。** `ensureService` 吞掉 `IsAlreadyExists`；漂移的 selector 永远不会被协调回来。控制器也不 watch Service | `pkg/controllers/disaggregatedset/service_manager.go` | 修法：加 `Owns(&corev1.Service{})` 并做 spec diff-and-patch | 🟡 |
| F6 | **`volumeClaimTemplates` 完全没有校验**——既没有 API 注释承诺的「名称与 `volumeMount` 交叉检查」，也没有不可变性检查。由于 StatefulSet 自己的该字段不可变，改动会以 `FailedUpdate` 事件而不是干净的准入错误呈现 | `pkg/webhooks/leaderworkerset_webhook.go` | 见[模块 7](07_scheduling_placement_and_networking.md) §6.1 | 🟡 |
| F7 | **PVC 透传有损且静默**——只复制 `accessModes`、`storageClassName`、`volumeMode` 和 `resources`。`selector`、`dataSource`、`dataSourceRef` 和 PVC 的 label/annotation 被无声丢弃 | `pkg/utils/controller/controller_utils.go` | 最低限度是写进文档；更好的是把字段透传过去，或者显式拒绝 | 🟡 |
| F8 | **没有名字长度校验。** 真实上限是 `51 - int(replicas/10)` 个字符，超了会以一个离 LWS 很远的 StatefulSet label 错误呈现 | `pkg/webhooks/leaderworkerset_webhook.go` | 见[模块 10](10_operations_and_troubleshooting.md) §5.2 | 🟡 |
| F9 | **KAL 不检查 `api/disaggregatedset/` 和 `api/config/`。** 排除规则是 `path-except: "api/leaderworkerset/*"`。而 DS 恰恰是最新、变动最快的 API | `.golangci-kal.yml` | 预计会有 `commentstart` 违规。见[模块 2](02_api_surface_anatomy.md) §8.2 | 🟡 |
| F10 | **没有 DisaggregatedSet 控制器的集成测试。** `test/integration/controllers/` 只有 `leaderworkerset_test.go`；DS 的覆盖靠单测加 e2e。KEP-766 的集成测试计划未达成 | `test/integration/controllers/` | | 🟠 |
| F11 | **实验性重启 annotation 不看自己的值**（只做存在性检查），因此设成 `false` 反而会启用该特性。既然 `RecreateGroupAfterStart` 已是一等枚举值，干净的修法是显式地大声废弃它 | `pkg/controllers/pod_controller.go:220` | 见[模块 5](05_pod_controller_and_failure_handling.md) §4.1 | 🟡 |
| F12 | **未声明的 Helm 值。** `metrics.prometheusNamespace` 和 `metrics.serviceMonitor` 有文档，却不在 `charts/lws/values.yaml` 和 chart README 里 | `charts/lws/` | 见[模块 10](10_operations_and_troubleshooting.md) §4 | 🟢 |
| F13 | **冗余的跨 slice `List` 调用**——`updateScalerStatus` 和 `seedForRole` 各在 per-slice 列举之上再发一次。虽走缓存，但 100 个 slice 时很浪费 | `pkg/controllers/disaggregatedset/` | | 🟡 |
| F14 | **DisaggregatedSet 完全没有不可变性校验。** `ValidateUpdate` 与 `ValidateCreate` 完全相同，`oldDisagg` 未被使用。role 名字可变，而改名会静默地重建该 role 的整个集群 | `pkg/webhooks/disaggregatedset/` | 先定下什么应当不可变，再去强制它 | 🟡 |

---

## 第 6 层 —— 更大的工作，需要 KEP

| # | 机会 | 理由 | ⬤ |
| :--- | :--- | :--- | :---: |
| P1 | **加一个 `revisionHistoryLimit` 字段。** LWS 只保留一个 revision，所以 `kubectl rollout undo` 无从下手，回滚只能手工重新 apply 旧 manifest。Deployment 和 StatefulSet 都有先例 | 🔴 |
| P2 | **实现 KEP-820 的 fail-fast 重启预算。** `maxGroupRestarts`、一个 `Failed` condition、一个组重启计数器。目前 `status: provisional` 且零实现——注意按 K5，它的后半部分（`publishNotReadyAddresses` 开关）应当被删掉 | 🔴 |
| P3 | **加第二个 `SchedulerProvider`。** 接口是为多个而设计的，`SupportedSchedulerProviders` 却只有一项。上游 coscheduling 插件只需要一个 label，不需要 CRD | 🟠 |
| P4 | **一个原生的「所有 worker 就绪」信号。** TensorRT-LLM 示例给 Pod 授予 RBAC 让 leader 去轮询 API server——这是缺少原语的有力证据。KEP-135 的 `LeaderReady` 只解决了反方向 | 🔴 |
| P5 | **真正的 `component-base` feature gate。** 目前完全没有 feature gate 机制；开关就是 `Configuration` CRD 加一个 `experimental-` annotation。先开 issue 讨论——这是架构而非清理 | 🟠 |
| P6 | **让 leader 和 worker 有各自的 `volumeClaimTemplates`。** KEP-622 明确共用一个字段。一个需要与 worker 不同存储的 leader 无法表达 | 🔴 |
| P7 | **给 DisaggregatedSet 的 slice 做 gang 调度。** KEP-848 承认只有硬约束的放置在缺少整 slice 前瞻时可能把 slice 卡死，并推迟了 gang 调度 | 🔴 |
| P8 | **多 slice 的 External 伸缩。** webhook 禁止 `slices > 1` 与任何 External role 并存，标注为 `// Alpha:` 并推迟到后续 KEP（issue #948） | 🔴 |

---

## 建议的顺序

1. **第 1 层里挑一个文档修复**——D1 的用户影响最大。在一件没争议的事情上把 PR 流程走熟。
2. **一条 KEP 评论**，而不是 PR——K5。两次 grep，真实价值，低风险。
3. **第 4 层里挑一个带测试的、边界清晰的代码修复**——C1、C4 或 C6。
4. **`CONTRIBUTING.md`**（G1），在你自己走过一遍流程之后。你正是写它的合适人选，恰恰因为你刚刚一路踩着坑学完。
5. **为你在意的第 5 层条目开一个 issue**，先看维护者怎么回应，再动手写代码。
6. **F1（DisaggregatedSet status）**，如果你想做点有分量的。它边界清晰，所有输入都已存在，API 也已经把形状声明好了。

!!! warning "动手之前"
    先搜 issue tracker 和已开的 PR。本附录是在某一个 commit 上读代码整理出来的，而项目推进得很快——其中若干条可能已经被修掉、被认领，或被刻意推迟。一个与既有工作重复的 PR 会浪费 reviewer 的时间，而在一个只有四位 approver 的项目里，那是最稀缺的资源。
