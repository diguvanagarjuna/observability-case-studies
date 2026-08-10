# On-Prem AI Stack Observability — Reference Architecture
### For: OpenShift on bare-metal (Dell/Lenovo/IBM), vLLM model serving

## Goal
Single, correlated view from network/hardware layer up to application and AI
workload layer — not necessarily one vendor tool doing everything natively, but
one place where all the data lands and connects.

## Approach: "System of record" + OTel feeding into it

Rather than waiting for one vendor to natively understand DCGM, Ceph, RDMA, and
vLLM metrics out of the box, pick one platform as your **topology and correlation
system of record**, and use OpenTelemetry/Prometheus to feed everything else into
it with consistent labels. This is the practical pattern being used across the
industry right now — no vendor tool natively covers hardware-to-app in one box
for on-prem AI stacks yet.

```
┌─────────────────────────────────────────────────────────────┐
│                    SYSTEM OF RECORD                         │
│         (AppDynamics Flow Maps / Dynatrace Smartscape)      │
│         — Topology, correlation, business-transaction view  │
└───────────────────────▲───────────────────────────────────────┘
                         │  OTLP / metrics ingestion
┌────────────────────────┴──────────────────────────────────────┐
│                  OTel Collector (Gateway pattern)              │
│         Central aggregation, consistent labeling               │
└──┬──────────┬──────────┬──────────┬──────────┬────────────────┘
   │          │          │          │          │
   ▼          ▼          ▼          ▼          ▼
Hardware   Storage    Network    Platform    Model Serving
(IPMI/     (Ceph      (RDMA/     (OpenShift  (vLLM native
Redfish    exporter   NIC        Prometheus  Prometheus
per        via ODF)   metrics)   built-in)   metrics)
vendor)
```

## Layer-by-layer

### 1. Hardware (Dell / Lenovo / IBM)
- `ipmi_exporter` — works across all three vendors for baseline health
  (temp, power, fan, ECC errors)
- Check vendor-specific integrations too — Dell iDRAC and Lenovo XClarity both
  have deeper telemetry than generic IPMI in some cases; worth evaluating if
  generic IPMI misses anything you need

### 2. Storage
- If storage is via OpenShift Data Foundation (Ceph-based) — Ceph has a native
  Prometheus exporter, gives I/O latency, throughput, capacity, OSD health
- If it's separate SAN/NAS from Dell/Lenovo/IBM directly — check if it exposes
  Prometheus/SNMP metrics natively before building custom collection

### 3. Network
- Standard interface/throughput metrics via `node_exporter`
- If you're running RDMA/InfiniBand or RoCE for GPU-to-GPU or storage traffic
  (common for multi-node AI workloads) — this needs fabric-specific monitoring,
  since standard network metrics won't show fabric-level congestion or errors
  that quietly degrade training/inference throughput

### 4. GPU
- DCGM Exporter as a DaemonSet across GPU nodes — vendor-agnostic for NVIDIA
  GPUs regardless of whether the chassis is Dell, Lenovo, or IBM

### 5. Platform (OpenShift)
- OpenShift ships with built-in Prometheus monitoring already — the question is
  whether that's being forwarded into your central platform or sitting siloed.
  OTel Collector's `prometheus` receiver can scrape OpenShift's existing
  Prometheus and forward it onward without duplicating collection

### 6. Model Serving (vLLM)
- Native Prometheus metrics endpoint — `vllm:time_to_first_token_seconds`,
  `vllm:gpu_cache_usage_perc`, `vllm:num_requests_running`, etc. — no custom
  instrumentation needed here

### 7. Tying it together
- One OTel Collector Gateway deployment aggregates all of the above
- Every metric tagged consistently: node name, rack, hardware vendor, GPU ID,
  namespace — this consistent tagging is what actually makes correlation work
  in your chosen system-of-record platform, not the platform itself
- Feed into AppDynamics or Dynatrace depending on which you already have license
  for, or are evaluating — see honest trade-offs below

## Honest trade-offs on the "system of record" choice

**AppDynamics** — strong app-to-business-transaction correlation, has added
GPU monitoring at node/cluster level in recent releases. Good fit if you're
already invested in the Cisco/Splunk ecosystem.

**Dynatrace (Smartscape)** — currently the strongest pure topology/dependency
mapping engine, auto-builds real-time dependency graphs across hosts, network,
Kubernetes without manual tagging, and has recently expanded network/cloud
entity discovery. Less proven specifically on deep GPU/AI-hardware telemetry as
a native feature — you'd still be feeding DCGM data in via OTel rather than
relying on native GPU discovery.

**Instana** — lighter-weight deployment, decent app+infra correlation. Would
need to verify current GPU/bare-metal AI hardware support before recommending
for this specific use case.

**Bottom line:** whichever platform you pick, treat GPU/storage/network/model
metrics as things you *feed in* via OTel with good labels, not things you expect
the platform to natively understand for AI workloads yet. The correlation value
comes from consistent tagging across all layers landing in one place, not from
any single tool's native AI awareness.

## Next steps if this is useful

1. Confirm which platform (AppD/Dynatrace/Instana) you already have, or are
   evaluating — that decides where the "system of record" data lands
2. Stand up DCGM + Ceph + ipmi_exporter + vLLM metrics via one OTel Collector
   Gateway first, with consistent labeling — this is platform-agnostic and
   useful regardless of which system of record you choose
3. Validate correlation works end-to-end on one node/rack before scaling to
   full cluster

## Detailed Implementation Guide

The sections below give concrete, working configuration for each layer on
OpenShift bare-metal. Adjust namespaces, node selectors, and endpoints to match
your actual cluster.

### 1. Hardware layer — IPMI exporter (Dell/Lenovo/IBM)

Deploy as a Deployment (not DaemonSet) since `ipmi_exporter` typically polls
BMCs remotely over the network rather than running per-node:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ipmi-exporter
  namespace: hw-monitoring
spec:
  replicas: 1
  selector:
    matchLabels: { app: ipmi-exporter }
  template:
    metadata:
      labels: { app: ipmi-exporter }
    spec:
      containers:
        - name: ipmi-exporter
          image: prometheuscommunity/ipmi-exporter:latest
          ports:
            - containerPort: 9290
          volumeMounts:
            - name: config
              mountPath: /etc/ipmi_exporter
      volumes:
        - name: config
          configMap:
            name: ipmi-exporter-config
```

`ipmi-exporter-config` ConfigMap lists each BMC IP + credentials per module —
one module block per Dell/Lenovo/IBM server's BMC. Since credentials are
involved, store them via a Secret mounted into the config, not hardcoded.

If you want richer telemetry than generic IPMI provides, check whether Dell
(iDRAC) or Lenovo (XClarity) expose a native Redfish/Prometheus integration —
Redfish is increasingly preferred over legacy IPMI for detail and reliability.

### 2. Storage layer — Ceph exporter (via OpenShift Data Foundation)

If storage is backed by ODF, the Ceph Prometheus exporter is typically already
running as part of the ODF operator — you don't need to deploy it separately.
Verify and expose it:

```bash
oc get servicemonitor -n openshift-storage | grep ceph
oc get svc -n openshift-storage | grep prometheus
```

Point your central OTel Collector's `prometheus` receiver at this existing
ServiceMonitor's target rather than duplicating Ceph metric collection.

### 3. Network layer — RDMA/InfiniBand (if applicable)

If GPU-to-GPU or GPU-to-storage traffic runs over RDMA/InfiniBand:

```bash
# Check current fabric counters directly on a node (diagnostic, not exporter)
ibstat
perfquery -x
```

For continuous export, `node_exporter`'s `infiniband` collector can expose
port counters if enabled:

```yaml
args:
  - --collector.infiniband
```

For standard Ethernet network monitoring, `node_exporter` defaults (interface
throughput, errors, drops) are sufficient.

### 4. GPU layer — DCGM Exporter

Deploy via the official Helm chart (recommended over raw manifests since it
handles node selection and Prometheus Operator integration):

```bash
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update

helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  --namespace gpu-monitoring \
  --create-namespace \
  --set serviceMonitor.enabled=true
```

This deploys one exporter pod per GPU node automatically. Verify:

```bash
oc get pods -n gpu-monitoring
curl <dcgm-exporter-pod-ip>:9400/metrics | grep DCGM_FI_DEV_GPU_UTIL
```

Key metrics to confirm are flowing: `DCGM_FI_DEV_GPU_UTIL`,
`DCGM_FI_DEV_FB_USED`, `DCGM_FI_DEV_GPU_TEMP`, `DCGM_FI_DEV_POWER_USAGE`.

### 5. Platform layer — OpenShift's built-in Prometheus

OpenShift's cluster monitoring stack is already running in
`openshift-monitoring`. Rather than standing up a second Prometheus, configure
your OTel Collector to federate from it:

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'openshift-federate'
          honor_labels: true
          metrics_path: '/federate'
          params:
            'match[]':
              - '{__name__=~"node_.*"}'
              - '{__name__=~"kube_pod_.*"}'
          static_configs:
            - targets: ['prometheus-k8s.openshift-monitoring.svc:9091']
```

You'll need a ServiceAccount token with appropriate RBAC to read from
`openshift-monitoring` — OpenShift's Prometheus requires bearer-token auth by
default.

### 6. Model serving layer — vLLM

vLLM exposes metrics natively on its OpenAI-compatible API server — no sidecar
needed:

```bash
vllm serve <model-name> --port 8000
curl localhost:8000/metrics | grep vllm:
```

OTel Collector scrape config:

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'vllm'
          static_configs:
            - targets: ['vllm-service:8000']
```

Key metrics: `vllm:num_requests_running`, `vllm:num_requests_waiting`,
`vllm:gpu_cache_usage_perc`, `vllm:time_to_first_token_seconds`,
`vllm:time_per_output_token_seconds`. Metric names have changed across vLLM
versions — confirm exact names against your deployed version's `/metrics`
output before building dashboards/alerts on them.

### 7. Central OTel Collector Gateway — tying it all together

Deploy as a separate Gateway instance (not per-node DaemonSet) since it's
aggregating from multiple sources rather than local host telemetry:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-gateway
  namespace: observability
spec:
  replicas: 2
  selector:
    matchLabels: { app: otel-gateway }
  template:
    metadata:
      labels: { app: otel-gateway }
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:latest
          args: ["--config=/etc/otel/config.yaml"]
          volumeMounts:
            - name: config
              mountPath: /etc/otel
      volumes:
        - name: config
          configMap:
            name: otel-gateway-config
```

Collector config combining all receivers with consistent labeling:

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'ipmi'
          static_configs: [{ targets: ['ipmi-exporter.hw-monitoring:9290'] }]
        - job_name: 'dcgm'
          static_configs: [{ targets: ['dcgm-exporter.gpu-monitoring:9400'] }]
        - job_name: 'vllm'
          static_configs: [{ targets: ['vllm-service:8000'] }]
        - job_name: 'ceph'
          static_configs: [{ targets: ['<ceph-exporter-endpoint>'] }]

processors:
  k8sattributes:
    extract:
      metadata: [k8s.node.name, k8s.namespace.name, k8s.pod.name]
  resource:
    attributes:
      - key: hardware.vendor
        value: "dell"          # set per node-group/collector instance
        action: upsert
  batch: {}

exporters:
  # Choose based on which platform is your "system of record"
  otlphttp/dynatrace:
    endpoint: "https://<your-environment-id>.live.dynatrace.com/api/v2/otlp"
    headers:
      Authorization: "Api-Token <your-token>"
  # OR, if using AppDynamics/Splunk O11y:
  otlphttp/splunk:
    endpoint: "https://ingest.<realm>.signalfx.com"
    headers:
      X-SF-Token: "<your-token>"

service:
  pipelines:
    metrics:
      receivers: [prometheus]
      processors: [k8sattributes, resource, batch]
      exporters: [otlphttp/dynatrace]   # or otlphttp/splunk
```

**Note on the `hardware.vendor` label:** since your fleet spans Dell, Lenovo,
and IBM, run separate Collector instances (or use relabeling rules) per
hardware group so this attribute is set correctly per node rather than
hardcoded globally — this is what lets you later filter/correlate "is this a
Dell-node-specific issue or fleet-wide."

### 8. Validation checklist for this specific stack

- [ ] `ipmi_exporter` reachable and returning data for all three hardware vendors
- [ ] Ceph ServiceMonitor confirmed active in `openshift-storage`
- [ ] DCGM Exporter pods running on every GPU node (`oc get pods -n gpu-monitoring -o wide`)
- [ ] OpenShift Prometheus federation endpoint accessible with correct RBAC token
- [ ] vLLM `/metrics` endpoint returning `vllm:*` series
- [ ] OTel Gateway successfully exporting to chosen system-of-record platform (check for ingestion errors in Collector logs)
- [ ] `hardware.vendor` / node / rack labels present and correct on exported metrics
- [ ] End-to-end test: induce a known GPU load, confirm it's visible and correlated with the right node/vendor label in the system-of-record UI

---
*General reference architecture based on publicly available product
capabilities and OpenTelemetry/Prometheus patterns — not tied to any specific
customer environment. Commands and configs above are illustrative starting
points; validate against your exact software versions before running in
production.*

## Useful Links

**GPU monitoring**
- DCGM Exporter (official NVIDIA repo): https://github.com/NVIDIA/dcgm-exporter
- DCGM Exporter install/Helm chart guide: https://docs.nvidia.com/datacenter/dcgm/latest/installation/install-dcgm-exporter.html

**Model serving (vLLM)**
- vLLM Production Metrics (official docs): https://docs.vllm.ai/en/latest/design/metrics/

**Topology / correlation platforms**
- AppDynamics Flow Maps overview: https://help.splunk.com/en/appdynamics-saas/application-performance-monitoring/25.4.0/business-applications/flow-maps/flow-map-overview
- Dynatrace Smartscape concepts: https://docs.dynatrace.com/docs/analyze-explore-automate/smartscape/smartscape-concepts
- Dynatrace Smartscape overview: https://www.dynatrace.com/hub/detail/smartscape/

**General**
- OpenTelemetry Collector documentation: https://opentelemetry.io/docs/collector/
- Prometheus exporters and integrations list: https://prometheus.io/docs/instrumenting/exporters/

*Note: exact metric names, install steps, and product capabilities change over
time — always confirm against the current version of each doc before
implementing.*

