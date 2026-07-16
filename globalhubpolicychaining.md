# ACM Policy Template Chaining with Global Hub and PromQL Federation

This is an experimental writeup based on ACM Observability (MCOA / Prometheus Agent), Global Hub policy templating, and Perses. It is **not** a supported product architecture for ACM-9685; treat it as a field design note.

Companion reading: [Mastering the Prometheus Agent in RHACM](https://github.com/ch-stark/mcoablog/blob/main/prometheusagent.md) — enforced vs open MCOA fields, `writeRelabelConfigs`, and `ScrapeConfig` for user-workload metrics.

## Table of Contents

1. [The Two Template Scopes](#1-the-two-template-scopes)
2. [Global Hub → Managed Hub → Leaf Cluster Topology](#2-global-hub--managed-hub--leaf-cluster-topology)
3. [Choose One Metrics Architecture](#3-choose-one-metrics-architecture)
4. [Chaining Phases Explained](#4-chaining-phases-explained)
5. [Practical Chaining Patterns](#5-practical-chaining-patterns)
6. [MCOA + Policy Chaining](#6-mcoa--policy-chaining)
7. [PromQL at the Query Layer](#7-promql-at-the-query-layer)
8. [Pushing Perses Config via Policy Chaining](#8-pushing-perses-config-via-policy-chaining)
9. [Key Constraints](#9-key-constraints)

---

## 1. The Two Template Scopes

ACM policy templating has two distinct resolution scopes, and understanding the difference is the foundation of chaining.

| Delimiter | Resolved on | When |
|---|---|---|
| `{{ ... }}` | **Managed cluster** | At enforcement time, locally per cluster |
| `{{hub ... hub}}` | **Hub cluster** | At propagation time, before the policy is sent down |

`{{hub…hub}}` resolves once on the hub and the result is **baked into** what gets sent to the managed cluster. `{{ }}` resolves later, locally on each managed cluster. You can nest both in a single policy to get a two-phase resolution.

### Hub template variable contexts

There are two different “current object” contexts. Do not mix them.

| Context | Typical variables | When |
|---|---|---|
| **Placement-targeted** policy (one managed cluster at a time) | `.ManagedClusterName`, `.ManagedClusterLabels` | Policy is bound via Placement; hub templates resolve per target cluster |
| **`range` over `lookup` ManagedCluster list** | `.metadata.name`, `.metadata.labels`, `.spec…` | You iterated `ManagedCluster` objects; use the ranged object’s fields |

---

## 2. Global Hub → Managed Hub → Leaf Cluster Topology

```
┌─────────────────────────────────────────────────────┐
│                    Global Hub                        │
│     Multicluster Global Hub · (optional) Thanos     │
│              {{hub … hub}} resolution                │
└──────────┬──────────────────┬──────────────────┬────┘
           │ policy +         │                  │
           │ resolved values  │                  │
           ▼                  ▼                  ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Managed Hub A│   │ Managed Hub B│   │ Managed Hub C│
│ Regional ACM │   │ Regional ACM │   │ Regional ACM │
│ MCO + MCOA   │   │ MCO + MCOA   │   │ MCO + MCOA   │
│ {{ }} tmpls  │   │ {{ }} tmpls  │   │ {{ }} tmpls  │
└──┬──────┬───┘   └──┬──────┬───┘   └──┬──────┬───┘
   │      │          │      │          │      │
   ▼      ▼          ▼      ▼          ▼      ▼
[C1]    [C2]       [C3]    [C4]      [C5]    [C6]
MCOA    MCOA       MCOA    MCOA      MCOA    MCOA
PrometheusAgent    PrometheusAgent   PrometheusAgent
```

The Global Hub manages Managed Hubs as its "managed clusters". Each Managed Hub manages its own fleet of leaf clusters.

- **Policy values flow down** through `{{hub}}` (then optional `{{ }}` on the Managed Hub for leaves).
- **Metrics flow up** through MCOA Prometheus Agents (`remoteWrite` to the hub that owns the leaf), then optionally further via **rollup** or **Store API federation** (see §3).

On each leaf, MCOA’s Prometheus Agent scrapes platform (and optional user-workload) targets and remote-writes to that leaf’s **Managed Hub** observatorium. MCOA enforces the base hub remote-write; you may add relabeling, scrape sources, and (carefully) extra remote-write entries. See the [Prometheus Agent writeup](https://github.com/ch-stark/mcoablog/blob/main/prometheusagent.md).

---

## 3. Choose One Metrics Architecture

Do not treat these as the same “federation layer.” Pick one primary design and keep Perses / policy examples aligned with it.

### Option A — Federate stores (no merge of TSDB data)

Metrics stay in each Managed Hub’s object store. Global Hub Thanos Query fans out to regional Store APIs.

```
Leaf PrometheusAgent ──remote_write──► Managed Hub MCO / Thanos
                                              │
                                              │  Thanos Store API (mTLS / mesh / LB)
                                              ▼
                                    Global Hub Thanos Query ──► Perses (on Global Hub)
```

Closest to ACM-9685 wording (“historical data remains at the original hub… we will not merge of data”). Requires reachable Store endpoints from the Global Hub (not in-cluster DNS to another hub).

### Option B — Remote-write rollup (central copy)

Regional hubs (or leaf agents) also remote-write into a **global** observatorium. Global Perses queries only the global Thanos.

```
Leaf PrometheusAgent ──remote_write──► Managed Hub MCO / Thanos
                         │
                         └──(optional second remote_write / hub rollup)──► Global Hub observatorium
                                                                                    │
                                                                                    ▼
                                                                          Perses (on Global Hub)
```

This is what field Autoshift “global observability” packaging typically implements. It **does** consolidate series at the global tier (extra network + storage cost). Apply `writeRelabelConfigs` before the second hop to control cardinality.

### Option C — Regional Perses UI, central query

Perses runs on each Managed Hub; datasources point at the **Global** Thanos Query URL (injected via `{{hub}}`). Same query backend as A or B; only the UI placement differs.

This doc’s Perses examples in §8 follow **Option C** with a Global Query URL. Collection still follows A or B underneath.

---

## 4. Chaining Phases Explained

### Phase 1 — Global Hub resolves `{{hub…hub}}`

The Global Hub reads values from its own cluster (Secrets, ConfigMaps, ManagedCluster objects) and bakes them into the policy before forwarding it to each Managed Hub. By the time the policy arrives at a Managed Hub, all `{{hub…hub}}` blocks are already resolved to concrete strings.

```yaml
# Policy authored on the Global Hub (placement-targeted → one Managed Hub at a time)
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-observability-config
  namespace: global-policies
spec:
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: observability-configmap
        spec:
          remediationAction: enforce
          severity: low
          object-templates-raw: |
            - complianceType: musthave
              objectDefinition:
                apiVersion: v1
                kind: ConfigMap
                metadata:
                  name: observability-config
                  namespace: open-cluster-management-observability
                data:
                  thanos-query-url: >-
                    {{hub fromConfigMap "global-policies"
                       "observability-endpoints"
                       (printf "%s-thanos-url" .ManagedClusterName) hub}}
                  global-thanos-url: >-
                    {{hub fromConfigMap "global-policies"
                       "observability-endpoints"
                       "global-thanos-url" hub}}
                  cluster-region: >-
                    {{hub index .ManagedClusterLabels "region" hub}}
                  cluster-tier: >-
                    {{hub index .ManagedClusterLabels "tier" hub}}
```

When this policy targets Managed Hub A (with label `region=eu-west`), the hub resolves the template and sends down a fully static ConfigMap — no template syntax visible to the Managed Hub.

### Phase 2 — Managed Hub distributes MCOA-compatible objects to leaves

Do **not** push a generic ConfigMap and expect it to change MCOA remote-write. MCOA owns destination; you customize **sources** (`ScrapeConfig`) and **open** PrometheusAgent fields (`writeRelabelConfigs`, additive remote-write, secrets).

Typical chain:

```
Global Hub value
    │
    └──{{hub fromConfigMap}}──► ConfigMap on Managed Hub
                                    │
                                    └── Managed Hub policy uses {{ fromConfigMap }}
                                         to render ScrapeConfig / Agent patches on leaves
```

Example: drop noisy series before they leave the leaf (open field; MCOA preserves custom `writeRelabelConfigs` on the enforced `acm-observability` remote-write):

```yaml
# Policy authored on the Managed Hub, targeting leaf clusters
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-leaf-agent-relabel
  namespace: regional-policies
spec:
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: mcoa-write-relabel
        spec:
          remediationAction: enforce
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: monitoring.rhobs/v1alpha1
                kind: PrometheusAgent
                metadata:
                  name: mcoa-default-platform-metrics-collector-global
                  namespace: open-cluster-management-observability
                spec:
                  remoteWrite:
                    - name: acm-observability
                      writeRelabelConfigs:
                        - action: drop
                          regex: ^Watchdog$
                          sourceLabels:
                            - alertname
```

Example: point user-workload collection at a COO `MonitoringStack` (custom `ScrapeConfig`; label is required for MCOA discovery):

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-leaf-uwl-scrapeconfig
  namespace: regional-policies
spec:
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: coo-uwl-scrapeconfig
        spec:
          remediationAction: enforce
          object-templates-raw: |
            - complianceType: musthave
              objectDefinition:
                apiVersion: monitoring.rhobs/v1alpha1
                kind: ScrapeConfig
                metadata:
                  name: coo-uwl-metrics
                  namespace: open-cluster-management-observability
                  labels:
                    app.kubernetes.io/component: user-workload-metrics-collector
                spec:
                  scheme: HTTP
                  staticConfigs:
                    - targets:
                        - >-
                          {{ fromConfigMap "open-cluster-management-observability"
                             "observability-config" "coo-monitoring-stack-svc" }}
```

The leaf never needs to know whether the service name originated on the Global Hub — it sees a resolved string in the ConfigMap, then a concrete `ScrapeConfig`.

---

## 5. Practical Chaining Patterns

Four techniques are available. **Pattern 3 is the recommended approach** for any Global Hub topology — it is the only one that scales dynamically with your fleet. Patterns 1, 2, and 4 are building blocks that you compose *inside* Pattern 3 rather than use standalone.

> **Note:** Pattern 3 generates objects **on the hub that evaluates the policy** (or on each placement target, depending on how you structure Placement). It is for **config fan-out**, not for teaching MCOA how to scrape. Leaf metric collection still uses PrometheusAgent + ScrapeConfig on the managed cluster.

---

### ⭐ Pattern 3 — Iterate over all managed clusters (recommended)

The `lookup` function combined with `range` generates a complete configuration block per cluster from a single policy. When a new Managed Hub is onboarded it automatically gets its config — no manual policy updates, no static inventory to maintain.

The label filter (fifth argument to `lookup`) is critical for fleet performance — only clusters opted in to observability are iterated. Confirm the `lookup` signature against your ACM Governance docs for your version.

```yaml
object-templates-raw: |
  {{hub range (lookup "cluster.open-cluster-management.io/v1"
                       "ManagedCluster" "" ""
                       "observability=enabled").items hub}}
  - complianceType: musthave
    objectDefinition:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: hub-meta-{{hub .metadata.name hub}}
        namespace: open-cluster-management-observability
      data:
        cluster-name: '{{hub .metadata.name hub}}'
        region: '{{hub index .metadata.labels "region" hub}}'
        tier: '{{hub index .metadata.labels "tier" hub}}'
  {{hub end hub}}
```

Every value is sourced live from the `ManagedCluster` object on the hub. Inside this `range`, use **`.metadata.*`**, not `.ManagedClusterName` / `.ManagedClusterLabels`.

---

### Building block: Pattern 1 — Secret fan-out

Use this technique **inside** a Pattern 3 loop (or in a placement-targeted policy) to inject credentials. Prefer `copySecretData` for multi-key secrets.

```yaml
# Inside a range loop or standalone for a single secret propagation
data: '{{hub copySecretData "global-policies" "thanos-tls" hub}}'
```

If you only need specific keys:

```yaml
data:
  tls.crt: '{{hub fromSecret "global-policies" "thanos-tls" "tls.crt" hub}}'
  tls.key: '{{hub fromSecret "global-policies" "thanos-tls" "tls.key" hub}}'
```

---

### Building block: Pattern 2 — Per-target label injection (placement context)

Use this in a **placement-targeted** policy (one Managed Hub / cluster at a time). Here the special hub variables apply:

```yaml
data:
  region:      '{{hub index .ManagedClusterLabels "region" hub}}'
  environment: '{{hub index .ManagedClusterLabels "environment" hub}}'
  tier:        '{{hub index .ManagedClusterLabels "tier" hub}}'
  thanos-url:  >-
    {{hub fromConfigMap "global-policies" "regional-endpoints"
       (printf "%s-thanos-url" .ManagedClusterName) hub}}
```

Inside a Pattern 3 `range` over `lookup` results, use `.metadata.labels` / `.metadata.name` instead (see composite example below).

---

### Building block: Pattern 4 — Conditional config by tier

Same dual-context rule: `.ManagedClusterLabels` in placement context; `.metadata.labels` inside a `range`.

```yaml
# Placement-targeted form
data:
  retention: >-
    {{hub if eq (index .ManagedClusterLabels "tier") "production" hub}}
    30d
    {{hub else hub}}
    7d
    {{hub end hub}}
```

---

### Composite example — Pattern 3 + building blocks

A single policy on the Global Hub generates a fully populated, per-cluster observability ConfigMap for every opted-in Managed Hub — with TLS credentials, regional metadata, and tier-appropriate retention:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-observability-per-hub
  namespace: global-policies
spec:
  remediationAction: enforce
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: observability-config-per-hub
        spec:
          remediationAction: enforce
          severity: low
          object-templates-raw: |
            {{hub range (lookup "cluster.open-cluster-management.io/v1"
                                 "ManagedCluster" "" ""
                                 "observability=enabled").items hub}}
            - complianceType: musthave
              objectDefinition:
                apiVersion: v1
                kind: ConfigMap
                metadata:
                  name: observability-config
                  namespace: open-cluster-management-observability
                data:
                  cluster-name: '{{hub .metadata.name hub}}'
                  region:       '{{hub index .metadata.labels "region" hub}}'
                  environment:  '{{hub index .metadata.labels "environment" hub}}'
                  thanos-url:   >-
                    {{hub fromConfigMap "global-policies" "regional-endpoints"
                       (printf "%s-thanos-url" .metadata.name) hub}}
                  retention: >-
                    {{hub if eq (index .metadata.labels "tier") "production" hub}}
                    30d
                    {{hub else hub}}
                    7d
                    {{hub end hub}}
                  scrape-interval: >-
                    {{hub if eq (index .metadata.labels "tier") "production" hub}}
                    15s
                    {{hub else hub}}
                    60s
                    {{hub end hub}}
            - complianceType: musthave
              objectDefinition:
                apiVersion: v1
                kind: Secret
                metadata:
                  name: thanos-tls
                  namespace: open-cluster-management-observability
                data: '{{hub copySecretData "global-policies" "thanos-tls" hub}}'
            {{hub end hub}}
```

> **Placement note:** A `range` that emits many objects in one ConfigurationPolicy is evaluated on the **cluster where the policy runs**. For per–Managed-Hub ConfigMaps that must land *on each* Managed Hub, prefer placement-targeted policies (Pattern 2 variables) or ensure your Placement + object namespace model matches how Governance delivers the rendered list. Validate in a lab before relying on the range-emits-everywhere pattern.

---

## 6. MCOA + Policy Chaining

Bridge between Global Hub policies and what MCOA will actually accept. Details and examples: [prometheusagent.md](https://github.com/ch-stark/mcoablog/blob/main/prometheusagent.md).

| Concern | Who owns it | Policy guidance |
|---|---|---|
| Hub remote-write destination + base TLS | **MCOA** (enforced) | Do not remove `acm-observability` remote-write; Addon Manager reverts that |
| Extra remote-write (e.g. global rollup) | Platform / GitOps | Additive `remoteWrite` entries + secrets in `open-cluster-management-observability`; MCOA can replicate listed secrets to leaves |
| Drop / reshape series | You | `writeRelabelConfigs` on the agent (supported / preserved) |
| What to scrape (UWL / COO / custom) | You | `ScrapeConfig` with `app.kubernetes.io/component: user-workload-metrics-collector` |
| Default UWL target | MCOA | No custom ScrapeConfig required for in-cluster UWL Prometheus |

Recommended Global Hub → Managed Hub → leaf sequence:

1. Ensure MCO / MCOA (and COO if needed) on Managed Hubs — Autoshift or equivalent packaging is fine as the installer.
2. Push shared ConfigMaps / TLS via `{{hub}}` into `open-cluster-management-observability` (or the policy namespace used for hub template sources).
3. On leaves, enforce only **open** customizations: `ScrapeConfig`, `writeRelabelConfigs`, optional additive remote-write for Option B.
4. Separately configure query (Option A Store list vs Option B global bucket) and Perses datasources (§7–§8).

---

## 7. PromQL at the Query Layer

MCOA’s Prometheus Agent on each leaf injects a `cluster` label into series remote-written to the hub. Once those series are visible to a single Thanos Query (via Option A federation or Option B rollup), Perses can run cross-cluster PromQL without per-cluster dashboards.

### Cross-cluster PromQL examples

**CPU saturation per cluster:**

```promql
sum by (cluster) (
  rate(container_cpu_usage_seconds_total{
    container!="",
    namespace!="kube-system"
  }[5m])
)
/
sum by (cluster) (
  machine_cpu_cores
)
```

**Memory pressure — flag any cluster above 90%:**

```promql
(
  sum by (cluster) (container_memory_working_set_bytes)
  /
  sum by (cluster) (machine_memory_bytes)
) > 0.90
```

**Pod restart rate across the entire fleet:**

```promql
sum by (cluster, namespace) (
  increase(kube_pod_container_status_restarts_total[1h])
) > 5
```

**Top-N clusters by network egress:**

```promql
topk(5,
  sum by (cluster) (
    rate(container_network_transmit_bytes_total[5m])
  )
)
```

### Option A — Thanos Query Store peers (external endpoints)

Managed Hub Store APIs are **not** reachable via Global Hub in-cluster DNS (`*.svc.cluster.local`) unless you have already meshed those services. Lead with external, authenticated endpoints:

```yaml
# Illustrative Global Hub Thanos Query args — use real hostnames you expose
args:
  - query
  - --store=thanos-store.hub-a.example.com:10901
  - --store=thanos-store.hub-b.example.com:10901
  - --store=thanos-store.hub-c.example.com:10901
  - --query.replica-label=replica
  - --query.replica-label=prometheus_replica
```

Expose each Managed Hub Store API with mTLS (LoadBalancer, Ingress, Submariner, or Skupper), then point Global Query at those addresses. Keeping Store lists in a ConfigMap and templating Query config via policy is optional follow-on work.

### Option B — Query only the global observatorium

If you roll up into the Global Hub, Perses’s datasource is simply the global Thanos Query / observatorium query frontend URL. No Store fan-out required (at the cost of duplicated storage and merge semantics).

---

## 8. Pushing Perses Config via Policy Chaining

Because Perses stores dashboards and datasources as Kubernetes CRs, they are first-class citizens in the ACM policy framework.

> **Schema check:** Confirm the installed Perses CRD group/version on the target cluster (`perses.dev/v1alpha1` vs `v1alpha2`) and the exact `PersesDashboard` / `PersesDataSource` shape for your operator build. Examples below are illustrative.

Architecture assumed here: **Option C** — Perses on each Managed Hub, datasource URL = Global Thanos Query (`global-thanos-url`).

### Push a Perses Datasource pointing at Global Thanos Query

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-perses-datasource
  namespace: global-policies
spec:
  remediationAction: enforce
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: perses-thanos-datasource
        spec:
          remediationAction: enforce
          severity: low
          object-templates-raw: |
            - complianceType: musthave
              objectDefinition:
                apiVersion: perses.dev/v1alpha1
                kind: PersesDataSource
                metadata:
                  name: thanos-global
                  namespace: open-cluster-management-observability
                spec:
                  plugin:
                    kind: PrometheusDataSource
                    spec:
                      directUrl: >-
                        {{hub fromConfigMap "global-policies"
                           "observability-endpoints"
                           "global-thanos-url" hub}}
```

For a **regional-only** datasource (Option A UI on the Managed Hub querying local Thanos), swap the ConfigMap key to a per-hub URL via `(printf "%s-thanos-url" .ManagedClusterName)`.

### Push a fleet-wide Perses Dashboard from the Global Hub

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-perses-fleet-dashboard
  namespace: global-policies
spec:
  remediationAction: enforce
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: fleet-cpu-dashboard
        spec:
          remediationAction: enforce
          severity: low
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: perses.dev/v1alpha1
                kind: PersesDashboard
                metadata:
                  name: fleet-cpu-saturation
                  namespace: open-cluster-management-observability
                spec:
                  datasources:
                    thanos:
                      default: true
                      plugin:
                        kind: PrometheusDataSource
                        spec:
                          directUrl: >-
                            {{hub fromConfigMap "global-policies"
                               "observability-endpoints"
                               "global-thanos-url" hub}}
                  panels:
                    fleet-cpu:
                      kind: TimeSeriesChart
                      spec:
                        title: CPU saturation by cluster
                        queries:
                          - kind: PrometheusTimeSeriesQuery
                            spec:
                              datasource:
                                name: thanos
                              query: >
                                sum by (cluster) (
                                  rate(container_cpu_usage_seconds_total[5m])
                                )
                                /
                                sum by (cluster) (machine_cpu_cores)
                    fleet-memory:
                      kind: TimeSeriesChart
                      spec:
                        title: Memory pressure by cluster
                        queries:
                          - kind: PrometheusTimeSeriesQuery
                            spec:
                              datasource:
                                name: thanos
                              query: >
                                sum by (cluster) (
                                  container_memory_working_set_bytes
                                )
                                /
                                sum by (cluster) (machine_memory_bytes)
```

This single policy, placed on the Global Hub and bound via a `Placement` targeting all Managed Hubs, ensures every regional Perses instance gets an identical fleet dashboard pointed at the same global query URL. Update `global-thanos-url` in one ConfigMap when the endpoint changes.

### Placement and PlacementBinding to target all Managed Hubs

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: all-managed-hubs
  namespace: global-policies
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            role: managed-hub

---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: bind-perses-policies
  namespace: global-policies
placementRef:
  name: all-managed-hubs
  apiGroup: cluster.open-cluster-management.io
  kind: Placement
subjects:
  - name: policy-perses-datasource
    apiGroup: policy.open-cluster-management.io
    kind: Policy
  - name: policy-perses-fleet-dashboard
    apiGroup: policy.open-cluster-management.io
    kind: Policy
```

---

## 9. Key Constraints

### Namespace scope

Hub templates are namespace-scoped. Any object referenced in a `{{hub…hub}}` block — ConfigMap, Secret, or any other resource looked up via `lookup` — must exist in the **same namespace** as the Policy object itself. Structure your Global Hub policy namespaces accordingly (e.g. `global-policies` for template sources; land rendered objects in `open-cluster-management-observability` on targets).

### Template resolution order

`{{hub…hub}}` is resolved **once at propagation time** and the result is immutable in transit. If the source ConfigMap or Secret on the hub changes, the policy must be re-propagated for managed clusters to see the updated value. ACM does this automatically on the next reconciliation cycle, but there is a short lag.

### `copySecretData` and `copyConfigMapData`

Prefer these bulk-copy functions over individual key templates when propagating multi-key objects:

```yaml
# Bulk copy — cleaner for multi-key secrets:
data: '{{hub copySecretData "global-policies" "thanos-tls" hub}}'

# Instead of templating each key individually:
data:
  tls.crt: '{{hub fromSecret "global-policies" "thanos-tls" "tls.crt" hub}}'
  tls.key: '{{hub fromSecret "global-policies" "thanos-tls" "tls.key" hub}}'
```

### `lookup` label filtering

The `lookup` function accepts a label selector as a fifth argument, which is critical for performance when iterating over large ManagedCluster fleets:

```yaml
# Only return clusters with the observability label — avoids iterating everything
{{hub range (lookup "cluster.open-cluster-management.io/v1"
                     "ManagedCluster" "" ""
                     "observability=enabled").items hub}}
```

### MCOA reconciliation

Customizations that touch **enforced** fields are reverted by the Addon Manager. Stick to documented open surfaces (`writeRelabelConfigs`, labeled `ScrapeConfig`s, additive remote-write). See [prometheusagent.md](https://github.com/ch-stark/mcoablog/blob/main/prometheusagent.md).

### Perses CR availability

Perses CRDs must be installed on the target cluster before ACM can enforce `PersesDashboard` or `PersesDataSource` objects. Use a prerequisite policy (with `dependencies`) to ensure the Perses operator is present before the dashboard policy is evaluated.

```yaml
metadata:
  annotations:
    policy.open-cluster-management.io/dependencies: |
      [{"name": "policy-perses-operator", "namespace": "global-policies",
        "compliance": "Compliant"}]
```

---

## Summary: the full chain at a glance

```
Global Hub
  └─ ConfigMap / Secret (source of truth)
       │
       └─ {{hub fromConfigMap / fromSecret / lookup hub}}
            │
            ├─ Resolved policy ──► Managed Hub A (MCO + MCOA)
            │                          │
            │                          └─ {{ }} / musthave
            │                               ├─ ScrapeConfig / Agent patches on leaves
            │                               └─ Perses CR (Option C) → global-thanos-url
            ├─ Resolved policy ──► Managed Hub B
            └─ Resolved policy ──► Managed Hub C

Metrics (pick one):
  Option A: Leaf Agent → Managed Hub Thanos ──Store API──► Global Thanos Query → Perses
  Option B: Leaf Agent → Managed Hub (+ rollup remote_write) → Global observatorium → Perses

Policy down · metrics up · query layer chosen explicitly
```
