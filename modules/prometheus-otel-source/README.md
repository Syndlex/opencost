# OpenCost Data Sources - Prometheus with OpenTelemetry

This module provides OpenCost with metrics and metadata for cost allocation
using OpenTelemetry Collector as the primary metrics pipeline. It extends the
standard Prometheus data source (`modules/prometheus-source`) by rewriting
PromQL queries to use OTel metric names and label conventions.

The OTel Collector translates metric names from dots to underscores when
exposing them via the Prometheus exporter (e.g. `container.cpu.time` becomes
`container_cpu_time`). Resource attributes become Prometheus target labels
(e.g. `k8s.node.name` becomes `k8s_node_name`).

## Required OTel Collector Components

### kubeletstats receiver

Container, pod, node, and volume runtime metrics from the Kubelet API.

- **Source:** [receiver/kubeletstatsreceiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/kubeletstatsreceiver)
- **Metrics:** [documentation.md](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/kubeletstatsreceiver/documentation.md)
- **Definitions:** [metadata.yaml](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/kubeletstatsreceiver/metadata.yaml)

| OTel Metric | Prometheus Name | Used For |
|---|---|---|
| `container.cpu.time` | `container_cpu_time` | CPU usage rate, CPU max (irate fallback) |
| `container.memory.usage` | `container_memory_usage` | RAM bytes allocated |
| `container.memory.working_set` | `container_memory_working_set` | RAM usage avg/max, data coverage |
| `k8s.pod.network.io` | `k8s_pod_network_io` | Network transfer/receive bytes |
| `k8s.volume.capacity` | `k8s_volume_capacity` | PV used average/max calculation |
| `k8s.volume.available` | `k8s_volume_available` | PV used average/max calculation |

Resource attributes exposed as Prometheus labels: `k8s_container_name`,
`k8s_pod_name`, `k8s_namespace_name`, `k8s_node_name`, `k8s_pod_uid`,
`k8s_volume_name`, `k8s_persistentvolumeclaim_name`.

> **Note:** As of OTel Collector v0.125.0+, `container.cpu.usage` replaces the
> deprecated `container.cpu.utilization`. See the
> [CPU metrics transition blog post](https://opentelemetry.io/blog/2025/kubeletstats-receiver-metrics-deprecation/).

### k8s_cluster receiver

Cluster-level metrics from the Kubernetes API: resource requests, limits,
allocatable capacity, and PV/PVC status.

- **Source:** [receiver/k8sclusterreceiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/k8sclusterreceiver)
- **Metrics:** [documentation.md](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/k8sclusterreceiver/documentation.md)
- **Definitions:** [metadata.yaml](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/k8sclusterreceiver/metadata.yaml)

| OTel Metric | Prometheus Name | Used For |
|---|---|---|
| `k8s.container.cpu_limit` | `k8s_container_cpu_limit` | CPU limits (not yet used — querier uses `kube_pod_container_resource_limits`) |
| `k8s.container.memory_limit` | `k8s_container_memory_limit` | RAM limits (not yet used — querier uses `kube_pod_container_resource_limits`) |
| `k8s.container.cpu_request` | `k8s_container_cpu_request` | CPU requests (not yet used — querier uses `kube_pod_container_resource_requests`) |
| `k8s.container.memory_request` | `k8s_container_memory_request` | RAM requests (not yet used — querier uses `kube_pod_container_resource_requests`) |
| `k8s.node.allocatable_cpu` | `k8s_node_allocatable_cpu` | Not yet used — querier uses `kube_node_status_allocatable` from KSM |
| `k8s.node.allocatable_memory` | `k8s_node_allocatable_memory` | Not yet used — querier uses `kube_node_status_allocatable` from KSM |
| `k8s.pod.phase` | `k8s_pod_phase` | Pod active minutes (phase == 2 = Running) — not used; querier uses `kube_pod_container_status_running` |

The following optional metrics should be enabled in the receiver config:

| OTel Metric | Prometheus Name | Used For |
|---|---|---|
| `k8s.persistentvolume.storage.capacity` | `k8s_persistentvolume_storage_capacity` | Not used — querier uses `kube_persistentvolume_capacity_bytes` from KSM |
| `k8s.persistentvolumeclaim.storage.request` | `k8s_persistentvolumeclaim_storage_request` | Not used — querier uses `kube_persistentvolumeclaim_resource_requests_storage_bytes` from KSM |

The `allocatable_types_to_report` config option must include `cpu` and `memory`
to emit the allocatable metrics.

> **Note:** Node allocatable is used as an approximation for node capacity.
> Allocatable = capacity minus system-reserved. The difference is typically
> small and acceptable for cost allocation purposes.

### host_metrics receiver

Node-level system metrics: CPU time by mode and filesystem usage.

- **Source:** [receiver/hostmetricsreceiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/hostmetricsreceiver)
- **Metrics:** [documentation.md](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/hostmetricsreceiver/documentation.md)

| OTel Metric | Prometheus Name | Used For |
|---|---|---|
| `system.cpu.time` | `system_cpu_time` | Node CPU mode breakdown (idle/system/user) |
| `system.filesystem.usage` | `system_filesystem_usage` | Local storage cost and usage |

### kube-state-metrics (KSM) -- Still Required

Several queries have no OTel Collector equivalent and still require
[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) scraped
via the Prometheus receiver or a Prometheus instance.

| KSM Metric | Used For | Why No OTel Equivalent |
|---|---|---|
| `kube_pod_container_resource_requests` | GPU requests (nvidia_com_gpu), GPU sharing (integer resources) | k8s_cluster emits separate per-resource metrics but not for GPU vendors |
| `kube_persistentvolumeclaim_info` | PVC-to-PV/StorageClass join | k8s_cluster provides this as resource attributes, not a joinable metric |
| `kube_node_labels` | Node label metadata | OTel uses resource attributes, not label metrics |
| `kube_namespace_labels`, `kube_namespace_annotations` | Namespace metadata | Same |
| `kube_pod_labels`, `kube_pod_annotations` | Pod label metadata | Same |
| `kube_pod_owner` | Pod-to-ReplicaSet/Deployment mapping | Same |
| `kube_service_labels` | Service metadata | Same |
| `kube_deployment_labels` | Deployment metadata | Same |
| `kube_statefulset_labels` | StatefulSet metadata | Same |
| `kube_daemonset_labels` | DaemonSet metadata | Same |
| `kube_job_labels` | Job metadata | Same |
| `kube_replicaset_created`, `kube_replicaset_owner` | ReplicaSet ownership | Same |

> **Label mismatch warning:** KSM uses classic Prometheus labels (`container`,
> `pod`, `namespace`, `node`), but this module's queries use OTel-style labels
> (`k8s_container_name`, `k8s_pod_name`, `k8s_namespace_name`, `k8s_node_name`).
> For KSM metrics to work, you must either:
> 1. Use `metric_relabel_configs` in the Prometheus scrape config to rename labels, or
> 2. Use an OTel `transform` processor to rename attributes before the Prometheus exporter.
>
> This label mismatch is tracked as open work (see below).

### OpenCost Custom Metrics

These metrics are emitted by OpenCost itself and are **not** from the OTel
Collector. They must be scraped from the OpenCost `/metrics` endpoint:

| Metric | Purpose |
|---|---|
| `node_total_hourly_cost` | Total node cost per hour |
| `node_cpu_hourly_cost` | Node CPU cost per hour |
| `node_ram_hourly_cost` | Node RAM cost per hour |
| `node_gpu_hourly_cost` | Node GPU cost per hour |
| `node_gpu_count` | GPU count per node |
| `kubecost_node_is_spot` | Spot/preemptible node indicator |
| `pv_hourly_cost` | PV cost per hour |
| `kubecost_pv_info` | PV provider/storage class info |
| `pod_pvc_allocation` | Pod-to-PVC binding |
| `container_cpu_allocation` | Container CPU allocation |
| `container_gpu_allocation` | Container GPU allocation |
| `kubecost_container_cpu_usage_irate` | Recording rule: CPU irate |
| `kubecost_pod_network_egress_bytes_total` | Network egress bytes |
| `kubecost_pod_network_ingress_bytes_total` | Network ingress bytes |
| `kubecost_network_*_cost` | Network cost per GiB (zone/region/internet/NAT gateway) |
| `kubecost_load_balancer_cost` | Load balancer cost |
| `kubecost_cluster_management_cost` | Cluster management cost |

> **Note:** OpenCost custom metrics use the classic `node` label. Queries use
> `label_replace()` to transform `node` to `k8s_node_name` for consistency.

### NVIDIA DCGM (Optional)

For GPU cost allocation, [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
metrics are needed:

| Metric | Used For |
|---|---|
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | GPU usage avg/max |
| `DCGM_FI_DEV_DEC_UTIL` | GPU device info (model, UUID) |

## OTel Label Mapping

This module uses OTel-style labels (from resource attributes) throughout all
PromQL queries. These replace the classic Prometheus/cAdvisor/KSM labels:

| OTel Resource Attribute | Prometheus Label | Replaces Classic |
|---|---|---|
| `k8s.container.name` | `k8s_container_name` | `container`, `container_name` |
| `k8s.pod.name` | `k8s_pod_name` | `pod`, `pod_name` |
| `k8s.namespace.name` | `k8s_namespace_name` | `namespace` |
| `k8s.node.name` | `k8s_node_name` | `node`, `kubernetes_node` |
| `k8s.persistentvolume.name` | `k8s_persistentvolume_name` | `persistentvolume` |
| `k8s.persistentvolumeclaim.name` | `k8s_persistentvolumeclaim_name` | `persistentvolumeclaim` |
| `k8s.volume.name` | `k8s_volume_name` | `volumename` |
| `k8s.storageclass.name` | `k8s_storageclass_name` | `storageclass` |

## Known Gaps and Open Work

### KSM queries use OTel labels but KSM emits classic labels

Several queries still reference KSM metrics (`kube_pod_container_resource_requests`
for GPU resources, `kube_persistentvolumeclaim_info`) but filter and group by
OTel-style labels (`k8s_container_name`, `k8s_node_name`). KSM does not emit
these labels natively. These queries will return empty results unless label
transformation is configured (via Prometheus `metric_relabel_configs` or an OTel
processor).

**Options to fix:**
- Require users to configure label relabeling for the remaining KSM metrics
- For metrics without OTel equivalents, the label mapping in the result parser
  could be extended

### Metrics without OTel equivalents

| Feature | Current Metric | Gap |
|---|---|---|
| PVC info joins | `kube_persistentvolumeclaim_info` | Resource attributes only, not joinable metric |
| CPU throttling diagnostic | `container_cpu_cfs_throttled_periods_total` | cAdvisor metric, not in kubeletstats (diagnostic removed) |
| Label/annotation metadata | `kube_*_labels`, `kube_*_annotations` | OTel uses resource attributes, not info metrics |
| GPU resource requests | `kube_pod_container_resource_requests{resource="nvidia_com_gpu"}` | k8s_cluster does not emit per-GPU-vendor metrics |
| PV used bytes | `k8s_volume_capacity` / `k8s_volume_available` | Available but cannot be joined to PVC names in PromQL (pod volumeMount name ≠ PVC name) |

### Metrics from KSM (kube-state-metrics)

The following KSM metrics are used directly. The k8scluster OTel receiver emits
alternative metrics (`k8s_container_cpu_request` etc.) but they are **not yet
equivalent** — the KSM variants carry more labels and are used by the allocation
pipeline. Future work may migrate to the OTel k8scluster receiver.

| KSM Metric | Used For |
|---|---|
| `kube_pod_container_resource_requests` | Container CPU and RAM requests |
| `kube_pod_container_resource_limits` | Container CPU and RAM limits |
| `kube_pod_container_status_running` | Container and pod active minutes |
| `kube_node_status_capacity` | Node CPU and RAM capacity |
| `kube_node_status_allocatable` | Node CPU and RAM allocatable |
| `kube_pod_owner` | Pod→ReplicaSet/DaemonSet/Job ownership |
| `kube_replicaset_owner` | ReplicaSet→Deployment ownership |
| `kube_persistentvolume_capacity_bytes` | PV capacity and active minutes |
| `kube_persistentvolumeclaim_info` | PVC→PV join and storage class |
| `kube_persistentvolumeclaim_resource_requests_storage_bytes` | PVC requested bytes |

## Example OTel Collector Configuration

Minimal configuration showing all required components:

```yaml
receivers:
  kubelet_stats:
    collection_interval: 30s
    auth_type: serviceAccount
    endpoint: "${env:K8S_NODE_NAME}:10250"
    insecure_skip_verify: true
    metric_groups:
      - container
      - pod
      - node
      - volume

  k8s_cluster:
    auth_type: serviceAccount
    collection_interval: 30s
    allocatable_types_to_report:
      - cpu
      - memory
    metrics:
      k8s.persistentvolume.storage.capacity:
        enabled: true
      k8s.persistentvolumeclaim.storage.request:
        enabled: true

  host_metrics:
    collection_interval: 30s
    scrapers:
      cpu: {}
      filesystem: {}

  prometheus:
    config:
      scrape_configs:
        - job_name: kube-state-metrics
          static_configs:
            - targets: ['kube-state-metrics:8080']
        - job_name: opencost
          static_configs:
            - targets: ['opencost:9003']

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
    resource_to_telemetry_conversion:
      enabled: true

service:
  pipelines:
    metrics:
      receivers: [kubelet_stats, k8s_cluster, host_metrics, prometheus]
      exporters: [prometheus]
```

> **Important:** `resource_to_telemetry_conversion: enabled: true` is required
> so that OTel resource attributes (like `k8s.node.name`) are promoted to
> Prometheus metric labels. Without this, the OTel-style labels will not appear
> on scraped metrics.

## Sharded Prometheus Best Practices

If running Prometheus in a sharded (HA) setup, each Prometheus pod only scrapes
a subset of targets. Set `PROMETHEUS_SERVER_ENDPOINT` to a global query endpoint
that aggregates all shards:

- [Thanos Query](https://thanos.io/tip/components/query.md/)
- [Cortex Query Frontend](https://cortexmetrics.io/docs/architecture/)
- [Mimir Query Frontend](https://grafana.com/docs/mimir/latest/operations/query-frontend/)

```
export PROMETHEUS_SERVER_ENDPOINT="http://thanos-query-frontend:9090"
```

For more details, see the
[OpenCost documentation](https://www.opencost.io/docs/installation/prometheus).
