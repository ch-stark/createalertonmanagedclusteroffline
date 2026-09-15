# RHACM Observability: cluster name in alert annotations

This is the customer workaround write-up in
[`ch-stark/createalertonmanagedclusteroffline`](https://github.com/ch-stark/createalertonmanagedclusteroffline/blob/main/managed_cluster_name_inalerts.md)
(`managed_cluster_name_inalerts.md`). It was originally written against **RHACM 2.17**.
Use the **ACM 2.13** section below if that is the hub version under support.

Native `managed_cluster_name` on `acm_managed_cluster_info` / `policyreport_info` is
[ACM-41089](https://issues.redhat.com/browse/ACM-41089) (currently targeted at ACM 5.1).
Until that ships on the customer's version, put the name on the **alert**, not the metric.

## Problem Statement
 
Several of the hub-side metrics used for alerting only carry an opaque `managed_cluster_id` (a UUID) — not the human-readable cluster name. If an alert template uses `{{ $labels.managed_cluster_id }}` directly, the summary/description shows something like `4f2a1c9e-7b3d-4e21-9f6a-...` instead of `east-region-prod`, which isn't useful when triaging an alert at 2am.
 
**Goal:** every alert annotation below should resolve to the real managed cluster *name*, not its ID.
 
| Metric | Has the name natively? | What it has instead |
|---|---|---|
| Forwarded fleet metrics (e.g. `kube_node_status_allocatable`) | ✅ `cluster` label | — |
| `acm_managed_cluster_info` | ❌ | `managed_cluster_id` only |
| `policyreport_info` | ❌ | `managed_cluster_id` only |
| `acm_managed_cluster_labels` (the lookup table) | ✅ `name` label | `managed_cluster_id` (used as the join key) |
| `acm_managed_cluster_status_condition` | ✅ `managed_cluster_name` label | — |
 
Where a metric is missing the name (rows 2–3 above), the workaround joins it against `acm_managed_cluster_labels` — the metric that maps `managed_cluster_id` → `name` — using `group_left`, so the real name rides along into the alert's labels and annotations.

A join **without** collapsing `acm_managed_cluster_labels` first will fail with Thanos `422` / `many-to-many matching not allowed` (duplicate series). That metric is one timeseries per cluster **and** label-set, so a day-2 ManagedCluster label change leaves two series for the same ID until the old set expires. This is [ACM-30479](https://issues.redhat.com/browse/ACM-30479), not a customer PromQL error.

| Hub version | Built-in `ViolatedPolicyReport` join | What to give the customer |
|---|---|---|
| **2.13** (and 2.14 / 2.15 until the clone ships) | **Unsafe** — no `max by` collapse | Custom rules only (this 2.13 section) |
| 2.16.3, 2.17.1, 5.0.0+ | Join fixed in default rules ([ACM-30479](https://issues.redhat.com/browse/ACM-30479)) | Workarounds 1–3 below; default policy rule is OK |
| 5.1+ (ACM-41089) | Native name labels (planned) | No join required once the metric carries `managed_cluster_name` |

---

## ACM 2.13 workaround 

On 2.13, `acm_managed_cluster_info` and `policyreport_info` are documented as **Stable** with `managed_cluster_id` and **no** cluster name
([Observability 2.13](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/observability/observing-environments-intro)).
Do **not** tell a 2.13 customer to rely on ConfigMap `thanos-ruler-default-rules` / alert `ViolatedPolicyReport` for the name — that default expr is the broken join.

Put custom rules only in ConfigMap `thanos-ruler-custom-rules` in namespace `open-cluster-management-observability`.
Have them confirm label names in Grafana Explore on **their** hub before they copy annotations (`managed_cluster_name` vs `name`).

### 2.13-A — Unavailable cluster (preferred: no join)

`acm_managed_cluster_status_condition` exists from ACM 2.7 and already carries `managed_cluster_name`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: thanos-ruler-custom-rules
  namespace: open-cluster-management-observability
data:
  custom_rules.yaml: |
    groups:
      - name: acm-cluster-availability
        rules:
          - alert: ManagedClusterUnavailable
            expr: |
              acm_managed_cluster_status_condition{
                condition="ManagedClusterConditionAvailable",
                status!="True"
              } == 1
            for: 5m
            labels:
              severity: critical
            annotations:
              summary: "Cluster {{ $labels.managed_cluster_name }} is unavailable"
              description: "Managed cluster {{ $labels.managed_cluster_name }} availability is not True."
```

Verify first:

```
count by (managed_cluster_name, condition, status) (acm_managed_cluster_status_condition)
```

### 2.13-B — Must use `acm_managed_cluster_info`

`max by` is required. `last_over_time` covers the ~10 minute overlap while both old and new `acm_managed_cluster_labels` series exist.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: thanos-ruler-custom-rules
  namespace: open-cluster-management-observability
data:
  custom_rules.yaml: |
    groups:
      - name: acm-cluster-info
        rules:
          - alert: ManagedClusterUnavailable
            expr: |
              acm_managed_cluster_info{available!="True"}
              * on (managed_cluster_id) group_left (name)
                max by (managed_cluster_id, name) (
                  last_over_time(acm_managed_cluster_labels[10m])
                )
            for: 5m
            labels:
              severity: critical
              cluster: "{{ $labels.name }}"
            annotations:
              summary: "Cluster {{ $labels.name }} is unavailable"
              description: "Managed cluster {{ $labels.name }} (ID: {{ $labels.managed_cluster_id }}) is not available."
```

A `group_left` **without** `max by` / `last_over_time` is what produces duplicate series.

### 2.13-C — Policy violations (custom rule only)

Do not use the 2.13 built-in `ViolatedPolicyReport` for this. Add a custom rule:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: thanos-ruler-custom-rules
  namespace: open-cluster-management-observability
data:
  custom_rules.yaml: |
    groups:
      - name: policy-reports-custom
        rules:
          - alert: CustomPolicyViolation
            expr: |
              sum by (name, policy, severity) (
                policyreport_info{result="fail"}
                * on (managed_cluster_id) group_left (name)
                  max by (managed_cluster_id, name) (
                    last_over_time(acm_managed_cluster_labels[10m])
                  )
              ) > 0
            for: 1m
            labels:
              severity: "{{ $labels.severity }}"
            annotations:
              summary: "Policy violation on cluster {{ $labels.name }}"
              description: "Policy {{ $labels.policy }} (severity: {{ $labels.severity }}) on cluster {{ $labels.name }}."
```

If they used `max by` and still see **two ALERT series** for one violation, check for leftover MCOA `PrometheusRules` plus MCO evaluating the same alert ([ACM-34481](https://issues.redhat.com/browse/ACM-34481)) — that is a second evaluator, not a bad join.

Apply on the hub:

```bash
oc apply -f thanos-ruler-custom-rules.yaml -n open-cluster-management-observability
```

Confirm in Grafana Explore: `ALERTS{alertname="ManagedClusterUnavailable"}`.
If Thanos ruler logs show `422` / `many-to-many matching not allowed`, the custom expr is still joining raw `acm_managed_cluster_labels` without the collapse.

This workaround puts the name on the **alert**. It does not change 2.13 metric cardinality or ship ACM-41089.

---

## Where Observability Alert Configuration Lives (2.17 and later)
 
All of the following resources are on the hub cluster, in namespace `open-cluster-management-observability`.
 
| Purpose | Resource | What to Set |
|---|---|---|
| Your custom alert rules | ConfigMap `thanos-ruler-custom-rules` | PromQL `expr`, annotations, and labels |
| Built-in alerts (e.g. `ViolatedPolicyReport`) | ConfigMap `thanos-ruler-default-rules` | View only — do not edit for custom work |
| Alert routing (Slack, email, PagerDuty) | Secret `alertmanager-config` | Receivers, routes, notification templates |
| Metric verification | Grafana → Explore | Ad-hoc PromQL queries |
 
Custom rules are evaluated by the `observability-thanos-rule` pods. After you update `thanos-ruler-custom-rules`, the config reloads automatically via the sidecar.
 
---
 
## Workaround 1: Alerts on Forwarded Fleet Metrics
 
**Where:** ConfigMap `thanos-ruler-custom-rules` on the hub cluster.
 
**What to set:** annotations and labels inside `data.custom_rules.yaml`.
 
Apply on the hub:
 
```bash
oc apply -f thanos-ruler-custom-rules.yaml -n open-cluster-management-observability
```
 
Example ConfigMap:
 
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: thanos-ruler-custom-rules
  namespace: open-cluster-management-observability
data:
  custom_rules.yaml: |
    groups:
      - name: cluster-health
        rules:
          - alert: HighCPUUsage
            expr: |
              sum by (cluster, clusterID) (cluster:cpu_usage_cores:sum)
              /
              sum by (cluster, clusterID) (kube_node_status_allocatable{resource="cpu"}) > 0.8
            for: 5m
            labels:
              severity: warning
              cluster: "{{ $labels.cluster }}"
              clusterID: "{{ $labels.clusterID }}"
            annotations:
              summary: "High CPU on cluster {{ $labels.cluster }}"
              description: "Cluster {{ $labels.cluster }} (ID: {{ $labels.clusterID }}) CPU usage is above 80%."
```
 
`{{ $labels.cluster }}` works here because metrics forwarded from managed clusters already carry the `cluster` label (ManagedCluster name) and `clusterID` label (UUID).
 
Verify in Grafana Explore (ACM console → Grafana link → Explore):
 
```
kube_node_status_allocatable{resource="cpu"}
```
 
Confirm `cluster` and `clusterID` appear in the label set.
 
> **Why no join is needed:** unlike Workarounds 2 and 3, metrics forwarded from managed clusters already carry the `cluster` label at the source — no lookup against `acm_managed_cluster_labels` required.
 
---
 
## Workaround 2: Alerts Based on `acm_managed_cluster_info`
 
**Where:** Same ConfigMap — `thanos-ruler-custom-rules` on the hub.
 
**What to set:** A join in the `expr` field; use `{{ $labels.name }}` in annotations.
 
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: thanos-ruler-custom-rules
  namespace: open-cluster-management-observability
data:
  custom_rules.yaml: |
    groups:
      - name: acm-cluster-info
        rules:
          - alert: ManagedClusterUnavailable
            expr: |
              acm_managed_cluster_info{available!="True"}
              * on (managed_cluster_id) group_left (name)
                max by (managed_cluster_id, name) (acm_managed_cluster_labels)
            for: 5m
            labels:
              severity: critical
              cluster: "{{ $labels.name }}"
            annotations:
              summary: "Cluster {{ $labels.name }} is unavailable"
              description: "Managed cluster {{ $labels.name }} (ID: {{ $labels.managed_cluster_id }}) is not available."
```
 
**Why this join is needed:** `acm_managed_cluster_info` only exposes `managed_cluster_id`, never a name. This part of the query:
 
```
* on (managed_cluster_id) group_left (name)
  max by (managed_cluster_id, name) (acm_managed_cluster_labels)
```
 
looks up that ID in `acm_managed_cluster_labels` and copies its `name` label onto the result. Without it, `{{ $labels.name }}` renders empty and you're stuck alerting on the raw UUID.
 
> **Shortcut:** if your alert condition can instead be expressed against `acm_managed_cluster_status_condition` (e.g. availability/condition-based alerts), that metric already carries `managed_cluster_name` natively — no join required.
 
Verify in Grafana Explore:
 
```
acm_managed_cluster_info
acm_managed_cluster_labels
```
 
---
 
## Workaround 3: Policy Violation Alerts (`policyreport_info`)

**2.13:** skip this section. The built-in `ViolatedPolicyReport` join is unsafe on 2.13. Use **2.13-C** above.

**Where (2.16.3 / 2.17.1 / 5.0+):** Built-in rule in ConfigMap `thanos-ruler-default-rules` on the hub.

**What to use:** `{{ $labels.cluster }}` in alert annotations — after [ACM-30479](https://issues.redhat.com/browse/ACM-30479), the default `ViolatedPolicyReport` rule joins `policyreport_info` to `acm_managed_cluster_labels` **and** collapses duplicate label-sets with `max by`.
 
View the built-in rule (do not edit unless directed by support):
 
```bash
oc -n open-cluster-management-observability \
  get cm thanos-ruler-default-rules \
  -o yaml | grep -A 20 ViolatedPolicyReport
```
 
The rule's `annotations.description` already references `{{ $labels.cluster }}`.
 
If you need a custom policy alert, add it to `thanos-ruler-custom-rules` (not the default ConfigMap):
 
```yaml
data:
  custom_rules.yaml: |
    groups:
      - name: policy-reports-custom
        rules:
          - alert: CustomPolicyViolation
            expr: |
              sum(
                policyreport_info
                * on (managed_cluster_id) group_left (cluster)
                  max by (managed_cluster_id, cluster) (
                    label_replace(acm_managed_cluster_labels, "cluster", "$1", "name", "(.*)")
                  )
              ) by (cluster, policy, severity) > 0
            for: 1m
            labels:
              severity: "{{ $labels.severity }}"
            annotations:
              summary: "Policy violation on cluster {{ $labels.cluster }}"
              description: "Policy {{ $labels.policy }} (severity: {{ $labels.severity }}) on cluster {{ $labels.cluster }}."
```
 
**Why this join is needed:** `policyreport_info` also only carries `managed_cluster_id`, not a name. `label_replace(acm_managed_cluster_labels, "cluster", "$1", "name", "(.*)")` copies the value of `acm_managed_cluster_labels`'s `name` label into a new label called `cluster` (matching what the rest of the query groups by), then `group_left (cluster)` attaches it the same way as Workaround 2. Net effect: `{{ $labels.cluster }}` in the annotation is the real name, not the ID.
 
---
 
## Where to Configure Notification Delivery (Slack, Email, etc.)
 
Alert rule annotations define *what* the alert says. Alertmanager defines *where* it is sent.
 
**Where:** Secret `alertmanager-config` in `open-cluster-management-observability` on the hub.
 
```bash
# Extract current config
oc -n open-cluster-management-observability \
  get secret alertmanager-config \
  --template='{{ index .data "alertmanager.yaml" }}' | base64 -d > alertmanager.yaml
 
# Edit alertmanager.yaml (add receivers, routes, etc.), then apply:
oc -n open-cluster-management-observability \
  create secret generic alertmanager-config \
  --from-file=alertmanager.yaml --dry-run=client -o yaml | \
  oc -n open-cluster-management-observability replace -f -
```
 
Custom notification templates go under the `templates:` key in `alertmanager.yaml` (mounted at `/etc/alertmanager/template/*.tmpl` in the Alertmanager pods).
 
---
 
## Where to Verify Alerts Are Working
 
| Check | Where |
|---|---|
| Metric labels | Hub → ACM console → Grafana → Explore |
| Active/pending alerts | Grafana Explore → query `ALERTS` or `ALERTS{alertname="YourAlertName"}` |
| Thanos Ruler rule load errors | Hub → `oc logs -n open-cluster-management-observability observability-thanos-rule-0 -c thanos-ruler` |
| Alertmanager delivery | Hub → `oc logs -n open-cluster-management-observability observability-alertmanager-0 -c alertmanager` |
 
---
 
## Important Notes

- Do not edit `thanos-ruler-default-rules` for custom alerts — use `thanos-ruler-custom-rules`. The operator manages the default ConfigMap.
- On **2.13**, do not point customers at the built-in `ViolatedPolicyReport` for cluster name. That default join lacks the ACM-30479 `max by` collapse.
- `acm_managed_cluster_info` does not have `managed_cluster_name` on 2.13 or 2.17. Use the collapsed join to `acm_managed_cluster_labels`, or use `acm_managed_cluster_status_condition` for availability (it does include `managed_cluster_name`).
- Do not add Prometheus external labels on managed clusters expecting them to appear on hub-scraped metrics like `acm_managed_cluster_info` — those metrics are emitted on the hub by `clusterlifecycle-state-metrics`, not on the managed cluster.
- Native name labels on those two metrics: [ACM-41089](https://issues.redhat.com/browse/ACM-41089) (ACM 5.1). Do not promise that on a 2.13 hub.
