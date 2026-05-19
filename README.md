# OpenTelemetry Observability

## Overview

This repository provides a comprehensive observability setup using OpenTelemetry (OTel) for collecting metrics, traces, and logs from various sources, including Kubernetes, nodes, and applications. The collected data is then exported to Prometheus, Loki, and Tempo for monitoring and analysis.

## Architecture Diagram

+-------------------+
|                   |
|   Application     |
|                   |
+--------+----------+
         |
         v
+--------+----------+
|                   |
|  OTel SDK         |
|                   |
+--------+----------+
         |
         v
+--------+----------+
|                   |
| OpenTelemetry Collector |
| (receives OTLP)   |
|                   |
+--------+----------+
         |        |        |
         v        v        v
+--------+----------+  +--------+----------+  +--------+----------+
|                   |  |                   |  |                   |
| Prometheus Exporter |  | Loki Exporter     |  | Tempo Exporter    |
|                   |  |                   |  |                   |
+-------------------+  +-------------------+  +-------------------+
         |        |        |
         v        v        v
+--------+----------+  +--------+----------+  +--------+----------+
|                   |  |                   |  |                   |
| Prometheus Server |  | Loki Server       |  | Tempo Server      |
|                   |  |                   |  |                   |
+-------------------+  +-------------------+  +-------------------+
         |        |        |
         v        v        v
+--------+----------+  +--------+----------+  +--------+----------+
|                   |  |                   |  |                   |
| Grafana           |  | Loki UI           |  | Tempo UI          |
|                   |  |                   |  |                   |
+-------------------+  +-------------------+  +-------------------+

## Components

### OpenTelemetry Collector Configuration

The collector is configured to receive OTLP (OpenTelemetry Protocol) data from various sources and export it to Prometheus, Loki, and Tempo.

receivers:
  otlp:
    protocols:
      grpc:

exporters:
  prometheus:
    endpoint: "0.0.0.0:8848"
  loki:
    url: "http://loki:3100/loki/api/v1/push"
  tempo:
    endpoint: "http://tempo:7269"

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [tempo]
    metrics:
      receivers: [otlp]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      exporters: [loki]

### kube-state-metrics

kube-state-metrics is a Prometheus exporter for Kubernetes cluster state.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
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
      containers:
      - name: kube-state-metrics
        image: quay.io/coreos/kube-state-metrics:v2.3.0

### node-exporter

node-exporter is a Prometheus exporter for machine metrics.

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.2.2

### Grafana Unified Dashboard

Grafana is used to create a unified dashboard for metrics, logs, and traces.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:8.3.2

## SLO Definition

- **Availability**: 99.9%
- **Latency (p99)**: <200ms

## Alerting Rules

Alerts are configured to notify via PagerDuty and Slack.

groups:
- name: example
  rules:
  - alert: HighCPUUsage
    expr: node_cpu_seconds_total{mode="user"} > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage on {{ $labels.instance }}"
      description: "{{ $labels.instance }} has been using more than 80% of its CPU for the last 5 minutes."

## Instrumenting an Application with OTel SDK

To instrument your application, follow these steps:

1. **Install OTel SDK**: Add the appropriate OTel SDK to your project.
2. **Initialize TracerProvider**: Initialize a `TracerProvider` and set up resource attributes.
3. **Create Tracers**: Create tracers for different parts of your application.
4. **Instrument Code**: Use the tracers to instrument your code.

Example in Python:

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

span_processor = BatchSpanProcessor(
    OTLPTraceExporter(endpoint="http://localhost:4317")
)
trace.get_tracer_provider().add_span_processor(span_processor)

with tracer.start_as_current_span("my span"):
    # Your code here

## Installation Commands

### OpenTelemetry Collector

docker run -d --name otel-collector \
  -p 4317:4317 \
  -v /path/to/config.yaml:/etc/otel-collector-config.yaml \
  otel/opentelemetry-collector:latest \
  --config=/etc/otel-collector-config.yaml

### kube-state-metrics

kubectl apply -f kube-state-metrics-deployment.yaml

### node-exporter

kubectl apply -f node-exporter-daemonset.yaml

### Grafana

kubectl apply -f grafana-deployment.yaml

## Skills Demonstrated

- OpenTelemetry Collector configuration
- Exporting metrics, traces, and logs to Prometheus, Loki, and Tempo
- kube-state-metrics setup
- node-exporter deployment
- Grafana dashboard creation
- SLO definition and alerting rules
- Instrumenting applications with OTel SDK
- Installation commands for various components

This README provides a comprehensive guide to setting up a production-ready observability stack using OpenTelemetry.