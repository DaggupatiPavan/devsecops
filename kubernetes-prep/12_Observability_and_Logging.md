# 12. Observability & Logging

## 1. Metrics (Prometheus)

Prometheus is the de-facto standard for Kubernetes monitoring.

### Architecture
- **Prometheus Server**: Scrapes metrics from targets at intervals. Stores them in a Time Series Database (TSDB).
- **Exporters**: Sidecars or agents that expose metrics (e.g., `node-exporter` for OS metrics, `kube-state-metrics` for K8s objects).
- **AlertManager**: Handles alerts sent by Prometheus (grouping, inhibition, routing to Slack/PagerDuty).
- **Grafana**: Visualizes the data.

### PromQL (Prometheus Query Language)
- `rate(http_requests_total[5m])`: Per-second rate of requests over the last 5 minutes.
- `sum by (pod) (container_memory_usage_bytes)`: Total memory per pod.

## 2. Logging Stacks

### EFK Stack (Legacy/Heavy)
- **Elasticsearch**: Search and storage engine (Java-based, resource heavy).
- **Fluentd/Fluentbit**: Log collector (DaemonSet) that tails `/var/log/containers/*.log`.
- **Kibana**: Visualization UI.

### PLG Stack (Modern/Lightweight)
- **Promtail**: Log collector (DaemonSet). Designed for Loki.
- **Loki**: "Prometheus for Logs". Does not index the text, only labels. Very efficient.
- **Grafana**: Visualization UI (Single pane of glass for Metrics + Logs).

### Logging Architecture
1.  **Node Level**: The container runtime writes stdout/stderr to `/var/log/containers/`.
2.  **Cluster Level**: An agent (Fluentbit/Promtail) runs on every node, reads these files, enriches them with K8s metadata (Pod name, Namespace), and pushes to the backend.

## 3. Distributed Tracing (OpenTelemetry)

Tracing follows a request across microservices to identify bottlenecks.

### OpenTelemetry (OTEL)
A vendor-neutral standard for generating and collecting telemetry (Metrics, Logs, Traces).

- **OTEL SDK**: Library inside your app to generate traces.
- **OTEL Collector**: Agent that receives traces, processes them, and exports to backends (Jaeger, Tempo, Datadog).

### Span Context
To trace a request, a **Trace ID** must be propagated via HTTP headers (e.g., `traceparent`) between services.

```
Frontend -> (Header: TraceID=123) -> Backend -> (Header: TraceID=123) -> DB
```
