# Appendix A: Glossary

Every term used in these notes, grouped by domain. Where a term has a specific LWS meaning that differs from its general Kubernetes meaning, both are given.

!!! info "Provenance"
    Verified against `kubernetes-sigs/lws` at `32a9c37` (2026-08-06), release v0.9.0.

---

## LeaderWorkerSet Core

| Term | Meaning |
| :--- | :--- |
| **LWS** | LeaderWorkerSet. Also the CRD short name (`kubectl get lws`) and the SSA field manager name |
| **Group** | One leader pod plus `size - 1` worker pods, managed as a single unit of replication. Also called a "replica" in the API docs and a "super pod" in the README |
| **Group index** | The ordinal `0..replicas-1` identifying a group. Comes from the leader StatefulSet's ordinal. Label `leaderworkerset.sigs.k8s.io/group-index` |
| **Worker index** | The ordinal within a group: `0` for the leader, `1..size-1` for workers. Label `leaderworkerset.sigs.k8s.io/worker-index`; env var `LWS_WORKER_INDEX` |
| **Group key** | `sha1("<namespace>/<leaderPodName>")` — a 40-character hex digest shared by all pods in a group, used as the exclusive-placement affinity key. Changes when the group is recreated |
| **Leader pod** | The pod with `worker-index=0`. Owns the worker StatefulSet, the per-group Service, and the Volcano PodGroup. Its name **is** the group name |
| **Leader StatefulSet** | The single StatefulSet per LWS whose replicas are the leader pods. Selector: `name=<lws>, worker-index=0` |
| **Worker StatefulSet** | One per group, named after its leader pod, owned by the leader **Pod**. `ordinals.start = 1`, `replicas = size - 1` |
| **Subgroup** | A partition of a group into equal-sized units, each able to claim its own topology domain. KEP-115, KEP-257 |
| **`size`** | `leaderWorkerTemplate.size` — total pods per group, leader included. Mutable (KEP-552) |
| **`replicas`** | `spec.replicas` — the number of **groups**, not pods. The scale-subresource target |
| **Burst / surge replica** | A group beyond `spec.replicas` created during a `maxSurge` rollout. Counted in `status.replicas` but excluded from the condition arithmetic |
| **Revision key** | The value of `leaderworkerset.sigs.k8s.io/template-revision-hash`. The **only** definition of "updated" in the rollout path |
| **Partition** | The group index below which nothing updates. Both a controller-internal variable and, since KEP-511, a user-settable field |
| **Rolling step** | `maxUnavailable + (surge already materialized)`. How far the partition descends per step |
| **Exclusive topology** | Annotation-driven placement: one group per topology domain, exclusively. Implemented as pod affinity on the leader plus a `nodeSelector` on the workers |

## DisaggregatedSet

| Term | Meaning |
| :--- | :--- |
| **DS** | DisaggregatedSet. Manages N version-coupled LeaderWorkerSets |
| **Role** | A named tier of a DisaggregatedSet — typically `prefill` and `decode`. 2–10 per DS |
| **Slice** | An independent copy of the whole role topology, rolling out independently. The durable identity that maps to a placement domain. KEP-846 |
| **Revision (DS)** | An 8-hex-character SHA-256 over role name plus `LeaderWorkerTemplate` only. Everything else is outside the hash |
| **`UpdateStep`** | The planner's output: absolute target replica counts (`Past`, `New`) per role index. Absolute, not deltas, so re-application is idempotent |
| **`totalSteps`** | `max` over roles of `ceil(maxReplicas / batchSize)`. The lockstep clock |
| **Batch size** | `maxSurge` if non-zero, else `max(1, maxUnavailable)` |
| **Orphan prevention** | The rule that no role may reach 0 while another is non-zero, unless all can go to 0 at once. KEP-766 Property 3 |
| **Force drain** | The off-trajectory step that drains old replicas just enough to admit the next scale-up when surge budget blocks it |
| **`initial-replicas`** | The annotation recording each role's replica count at rollout start — the drain interpolation's denominator |
| **`dsrs`** | `DisaggregatedSetRoleScaler`, the per-role `/scale` target created for `scaling.mode: External`. KEP-849 |
| **Static / External scaling** | `RoleScalingMode`. Static uses inline `spec.replicas`; External delegates to a `dsrs` |
| **No-shrink guard** | The mid-rollout rule preventing an External role's new-revision fleet from shrinking when HPA writes a smaller value |
| **`-prv` Service** | The per-`(slice, revision, role)` headless Service. Enables revision-aware routing |

## Kubernetes

| Term | Meaning |
| :--- | :--- |
| **CRD** | CustomResourceDefinition — the schema registering `LeaderWorkerSet` and friends |
| **CEL** | Common Expression Language. Used for the DS all-or-nothing replicas rule. Runs **after** structural defaulting |
| **ControllerRevision** | The built-in type LWS uses to snapshot `leaderWorkerTemplate` + `networkConfig`. KEP-238 |
| **SSA** | Server-Side Apply. LWS applies the leader StatefulSet with field manager `lws` and `Force: true` |
| **Field manager** | The `metadata.managedFields` attribution recording which actor owns which field |
| **Apply configuration** | Generated builder types (`appsapplyv1.StatefulSet(...)`) used to construct SSA patches |
| **Owner reference** | The link that drives garbage collection. In LWS the worker StatefulSet's owner is the leader **Pod** |
| **Foreground propagation** | Delete semantics ensuring dependents are gone before the owner. Used when deleting a leader for group recreation |
| **envtest** | A test harness running a real apiserver and etcd without a kubelet or webhooks |
| **Ginkgo / Gomega** | The BDD test framework and matcher library used in `test/integration` and `test/e2e` |
| **kind** | Kubernetes in Docker. The e2e substrate |
| **Prow** | The Kubernetes CI system. LWS's heavy presubmits run there, configured out-of-tree |
| **KAL** | Kube API Linter (`sigs.k8s.io/kube-api-linter`). Enforces API conventions; runs only on `api/leaderworkerset/*` |
| **KEP** | Kubernetes Enhancement Proposal. `keps/<number>-<title>/{README.md,kep.yaml}` |
| **PRR** | Production Readiness Review. The `feature-gates`, `disable-supported`, and `metrics` fields in `kep.yaml` |
| **`StatefulSetStartOrdinal`** | The upstream feature letting a StatefulSet's ordinals start at a non-zero value. LWS depends on it for worker ordinals |
| **`MaxUnavailableStatefulSet`** | The upstream feature enabling `maxUnavailable > 1` on a StatefulSet. Beta and default-on since Kubernetes 1.35 |
| **`HPAScaleToZero`** | The upstream gate letting HPA scale to and from zero. Its absence is why DS scalers seed at 1 |
| **Downward API** | The mechanism for exposing pod labels and annotations to a container as files or env vars |

## Inference and Parallelism

| Term | Meaning |
| :--- | :--- |
| **TP** | Tensor parallelism. Splits every weight matrix across ranks; two `all-reduce` operations per transformer layer. Extremely bandwidth-sensitive |
| **PP** | Pipeline parallelism. Splits layers into sequential stages with point-to-point activation hand-off. Tolerant of cross-host links |
| **EP** | Expert parallelism. Splits MoE experts across ranks; `all-to-all` token dispatch per MoE layer. Bandwidth-hungry and bursty |
| **DP** | Data parallelism. Independent replicas with no inter-replica communication. What `spec.replicas` counts |
| **Prefill** | The phase that processes the whole prompt and builds the KV cache. Compute-bound |
| **Decode** | The phase that generates one token at a time. Memory-bandwidth-bound |
| **P/D disaggregation** | Running prefill and decode in separate pools with KV-cache transfer between them. The reason DisaggregatedSet exists |
| **KV cache** | Cached attention keys and values, avoiding recomputation across decoding steps |
| **TTFT** | Time to first token. The prefill-phase SLO |
| **ITL / TPOT** | Inter-token latency / time per output token. The decode-phase SLO |
| **Continuous batching** | Adding and removing sequences from a running batch each step. Interleaves prefill and decode, causing head-of-line blocking |
| **Rendezvous** | The startup handshake where ranks discover each other and form a collective |
| **World size** | The total number of participating ranks. `LWS_GROUP_SIZE` maps to it |
| **NVLink / NVSwitch** | NVIDIA's intra-host GPU interconnect, hundreds of GB/s. The boundary that makes multi-host hard |
| **RDMA / RoCE / EFA** | Remote direct memory access fabrics used for cross-host GPU communication |
| **HBM** | High-bandwidth memory — the accelerator's on-package memory |
| **MoE** | Mixture of Experts. Routes each token to a subset of expert sub-networks |

## Ecosystem

| Project | Relationship to LWS |
| :--- | :--- |
| **JobSet** | Sibling SIG-Apps API for multi-host *training*. Shares design DNA; not a competitor |
| **Kueue** | Quota, admission, Topology Aware Scheduling, DWS flex-start. LWS integrates by wearing labels |
| **Volcano** | Batch scheduler providing `PodGroup` gang scheduling. The only implemented `SchedulerProvider` |
| **TAS** | Kueue's Topology Aware Scheduling. Placement within a hierarchy level, driven by pod-template annotations |
| **DWS** | Dynamic Workload Scheduler. Google Cloud capacity provisioning, including flex-start |
| **HPA** | HorizontalPodAutoscaler. Drives `spec.replicas` through the scale subresource, with a leader-only selector |
| **KEDA** | Event-driven autoscaler. An alternative `/scale` writer, and one that supports scale-to-zero |
| **llm-d** | CNCF sandbox serving stack; the reference consumer both APIs are co-designed with |
| **Gateway API Inference Extension** | Model-aware and prefix-cache-aware routing. Consumes LWS pods as backends |
| **vLLM / SGLang / TensorRT-LLM / llama.cpp** | Inference engines with upstream LWS examples ([Module 9](09_inference_engine_integration.md)) |
| **NVIDIA Dynamo / llmaz / OME / AXLearn / Kubeflow Trainer** | Listed upstream integrations |
| **Ray** | The distributed runtime vLLM uses for multi-host. Leader is the head node |
| **cert-manager** | Optional replacement for LWS's internal certificate rotation |
| **Prometheus Operator** | Consumes the `ServiceMonitor` shipped in `config/components/prometheus` |

## Files and Identifiers

| Identifier | What it is |
| :--- | :--- |
| `leaderworkerset.x-k8s.io/v1` | The LWS API group and version |
| `disaggregatedset.x-k8s.io/v1` | The DS API group and version |
| `config.lws.x-k8s.io/v1alpha1` | The controller's `Configuration` file format — **not** a cluster resource |
| `leaderworkerset.sigs.k8s.io/*` | The label and annotation domain. **Different from the API group** |
| `b8b2488c.x-k8s.io` | The default leader-election resource name |
| `lws-system` | The default install namespace |
| `lws-ca` / `lws` | The internal CA name and organization |
| `rollingUpdateParameters()` | The function computing `(partition, replicas)` — [Module 6](06_rollout_and_revisions.md) |
| `ComputeNextStep()` | The DS planner's single entry point — [Module 8](08_disaggregatedset.md) |
| `SetExclusiveAffinities()` | Injects group and subgroup placement affinity — [Module 7](07_scheduling_placement_and_networking.md) |
| `handleRestartPolicy()` | The group-recreation decision — [Module 5](05_pod_controller_and_failure_handling.md) |
| `AddLWSVariables()` | Injects the three `LWS_*` env vars — [Module 3](03_group_lifecycle_and_identity.md) |
| `GetParentNameAndOrdinal()` | Parses `<parent>-<ordinal>` from a pod name. Two meanings depending on caller |
