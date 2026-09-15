# RHACM Observability: cluster name in alert annotations

This is the customer workaround write-up in
[`ch-stark/createalertonmanagedclusteroffline`](https://github.com/ch-stark/createalertonmanagedclusteroffline/blob/main/managed_cluster_name_inalerts.md)
(`managed_cluster_name_inalerts.md`). It was originally written against **RHACM 2.17**.
Use the **ACM 2.13 analysis** and **workaround** sections below if that is the hub version under support.

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
| `acm_managed_cluster_labels` | `name` **only if** that key exists on the ManagedCluster object | `managed_cluster_id` plus every MC label; not a reliable name source on 2.13 |
| `acm_managed_cluster_status_condition` | ✅ `managed_cluster_name` (`metadata.name`) | also `managed_cluster_id` on 2.13 |

Where a metric is missing the name (rows 2–3), put it on the **alert** with a join. On **2.13**, use `acm_managed_cluster_status_condition` as the lookup (`on (managed_cluster_id)` → `managed_cluster_name`). Do not join `acm_managed_cluster_labels` on `name`: that label is not first-class, and a join that is not keyed on `managed_cluster_id` fails with Thanos `422` / `match group {}` / `many-to-many matching not allowed`.

[ACM-30479](https://issues.redhat.com/browse/ACM-30479) is a **different** failure: the default `ViolatedPolicyReport` rule joined `policyreport_info` to the labels metric without collapsing extra labels. That error is `match group {managed_cluster_id="…"}` with the **same** cluster name. It is fixed in 2.16.3 / 2.17.1 / 5.0 ([ACM-42977](https://issues.redhat.com/browse/ACM-42977) tracks 2.14.z / 2.15.z). It does not add names to `acm_managed_cluster_info`, does not land on 2.13, and does not fix a custom join with an empty match group.

| Hub version | Built-in `ViolatedPolicyReport` join | What to give the customer |
|---|---|---|
| **2.13** (and 2.14 / 2.15 until the clone ships) | **Unsafe** — no `max by` collapse | Custom rules only (this 2.13 section) |
| 2.16.3, 2.17.1, 5.0.0+ | Join fixed in default rules ([ACM-30479](https://issues.redhat.com/browse/ACM-30479)) | Workarounds 1–3 below; default policy rule is OK |
| 5.1+ (ACM-41089) | Native name labels (planned) | No join required once the metric carries `managed_cluster_name` |

---

## ACM 2.13 analysis

The customer claim is valid: on a 2.13 hub, `acm_managed_cluster_info` and `policyreport_info` do not carry a human-readable cluster name. That is documented as **Stable**
([Observability 2.13](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/observability/observing-environments-intro)).
Native `managed_cluster_name` on those two metrics is [ACM-41089](https://issues.redhat.com/browse/ACM-41089) (ACM 5.1). Do not promise it on 2.13.

These hub metrics are emitted by `clusterlifecycle-state-metrics` on the hub (`backplane-2.8` ≈ ACM 2.13), not by Prometheus on the managed cluster. Adding Prometheus external labels on spokes will not appear on `acm_managed_cluster_info`.

### What 2.13 actually emits

Confirmed from `stolostron/clusterlifecycle-state-metrics` branch [`backplane-2.8`](https://github.com/stolostron/clusterlifecycle-state-metrics/tree/backplane-2.8):
[`managedclusterinfo.go`](https://github.com/stolostron/clusterlifecycle-state-metrics/blob/backplane-2.8/pkg/generators/cluster/managedclusterinfo.go),
[`managedclusterlabels.go`](https://github.com/stolostron/clusterlifecycle-state-metrics/blob/backplane-2.8/pkg/generators/cluster/managedclusterlabels.go),
[`managedclusterstatus.go`](https://github.com/stolostron/clusterlifecycle-state-metrics/blob/backplane-2.8/pkg/generators/cluster/managedclusterstatus.go).

**`acm_managed_cluster_info`** — no name. Labels: `hub_cluster_id`, `managed_cluster_id`, `vendor`, `cloud`, `service_name`, `version`, `available`, `created_via`, `core_worker`, `socket_worker`, `hub_type`, `product`.

When `available`, `version`, vendor, or cores change, Prometheus/Thanos can keep the old and new label sets alive for a few minutes. That is two series for the **same** ID with different *other* labels — not two names.

**`acm_managed_cluster_labels`** — defaults are only `hub_cluster_id` and `managed_cluster_id`. Every label on the ManagedCluster object is then copied on as a Prometheus label. `name` exists **only if** the object has a `name=` label; it is not a dedicated field. This metric is one series per cluster **and** label-set, so a day-2 label edit leaves two series for the same ID until the old set expires.

**`acm_managed_cluster_status_condition`** — present since ACM 2.7. On 2.13 it has `managed_cluster_id` (from the clusterID claim; falls back to `metadata.name` if empty) and `managed_cluster_name` (`mc.GetName()`, the ManagedCluster name). Extra labels are `condition` and `status`. Many series per cluster is **by design**, not label churn. ManagedCluster `metadata.name` is immutable, so this metric cannot grow a second name for one cluster because of a rename.

**`policyreport_info`** — `managed_cluster_id` only. No name on 2.13.

**Forwarded fleet metrics** (for example `kube_node_status_allocatable`) — already have `cluster` (ManagedCluster name) and `clusterID`. No join needed. Workaround 1 below is safe on 2.13.

`getClusterID()` uses the OpenShift clusterID claim when present, otherwise `metadata.name`. Two OpenShift clusters that somehow share a clusterID can collide on `managed_cluster_id`; that is a data problem, not stale Prometheus names.

### Two different 422 errors

Thanos `many-to-many matching not allowed` is not one bug. Read the match group.

**Empty match group — customer PromQL, not ACM-30479**

```
found duplicate series for the match group {} on the right-hand side of the operation:
[{name="ilab-ctigtdc15d"}, {name="ilab-ctigtdcspk1d"}];
many-to-many matching not allowed: matching labels must be unique on one side
```

`{}` means the join is **not keyed on `managed_cluster_id`**: `on()`, `on(name)`, or the right side dropped the ID (`max by (name)`). Prometheus puts every right-hand series in one bucket. The two `name` values are two ManagedClusters (typical lab hostnames). That is not a rename with a leftover series.

`max by (managed_cluster_id, name)` does not fix this. If the join is not `on (managed_cluster_id)`, collapse never runs on the ID. If two different names share an ID, `max by (id, name)` **keeps both rows**.

**Match group with the ID — ACM-30479 shape**

```
match group {managed_cluster_id="62f2c026-…"}
```

same cluster name, extra labels (for example a day-2 ManagedCluster label). That is labels-metric cardinality. The default `ViolatedPolicyReport` expr on 2.13 joins without `max by`, so it hits this. [ACM-30479](https://issues.redhat.com/browse/ACM-30479) fixed that default rule in **2.16.3 / 2.17.1 / 5.0**. [ACM-42977](https://issues.redhat.com/browse/ACM-42977) is the 2.14.z / 2.15.z clone (no 2.13 clone). ACM-30479 does **not** add names to `acm_managed_cluster_info`, does not change status-condition cardinality, and does not fix a custom join with `match group {}`.

Do not tell the customer “Prometheus kept two names for one cluster because labels changed” and do not point them at ACM-30479 for the empty-match-group error.

### What collapses what

| Construct | Fixes | Does not fix |
|---|---|---|
| `on (managed_cluster_id)` | empty match group `{}` | extra labels on the right side |
| `max by (managed_cluster_id, name)` | extra labels, **same** name (ACM-30479) | `{}`; two different names for one ID |
| `topk by (managed_cluster_id) (1, …)` | forces one right-hand row per ID | if two clusters share a clusterID, the name is arbitrary |
| `last_over_time(...[10m])` | Thanos scrape gaps / brief overlap | a missing join key |

On 2.13 the lookup table must be `acm_managed_cluster_status_condition` (`managed_cluster_name`), not `acm_managed_cluster_labels` (`name`).

### Jiras (do not mix)

| Jira | What it is | On 2.13? |
|---|---|---|
| [ACM-41089](https://issues.redhat.com/browse/ACM-41089) | Native `managed_cluster_name` on `acm_managed_cluster_info` / `policyreport_info` ([clusterlifecycle-state-metrics#677](https://github.com/stolostron/clusterlifecycle-state-metrics/pull/677), [insights-metrics#554](https://github.com/stolostron/insights-metrics/pull/554)) | **No** — targeted at ACM 5.1 |
| [ACM-30479](https://issues.redhat.com/browse/ACM-30479) | Default `ViolatedPolicyReport` missing `max by` | **No** — 2.16.3 / 2.17.1 / 5.0 |
| [ACM-42977](https://issues.redhat.com/browse/ACM-42977) | ACM-30479 clone for 2.14.z / 2.15.z | **No** 2.13 clone |
| [ACM-34481](https://issues.redhat.com/browse/ACM-34481) | Leftover MCOA `PrometheusRules` plus MCO evaluating the same alert (two ALERT series after a *good* join) | Possible; check if they still see duplicates after 2.13-C |

### What to send / not send

| Send | Do not send |
|---|---|
| **2.13-A** for unavailable-cluster (preferred, no join) | Workaround 2 / 3 `acm_managed_cluster_labels` + `name` join |
| **2.13-B / 2.13-C** if they must alert on `acm_managed_cluster_info` or `policyreport_info` | Built-in `ViolatedPolicyReport` on 2.13 (`thanos-ruler-default-rules`) |
| Custom rules only in `thanos-ruler-custom-rules` | “This is ACM-30479; it will be fixed in 2.16.3 / 2.17.1 / 5.0” as the answer to `match group {}` |
| | ACM-41089 as a 2.13 fix |

Have them confirm labels in Grafana Explore on **their** hub before copying annotations (`managed_cluster_name` vs `name`).

### Support reply (copy)

**Summary**

On ACM 2.13, `acm_managed_cluster_info` and `policyreport_info` only have `managed_cluster_id`, not a cluster name. That is expected ([Observability 2.13](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/observability/observing-environments-intro)). Native `managed_cluster_name` on those metrics is ACM-41089 and is not in 2.13.

The error is not “one cluster with two names because Prometheus kept a stale series.” An empty match group `{}` means the join is **not keyed on `managed_cluster_id`**. Prometheus then puts every right-hand series in one bucket, so two different clusters collide:

```
found duplicate series for the match group {} on the right-hand side of the operation:
[{name="ilab-ctigtdc15d"}, {name="ilab-ctigtdcspk1d"}];
many-to-many matching not allowed: matching labels must be unique on one side
```

Those are two ManagedCluster names. ManagedCluster `metadata.name` does not change, so this is not a rename with a leftover series. On 2.13 the `name` label comes from `acm_managed_cluster_labels` (only if that key exists on the object). `acm_managed_cluster_info` has no `name`.

This is **not** ACM-30479. That bug is the default `ViolatedPolicyReport` rule: extra labels, **same** cluster name, match group `{managed_cluster_id="…"}`. It was fixed for 2.16.3 / 2.17.1 / 5.0; ACM-42977 covers 2.14.z / 2.15.z. That change does not add names to `acm_managed_cluster_info`, does not land on 2.13, and will not fix this custom join.

**Workaround (ACM 2.13)**

Preferred — no join. Alert on status condition; it already has `managed_cluster_name`:

```promql
acm_managed_cluster_status_condition{
  condition="ManagedClusterConditionAvailable",
  status!="True"
} == 1
```

If you must use `acm_managed_cluster_info`, look the name up from status condition. Join **only** on the ID:

```promql
acm_managed_cluster_info{available!="True"}
* on(managed_cluster_id) group_left(managed_cluster_name)
  topk by (managed_cluster_id) (1,
    max by (managed_cluster_id, managed_cluster_name) (
      last_over_time(acm_managed_cluster_status_condition[10m])
    )
  )
```

Do not join `acm_managed_cluster_labels` on `name`, and do not use `on()` with no labels. `topk` is only a guard if two ManagedClusters share a `managed_cluster_id`; it picks one name. Please try this in the hub Console and send the result (or a new error).

---

## ACM 2.13 workaround (what to send the customer)

Put custom rules only in ConfigMap `thanos-ruler-custom-rules` in namespace `open-cluster-management-observability`.
Do **not** tell a 2.13 customer to rely on ConfigMap `thanos-ruler-default-rules` / alert `ViolatedPolicyReport` for the name — that default expr is the ACM-30479 join (no `max by`).

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

Join **only** on `managed_cluster_id`. Copy `managed_cluster_name` from `acm_managed_cluster_status_condition` (immutable ManagedCluster name). `max by` collapses the many condition series; `topk` keeps one row per ID if two names ever share an ID; `last_over_time` covers Thanos scrape gaps.

Do **not** join `acm_managed_cluster_labels` on `name`. `max by (managed_cluster_id, name)` does not fix an empty match group `{}`, and it does not collapse two different names for one ID.

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
              * on (managed_cluster_id) group_left (managed_cluster_name)
                topk by (managed_cluster_id) (1,
                  max by (managed_cluster_id, managed_cluster_name) (
                    last_over_time(acm_managed_cluster_status_condition[10m])
                  )
                )
            for: 5m
            labels:
              severity: critical
              cluster: "{{ $labels.managed_cluster_name }}"
            annotations:
              summary: "Cluster {{ $labels.managed_cluster_name }} is unavailable"
              description: "Managed cluster {{ $labels.managed_cluster_name }} (ID: {{ $labels.managed_cluster_id }}) is not available."
```

### 2.13-C — Policy violations (custom rule only)

Do not use the 2.13 built-in `ViolatedPolicyReport` for this. Use the same status-condition lookup as 2.13-B (`managed_cluster_name`, not `name`).

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
              sum by (managed_cluster_name, policy, severity) (
                policyreport_info{result="fail"}
                * on (managed_cluster_id) group_left (managed_cluster_name)
                  topk by (managed_cluster_id) (1,
                    max by (managed_cluster_id, managed_cluster_name) (
                      last_over_time(acm_managed_cluster_status_condition[10m])
                    )
                  )
              ) > 0
            for: 1m
            labels:
              severity: "{{ $labels.severity }}"
            annotations:
              summary: "Policy violation on cluster {{ $labels.managed_cluster_name }}"
              description: "Policy {{ $labels.policy }} (severity: {{ $labels.severity }}) on cluster {{ $labels.managed_cluster_name }}."
```

If they used `max by` and still see **two ALERT series** for one violation, check for leftover MCOA `PrometheusRules` plus MCO evaluating the same alert ([ACM-34481](https://issues.redhat.com/browse/ACM-34481)) — that is a second evaluator, not a bad join.

Apply on the hub:

```bash
oc apply -f thanos-ruler-custom-rules.yaml -n open-cluster-management-observability
```

Confirm in Grafana Explore: `ALERTS{alertname="ManagedClusterUnavailable"}`.

On-hub checks before arguing about duplicates:

```
# info: should be one series per ID unless available/version/cores just changed
count by (managed_cluster_id) (acm_managed_cluster_info)

# status condition: many series per cluster is expected (condition × status)
count by (managed_cluster_id, managed_cluster_name) (acm_managed_cluster_status_condition)

# labels: name is optional; this is not the 2.13 lookup table
count by (managed_cluster_id, name) (acm_managed_cluster_labels)
```

If Thanos ruler logs show `422` / `many-to-many matching not allowed`:

- **`match group {}`** with two different cluster names — the join is not keyed on `managed_cluster_id` (for example `on()`, or the right side dropped the ID). Use 2.13-B/C as written. This is not ACM-30479.
- **`match group {managed_cluster_id="…"}`** with the **same** name and extra labels — labels-metric cardinality; collapse with `max by`. That is the ACM-30479 shape (default `ViolatedPolicyReport` on 2.13).

If the join is good and they still see **two ALERT series**, that is [ACM-34481](https://issues.redhat.com/browse/ACM-34481) (two evaluators), not a bad join.

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

**2.13:** skip this section. Use **2.13-A** (preferred) or **2.13-B** (status-condition lookup). Do not copy the `acm_managed_cluster_labels` / `name` join below onto a 2.13 hub.

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
- On **2.13**, read **ACM 2.13 analysis** above before sending a Jira. Empty `match group {}` is a missing join key, not ACM-30479. Do not point customers at the built-in `ViolatedPolicyReport` for cluster name.
- `acm_managed_cluster_info` does not have `managed_cluster_name` on 2.13 or 2.17. On 2.13, look the name up from `acm_managed_cluster_status_condition` with `on (managed_cluster_id)` (2.13-B/C). For availability-only alerts, use 2.13-A and skip the join. Do not join `acm_managed_cluster_labels` on `name`.
- Do not add Prometheus external labels on managed clusters expecting them to appear on hub-scraped metrics like `acm_managed_cluster_info` — those metrics are emitted on the hub by `clusterlifecycle-state-metrics`, not on the managed cluster.
- Native name labels on those two metrics: [ACM-41089](https://issues.redhat.com/browse/ACM-41089) (ACM 5.1). Do not promise that on a 2.13 hub.
