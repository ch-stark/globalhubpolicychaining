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
Option B is simpler to operate day-to-day but deliberately consolidates a second copy.

```
A: Leaf Agent → Managed Hub Thanos ──Store API──► Global Query → Perses
B: Leaf Agent → Managed Hub Thanos
                    └── remote_write ──► Global observatorium → Perses
```

Perses UI placement (Global Hub vs regional) is orthogonal — see Option C in the chaining doc. Collection is still A or B underneath.

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
- Query performance and availability depend only on Global Hub (regional hub outage does not break global dashboards for already-ingested data).
- Matches common Autoshift / field packaging.
- Cardinality can be cut on the second hop with `writeRelabelConfigs`.

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

| Prefer **A** when… | Prefer **B** when… |
|---|---|
| Product / RFE language is “no merge” | You need a reliable global UI first |
| Data residency / no central copy is a hard requirement | Cross-region Store reachability is painful |
| Hub move / split retention matters | Cardinality at global is a curated subset |

Keep architecture docs to these snippets (Query `--store` list or second `remoteWrite` + relabel). Distribute them later via policy chaining if needed; do not explain the metrics design through full ACM Policy wrappers.
