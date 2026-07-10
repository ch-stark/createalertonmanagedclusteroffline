# Catching Offline Clusters Before They Become Incidents: Fleet Alerting in Red Hat ACM

If you're running Red Hat Advanced Cluster Management (ACM) across a fleet of tens or hundreds of managed clusters, "is everything still connected?" shouldn't be a question you answer by opening the console and scrolling. It should be a question your observability stack answers for you, automatically, the moment something goes wrong. This post walks through how to build that alerting, where teams typically get tripped up, and how to manage it at scale.

## Why fleet-level alerting is different

A single-cluster alerting rule (disk pressure, pod crash loops, node not-ready) is straightforward: the metrics come from that cluster's own Prometheus. Fleet-level alerting is different because the thing you actually care about — *is the managed cluster still there and healthy from the hub's point of view* — lives in the hub's own telemetry, not in the spoke cluster's metrics.

ACM's observability stack centralizes this by running Thanos Ruler on the hub, evaluating rules against metrics that describe the fleet itself: cluster availability, addon status, and so on. That means the alerting logic for "cluster went offline" isn't something you bolt onto each managed cluster — it's a small set of rules you maintain once, centrally, and it covers every cluster the hub knows about.

## The building blocks

There are three pieces that need to line up:

1. **The rule itself**, delivered via a `ConfigMap` (`thanos-ruler-custom-rules`) in the `open-cluster-management-observability` namespace, consumed by Thanos Ruler.
2. **The metric the rule depends on**, which has to actually be flowing into observability storage.
3. **The routing**, so a firing alert reaches a human (or a ticketing system) instead of sitting quietly in an Alertmanager UI nobody's watching.

### The rule

A minimal "cluster went offline" rule looks like this:

```yaml
groups:
  - name: fleet-lifecycle-alerts
    rules:
      - alert: ManagedClusterOffline
        expr: acm_managed_cluster_info{available!="True"} == 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Managed Cluster {{ $labels.managed_cluster_id }} is Offline"
          description: "The managed cluster {{ $labels.managed_cluster_id }} (cloud: {{ $labels.cloud }}, vendor: {{ $labels.vendor }}) is reporting availability as '{{ $labels.available }}' for more than 5 minutes."
```

The `for: 5m` matters more than it looks. Managed clusters can blip — a network hiccup, a control plane restart — and you don't want a page for every transient flap. Five minutes is a reasonable starting point; tune it against your own environment's noise floor.

It's also worth pairing this with an addon-health rule, since a cluster can be technically "available" while its observability pipeline itself has quietly stopped reporting:

```yaml
      - alert: ObservabilityAddonDegraded
        expr: acm_managed_cluster_labels{feature_open_cluster_management_io_addon_multicluster_observability_addon!="available"} == 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Telemetry Pipeline Degraded on Cluster {{ $labels.cluster }}"
          description: "The observability addon on cluster {{ $labels.cluster }} is reporting as '{{ $labels.feature_open_cluster_management_io_addon_multicluster_observability_addon }}' instead of 'available'."
```

Without this second rule, a cluster whose observability addon has died can look "fine" right up until the moment you actually need data from it.

## The gotcha almost everyone hits

Here's the part that catches teams out: writing the rule is the easy 20%. The rule is only as good as the metric it queries, and `acm_managed_cluster_info` — the metric that carries per-cluster availability — is not always flowing into the hub's metrics storage by default, depending on how the observability stack's metrics allowlist is configured.

If your alert never fires (even when you can see in the console that a cluster is genuinely offline), the first thing to check isn't the alerting rule — it's whether the underlying metric is actually being collected. That usually means:

- Confirming `acm_managed_cluster_info` is present in the observability custom metrics allowlist (or the relevant `ScrapeConfig`), rather than being filtered out upstream.
- Querying the metric directly in the observability Grafana/Thanos query endpoint to confirm data points exist for the cluster in question, before assuming the alert rule is broken.

This is a good general lesson for fleet observability: debug top-down. Confirm the metric exists and has fresh data *before* debugging the rule expression, the routing, or Alertmanager. A perfectly correct PromQL expression against a metric that isn't being scraped will never fire, and it's easy to burn an afternoon staring at rule syntax when the actual gap is upstream.

## Routing: getting the alert to a human

Once the rule is firing correctly, routing is comparatively simple — it's standard Alertmanager configuration, just scoped to the hub's Alertmanager instance:

```yaml
route:
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default-receiver'
  routes:
    - matchers:
        - destination="netcool"
      receiver: 'ticketing-gateway'
receivers:
  - name: 'default-receiver'
  - name: 'ticketing-gateway'
    webhook_configs:
      - url: 'https://ticketing-gateway.internal.example.com/api/v1/alerts'
        send_resolved: true
```

Tagging the alert rule itself with a `destination` label (as shown in the earlier rule example) lets you route different alert types to different downstream systems — critical fleet alerts to a ticketing system, lower-severity noise to a Slack channel, and so on — without maintaining separate rule files per destination.

## Managing this at scale with GitOps

Hand-editing the `thanos-ruler-custom-rules` ConfigMap on a live hub cluster works for a proof of concept, but it doesn't survive contact with multiple environments, change review, or an accidental `oc edit`. The more durable pattern is to manage the ConfigMap, the Alertmanager secret, and the `MultiClusterObservability` CR patch as GitOps-managed manifests — wrapped in an ACM `Policy` (or generated via the `PolicyGenerator` Kustomize plugin) and rolled out with `remediationAction: enforce` so drift gets corrected automatically instead of paged on.

That gets you the best of both worlds: alerting logic that's version-controlled and reviewable like any other code, and a hub that self-heals if someone changes the rule set out-of-band.

## The takeaway

Fleet-level "is it still there" alerting in ACM comes down to three things lining up: a rule that queries the right metric with a sensible debounce window, confirmation that the metric is actually being scraped (the step most people skip), and routing that gets the alert somewhere a human will see it. Get those three right, managed through Git rather than ad hoc edits, and "which clusters are offline right now" stops being a question you have to go looking for the answer to.
