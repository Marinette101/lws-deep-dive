# Appendix B: PR Opportunity Backlog

Every concrete upstream contribution opportunity surfaced while writing these notes, with the file path, the evidence, and a difficulty rating. Each entry was found by reading the code, not by reading the issue tracker — so before starting anything, **check whether an issue or PR already exists**, and open an issue first for anything beyond the trivial tier.

!!! info "Provenance"
    All findings verified against `kubernetes-sigs/lws` at `32a9c37` (2026-08-06), release v0.9.0. Line references may drift; the greps given will not.

**Difficulty legend**

| | Meaning |
| :--- | :--- |
| 🟢 | An afternoon. No design discussion needed. Evidence fits in the PR description |
| 🟡 | A day or two. May need a small test, or a reviewer conversation about approach |
| 🟠 | A week or more. Needs an issue first, and possibly a KEP |
| 🔴 | Needs a KEP and a design discussion before any code |

---

## Tier 1 — Documentation

The highest ratio of user impact to effort in the project. Every one of these is a place where following the docs produces a wrong result.

| # | Finding | Where | Evidence | ⬤ |
| :--- | :--- | :--- | :--- | :---: |
| D1 | **Documents a `SubGroupType: LeaderOnly` value that does not exist**, and describes the *opposite* semantics. The real value is `LeaderExcluded`, meaning the leader is in **no** subgroup | `site/content/en/docs/concepts/_index.md` | `grep -n "LeaderExcluded\|LeaderOnly" api/leaderworkerset/v1/leaderworkerset_types.go`. See [Module 3](03_group_lifecycle_and_identity.md), §6.2 | 🟢 |
| D2 | **Two pages contradict each other on `MaxUnavailableStatefulSet`**: `rollout-strategy` says beta and default-on since 1.35; `installation` says "still in alpha since v1.24" | `concepts/rollout-strategy/_index.md` vs `installation/_index.md` | Both ship in v0.9.0. See [Module 6](06_rollout_and_revisions.md), §9 | 🟢 |
| D3 | **Four keys missing from the reference table**: `subgroup-policy-type`, `experimental-recreate-group-after-start`, `TPU_PROCESS_ADDRESSES`, `TPU_PROCESS_PORT` | `reference/labels-annotations-and-environment-variables.md` | Constants in `leaderworkerset_types.go:90,98` and `pkg/utils/accelerators/tpu.go` | 🟢 |
| D4 | **Says to disable `internalCertManager`**; the real field is `internalCertManagement.enable`. Also an ungrammatical opening sentence, and kustomize steps less complete than what `hack/e2e-test.sh` actually does | `manage/cert_manager.md` | `api/config/v1alpha1/configuration_types.go`. See [Module 10](10_operations_and_troubleshooting.md), §3 | 🟢 |
| D5 | **The Helm CRD deletion hazard is absent from the troubleshooting page** — the most destructive known failure mode, documented only in installation and the chart README | `troubleshooting/_index.md` | Upstream issue #880. See [Module 10](10_operations_and_troubleshooting.md), §2.1 | 🟢 |
| D6 | **Copy-pasted port-forward**: `svc/vllm-leader` on the TensorRT-LLM page, where no such Service exists. The `kubectl get pods` output also contradicts the prose | `examples/tensorrt-llm.md` | See [Module 9](09_inference_engine_integration.md), §2.3 | 🟢 |
| D7 | **The TAS example cannot work as written**: topology levels are declared as `cloud.google.com/gce-topology-block` but the LWS annotations reference `cloud.provider.com/topology-block`. Also pins `vllm/vllm-openai:v0.8.5`, and has a one-tab `tabpane` | `examples/tas.md` | See [Module 7](07_scheduling_placement_and_networking.md), §4 | 🟢 |
| D8 | **The deprecated `Default` restart-policy value is undocumented site-wide.** Users cannot learn that it exists or that it means `None`, not "the default" | `concepts/failure-handling/_index.md` | `leaderworkerset_types.go:346`. Also fix the "distributed influence" typo | 🟢 |
| D9 | **The experimental annotation's documented value is misleading.** The page shows `: true` and the test wrapper sets `"enable"`; the code is a **presence check** and ignores the value entirely | `concepts/failure-handling/_index.md` | `pkg/controllers/pod_controller.go:220`. See [Module 5](05_pod_controller_and_failure_handling.md), §4.1 | 🟢 |
| D10 | **Stale image**: `nginx:1.14.2`, where the rest of the site uses `nginxinc/nginx-unprivileged:1.27` | `examples/hpa.md` | | 🟢 |
| D11 | **`kubectl get endpoints` is deprecated** (use EndpointSlice); plus a "llamap.cpp" typo | `examples/llamacpp.md` | | 🟢 |
| D12 | **Two pages both have `weight: 5`**, so their ordering is undefined | `examples/hpa.md`, `examples/tas.md` | | 🟢 |
| D13 | **A directory name contains a space**: `site/content/en/docs/contribution guidelines/` | | | 🟢 |
| D14 | **Contradictory community contacts**: `CONTRIBUTING.md` says `#sig-apps` / `kubernetes-sig-apps`; the site's contribution guidelines say Slack `C071WA7R9LY` / `wg-serving` | | See [Module 11](11_contributor_workflow.md), §7.3 | 🟢 |
| D15 | **The troubleshooting page contradicts itself**: cause says "< 1.27", fix says "≥ 1.26" | `troubleshooting/_index.md` | See [Module 10](10_operations_and_troubleshooting.md), §5.1 | 🟢 |
| D16 | **Four shipped features have no conceptual documentation**: `subdomainPolicy`, user-settable `partition`, resizable `size`, and gang scheduling | `site/content/en/docs/concepts/` | Modules [3](03_group_lifecycle_and_identity.md), [6](06_rollout_and_revisions.md), [7](07_scheduling_placement_and_networking.md) | 🟡 |
| D17 | **The overview's feature list omits gang scheduling**, which `README.md` does list. The two lists have drifted | `overview/_index.md` | | 🟢 |
| D18 | **A vestigial trailing section** ("Install with Helm chart — Please refer to the release page") duplicating the full Helm section above it | `installation/_index.md` | | 🟢 |
| D19 | **The pre-v0.9.0 DS upgrade snippet applies only one of two DS CRDs**, omitting `disaggregatedsetrolescalers` which the chart ships | `installation/_index.md` | `charts/lws/crds/` has three files | 🟢 |

---

## Tier 2 — Project Metadata and Governance

| # | Finding | Where | Evidence | ⬤ |
| :--- | :--- | :--- | :--- | :---: |
| G1 | **`CONTRIBUTING.md` is entirely unmodified boilerplate**, down to a leftover HTML comment. Nothing about `make test`, `make verify`, envtest, wrappers, KEPs, or the GNU sed requirement | `CONTRIBUTING.md` | See [Module 11](11_contributor_workflow.md), §7.4. **Arguably the highest-leverage contributor PR in the repo** | 🟡 |
| G2 | **`RELEASE.md` is template text and actively wrong** — still says "The Kubernetes **Template Project**", describes a `git tag -s` flow that does not match `make artifacts` / `make prepare-release-branch` / `cloudbuild.yaml` / `hack/cherry_pick_pull.sh` | `RELEASE.md` | | 🟡 |
| G3 | **`SECURITY_CONTACTS` lists an emeritus approver** — `liurupeng` is `emeritus_approvers` in `OWNERS` | `SECURITY_CONTACTS` | | 🟢 |
| G4 | **`hack/genref/config.yaml` targets the Kubernetes 1.28 API reference** while the project builds against `k8s.io/api v0.36.3`. Every cross-link in the generated reference points at 1.28 docs | `hack/genref/config.yaml` | One-line change | 🟢 |

---

## Tier 3 — KEP Corpus Hygiene

Every one of these is a self-contained documentation fix, and each is a good excuse to read a KEP closely.

| # | Finding | Where | ⬤ |
| :--- | :--- | :--- | :---: |
| K1 | **`kep-number: 127` in a directory named `115`**, and `creation-date: yyyy-mm-dd` never filled in | `keps/115-Subgroup-support/kep.yaml` | 🟢 |
| K2 | **Title and number copy-pasted from KEP-238** | `keps/257-Subgroup-leader-only/kep.yaml` | 🟢 |
| K3 | **Describes a `ResizePolicy` API field that was never built.** Implementation History records the pivot (`2025-08-05: Implementation revised to avoid additions to the API surface`) but the body was never updated. Also the only content KEP with no `latest-milestone`, no `stage`, and no `milestone` block | `keps/552-worker-resizing/` | 🟢 |
| K4 | **Describes a `PodGroupPerReplica` feature gate, three provider interfaces, and a PodGroup naming scheme that all differ from what shipped.** `grep -rn "PodGroupPerReplica"` matches only `kep.yaml:35` | `keps/407-gang-scheduling/` | 🟡 |
| K5 | **The `publishNotReadyAddresses` premise is stale.** LWS's own headless Services hardcode `true` and a test asserts it, so a knob defaulting to `false` would be a regression risk. Also, Implementation History dates predate the `creation-date` in `kep.yaml` | `keps/820-distributed-preflight-check/` | 🟢 |
| K6 | **KEP-766 says `RolloutStrategy` is not propagated to child LWSes; `lws_manager.go` propagates it** (inertly). Either the text or the code should change | `keps/766-DisaggregatedSet/` | 🟢 |
| K7 | **`status.selector` "written once at scaler creation"** per KEP-849, but the implementation rewrites it identically on every reconcile | `keps/849-DisaggregatedSet-HPA/` | 🟢 |
| K8 | **Several KEPs at `status: implementable` still have `<test>: <link to test coverage>` placeholders**, and 135/173/511/552/622 lack `stage:` / `milestone:` blocks | `keps/` | 🟢 |
| K9 | **`spec.slices` has `Maximum=100` in code but no maximum in the KEP's API block** | `keps/846-disaggregatedset-slices/` | 🟢 |

!!! tip "K5 is the best first interaction with the maintainers"
    It costs two greps, improves a live proposal, and is a comment rather than a PR — a low-risk way to introduce yourself.

---

## Tier 4 — Code: Contained Fixes

| # | Finding | Where | Evidence | ⬤ |
| :--- | :--- | :--- | :--- | :---: |
| C1 | **Latent nil dereference in the LWS validator.** `generalValidate` reads `RollingUpdateConfiguration.MaxUnavailable` **before** the `!= nil` guard two lines below. Unreachable while the defaulter runs, but a bypassed mutating webhook panics the validator instead of rejecting | `pkg/webhooks/leaderworkerset_webhook.go` | See [Module 2](02_api_surface_anatomy.md), §5.3 | 🟢 |
| C2 | **`topologyValueFromPod` swallows a missing Node** via `IgnoreNotFound`, returning `("", nil)` and setting an **empty node selector value** that matches nothing — an unschedulable worker StatefulSet with no clear error | `pkg/controllers/pod_controller.go` | See [Module 5](05_pod_controller_and_failure_handling.md), §6 | 🟢 |
| C3 | **The topology error message interpolates the empty value, not the key**: `"node does not have topology label: %s"` tells you nothing about which label is missing | `pkg/controllers/pod_controller.go` | Same function as C2 | 🟢 |
| C4 | **Hardcoded envtest `1.28.3`** while `ENVTEST_K8S_VERSION = 1.36.0`. Harmless via `make test-integration` (KUBEBUILDER_ASSETS wins), but the documented direct-run path is broken | `test/integration/controllers/suite_test.go` | See [Module 11](11_contributor_workflow.md), §3.2 | 🟢 |
| C5 | **A stale Ginkgo entry name** references `subdomain policy LeadersSharedWorkersDedicated`, a value rejected in KEP-173 and never shipped. The real value is `UniquePerReplica` | `test/integration/controllers/leaderworkerset_test.go` | | 🟢 |
| C6 | **Dead code in the DisaggregatedSet subsystem**: `NumRequiredRoles` (zero references), `ComputeInitialReplicaState` (tests only), the free `SetInitialReplicas` (unreferenced), `ServiceManager.scheme` (stored, never used), `sortByNewestTimestamp`'s `roleNames` parameter, `ValidateUpdate`'s `oldDisagg` parameter | `pkg/utils/disaggregatedset/`, `pkg/controllers/disaggregatedset/` | See [Module 8](08_disaggregatedset.md), §10.1 | 🟢 |
| C7 | **A vestigial field index.** `lwsOwnerKey` is registered but never used in any `client.MatchingFields` query — pure informer-cache overhead. Either wire it up or remove it | `pkg/controllers/leaderworkerset_controller.go` | `grep -rn "lwsOwnerKey"` finds only the definition and the registration. See [Module 4](04_lws_reconciler_internals.md), §1.4 | 🟢 |
| C8 | **`SetupIndexes`' error is logged but not fatal**, leaving the manager running partially initialised after a startup-time failure | `cmd/main.go` | | 🟢 |
| C9 | **Over-broad RBAC**: the pod controller holds `update` and `patch` on `nodes`, exercised by no code path in the core control plane | `pkg/controllers/pod_controller.go:67` | See [Module 5](05_pod_controller_and_failure_handling.md), §5 | 🟢 |
| C10 | **A latent bug in `hashRevision`**: `deepHashObject` calls `hasher.Reset()` before writing, so if both `Data.Raw` and `Data.Object` were populated the `Raw` bytes would be silently discarded. Unreachable today | `pkg/utils/revision/revision_utils.go` | See [Module 4](04_lws_reconciler_internals.md), §5.2 | 🟢 |
| C11 | **Float arithmetic in the DS planner** — `int(float64(a) * float64(b) / float64(c))`. Integer math is exactly equivalent and immune to FP edge cases | `pkg/controllers/disaggregatedset/planner.go` | | 🟡 |
| C12 | **The pod validating webhook is a no-op** — `validate()` returns `(nil, nil)`. Either remove it or give it a purpose | `pkg/webhooks/pod_webhook.go` | | 🟡 |
| C13 | **A misleading comment** in `setNodeSelectorForWorkerPods` describes update behaviour ("the following applying logic will automatically update it") that the create-only code does not implement | `pkg/controllers/pod_controller.go` | See [Module 5](05_pod_controller_and_failure_handling.md), §2 | 🟢 |
| C14 | **`initial-replicas` write failures are swallowed** — only `log.Error`, silently degrading that role's drain baseline to `spec.Replicas` and producing a wrong drain trajectory with no surfaced error | `pkg/controllers/disaggregatedset/executor.go` | See [Module 8](08_disaggregatedset.md), §4.6 | 🟡 |
| C15 | **`extractRollingUpdateConfig` asymmetry**: `unavail > 0` sets both fields; only-`surge > 0` sets one; both-zero silently reverts to `MaxSurge=1` | `pkg/controllers/disaggregatedset/executor.go` | | 🟡 |

---

## Tier 5 — Code: Feature and Validation Gaps

| # | Finding | Where | Rationale | ⬤ |
| :--- | :--- | :--- | :--- | :---: |
| F1 | **`DisaggregatedSet.status` is never written.** The API declares `RoleStatuses` and `Conditions`, the CRD has the subresource, RBAC is granted, and the upstream docs tell users to look at it — but no code writes it | `pkg/controllers/disaggregatedset/` | **The single best contribution opportunity in the project.** All inputs are already gathered per reconcile. See [Module 8](08_disaggregatedset.md), §10 | 🟠 |
| F2 | **No printcolumns on `DisaggregatedSet`**, so `kubectl get disaggregatedset` shows only NAME and AGE. Naturally pairs with F1 | `api/disaggregatedset/v1/disaggregatedset_types.go` | | 🟢 |
| F3 | **No LWS-specific or DS-specific Prometheus metrics.** KEP-766, KEP-846, and KEP-849 all list metrics as a Beta criterion | | KEP-backed, well-motivated. See [Module 10](10_operations_and_troubleshooting.md), §4 | 🟠 |
| F4 | **Non-template spec changes on a DS role are silently dropped.** `ComputeRevision` hashes only role name + `LeaderWorkerTemplate`, so changing `networkConfig`, `startupPolicy`, `rolloutStrategy`, or role `metadata` produces the same revision, creates no new LWS, and never patches the existing one | `pkg/utils/disaggregatedset/utils.go` | **The sharpest user-visible bug in the DS subsystem.** Open an issue first — the fix has design choices | 🟠 |
| F5 | **DS Services are create-only and never repaired.** `ensureService` swallows `IsAlreadyExists`; a drifted selector is never reconciled back. The controller does not watch Services either | `pkg/controllers/disaggregatedset/service_manager.go` | Fix: add `Owns(&corev1.Service{})` plus a spec diff-and-patch | 🟡 |
| F6 | **No validation of `volumeClaimTemplates` at all** — not the name/`volumeMount` cross-check the API doc comment promises, and not immutability. Since StatefulSet's own field is immutable, an edit surfaces as a `FailedUpdate` event rather than a clean admission error | `pkg/webhooks/leaderworkerset_webhook.go` | See [Module 7](07_scheduling_placement_and_networking.md), §6.1 | 🟡 |
| F7 | **PVC forwarding is lossy and silent** — only `accessModes`, `storageClassName`, `volumeMode`, and `resources` are copied. `selector`, `dataSource`, `dataSourceRef`, and PVC labels/annotations are dropped without warning | `pkg/utils/controller/controller_utils.go` | At minimum document it; better, forward the fields or reject them | 🟡 |
| F8 | **No name-length validation.** The real limit is `51 - int(replicas/10)` characters, and exceeding it surfaces as a StatefulSet label error far from the LWS | `pkg/webhooks/leaderworkerset_webhook.go` | See [Module 10](10_operations_and_troubleshooting.md), §5.2 | 🟡 |
| F9 | **KAL does not lint `api/disaggregatedset/` or `api/config/`.** The exclusion is `path-except: "api/leaderworkerset/*"`. DS is the newest and fastest-changing API | `.golangci-kal.yml` | Expect `commentstart` violations. See [Module 2](02_api_surface_anatomy.md), §8.2 | 🟡 |
| F10 | **No DisaggregatedSet controller integration tests.** `test/integration/controllers/` has only `leaderworkerset_test.go`; DS coverage is unit tests plus e2e. KEP-766's integration test plan is unmet | `test/integration/controllers/` | | 🟠 |
| F11 | **The experimental restart annotation ignores its value** (presence check only), so setting `false` enables the feature. Now that `RecreateGroupAfterStart` is a first-class enum value, the clean fix is a loud deprecation | `pkg/controllers/pod_controller.go:220` | See [Module 5](05_pod_controller_and_failure_handling.md), §4.1 | 🟡 |
| F12 | **Undeclared Helm values.** `metrics.prometheusNamespace` and `metrics.serviceMonitor` are documented but absent from `charts/lws/values.yaml` and the chart README | `charts/lws/` | See [Module 10](10_operations_and_troubleshooting.md), §4 | 🟢 |
| F13 | **Redundant cross-slice `List` calls** — `updateScalerStatus` and `seedForRole` each issue one on top of the per-slice lists. Cache-backed, but wasteful at 100 slices | `pkg/controllers/disaggregatedset/` | | 🟡 |
| F14 | **No immutability validation on DisaggregatedSet at all.** `ValidateUpdate` is identical to `ValidateCreate` and `oldDisagg` is unused. Role names are mutable, and a rename silently recreates that role's entire fleet | `pkg/webhooks/disaggregatedset/` | Decide what should be immutable, then enforce it | 🟡 |

---

## Tier 6 — Larger Work, KEP Required

| # | Opportunity | Rationale | ⬤ |
| :--- | :--- | :--- | :---: |
| P1 | **A `revisionHistoryLimit` field.** LWS keeps exactly one revision, so `kubectl rollout undo` has nothing to work with and rollback means re-applying an old manifest by hand. Deployment and StatefulSet both have precedent | 🔴 |
| P2 | **Implement KEP-820's fail-fast restart budget.** `maxGroupRestarts`, a `Failed` condition, and a group-restart counter. Currently `status: provisional` with zero implementation — and note its second half (the `publishNotReadyAddresses` knob) should be dropped, per K5 | 🔴 |
| P3 | **A second `SchedulerProvider`.** The interface is designed for more than one; `SupportedSchedulerProviders` has a single entry. The upstream coscheduling plugin needs only a label, not a CRD | 🟠 |
| P4 | **A native "all workers ready" signal.** The TensorRT-LLM example grants the pod RBAC so the leader can poll the API server — strong evidence of a missing primitive. KEP-135's `LeaderReady` solves the inverse direction only | 🔴 |
| P5 | **Real `component-base` feature gates.** There is no feature-gate machinery at all; toggles are the `Configuration` CRD plus one `experimental-` annotation. Discuss in an issue first — this is architecture, not cleanup | 🟠 |
| P6 | **Separate leader and worker `volumeClaimTemplates`.** KEP-622 explicitly shares one field. A leader that needs different storage from its workers cannot be expressed | 🔴 |
| P7 | **Gang scheduling for DisaggregatedSet slices.** KEP-848 acknowledges that hard-only placement can wedge a slice with no whole-slice look-ahead, and defers gang scheduling | 🔴 |
| P8 | **Multi-slice External scaling.** The webhook forbids `slices > 1` with any External role, tagged `// Alpha:` and deferred to a follow-up KEP (issue #948) | 🔴 |

---

## Suggested Sequence

1. **One documentation fix** from Tier 1 — D1 is the highest user impact. Learn the PR mechanics on something uncontroversial.
2. **A KEP comment**, not a PR — K5. Two greps, real value, low risk.
3. **One contained code fix with a test** from Tier 4 — C1, C4, or C6.
4. **`CONTRIBUTING.md`** (G1) once you have been through the workflow yourself. You are the right person to write it precisely because you just learned it the hard way.
5. **Open an issue** for a Tier 5 item you care about, and see how the maintainers respond before writing code.
6. **F1 (DisaggregatedSet status)** if you want something substantial. It is well-scoped, all the inputs already exist, and the API already declares the shape.

!!! warning "Before starting anything"
    Search the issue tracker and open PRs. This appendix was compiled by reading the tree at one commit, and the project moves quickly — several of these may already be fixed, claimed, or deliberately deferred. A PR that duplicates existing work wastes reviewer time, which in a project with four approvers is the scarcest resource there is.
