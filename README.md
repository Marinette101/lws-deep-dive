# lws-deep-dive

Personal deep-dive notes on **[LeaderWorkerSet (LWS)](https://github.com/kubernetes-sigs/lws)** — the
Kubernetes SIG-Apps API for deploying a group of pods as a unit of replication, built for multi-host
LLM inference, together with the **DisaggregatedSet (DS)** API layered on top of it for
prefill/decode disaggregated serving.

These notes are written from the source, not from the release notes. They cover the API surface, the
reconciler internals, the rollout algorithm, the failure-handling machinery, the DisaggregatedSet
planner/executor, and the contributor workflow — with the explicit goal of being able to open a
useful upstream PR.

Published site: <https://marinette101.github.io/lws-deep-dive/>

Available in English and 简体中文 (use the language switcher in the site header).

## Provenance

| | |
| :--- | :--- |
| Upstream repository | `kubernetes-sigs/lws` |
| Latest release described | **v0.9.0** |
| Verified against commit | **`32a9c37`** ("Run make manifest (#967)") |
| Commit date | **2026-08-06** |
| Notes written | **2026-08** |

Every mechanism claim in these notes is traceable to a file in the upstream tree at that commit.
Where the upstream documentation and the upstream code disagree, the notes follow the code and record
the discrepancy in [Appendix B](docs/en/appendix_pr_opportunities.md).

## Build locally

```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
mkdocs serve          # http://127.0.0.1:8000
mkdocs build --strict # fail on broken links / nav entries
```

Content lives in `docs/en/` and `docs/zh/` (same filenames per locale). Deployment is
`mkdocs gh-deploy` via `.github/workflows/publish-docs.yml` on every push to `main`.
