# Global Hub metrics: Option A vs Option B

Field design note for querying observability across Managed Hubs from an ACM Global Hub.
Companion: [globalhubpolicychaining.md](https://github.com/ch-stark/globalhubpolicychaining/blob/main/globalhubpolicychaining.md).
Related RFE: [ACM-9685](https://redhat.atlassian.net/browse/ACM-9685) — *query across multiple hubs for observability data*.

**Pick one primary design.** Do not treat Store federation and remote-write rollup as the same “federation layer.”

---

## Verdict

| | **A — Federate query-frontends** | **B — Remote-write rollup** |
|---|---|---|
| Where TSDB lives | Only on each Managed Hub | Managed Hub **and** Global Hub |
| What Global Thanos talks to | Managed Hub query-frontends (reachable from Global Hub) | Local global Thanos only |
| Who does the hard work | Networking + Global Hub Thanos config (depends who owns that) | Ingest path; Global Hub pays ongoing resources |
| ACM-9685 “no merge” | Yes | No (central copy) |

Option A matches ACM-9685 wording (“historical data remains at the original hub… we will not merge of data”).
Option B consolidates a second copy on the Global Hub.

**Expert lean (field / platform review):** Prefer **Option B** when you want **less to do**. For Option A you must (1) give the Global Hub access to each Managed Hub’s **query-frontend**, and (2) specially configure Global Hub Thanos to target those endpoints — effort depends on who owns that networking/config. If instead the Global Hub **ingests** the metrics (Option B), there is less cross-hub plumbing; the tradeoff is **Global Hub resource usage** (ingest, storage, query CPU/memory). A curated subset still fits B via `writeRelabelConfigs` on the second hop.

```
A: Leaf Agent → Managed Hub Thanos
                      │
                      │  query-frontend (access from Global Hub + Thanos config)
                      ▼
            Global Hub Thanos Query ──► Perses

B: Leaf Agent → Managed Hub Thanos
                    └── remote_write (subset or full) ──► Global observatorium → Perses
```

Perses UI placement (Global Hub vs regional) is orthogonal — see Option C in the chaining doc. Collection is still A or B underneath.

**TSDB** = Time Series Database — the Prometheus/Thanos object-store blocks that hold metric samples. “No merge of TSDB data” means do not copy those series into a second central store.

---

## Effort: who does what?

**Option A is more operational work** (access + Thanos targeting). **Option B is less work**, at the expense of Global Hub resources.

| | Setup / ownership | Ongoing cost |
|---|---|---|
| **A — Federate query-frontends** | Must provide Global Hub **access** to each Managed Hub query-frontend, then specially configure Global Hub Thanos to target them. Effort depends on who owns networking vs Thanos config. | Query fan-out across hubs (latency, partial failure). No second copy of TSDB. |
| **B — Rollup / ingest** | Point a second remote-write (or hub rollup) at Global Hub. Less cross-hub query plumbing. | **Global Hub resources**: ingest bandwidth, object store, Query CPU/memory — higher if you ingest “everything,” lower if you roll up a curated subset. |

MCOA already covers **leaf → its Managed Hub**. Crossing Managed Hubs (A or B) is extra architecture on top.

---

## What MCOA already gives you

Companion: [Mastering the Prometheus Agent in RHACM](https://github.com/ch-stark/mcoablog/blob/main/prometheusagent.md).

On each leaf / Managed Hub pair:

| Already (supported surfaces) | Not MCOA |
|---|---|
| Enforced `remoteWrite` to **that** hub’s observatorium + TLS | Global Hub access to Managed Hub query-frontends (Option A) |
| `writeRelabelConfigs` on the hub remote-write | Product “no-merge multi-hub query” (ACM-9685) |
| Custom `ScrapeConfig` (UWL / COO / extra targets) | Configuring Global Hub Thanos to target those frontends |
| `cluster` label on series | Exposing query-frontends across regions (LB / mesh / mTLS) |
| Carefully: **additive** `remoteWrite` (helps Option B) | Automatic rollup into a Global Hub bucket |

Default path today (full regional observability, **not** Global Hub cross-hub dashboards):

```
Leaf PrometheusAgent ──remote_write──► Managed Hub MCO / Thanos → Perses (on that hub)
```

Practical takeaway:

- Need fleet metrics **per Managed Hub only** → MCOA as-is; no A/B.
- Want a **global view with less cross-hub plumbing** → **Option B** (Global Hub ingests; you pay Global Hub resources). Prefer a curated subset via `writeRelabelConfigs` to limit that bill. Default recommendation from expert review.
- Need **ACM-9685 no-merge** as a hard product constraint → Option A; plan who provides Global Hub **access** to Managed Hub query-frontends and who configures Global Hub Thanos to target them.

---

## Option A — Federate Managed Hub query-frontends (no merge of TSDB data)

Metrics stay in each Managed Hub’s object store. Global Hub Thanos is configured to target each Managed Hub’s query-frontend (reachable from the Global Hub — not in-cluster DNS to another hub).

```
Leaf PrometheusAgent ──remote_write──► Managed Hub MCO / Thanos
                                              │
                                              │  query-frontend (mTLS / mesh / LB)
                                              ▼
                                    Global Hub Thanos Query ──► Perses (on Global Hub)
```

Two concrete jobs for Option A:

1. **Access** — Global Hub must reach each Managed Hub query-frontend.
2. **Thanos config** — Global Hub Thanos must be specially configured to target those endpoints.

Who owns (1) vs (2) drives the real effort.

### Pros

- Aligns with ACM-9685: no second copy of series.
- Regional retention / sovereignty stay where data was written.
- Hub migration stays natural: old hub keeps history; new hub gets new writes; Global Thanos just gains another target.
- Global Hub does **not** pay full ingest/storage for every regional series.

### Cons

- More to do: access + Global Hub Thanos targeting (depends who owns each part).
- Query path fans out: latency and partial failure (one hub down → gaps) show up in the UI.
- Ops surface: endpoint list, certs, firewalls live at Global Hub.

### Snippet — Global Hub Thanos targets Managed Hub query-frontends

```yaml
# Global Hub Thanos Query / store targets (illustrative)
args:
  - query
  - --endpoint=thanos-query-frontend.hub-a.example.com:9090
  - --endpoint=thanos-query-frontend.hub-b.example.com:9090
  - --query.replica-label=replica
```

(Exact flag names — `--store`, `--endpoint`, query-frontend URL — depend on your Thanos / observatorium build; the requirement is the same: Global Hub Thanos must be pointed at reachable Managed Hub query-frontends.)
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

- Simplest Global Perses setup: one local datasource; no Managed Hub query-frontend access list.
- **Less to do** than Option A: no cross-hub access + Thanos targeting project.
- Query availability for already-ingested data depends only on Global Hub.
- Matches common Autoshift / field packaging.
- Cardinality can be cut on the second hop with `writeRelabelConfigs` — right tool when Global Hub only needs a **subset** (limits Global Hub resource bill).

### Cons

- Explicitly merges / duplicates data — conflicts with ACM-9685 wording.
- **Global Hub resource usage** rises with how much you ingest (network, object store, Query).
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

Default when you want **less operational work**: **Option B** (Global Hub ingests; pay with Global Hub resources — prefer a curated subset).

| Prefer **A** when… | Prefer **B** when… |
|---|---|
| Product / RFE language is “no merge” (hard constraint) | You want less to do than access + Thanos targeting |
| Data residency / no central copy is a hard requirement | You accept Global Hub resource cost for ingest |
| Someone will own Global Hub → Managed Hub query-frontend access **and** Thanos config | Cross-region query-frontend reachability is painful |
| You must avoid centralizing series | Global Hub only needs a **subset** (`writeRelabelConfigs`) |

Keep architecture docs to these snippets (Global Thanos → Managed Hub query-frontends, or second `remoteWrite` + relabel). Distribute them later via policy chaining if needed; do not explain the metrics design through full ACM Policy wrappers.
