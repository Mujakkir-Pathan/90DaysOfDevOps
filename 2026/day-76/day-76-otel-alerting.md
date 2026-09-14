# Day 76 -- OpenTelemetry and Alerting

## Task

Today I added the third pillar of observability -- **traces** -- using OpenTelemetry and then configured alerting with Prometheus and Grafana.

By the end of the task, the observability stack covered:

- Metrics with Prometheus
- Logs with Loki and Promtail
- Traces with OpenTelemetry
- Alerting with Prometheus and Grafana

---

## All project

[today's stack](observability-stack/)

---

## All screenshot's

[screenshots of task](screenshots/)

---

# Task 1: Understand OpenTelemetry

## What is OpenTelemetry?

**OpenTelemetry (OTEL)** is a vendor-neutral, open-source framework for generating, collecting, and exporting telemetry data.

It supports three main types of telemetry:

- **Metrics** -- numerical measurements such as CPU and memory usage
- **Logs** -- application and system event records
- **Traces** -- the path of a request through services

OpenTelemetry is **not a backend**. It collects and ships telemetry to backends such as Prometheus, Jaeger, Loki, and Datadog.

## What is the OTEL Collector?

The **OpenTelemetry Collector** is a standalone service that receives, processes, and exports telemetry data.

Its pipeline has three main components:

| Component | Purpose |
|-----------|---------|
| Receivers | Accept telemetry data |
| Processors | Transform or batch telemetry |
| Exporters | Send telemetry to another system |

For this task:

- The **OTLP receiver** accepted telemetry.
- The **batch processor** grouped telemetry before export.
- The **Prometheus exporter** exposed metrics for Prometheus to scrape.
- The **debug exporter** printed traces and logs to the Collector's console.

## What is OTLP?

**OTLP (OpenTelemetry Protocol)** is the standard protocol used to send OpenTelemetry telemetry data.

The Collector was configured to accept:

- **gRPC:** port `4317`
- **HTTP:** port `4318`

## What are distributed traces?

A **trace** follows one request as it travels through multiple services.

Each operation in the trace is represented by a **span**.

A span can contain:

- Trace ID
- Span ID
- Parent span ID
- Start time
- Duration
- Attributes

Example:

```text
User request
     |
     v
API Gateway (span 1)
     |
     v
Auth Service (span 2)
     |
     v
Database (span 3)
```

---

# Task 2: Add the OpenTelemetry Collector

I created the Collector configuration in:

`otel-collector/otel-collector-config.yml`

## OpenTelemetry Architecture

```text
                    OTEL COLLECTOR

Telemetry
   |
   v
[Receivers]
   |
   v
[Processors]
   |
   v
[Exporters]
   |
   +---------------------> Prometheus
   |
   +---------------------> Debug Output
```

## My `otel-collector-config.yml`

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  debug:
    verbosity: detailed

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```

## Configuration Explanation

### Receivers

```yaml
receivers:
  otlp:
```

The Collector receives OTLP telemetry.

```yaml
grpc:
  endpoint: 0.0.0.0:4317

http:
  endpoint: 0.0.0.0:4318
```

This allows applications to send telemetry using either OTLP gRPC or OTLP HTTP.

### Processor

```yaml
processors:
  batch:
```

The batch processor groups telemetry before exporting it. This reduces unnecessary overhead when sending data.

### Exporters

```yaml
prometheus:
  endpoint: "0.0.0.0:8889"
```

The Prometheus exporter exposes Collector metrics on port `8889`, which Prometheus scrapes.

```yaml
debug:
  verbosity: detailed
```

The debug exporter prints detailed telemetry to the Collector's stdout.

### Pipelines

Metrics use:

```text
OTLP -> batch -> Prometheus exporter
```

Traces use:

```text
OTLP -> batch -> debug exporter
```

Logs use:

```text
OTLP -> batch -> debug exporter
```

In a production setup, traces could be sent to a trace backend such as Jaeger or Grafana Tempo instead of only the debug output.

## Verification

The Collector started successfully.

The Collector logs showed:

```text
TSDB started
Loading configuration file
Completed loading of configuration file
Server is ready to receive web requests.
Starting rule manager...
```

The Collector itself also reported that its receivers were ready, including OTLP gRPC on `4317` and OTLP HTTP on `4318`.

The `otel-collector` target appeared as **UP** in Prometheus.

---

# Task 3: Send Test Traces to the Collector

I sent a test OTLP trace to:

```text
http://localhost:4318/v1/traces
```

The request returned:

```json
{"partialSuccess":{}}
```

This confirmed that the Collector accepted the trace request.

## Trace Verification

I searched the Collector logs for `test-span`.

The Collector displayed:

```text
Name : test-span
Kind : Internal
Start time : 2018-12-13 14:51:00 +0000 UTC
End time : 2018-12-13 14:51:01 +0000 UTC
Status code : Unset
Status message :
DroppedAttributesCount: 0
DroppedEventsCount: 0
DroppedLinksCount: 0
Attributes:
     -> http.method: Str(GET)
```

This confirmed that the test span reached the OpenTelemetry Collector.

## Send OTLP Metrics

I also sent the test metric:

```text
test_requests_total
```

The request returned:

```json
{"partialSuccess":{}}
```

I then queried Prometheus using:

```promql
test_requests_total
```

The result was:

```text
test_requests_total{exported_job="my-test-service", instance="otel-collector:8889", job="otel-collector"} 42
```

Therefore, the complete metric flow worked:

```text
curl
  |
  v
OTEL Collector
  |
  v
Prometheus exporter
  |
  v
Prometheus
```

---

# Task 4: Set Up Prometheus Alerting Rules

I created:

`alert-rules.yml`

Prometheus was configured to load this file using:

```yaml
rule_files:
  - /etc/prometheus/alert-rules.yml
```

The rules file was also mounted into the Prometheus container.

## My `alert-rules.yml`

```yaml
groups:
  - name: system-alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage has been above 80% for more than 2 minutes. Current value: {{ $value }}%"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Memory usage is above 85%. Current value: {{ $value }}%"

      - alert: ContainerDown
        expr: absent(container_last_seen{name="notes-app"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container is down"
          description: "The notes-app container has not been seen for over 1 minute"

      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Scrape target is down"
          description: "{{ $labels.job }} target {{ $labels.instance }} is unreachable"

      - alert: HighDiskUsage
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space running low"
          description: "Root filesystem usage is above 90%. Current value: {{ $value }}%"
```

## Explanation of Each Alert

### 1. HighCPUUsage

```text
HighCPUUsage
```

Checks whether average CPU usage is above **80%**.

The condition must remain true for **2 minutes** before the alert fires.

Severity:

```text
warning
```

### 2. HighMemoryUsage

```text
HighMemoryUsage
```

Checks whether memory usage is above **85%**.

The condition must remain true for **2 minutes**.

Severity:

```text
warning
```

### 3. ContainerDown

```text
ContainerDown
```

Uses `absent()` to detect when the `notes-app` container's metric disappears.

The alert fires after the condition remains true for **1 minute**.

Severity:

```text
critical
```

### 4. TargetDown

```text
TargetDown
```

Checks:

```promql
up == 0
```

This detects a Prometheus scrape target that is unreachable.

The condition must remain true for **1 minute**.

Severity:

```text
critical
```

### 5. HighDiskUsage

```text
HighDiskUsage
```

Checks whether root filesystem usage is above **90%**.

The condition must remain true for **5 minutes**.

Severity:

```text
critical
```

## Meaning of the Alert Fields

| Field | Purpose |
|-------|---------|
| `expr` | PromQL condition that triggers the alert |
| `for` | Time the condition must remain true |
| `labels` | Metadata used for classification and routing |
| `annotations` | Human-readable alert information |

The `for` period helps avoid firing alerts for short-lived spikes.

## Verification

All five rules were visible in Prometheus under:

```text
system-alerts
```

The Prometheus Alerts page showed the rules correctly loaded.

I then tested the `notes-app` failure condition by stopping the container:

```bash
docker compose stop notes-app
```

The Prometheus Alerts page showed:

```text
FIRING (2)
INACTIVE (3)
```

Both:

```text
ContainerDown
TargetDown
```

were shown as **FIRING**.

The `notes-app` container was then started again.

---

# Task 5: Set Up Grafana Alerts

## Contact Point

I created the Grafana contact point:

```text
DevOps Team
```

Integration:

```text
Email
```

The contact point was successfully used by the Grafana notification policy.

## Grafana Alert Rule

I created:

```text
High Container Memory
```

The rule used the query:

```promql
container_memory_usage_bytes{name="notes-app"} / 1024 / 1024
```

Condition:

```text
IS ABOVE 100
```

Evaluation:

```text
Every 1m
```

Pending period:

```text
1m
```

The rule was configured with:

```text
severity = warning
```

Notifications were delivered to:

```text
DevOps Team
```

During verification, the Grafana alert instance displayed **NoData**. The rule itself was created successfully and showed the expected query, evaluation interval, label, and notification contact.

## Notification Policy

The default Grafana notification policy was configured with the `DevOps Team` contact point.

A nested policy was also created for:

```text
severity = critical
```

and routed to:

```text
DevOps Team
```

This demonstrated how Grafana can route notifications differently based on alert labels.

## Prometheus Alerts vs Grafana Alerts

| Prometheus Alerts | Grafana Alerts |
|-------------------|-----------------|
| Rules are evaluated by Prometheus | Rules are evaluated by Grafana |
| Naturally integrated with PromQL and Prometheus metrics | Can evaluate data from Grafana-supported data sources |
| Good for infrastructure and metric-based conditions close to Prometheus | Good for centralized visualization and notification management |
| Usually paired with Alertmanager for production notification routing | Has built-in contact points and notification policies |
| Best when the alert logic is tightly coupled to Prometheus metrics | Useful when alerting across multiple data sources |

### When to use each

I would use **Prometheus alerting** for infrastructure and service conditions based directly on Prometheus metrics, such as CPU, memory, disk, and target availability.

I would use **Grafana alerting** when I want a centralized alerting layer with contact points, notification policies, and alerts across multiple data sources.

---

# Task 6: Review the Full Stack Architecture

The final observability stack covers all three pillars:

```text
                    METRICS PIPELINE

[Node Exporter] ----------------> [Prometheus] -----> [Grafana Dashboards]
[cAdvisor] ---------------------> [Prometheus] -----> [Grafana Dashboards]
[OTEL Collector:8889] ---------> [Prometheus] -----> [Grafana Dashboards]
                                         |
                                         +----------> [Prometheus Alert Rules]
                                                          |
                                                          v
                                                   [Notifications]


                    LOGS PIPELINE

[Docker Containers] -> [Promtail] -> [Loki] -> [Grafana Explore/Dashboards]


                    TRACES PIPELINE

[curl / App OTLP] -----> [OTEL Collector] -----> [Debug Output]
                                  |
                                  |
                                  +--------------> Future: Jaeger / Tempo
```

## Complete Data Flow

```text
                         OBSERVABILITY STACK

  METRICS
  --------
  Node Exporter --------\
  cAdvisor --------------> Prometheus -------> Grafana
  OTEL Collector:8889 --/      |
                               |
                               +-------------> Alert Rules
                                                   |
                                                   v
                                             Notifications


  LOGS
  ----
  Docker Containers -> Promtail -> Loki -> Grafana


  TRACES
  ------
  Application / curl -> OTEL Collector -> Debug Output
                                         |
                                         +-> Future Jaeger / Tempo
```

## Services Running

| Service | Port | Purpose |
|---------|------|---------|
| Prometheus | 9090 | Metrics storage and querying |
| Node Exporter | 9100 | Host system metrics |
| cAdvisor | 8080 | Container metrics |
| Grafana | 3000 | Visualization and alerting |
| Loki | 3100 | Log storage |
| Promtail | 9080 | Log collection agent |
| OTEL Collector | 4317/4318/8889 | Telemetry collection |
| Notes App | 8000 | Sample application |

## Final Verification

I ran:

```bash
docker compose ps
```

The final stack showed all 8 services running:

```text
NAME             SERVICE          STATUS
cadvisor         cadvisor         Up ... (healthy)
grafana          grafana          Up ...
loki             loki             Up ...
node-exporter    node-exporter    Up ...
notes-app        notes-app        Up ...
otel-collector   otel-collector   Up ...
prometheus       prometheus       Up ...
promtail         promtail         Up ...
```

The required eight containers were running:

1. Prometheus
2. Node Exporter
3. cAdvisor
4. Grafana
5. Loki
6. Promtail
7. OTEL Collector
8. Notes App

---

# Final Takeaways

The biggest takeaway from Day 76 was understanding how the three observability pillars fit together:

```text
Metrics -> What is happening?
Logs    -> Why did it happen?
Traces  -> Where did the request spend time / fail?
```

OpenTelemetry provided a standard way to collect telemetry, while Prometheus and Grafana provided metric storage, visualization, and alerting.

I also learned that alerting is not only about creating a condition. The complete alerting flow includes:

```text
Metric
  |
  v
Alert Rule
  |
  v
Alert State
  |
  v
Label / Severity
  |
  v
Notification Policy
  |
  v
Contact Point
  |
  v
Notification
```

Day 76 completed the transition from simply **observing the system** to also **being notified when something goes wrong**.
