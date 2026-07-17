# Global Hub metrics: Option A vs Option B

Field design note for querying observability across Managed Hubs from an ACM Global Hub.
Companion: [globalhubpolicychaining.md](https://github.com/ch-stark/globalhubpolicychaining/blob/main/globalhubpolicychaining.md).
Related RFE: [ACM-9685](https://redhat.atlassian.net/browse/ACM-9685) — *query across multiple hubs for observability data*.

**Pick one primary design.** Do not treat Store federation and remote-write rollup as the same “federation layer.”

---

## Verdict

| | **A — Federate stores** | **B — Remote-write rollup** |
|---|---|---|
| Where TSDB lives | Only on each Managed Hub | Managed Hub **and** Global Hub |
| What Global Query talks to | Regional Store APIs | Local global Thanos only |
| ACM-9685 “no merge” | Yes | No (central copy) |

Option A matches ACM-9685 wording (“historical data remains at the original hub… we will not merge of data”).
Option B deliberately consolidates a second copy — but for Global Hub query load it is usually the better engineering trade.

**Expert lean (field / platform review):** Prefer **Option B** for resource efficiency. Global Query that fans out to every Managed Hub Store (A) burns CPU, memory, and cross-region bandwidth on every dashboard refresh. A curated rollup (B) pays write + storage once for a subset, then queries stay local and cheap. If the goal is only a **subset** of Managed Hub metrics at Global Hub, Option B still fits — filter with `writeRelabelConfigs` on the second hop; you do not need Store federation for that.

```
A: Leaf Agent → Managed Hub Thanos ──Store API──► Global Query → Perses
B: Leaf Agent → Managed Hub Thanos
                    └── remote_write (subset) ──► Global observatorium → Perses
```

Perses UI placement (Global Hub vs regional) is orthogonal — see Option C in the chaining doc. Collection is still A or B underneath.

**TSDB** = Time Series Database — the Prometheus/Thanos object-store blocks that hold metric samples. “No merge of TSDB data” means do not copy those series into a second central store.

---

## Effort: which option costs more?

**Option A is more effort.** Option B reuses MCOA’s write path more; A adds a cross-hub query/networking layer that MCOA does not provide.

| | Effort / runtime cost | Why |
|---|---|---|
| **A — Federate stores** | Higher to build **and** to run | Expose each Managed Hub Store (LB/Ingress/mesh + mTLS), maintain Global Query `--store` list, live with fan-out latency / partial outages. Query-time fan-out is the ongoing resource tax. Almost none of that is MCOA. |
| **B — Rollup** | Lower for global UI; cheaper at query time | Keep one Global Query + Perses. Extra work is a second `remoteWrite` (and storage for a **subset**), which sits closer to existing MCOA Agent surfaces. Pay once on write; Global Query stays local. |

MCOA already covers **leaf → its Managed Hub**. Crossing Managed Hubs (A or B) is extra architecture on top.

---

## What MCOA already gives you

Companion: [Mastering the Prometheus Agent in RHACM](https://github.com/ch-stark/mcoablog/blob/main/prometheusagent.md).

On each leaf / Managed Hub pair:

| Already (supported surfaces) | Not MCOA |
|---|---|
| Enforced `remoteWrite` to **that** hub’s observatorium + TLS | Global Hub querying **other** hubs’ Stores (Option A) |
| `writeRelabelConfigs` on the hub remote-write | Product “no-merge multi-hub query” (ACM-9685) |
| Custom `ScrapeConfig` (UWL / COO / extra targets) | Exposing Store APIs across regions |
| `cluster` label on series | Global Perses pointing at federated Stores |
| Carefully: **additive** `remoteWrite` (helps Option B) | Automatic rollup into a Global Hub bucket |

Default path today (full regional observability, **not** Global Hub cross-hub dashboards):

```
Leaf PrometheusAgent ──remote_write──► Managed Hub MCO / Thanos → Perses (on that hub)
```

Practical takeaway:

- Need fleet metrics **per Managed Hub only** → MCOA as-is; no A/B.
- Need a **global view** (full fleet or a curated subset) with efficient resource use → **Option B** (additive remote-write / hub rollup + `writeRelabelConfigs` + global Thanos). Default recommendation from expert review.
- Need **ACM-9685 no-merge** as a hard product constraint → Option A; plan for Store exposure and Query fan-out — that effort is networking/Thanos, not MCOA feature flags.

---

## Option A — Federate stores (no merge of TSDB data)

Metrics stay in each Managed Hub’s object store. Global Hub Thanos Query fans out to regional Store APIs.

```
Leaf PrometheusAgent ──remote_write──► Managed Hub MCO / Thanos
                                              │
                                              │  Thanos Store API (mTLS / mesh / LB)
                                              ▼
                                    Global Hub Thanos Query ──► Perses (on Global Hub)
```

Requires reachable Store endpoints from the Global Hub (not in-cluster DNS to another hub).

### Pros

- Aligns with ACM-9685: no second copy of series.
- Regional retention / sovereignty stay where data was written.
- Hub migration stays natural: old hub keeps history; new hub gets new writes; Query just adds another `--store`.

### Cons

- Hardest networking: Global Hub must reach each regional Store (LB / Ingress / mesh + mTLS).
- Query path is chatty: every dashboard fans out to N stores; latency and partial failure (one hub down → gaps) show up in the UI.
- Ops surface: Store list, certs, firewalls, and Store discovery all live at Global Hub.

### Snippet — Global Query store list

```yaml
# thanos-query args (illustrative)
args:
  - query
  - --store=thanos-store.hub-a.example.com:10901
  - --store=thanos-store.hub-b.example.com:10901
  - --query.replica-label=replica
```

---

## Option B — Remote-write rollup (central copy)

Regional hubs (or leaf agents) also remote-write into a global observatorium. Global Perses queries only the global Thanos.

```
Leaf PrometheusAgent ──remote_write──► Managed Hub MCO / Thanos
                         │
                         └──(optional second remote_write / hub rollup)──► Global Hub observatorium
                                                                                    │
                                                                                    ▼
                                                                          Perses (on Global Hub)
```

What field Autoshift “global observability” packaging typically implements. Apply `writeRelabelConfigs` before the second hop to control cardinality.

### Pros

- Simplest Global Perses setup: one datasource URL, no Store fan-out.
- More efficient resource use at query time: Global Query does not fan out to N regional Stores on every refresh.
- Query performance and availability depend only on Global Hub (regional hub outage does not break global dashboards for already-ingested data).
- Matches common Autoshift / field packaging.
- Cardinality can be cut on the second hop with `writeRelabelConfigs` — right tool when Global Hub only needs a **subset** of Managed Hub metrics.

### Cons

- Explicitly merges / duplicates data — conflicts with ACM-9685 wording.
- Extra network + object-store cost at Global Hub.
- Two write paths to keep healthy (regional + global); drop/duplicate series if labels or relabeling diverge.
- Hub migration is messier: “history stays on old hub” is no longer true for anything already rolled up.

### Snippet — second hop (cardinality control)

```yaml
# Extra remote_write on Managed Hub → Global (illustrative)
remoteWrite:
  - url: https://observatorium.global.example.com/api/metrics/v1/write
    writeRelabelConfigs:
      - action: keep
        regex: "cluster|namespace|pod|__name__"
        sourceLabels: [__name__]   # tighten to allowlisted metrics in practice
```

Prefer hub-side rollup over a second leaf `remote_write` unless you must bypass the regional store.

---

## Decision guide

Default engineering recommendation: **Option B**, especially for a curated subset at Global Hub.

| Prefer **A** when… | Prefer **B** when… |
|---|---|
| Product / RFE language is “no merge” (hard constraint) | You want efficient Global Hub query / resource use |
| Data residency / no central copy is a hard requirement | You need a reliable global UI first |
| Hub move / split retention matters and no central copy is allowed | Cross-region Store reachability is painful |
| | Global Hub only needs a **subset** of Managed Hub metrics (`writeRelabelConfigs`) |

Keep architecture docs to these snippets (Query `--store` list or second `remoteWrite` + relabel). Distribute them later via policy chaining if needed; do not explain the metrics design through full ACM Policy wrappers.
