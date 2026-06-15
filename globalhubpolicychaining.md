# ACM Policy Template Chaining with Global Hub and PromQL Federation

## Table of Contents

1. [The Two Template Scopes](#1-the-two-template-scopes)
2. [Global Hub → Managed Hub → Leaf Cluster Topology](#2-global-hub--managed-hub--leaf-cluster-topology)
3. [Chaining Phases Explained](#3-chaining-phases-explained)
4. [Practical Chaining Patterns](#4-practical-chaining-patterns)
5. [PromQL Federation with Perses](#5-promql-federation-with-perses)
6. [Pushing Perses Config via Policy Chaining](#6-pushing-perses-config-via-policy-chaining)
7. [Key Constraints](#7-key-constraints)

---

## 1. The Two Template Scopes

ACM policy templating has two distinct resolution scopes, and understanding the difference is the foundation of chaining.

| Delimiter | Resolved on | When |
|---|---|---|
| `{{ ... }}` | **Managed cluster** | At enforcement time, locally per cluster |
| `{{hub ... hub}}` | **Hub cluster** | At propagation time, before the policy is sent down |

`{{hub…hub}}` resolves once on the hub and the result is **baked into** what gets sent to the managed cluster. `{{ }}` resolves later, locally on each managed cluster. You can nest both in a single policy to get a two-phase resolution.

---

## 2. Global Hub → Managed Hub → Leaf Cluster Topology

```
┌─────────────────────────────────────────────────────┐
│                    Global Hub                        │
│         Multicluster Global Hub · Thanos · Perses    │
│              {{hub … hub}} resolution                │
└──────────┬──────────────────┬──────────────────┬────┘
           │ policy +         │                  │
           │ resolved values  │                  │
           ▼                  ▼                  ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Managed Hub A│   │ Managed Hub B│   │ Managed Hub C│
│ Regional ACM │   │ Regional ACM │   │ Regional ACM │
│ {{ }} tmpls  │   │ {{ }} tmpls  │   │ {{ }} tmpls  │
└──┬──────┬───┘   └──┬──────┬───┘   └──┬──────┬───┘
   │      │          │      │          │      │
   ▼      ▼          ▼      ▼          ▼      ▼
[C1]    [C2]       [C3]    [C4]      [C5]    [C6]
Prom    Prom       Prom    Prom      Prom    Prom

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                  PromQL federation layer
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[C1][C2] ──remote_write──► Hub A Thanos ─┐
[C3][C4] ──remote_write──► Hub B Thanos ─┼──► Global Hub Thanos Query ──► Perses
[C5][C6] ──remote_write──► Hub C Thanos ─┘
```

The Global Hub manages Managed Hubs as its "managed clusters". Each Managed Hub in turn manages its own fleet of leaf clusters. Policy values flow **down** through `{{hub}}` resolution; metrics flow **up** through Thanos remote_write and federation.

---

## 3. Chaining Phases Explained

### Phase 1 — Global Hub resolves `{{hub…hub}}`

The Global Hub reads values from its own cluster (Secrets, ConfigMaps, ManagedCluster objects) and bakes them into the policy before forwarding it to each Managed Hub. By the time the policy arrives at a Managed Hub, all `{{hub…hub}}` blocks are already resolved to concrete strings.

```yaml
# Policy authored on the Global Hub
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
                  namespace: monitoring
                data:
                  thanos-query-url: >-
                    {{hub fromConfigMap "global-policies"
                       "observability-endpoints"
                       (printf "%s-thanos-url" .ManagedClusterName) hub}}
                  cluster-region: >-
                    {{hub index .ManagedClusterLabels "region" hub}}
                  cluster-tier: >-
                    {{hub index .ManagedClusterLabels "tier" hub}}
```

When this policy targets Managed Hub A (with label `region=eu-west`), the hub resolves the template and sends down a fully static ConfigMap — no template syntax visible to the Managed Hub.

### Phase 2 — Managed Hub re-templates for its leaf clusters

The Managed Hub can author its own policies that use `{{ }}` (managed-cluster-side) templates referencing the ConfigMap the Global Hub just pushed down. This creates a clean two-stage chain:

```
Global Hub value
    │
    └──{{hub fromConfigMap}}──► ConfigMap on Managed Hub
                                    │
                                    └──{{ fromConfigMap }}──► Config on Leaf Cluster
```

```yaml
# Policy authored on the Managed Hub, targeting leaf clusters
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-leaf-scrape-config
  namespace: regional-policies
spec:
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: scrape-config
        spec:
          remediationAction: enforce
          object-templates-raw: |
            - complianceType: musthave
              objectDefinition:
                apiVersion: v1
                kind: ConfigMap
                metadata:
                  name: prometheus-additional-scrape
                  namespace: monitoring
                data:
                  # Reads the value the Global Hub already pushed to the Managed Hub
                  remote-write-url: >-
                    {{ fromConfigMap "monitoring" "observability-config"
                       "thanos-query-url" }}
                  region: >-
                    {{ fromConfigMap "monitoring" "observability-config"
                       "cluster-region" }}
```

The leaf cluster never needs to know where the value originated — it just sees a resolved string.

---

## 4. Practical Chaining Patterns

### Pattern 1 — Secret fan-out from Global Hub to leaves

A pull secret or TLS certificate lives **once** on the Global Hub. A `{{hub fromSecret … hub}}` propagates it to Managed Hubs; a second policy on each Managed Hub fans it further to leaf clusters using `{{ fromSecret }}`. The secret is never stored in Git.

```yaml
# On Global Hub: push TLS secret to each Managed Hub
data:
  tls.crt: '{{hub fromSecret "global-policies" "thanos-tls" "tls.crt" hub}}'
  tls.key: '{{hub fromSecret "global-policies" "thanos-tls" "tls.key" hub}}'
```

Use `copySecretData` to copy all keys at once, instead of templating each key individually:

```yaml
# Copies every key from the source secret — much cleaner for multi-key secrets
data: '{{hub copySecretData "global-policies" "thanos-tls" hub}}'
```

### Pattern 2 — Dynamic per-cluster config from ManagedCluster labels

`.ManagedClusterLabels` gives access to all labels on the target ManagedCluster object. This means the Global Hub can inject per-hub metadata without maintaining a separate ConfigMap per hub.

```yaml
data:
  region:      '{{hub index .ManagedClusterLabels "region" hub}}'
  environment: '{{hub index .ManagedClusterLabels "environment" hub}}'
  tier:        '{{hub index .ManagedClusterLabels "tier" hub}}'
  thanos-url:  >-
    {{hub fromConfigMap "global-policies" "regional-endpoints"
       (printf "%s-thanos-url" .ManagedClusterName) hub}}
```

### Pattern 3 — Iterating over all managed clusters to build federation config

The `lookup` function combined with `range` lets you iterate over every ManagedCluster on the hub and generate a configuration block per cluster in a single policy. This is the key building block for generating Prometheus `scrape_config` or Perses datasource entries dynamically.

```yaml
object-templates-raw: |
  {{hub range (lookup "cluster.open-cluster-management.io/v1"
                       "ManagedCluster" "" "").items hub}}
  {{hub if eq (index .metadata.labels "observability") "enabled" hub}}
  - complianceType: musthave
    objectDefinition:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: scrape-target-{{hub .metadata.name hub}}
        namespace: monitoring
      data:
        cluster-name: '{{hub .metadata.name hub}}'
        api-server:   '{{hub (index .spec.managedClusterClientConfigs 0).url hub}}'
        ca-bundle:    >-
          {{hub (index .spec.managedClusterClientConfigs 0).caBundle hub}}
  {{hub end hub}}
  {{hub end hub}}
```

Only clusters labelled `observability=enabled` get a scrape target ConfigMap, and every value is sourced live from the ManagedCluster object — no static inventory to maintain.

### Pattern 4 — Conditional config based on cluster tier

```yaml
object-templates-raw: |
  - complianceType: musthave
    objectDefinition:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: retention-config
        namespace: monitoring
      data:
        retention: >-
          {{hub if eq (index .ManagedClusterLabels "tier") "production" hub}}
          30d
          {{hub else hub}}
          7d
          {{hub end hub}}
        scrape-interval: >-
          {{hub if eq (index .ManagedClusterLabels "tier") "production" hub}}
          15s
          {{hub else hub}}
          60s
          {{hub end hub}}
```

---

## 5. PromQL Federation with Perses

### The full observability stack

```
Leaf cluster Prometheus
    │
    │  remote_write  (ACM metrics collector injects `cluster` label)
    ▼
Managed Hub · Thanos Receiver
    │
    │  Thanos Store API
    ▼
Global Hub · Thanos Query Frontend
    │
    │  PromQL
    ▼
Perses  (datasource = Global Hub Thanos Query endpoint)
```

Because the ACM metrics collector on each leaf injects a `cluster` label into every time series, all metrics arriving at the Global Hub Thanos carry that label. A Perses dashboard querying the Global Hub Thanos Query endpoint can do **cross-cluster PromQL natively** — no per-cluster dashboards needed.

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

**Compare a specific metric between two regions using label filtering:**

```promql
sum by (cluster) (
  rate(http_requests_total{status=~"5.."}[5m])
)
* on(cluster) group_left(region)
  kube_node_labels{label_region="eu-west"}
```

**Top-N clusters by network egress:**

```promql
topk(5,
  sum by (cluster) (
    rate(container_network_transmit_bytes_total[5m])
  )
)
```

### Thanos Query federation between hub tiers

If you run a Thanos instance on each Managed Hub (aggregating its leaf clusters), the Global Hub Thanos Query can federate across all of them using the Thanos Store API. Configure this in the Global Hub's Thanos Query deployment:

```yaml
# Global Hub Thanos Query additional stores
args:
  - query
  - --store=thanos-store.hub-a.svc.cluster.local:10901
  - --store=thanos-store.hub-b.svc.cluster.local:10901
  - --store=thanos-store.hub-c.svc.cluster.local:10901
  - --query.replica-label=replica
  - --query.replica-label=prometheus_replica
```

Or if the Managed Hubs are external, expose their Store APIs via a Service of type `LoadBalancer` or through a Submariner/Skupper service mesh, and reference the external addresses in the Global Hub query config.

---

## 6. Pushing Perses Config via Policy Chaining

Because Perses stores dashboards and datasources as Kubernetes CRs, they are first-class citizens in the ACM policy framework. You manage them exactly like any other configuration object.

### Push a Perses Datasource pointing at the regional Thanos Query

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
                  name: thanos-regional
                  namespace: monitoring
                spec:
                  plugin:
                    kind: PrometheusDataSource
                    spec:
                      directUrl: >-
                        {{hub fromConfigMap "global-policies"
                           "observability-endpoints"
                           (printf "%s-thanos-url" .ManagedClusterName) hub}}
```

Each Managed Hub receives its own resolved Thanos URL — the `printf` builds the ConfigMap key from `.ManagedClusterName`, so a single policy handles the entire Managed Hub fleet without repetition.

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
                  namespace: monitoring
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

This single policy, placed on the Global Hub and bound via a `Placement` targeting all Managed Hubs, ensures every regional Perses instance gets an identical fleet dashboard. The `global-thanos-url` key in the ConfigMap is the one place you update when the Global Hub Thanos endpoint changes.

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

## 7. Key Constraints

### Namespace scope

Hub templates are namespace-scoped. Any object referenced in a `{{hub…hub}}` block — ConfigMap, Secret, or any other resource looked up via `lookup` — must exist in the **same namespace** as the Policy object itself. Structure your Global Hub policy namespaces accordingly (e.g. one namespace per concern: `global-observability`, `global-security`, `global-networking`).

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
            ├─ Resolved policy ──► Managed Hub A
            │                          │
            │                          └─ {{ fromConfigMap }}
            │                               └─ Config on Leaf Cluster 1, 2
            │
            ├─ Resolved policy ──► Managed Hub B
            │                          └─ ...
            │
            └─ Resolved policy ──► Managed Hub C
                                       └─ ...

Perses (on each Managed Hub)
  └─ PersesDataSource  ◄── policy from Global Hub (Thanos URL injected via {{hub}})
  └─ PersesDashboard   ◄── policy from Global Hub (shared fleet dashboards)
       │
       └─ PromQL ──► Managed Hub Thanos Query
                          │
                          └─ Thanos Store API ──► Global Hub Thanos Query
                                                       │
                                                       └─ fan-out across all leaf
                                                          cluster Prometheus instances
                                                          (each carrying `cluster` label)
```
