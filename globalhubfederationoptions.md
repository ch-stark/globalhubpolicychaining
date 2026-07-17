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
