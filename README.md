# opentelemetry-observability

![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-Collector-000000?logo=opentelemetry)
![Prometheus](https://img.shields.io/badge/Prometheus-Metrics-E6522C?logo=prometheus)
![Grafana](https://img.shields.io/badge/Grafana-Dashboards-F46800?logo=grafana)
![Jaeger](https://img.shields.io/badge/Jaeger-Traces-60D0E4)
![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-326CE5?logo=kubernetes)
![Helm](https://img.shields.io/badge/Helm-3.14-0F1689?logo=helm)

> OpenTelemetry Collector on Kubernetes — unified traces, metrics, and logs

---

## What This Does

Full observability stack using OpenTelemetry as the collection backbone. Unified pipeline for distributed traces (Jaeger), metrics (Prometheus + Grafana), and structured logs (Loki). Auto-instrumentation for Python, Go, and Java services. Exemplars link live traces directly to Prometheus metrics.

---

## Architecture

OTel Collector receives spans/metrics/logs from instrumented apps, fans out to Jaeger (traces), Prometheus remote-write (metrics), and Loki (logs). Grafana is the single pane of glass.

## What's Inside

- OTel Collector Helm chart with custom pipelines
- Jaeger all-in-one for trace storage and UI
- Prometheus + Grafana dashboards (latency, error rate, saturation)
- Loki + Promtail for log aggregation
- Auto-instrumentation via OTel Operator for Python/Go/Java
- Exemplars: link Prometheus metrics to live Jaeger traces

## Quick Start

```bash
# Deploy full stack
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install otel-collector open-telemetry/opentelemetry-collector -f values.yaml

# Deploy Grafana + Prometheus
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana -f grafana-values.yaml
```

---

## Skills Demonstrated

`OpenTelemetry Collector` · `Jaeger` · `Prometheus` · `Grafana` · `Loki` · `Helm` · `Kubernetes`
