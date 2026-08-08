# Appendix C: Primary Source Reading List

Annotated sources so that any claim in these notes can be re-verified independently. Entries are ordered by how useful they are to someone preparing an upstream PR, not alphabetically.

!!! info "Provenance"
    Links current as of 2026-08. LWS references point at `kubernetes-sigs/lws` commit `32a9c37` (2026-08-06), release v0.9.0.

---

## Part 1: The Source of Truth

The repository is the only authoritative source, and these notes exist to make it readable — not to replace it.

| Source | Why read it |
| :--- | :--- |
| [`kubernetes-sigs/lws`](https://github.com/kubernetes-sigs/lws) | The repository. Pin a commit when citing it; `main` moves weekly |
| `api/leaderworkerset/v1/leaderworkerset_types.go` | 459 lines that define the entire core API, plus every label and annotation constant. **Read this first.** [Module 2](02_api_surface_anatomy.md) |
| `pkg/controllers/leaderworkerset_controller.go` | The twelve-step reconcile and the rolling-update arithmetic. The densest file in the project. [Modules 4](04_lws_reconciler_internals.md) and [6](06_rollout_and_revisions.md) |
| `pkg/controllers/pod_controller.go` | Group creation, restart policy, topology pinning. [Modules 3](03_group_lifecycle_and_identity.md) and [5](05_pod_controller_and_failure_handling.md) |
| `pkg/controllers/disaggregatedset/planner.go` | A pure, client-free implementation of the N-dimensional rollout. The best-documented algorithm in the project. [Module 8](08_disaggregatedset.md) |
| `pkg/webhooks/pod_webhook.go` | Identity injection: env vars, group keys, subgroup math, affinity, TPU. [Module 3](03_group_lifecycle_and_identity.md) |
| [Documentation site](https://lws.sigs.k8s.io/docs/) | The user-facing docs. Accurate on the whole, with the specific exceptions catalogued in [Appendix B](appendix_pr_opportunities.md) |
| [Releases](https://github.com/kubernetes-sigs/lws/releases) | `manifests.yaml` and the packaged Helm chart per version |

---

## Part 2: The LWS KEP Corpus

All under `keps/` in the repository. Ordered by how much they explain.

| KEP | Status | Why it is worth reading |
| :--- | :--- | :--- |
| **766 — DisaggregatedSet** | Alpha, implemented | The best-argued KEP in the corpus. Its **Alternatives** section — specifically Alternative 4 on why `partition` was rejected — explains the existence of the entire planner. Read this even if you never touch DisaggregatedSet |
| **849 — DisaggregatedSet HPA** | Alpha, implemented | A pitfalls list where every item is a real bug someone hit, with mitigation and reasoning. The CEL-versus-defaulting trap is a broadly applicable Kubernetes API lesson |
| **820 — Fail-Fast Restart Budget** | **Provisional, NOT implemented** | Its "Why No `Failed` Before" section is unusually good historical context. Note the stale `publishNotReadyAddresses` premise ([Appendix B](appendix_pr_opportunities.md), K5) |
| **238 — ControllerRevision** | Stable | Why LWS needs revision history at all, and what a revision contains |
| **407 — Gang Scheduling** | Alpha | Read it **alongside the code** — the divergence is large ([Module 7](07_scheduling_placement_and_networking.md), §3.1) |
| **511 — Partition Update** | Implementable | The condition-semantics and truncation changes are the subtle part, and the KEP spells them out with before/after code |
| **846 — Slices** | Alpha | "A slice cycles through revisions, not the reverse" is the sentence that makes the naming scheme obvious |
| **848 — Placement Policy** | Alpha | The `NotIn`-matches-missing-keys subtlety, and an honest Risks section about hard-only placement |
| **115 / 257 — Subgroups** | Stable | Short. 257 explains why `LeaderExcluded` exists |
| **135 — Startup Policy** | Stable | Short. Why `LeaderReady` exists and what it costs |
| **173 — Headless Service Per Group** | Stable | The DNS-scale argument for `UniquePerReplica` |
| **552 — Resizable Workers** | Implementable | Read the Implementation History, not the body — the body describes an API that was never built |
| **622 — VolumeClaimTemplates** | Implementable | Short; the shared-template design decision is the whole content |
| **NNNN — Template** | — | The required sections and the `kep.yaml` schema. Read before writing one |

---

## Part 3: Upstream Kubernetes Dependencies

LWS's behaviour is partly inherited. These are the upstream features it depends on.

| Feature | Relevance |
| :--- | :--- |
| [StatefulSet ordinals (`StatefulSetStartOrdinal`)](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#start-ordinal) | Worker StatefulSets set `ordinals.start = 1`. Without it, LWS recurses infinitely — the hard 1.26 minimum ([Module 3](03_group_lifecycle_and_identity.md), §1.3) |
| [`maxUnavailable` for StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#maximum-unavailable-pods) | Beta and default-on since 1.35. Without it, `maxUnavailable > 1` silently behaves as 1 ([Module 6](06_rollout_and_revisions.md), §9) |
| [Server-Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/) | Field managers and conflict resolution. Explains why manual StatefulSet edits revert ([Module 4](04_lws_reconciler_internals.md), §4) |
| [Pod affinity and anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity) | Especially that `NotIn` **also matches objects missing the key** — the source of two subtle guards in LWS and DisaggregatedSet |
| [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | The per-pod-metric averaging formula is why the selector must match one pod per group ([Module 2](02_api_surface_anatomy.md), §4) |
| [CRD structural schemas and CEL validation](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation-rules) | Defaulting runs **before** CEL — the KEP-849 trap |
| [`kubernetes/kubernetes#64023`](https://github.com/kubernetes/kubernetes/issues/64023) | The 57-character StatefulSet name limit behind the LWS name-length failure |
| [`kubernetes/kubernetes#135017`](https://github.com/kubernetes/kubernetes/issues/135017) | Serialization drift; cited by `SetMatchesRevision` ([Module 4](04_lws_reconciler_internals.md), §5.3) |
| [API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) | What KAL enforces mechanically and what reviewers enforce by hand |

---

## Part 4: Sibling and Adjacent Projects

| Project | Why it matters |
| :--- | :--- |
| [JobSet](https://github.com/kubernetes-sigs/jobset) | The training counterpart. Shared design DNA; comparing the two clarifies which LWS decisions are serving-specific |
| [Kueue](https://kueue.sigs.k8s.io/) | Quota and admission. LWS integrates by wearing labels — no LWS code involved |
| [Kueue Topology Aware Scheduling](https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/) | The `podset-required-topology` and `podset-group-name` annotations from [Module 7](07_scheduling_placement_and_networking.md), §4 |
| [Volcano](https://volcano.sh/) | The `PodGroup` API and its `minMember` / `minResources` semantics — the two fields that make gang scheduling work correctly under `LeaderReady` |
| [llm-d](https://github.com/llm-d/llm-d) | The CNCF sandbox reference consumer both APIs are co-designed with. Several KEPs cite its requirements |
| [Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/) | Model-aware routing. Consumes LWS pods; revision-aware routing for DisaggregatedSet is its problem to solve |
| [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) | Manager wiring, watches, predicates, the client cache. Its docs explain half of `cmd/main.go` |
| [kube-api-linter](https://github.com/kubernetes-sigs/kube-api-linter) | The rules that block API PRs, and the rationale for each |

---

## Part 5: Inference Engines

Engine flags change fast. **Always check current documentation before copying a command** — several upstream LWS examples pin versions that are well behind.

| Engine | Read for |
| :--- | :--- |
| [vLLM distributed serving](https://docs.vllm.ai/en/latest/serving/distributed_serving.html) | The Ray head/worker model, and the TP-within-pod / PP-across-pods choice |
| [vLLM disaggregated prefilling](https://docs.vllm.ai/en/latest/features/disagg_prefill.html) | What DisaggregatedSet is orchestrating, from the engine's side |
| [SGLang multi-node](https://docs.sglang.ai/) | `--dist-init-addr`, `--nnodes`, `--node-rank` — the cleanest mapping onto the `LWS_*` contract |
| [TensorRT-LLM](https://nvidia.github.io/TensorRT-LLM/) | The MPI launcher model and why the leader must poll worker readiness |
| [llama.cpp RPC](https://github.com/ggml-org/llama.cpp) | The only CPU-only, kind-runnable example; the clearest illustration of LWS's two-Service model |
| [Ray](https://docs.ray.io/) | The head/worker rendezvous vLLM depends on |
| [NVIDIA NIM multi-node](https://docs.nvidia.com/nim/) | A production deployment that recommends LWS |

---

## Part 6: Talks

From the upstream adoption page. Useful for design intent that never made it into a KEP.

| Talk | Speakers |
| :--- | :--- |
| KubeCon NA 2024 — "Distributed Multi-Node Model Inference Using the LeaderWorkerSet API" | @ahg-g, @liurupeng |
| KubeCon EU 2025 — in-memory data caching for distributed training | @akshaychitneni, @bigsur0 |
| KubeCon EU 2025 — lightning talk | @kerthcet |
| KubeCon HK 2025 — "More Than Model Sharding" (in Chinese) | @panpan0000, @nicole-lihui |
| KubeCon HK 2025 | @kerthcet |
| KubeCon JP 2025 | @yankay |

The 2024 KubeCon NA talk is the original design presentation and the best available statement of why the API looks the way it does.

---

## Part 7: Companion Notes

| Source | Relationship |
| :--- | :--- |
| [Confidential Computing for LLM Serving on GKE](https://marinette101.github.io/llm-cc-in-k8s/) | The sibling notes to these. Its Module 6 uses LWS as the orchestration layer inside a confidential trust boundary — hardware TEEs, GPU attestation, and attested weight delivery, with LWS handling the multi-host group unchanged. See [Module 9](09_inference_engine_integration.md), §5.4 |

---

## Part 8: How to Verify a Claim in These Notes

Every mechanism claim here is traceable. The workflow:

```bash
git clone https://github.com/kubernetes-sigs/lws && cd lws
git checkout 32a9c37          # the commit these notes describe
```

Then, depending on what you are checking:

| Claim type | How to verify |
| :--- | :--- |
| An API field or constant | `grep -n "<name>" api/leaderworkerset/v1/leaderworkerset_types.go` |
| A validation rule | Read `pkg/webhooks/leaderworkerset_webhook.go`, then try to violate it on a live cluster |
| Rollout arithmetic | Read `rollingUpdateParameters` and `rollingUpdatePartition`, then trace the [Module 6](06_rollout_and_revisions.md), §6 worked example |
| A DisaggregatedSet claim | The planner is a pure function — read `planner.go` and its 38 KB of table tests |
| "X is not implemented" | `grep -rn "<identifier>" api/ pkg/ config/ charts/`. Absence in all four is the evidence |
| A documentation defect | Compare `site/content/en/docs/...` against the code path it describes |

If a claim in these notes does not survive that check, the notes are wrong — the code is not. Please open an issue against these notes, and consider whether the discrepancy is also a PR opportunity upstream.
