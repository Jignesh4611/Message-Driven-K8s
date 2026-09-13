# OpenTelemetry Monitoring with Prometheus and Grafana

This project demonstrates how to collect application metrics using
**OpenTelemetry**, expose them in a **Prometheus-compatible format**,
scrape them with **Prometheus**, and visualize them in **Grafana**.

Source For Application Code : https://github.com/Jignesh4611/Message-Driven
For video Understading visit : https://x.com/JIGNESH4611

## Architecture

``` text
Node.js Application
        |
        | OTLP/HTTP
        | POST /v1/metrics
        | :4318
        v
OpenTelemetry Collector
        |
        | OTLP receiver
        | batch processor
        v
Prometheus Exporter
        |
        | :8889/metrics
        v
Kubernetes Service
        |
        v
ServiceMonitor
        |
        v
Prometheus
        |
        v
Grafana
```

## Components

-   **Application** - Generates application metrics.
-   **OpenTelemetry Collector** - Receives, processes, and exports
    telemetry.
-   **Prometheus** - Scrapes and stores metrics.
-   **ServiceMonitor** - Configures Prometheus to discover and scrape
    the OpenTelemetry Collector.
-   **Grafana** - Visualizes metrics stored in Prometheus.

## OpenTelemetry Collector Configuration

The Collector runs as a Kubernetes Deployment using the OpenTelemetry
Collector Contrib image.

``` yaml
mode: deployment

image:
  repository: otel/opentelemetry-collector-contrib
```

### OTLP Receiver

The Collector accepts OTLP telemetry over gRPC and HTTP:

``` yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
```

The important OTLP/HTTP metrics endpoint is:

``` text
http://<collector-service>:4318/v1/metrics
```

`/v1/metrics` is the standard OTLP/HTTP metrics endpoint. It is provided
by the OTLP HTTP receiver; it does not need to be manually declared in
the configuration.

### Batch Processor

``` yaml
processors:
  batch: {}
```

The batch processor groups telemetry before it is exported.

### Prometheus Exporter

``` yaml
exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
```

The Prometheus exporter exposes the collected metrics in Prometheus
format at:

``` text
http://<collector>:8889/metrics
```

This endpoint is different from the OTLP/HTTP endpoint:

``` text
4318/v1/metrics  -> application sends metrics to the Collector
8889/metrics     -> Prometheus scrapes metrics from the Collector
```

## Metrics Pipeline

The Collector connects the receiver, processor, and exporter through a
pipeline:

``` yaml
service:
  pipelines:
    metrics:
      receivers:
        - otlp
      processors:
        - batch
      exporters:
        - prometheus
```

The flow is:

``` text
OTLP Receiver
     |
     v
Batch Processor
     |
     v
Prometheus Exporter
```

The `prometheus` exporter is defined under `exporters` and referenced by
name inside the metrics pipeline.

## Kubernetes Service Ports

The Collector Service exposes:

``` yaml
ports:
  otlp:
    enabled: true
    containerPort: 4317
    servicePort: 4317
    protocol: TCP

  otlp-http:
    enabled: true
    containerPort: 4318
    servicePort: 4318
    protocol: TCP

  metrics:
    enabled: true
    containerPort: 8889
    servicePort: 8889
    protocol: TCP
```

Port meanings:

  Port     Purpose
  -------- --------------------------------------
  `4317`   OTLP over gRPC
  `4318`   OTLP over HTTP
  `8889`   Prometheus exporter metrics endpoint

## ServiceMonitor

Prometheus uses a `ServiceMonitor` to discover the Collector's metrics
endpoint.

Example:

``` yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: otel-servicemonitor
  namespace: monitoring
  labels:
    release: monitoring

spec:
  selector:
    matchLabels:
      app.kubernetes.io/instance: my-opentelemetry
      app.kubernetes.io/name: opentelemetry-collector

  namespaceSelector:
    matchNames:
      - monitoring

  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
```

### Why `release: monitoring` is required

The Prometheus instance from `kube-prometheus-stack` is configured with:

``` yaml
serviceMonitorSelector:
  matchLabels:
    release: monitoring
```

Therefore, Prometheus only discovers ServiceMonitors having:

``` yaml
release: monitoring
```

Without this label, the `otel-servicemonitor` exists in Kubernetes but
Prometheus does not select it.

### Service Selector

The ServiceMonitor also needs to select the correct Collector Service.

The Collector Service has:

``` text
app.kubernetes.io/instance=my-opentelemetry
app.kubernetes.io/name=opentelemetry-collector
```

Therefore the ServiceMonitor uses the same labels:

``` yaml
selector:
  matchLabels:
    app.kubernetes.io/instance: my-opentelemetry
    app.kubernetes.io/name: opentelemetry-collector
```

## Verify the Deployment

Check the Collector pod:

``` bash
kubectl get pods -n monitoring
```

Check the Collector Service:

``` bash
kubectl get svc -n monitoring
```

Check ServiceMonitor:

``` bash
kubectl get servicemonitor -n monitoring
```

Check its labels:

``` bash
kubectl get servicemonitor otel-servicemonitor \
  -n monitoring \
  --show-labels
```

## Verify Prometheus Target

Open Prometheus and go to:

``` text
Status -> Targets
```

The expected target is similar to:

``` text
serviceMonitor/monitoring/otel-servicemonitor/0
```

It should show:

``` text
1/1 up
```

The endpoint should look similar to:

``` text
http://<pod-ip>:8889/metrics
```

## Verify Metrics in Prometheus

Check whether Prometheus is scraping the Collector:

``` promql
up{job="my-opentelemetry-opentelemetry-collector"}
```

Expected result:

``` text
1
```

Check the number of samples being scraped:

``` promql
scrape_samples_scraped{job="my-opentelemetry-opentelemetry-collector"}
```

List metric names:

``` promql
count by (__name__) ({job="my-opentelemetry-opentelemetry-collector"})
```

Example application metrics may include:

``` text
http_client_request_duration_seconds
db_client_connection_count
nodejs_eventloop_delay_max_seconds
```

## Grafana

Grafana is provided by `kube-prometheus-stack`.

Check the Grafana pod:

``` bash
kubectl get pods -n monitoring
```

Access Grafana locally:

``` bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Open:

``` text
http://localhost:3000
```

Get the default Grafana admin password:

``` bash
kubectl get secret monitoring-grafana \
  -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 -d
```

Default username:

``` text
admin
```

In Grafana, use **Explore** with the Prometheus data source and query
metrics such as:

``` promql
nodejs_eventloop_delay_max_seconds
```

## Troubleshooting

### ServiceMonitor does not appear in Prometheus

Check:

``` bash
kubectl get servicemonitor -n monitoring
```

Then verify the required label:

``` bash
kubectl get servicemonitor otel-servicemonitor \
  -n monitoring \
  --show-labels
```

It must contain:

``` text
release=monitoring
```

### ServiceMonitor exists but target is missing

Check the Collector Service labels:

``` bash
kubectl get svc -n monitoring --show-labels
```

Make sure they match the ServiceMonitor selector.

### Target is `UNKNOWN`

Check that the Collector exposes port `8889`:

``` bash
kubectl get svc -n monitoring
```

Check the Collector logs:

``` bash
kubectl logs -n monitoring <otel-collector-pod>
```

You can also test the endpoint from inside the cluster:

``` bash
kubectl run curl-test \
  --image=curlimages/curl \
  --restart=Never \
  -n monitoring \
  --rm -it \
  -- curl -v http://my-opentelemetry-opentelemetry-collector:8889/metrics
```

## Important Endpoint Difference

There are two different metrics endpoints in this architecture:

``` text
OTLP/HTTP:
:4318/v1/metrics
```

Used by the application to **send metrics to OpenTelemetry Collector**.

``` text
Prometheus:
:8889/metrics
```

Used by Prometheus to **scrape metrics from OpenTelemetry Collector**.

They are not the same endpoint.

## Final Data Flow

``` text
                    SEND
Node.js App ---------------------> OTel Collector
             :4318/v1/metrics          |
                                        |
                                  batch processor
                                        |
                                        v
                                Prometheus Exporter
                                        |
                                    :8889/metrics
                                        |
                                        v
                                  ServiceMonitor
                                        |
                                        v
                                    Prometheus
                                        |
                                        v
                                     Grafana
```

This setup provides a complete metrics pipeline from application
instrumentation to visualization.
