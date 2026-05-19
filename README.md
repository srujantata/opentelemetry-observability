# OpenTelemetry Observability Stack

[![CI](https://github.com/srujantata/opentelemetry-observability/actions/workflows/deploy.yml/badge.svg)](https://github.com/srujantata/opentelemetry-observability/actions)

Full-stack Kubernetes observability using **OpenTelemetry** as the single collection layer,
routing metrics → Prometheus, logs → Loki, traces → Tempo, all visualised in Grafana.
One collector config. No vendor lock-in.

---

## Architecture

```
Your Application
  └── OTel SDK (traces + metrics + logs)
        │
        ▼  OTLP (gRPC :4317 / HTTP :4318)
  OpenTelemetry Collector
  ├── Prometheus exporter  ──►  Prometheus  ──┐
  ├── Loki exporter        ──►  Loki        ──┤──► Grafana (unified dashboard)
  └── OTLP/Tempo exporter  ──►  Tempo       ──┘

Cluster-level scraping:
  kube-state-metrics  ──► Prometheus (Deployment/Pod/Node state)
  node-exporter       ──► Prometheus (CPU, memory, disk, network)
```

---

## Components

| Component | Purpose | Port |
|-----------|---------|------|
| OTel Collector | Receive OTLP, fan-out to backends | 4317 (gRPC), 4318 (HTTP) |
| Prometheus | Metrics storage + alerting | 9090 |
| Loki | Log aggregation | 3100 |
| Tempo | Distributed tracing | 3200 |
| Grafana | Unified dashboards | 3000 |
| kube-state-metrics | Kubernetes object metrics | 8080 |
| node-exporter | Node-level hardware metrics | 9100 |

---

## OTel Collector Configuration

```yaml
# otel-collector/config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  prometheus:
    config:
      scrape_configs:
      - job_name: kube-state-metrics
        static_configs:
        - targets: [kube-state-metrics:8080]
      - job_name: node-exporter
        static_configs:
        - targets: [node-exporter:9100]

processors:
  batch:
    timeout: 5s
  memory_limiter:
    limit_mib: 512

exporters:
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

service:
  pipelines:
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/tempo]
```

---

## Instrumenting Your Application

### Python

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317"))
)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("process-order") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute("order.total", total)
    # your business logic here
```

### Go

```go
exp, _ := otlptracegrpc.New(ctx,
    otlptracegrpc.WithEndpoint("otel-collector:4317"),
    otlptracegrpc.WithInsecure(),
)
tp := trace.NewTracerProvider(trace.WithBatcher(exp))
otel.SetTracerProvider(tp)

tracer := otel.Tracer("my-service")
ctx, span := tracer.Start(ctx, "handle-request")
defer span.End()
```

---

## Kubernetes Manifests

### kube-state-metrics

```yaml
# k8s/kube-state-metrics.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
      - name: kube-state-metrics
        image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.10.1
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 200m
            memory: 256Mi
```

### node-exporter (DaemonSet)

```yaml
# k8s/node-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.7.0
        args:
        - --path.rootfs=/host
        ports:
        - containerPort: 9100
        volumeMounts:
        - name: rootfs
          mountPath: /host
          readOnly: true
      volumes:
      - name: rootfs
        hostPath:
          path: /
```

---

## SLO Definitions

```yaml
# slo/http-availability.yaml
# 99.9% availability SLO — allows ~43 minutes downtime/month
- slo_name: http_availability
  objective: 0.999
  indicator:
    ratio:
      good:
        metric: http_requests_total{status=~"2..|3.."}
      total:
        metric: http_requests_total
  alerting:
    burn_rate_alerts:
    - name: HighErrorBudgetBurn
      short_window: 5m
      long_window: 1h
      burn_rate_threshold: 14.4   # burns 1% budget in 5 min
      severity: critical
```

---

## Alerting Rules

```yaml
# prometheus/rules/alerts.yaml
groups:
- name: slos
  rules:
  - alert: HighErrorRate
    expr: |
      sum(rate(http_requests_total{status=~"5.."}[5m]))
      / sum(rate(http_requests_total[5m])) > 0.01
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Error rate above 1% for 2 minutes"
      runbook: "https://wiki.internal/runbooks/high-error-rate"

  - alert: P99LatencyHigh
    expr: |
      histogram_quantile(0.99,
        sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
      ) > 0.2
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "P99 latency above 200ms on {{ $labels.service }}"
```

---

## Install (Helm)

```bash
# Prometheus + Grafana stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# Loki
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack -n monitoring

# Tempo
helm install tempo grafana/tempo -n monitoring

# OTel Collector (via operator)
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install opentelemetry-operator open-telemetry/opentelemetry-operator -n monitoring
kubectl apply -f otel-collector/collector.yaml
```

---

## Skills Demonstrated

- OpenTelemetry Collector configuration (multi-pipeline: metrics + logs + traces)
- Kubernetes cluster observability (kube-state-metrics + node-exporter)
- Distributed tracing with Tempo and OTel SDK instrumentation (Python + Go examples)
- Log aggregation to Loki
- SLO definition and burn-rate alerting
- Grafana unified dashboard connecting all three signal types
