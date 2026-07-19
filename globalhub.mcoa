# Declarative Telemetry Fabric: MultiCluster Observability Addon (MCOA)

Experimental field writeup. Companion docs:

- [Policy template chaining + Perses / Global Hub](./globalhubpolicychaining.md) — how to distribute config and choose query architectures
- [Prometheus Agent customization](https://github.com/ch-stark/mcoablog/blob/main/prometheusagent.md) — `writeRelabelConfigs`, `ScrapeConfig`, enforced vs open fields

The MultiCluster Observability Addon (MCOA) provides a scalable, secure, declarative framework for collecting metrics (and related signals) across hybrid fleets managed by RHACM.

---

## Table of Contents

1. [Declarative Upstream APIs & Architecture](#1-declarative-upstream-apis--architecture)
2. [Sharded Metrics Federation & Cardinality Management](#2-sharded-metrics-federation--cardinality-management)
3. [Resiliency & Pipeline Health](#3-resiliency--pipeline-health)
4. [Multi-Platform Support & Naming Consistency](#4-multi-platform-support--naming-consistency)
5. [MultiCluster Ingestion with ACM Global Hub](#5-multicluster-ingestion-with-acm-global-hub)
6. [How This Maps to Policy Chaining](#6-how-this-maps-to-policy-chaining)

---

## 1. Declarative Upstream APIs & Architecture

MCOA orchestrates collection using standard, fine-grained Kubernetes CRDs (typically authored or templated on the hub, enforced on spokes):

| CRD | Role |
|---|---|
| **PrometheusAgent** | Core metrics collector workload on managed spoke clusters, running with the Prometheus Operator stack |
| **ScrapeConfig** | Independent scrape / federation targets (for example user-workload or COO `MonitoringStack` endpoints) |
| **PrometheusRule** | Recording and alerting rules evaluated closer to the data on managed clusters |
| **AddOnDeploymentConfig** | Placement-scoped install parameters: HTTP proxies, node selectors, customized agent namespaces (default `open-cluster-management-agent-addon`) |

Hub-side templates for `PrometheusAgent` live in `open-cluster-management-observability`. MCOA replicates agents (and secrets listed in `spec.secrets`) into the spoke addon namespace. See the [Prometheus Agent writeup](https://github.com/ch-stark/mcoablog/blob/main/prometheusagent.md) for what MCOA **enforces** (hub remote-write + base TLS) versus what you may customize.

---

## 2. Sharded Metrics Federation & Cardinality Management

**Sharded ingestion.** MCOA processes each `ScrapeConfig` independently. That sharded federation model lets remote clusters collect, optionally downsample, and forward metrics with controlled CPU and memory cost.

**Curated platform profiles.** Default platform scrape profiles are tuned to limit cardinality so platform dashboards can keep pod-level visibility without unbounded global storage growth. Prefer `writeRelabelConfigs` on the agent (and disciplined allowlists) before adding a second remote-write hop to a Global Hub.

---

## 3. Resiliency & Pipeline Health

**WAL disk buffering.** Prometheus Agents buffer samples in a local disk-backed Write-Ahead Log (WAL).

**Outage survival.** If connectivity to the central ingest endpoint is lost, agents retain samples on disk so spokes can survive network partitions for on the order of **up to two hours** without losing buffered data (exact endurance depends on disk, volume, and WAL settings).

**Active pipeline alerting.** Spokes include built-in alerts to avoid silent ingestion failures, including:

| Alert | Intent |
|---|---|
| `MetricsCollectorNotIngestingSamples` | Agent fails to collect metrics locally |
| `MetricsCollectorRemoteWriteFailures` | High rate of failed HTTP remote-writes |
| `MetricsCollectorRemoteWriteBehind` | Remote-write queue latency exceeds thresholds |

---

## 4. Multi-Platform Support & Naming Consistency

MCOA supports observability on standard, non-OpenShift (CNCF-certified) managed clusters. On non-OCP platforms the addon can provision:

- **Node Exporter** and **Kube State Metrics** for platform-level metrics
- A local **Prometheus Server** as scrape / federation coordinator

**Stable identifiers.** Prefer the managed cluster’s stable ID (for example the `id.k8s.io` claim) over loose, mutable display names so fleet queries and tenancy labels stay consistent when clusters are renamed or re-imported.

---

## 5. Multicluster Ingestion with ACM Global Hub

When fleets grow large, sending telemetry **directly** to a central Global Hub avoids turning every regional ACM hub into a full Thanos receive / store tier.

### Topology (edge → Global Hub)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CENTRAL GLOBAL HUB                            │
│                                                                         │
│   ┌─────────────────────┐       ┌─────────────────┐                     │
│   │  Observatorium API  │◄──────┤ Thanos Storage  │                     │
│   │ (Secure Ingestion)  │       │  & Compaction   │                     │
│   └──────────▲──────────┘       └─────────────────┘                     │
└──────────────┼──────────────────────────────────────────────────────────┘
               │  Direct mTLS remote_write (bypass regional receive)
               │
┌──────────────┴──────────────────────────┐   ┌───────────────────────────┐
│           REGIONAL HUB                  │   │     MANAGED SPOKE         │
│                                         │   │                           │
│  disableHubSelfManagement: true         │   │  Prometheus Agent writes  │
│  MCOA Manager orchestrates placements   ├──►│  directly to Global Hub   │
│  and secret replication templates       │   │  Observatorium mTLS URL   │
└─────────────────────────────────────────┘   └───────────────────────────┘
```

This is **Option B (direct)** in [globalhubpolicychaining.md](./globalhubpolicychaining.md): spokes remote-write to the Global Hub Observatorium; the regional hub is primarily a **control plane** for MCOA placement and secret fan-out, not an intermediate metrics receive path.

Contrast with:

- **Option A** — metrics stay on regional hubs; Global Hub **queries** via Thanos Store API (no merge of TSDB data)
- **Option B (hub-hop rollup)** — spokes write to regional Thanos; regional hubs also forward to global (extra hop and storage)

### 5.1 Direct global remote-write

- Configure managed-cluster Prometheus Agents so the (or an additive) remote-write target is the **Global Hub Observatorium API** route, not the regional hub receive gateway.
- On the regional hub MultiClusterHub, `disableHubSelfManagement: true` can disable local hub self-management / local metric storage and receiver pipelines so regional managers do not need full Thanos Receiver PVC footprints for this design.
- Trade-off: you **do** centralize time series at the Global Hub (unlike ACM-9685’s “do not merge data” framing). Use relabeling and allowlists to control cardinality before the long haul.

### 5.2 Automated mTLS credential replication

Spokes need client mTLS material to talk to the Global Hub Observatorium:

1. **Single-point secret on the regional hub** — place the Global Hub client credentials (`ca.crt`, `tls.crt`, `tls.key`, and typically the Observatorium URL) in the regional hub’s `open-cluster-management-observability` namespace (often assembled once from the Global Hub signer / managed-cluster certs, then distributed).
2. **List the secret on `PrometheusAgent.spec.secrets`** — MCOA replicates those secrets into the spoke addon namespace.
3. **Mount + reload** — secrets appear under `/etc/prometheus/secrets/<secret-name>/` in the agent pod; the operator rolls pods to pick up certs without dropping the scrape pipeline.

Policy / Autoshift packaging can assemble that coalesced secret on the Global Hub and use `{{hub copySecretData}}` so every regional hub gets the same rollup credentials (see chaining doc §6).

### 5.3 Secure centralized ingestion gateway

- **Observatorium API** on the Global Hub is the secure entry point: mTLS verification of client payloads.
- **Tenancy & labeling** — the path validates streams and relies on standard `cluster` / `clusterID` labels so Thanos Query and Perses can filter multi-tenant fleet views safely.

---

## 6. How This Maps to Policy Chaining

| Concern | Where it lives |
|---|---|
| Distribute Observatorium URL + mTLS secrets to regional hubs | Global Hub `{{hub}}` policies → ConfigMap/Secret on Managed Hubs |
| Patch `PrometheusAgent` remote-write / `spec.secrets` on hubs or leaves | ConfigurationPolicy musthave on open fields only |
| COO / UWL scrape targets | Labeled `ScrapeConfig` on spokes ([prometheusagent.md](https://github.com/ch-stark/mcoablog/blob/main/prometheusagent.md)) |
| Dashboards / datasources | Perses CRs via policy ([globalhubpolicychaining.md](./globalhubpolicychaining.md) §8) |
| Cross-hub query without central TSDB | Option A Store federation (chaining doc §3 / §7) |

**Recommended reading order:** this fabric overview → Prometheus Agent customization → policy chaining for Global Hub distribution and Perses.
