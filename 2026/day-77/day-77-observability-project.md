# Day 77 -- Observability Project: Full Stack with Docker Compose

## Task

Four days of observability work were integrated into one complete stack
using Docker Compose. The stack combined Prometheus, Node Exporter,
cAdvisor, Grafana, Loki, Promtail, OpenTelemetry Collector, and the
Notes application.

The complete 8-service observability stack was cloned, launched,
validated across metrics, logs, and traces, and used to build a unified
Grafana dashboard.

------------------------------------------------------------------------

## All project

[today's stack](observability-for-devops/)

---

## All screenshot's

[screenshots of task](screenshots/)

------------------------------------------------------------------------

## Challenge Tasks

### Task 1: Clone and Launch the Reference Stack

The reference repository was cloned and the project structure was
inspected.

The repository contained the Docker Compose configuration, Prometheus
configuration, Grafana provisioning, Loki configuration, Promtail
configuration, OpenTelemetry Collector configuration, and Notes
application.

The stack was launched with:

``` bash
docker compose up -d
```

The services were verified with:

``` bash
docker compose ps
```

All 8 services were running:

  Service          Port        Status
  ---------------- ----------- ---------
  Prometheus       9090        Running
  Node Exporter    9100        Running
  cAdvisor         8080        Healthy
  Grafana          3000        Running
  Loki             3100        Running
  Promtail         9080        Running
  OTEL Collector   4317/4318   Running
  Notes App        8000        Running

Grafana was accessed using the provided `admin/admin` credentials.

------------------------------------------------------------------------

### Task 2: Validate the Metrics Pipeline

Prometheus was opened at `/targets` and all four required scrape jobs
were verified as UP:

-   `prometheus` -- self-monitoring
-   `node-exporter` -- host metrics
-   `docker` / `cadvisor` -- container metrics
-   `otel-collector` -- OTLP metrics

The following PromQL queries were executed successfully:

``` promql
up
```

``` promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

``` promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

``` promql
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
```

``` promql
topk(3, container_memory_usage_bytes{name!=""})
```

These queries validated target health, host CPU usage, memory usage,
container CPU usage, and the top memory-consuming containers.

The reference Prometheus configuration was used as part of the
integrated stack. A complete line-by-line comparison with the previous
Day 73-76 configuration was not performed.

------------------------------------------------------------------------

### Task 3: Validate the Logs Pipeline

Traffic was generated against the Notes application so that application
logs were available:

``` bash
for i in $(seq 1 50); do
  curl -s http://localhost:8000 > /dev/null
  curl -s http://localhost:8000/api/ > /dev/null
done
```

Grafana Explore was opened with Loki selected as the datasource.

The following LogQL queries were executed:

``` logql
{job="docker"}
```

``` logql
{container_name="notes-app"}
```

``` logql
{job="docker"} |= "error"
```

``` logql
{container_name="notes-app"} |= "GET"
```

``` logql
sum by (container_name) (rate({job="docker"}[5m]))
```

The Promtail configuration was verified as:

``` yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: container_name
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: stream
      - action: replace
        replacement: docker
        target_label: job
    pipeline_stages:
      - docker: {}
```

The configuration used Docker service discovery through the Docker
socket and sent container logs to Loki.

The Promtail service itself was running, while port `9080` was kept
internal in the reference Compose setup. Therefore, the local `/targets`
endpoint was not exposed on the host.

A complete configuration comparison with Day 75 was not performed.

------------------------------------------------------------------------

### Task 4: Validate the Traces Pipeline

An OTLP trace was sent to the OpenTelemetry Collector through the HTTP
receiver.

The collector debug output was checked with:

``` bash
docker logs otel-collector 2>&1 | grep -A 20 "GET /api/notes"
```

The collector output showed the following spans:

``` text
Name           : GET /api/notes
Kind           : Server
Status code    : Ok
```

The HTTP span contained attributes including:

``` text
http.method: GET
http.route: /api/notes
http.status_code: 200
```

A second span was received:

``` text
Name           : SELECT notes FROM database
Kind           : Client
```

Both spans used the same Trace ID:

``` text
aaaabbbbccccdddd1111222233334444
```

The database span used the HTTP span ID as its parent:

``` text
Parent ID      : 1111222233334444
```

This verified the parent-child relationship between the HTTP request
span and database query span.

The trace also contained start and end timing information.

A complete configuration comparison with the Day 76 OTEL Collector
configuration was not performed.

------------------------------------------------------------------------

### Task 5: Build a Unified "Production Overview" Dashboard

A unified Grafana dashboard was created containing system health,
container metrics, application logs, and service overview panels.

#### Row 1 -- System Health

  -------------------------------------------------------------------------------------------------------------------------------------------------------
  Panel                    Type                  Query
  ------------------------ --------------------- --------------------------------------------------------------------------------------------------------
  CPU Usage                Gauge                 `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`

  Memory Usage             Gauge                 `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100`

  Disk Usage               Gauge                 `(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100`

  Targets Up               Stat                  `sum(up) / count(up)`
  -------------------------------------------------------------------------------------------------------------------------------------------------------

The CPU, memory, and disk panels displayed system resource usage.

The dashboard displayed the Prometheus target status information.

#### Row 2 -- Container Metrics

  --------------------------------------------------------------------------------------------------------------
  Panel                    Type                  Query
  ------------------------ --------------------- ---------------------------------------------------------------
  Container CPU            Time series           `rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100`

  Container Memory         Bar chart             `container_memory_usage_bytes{name!=""} / 1024 / 1024`

  Container Count          Stat                  `count(container_last_seen{name!=""})`
  --------------------------------------------------------------------------------------------------------------

The container panels displayed CPU usage, memory usage, and the number
of containers monitored by cAdvisor.

#### Row 3 -- Application Logs

  -----------------------------------------------------------------------------------------------------
  Panel                    Type                  Query
  ------------------------ --------------------- ------------------------------------------------------
  App Logs                 Logs                  `{container_name="notes-app"}`

  Error Rate               Time series           `sum(rate({job="docker"} |= "error" [5m]))`

  Log Volume               Time series           `sum by (container_name) (rate({job="docker"}[5m]))`
  -----------------------------------------------------------------------------------------------------

The application logs and LogQL-based panels were added using Loki.

#### Row 4 -- Service Overview

  -------------------------------------------------------------------------------------------------------------
  Panel                    Type                  Query
  ------------------------ --------------------- --------------------------------------------------------------
  Prometheus Scrape        Time series           `prometheus_target_interval_length_seconds{quantile="0.99"}`
  Duration                                       

  OTEL Metrics Received    Stat                  `otelcol_receiver_accepted_metric_points`
  -------------------------------------------------------------------------------------------------------------

The OTEL Metrics Received panel showed `No data`, because the metric was
not available in the configured Prometheus data.

The dashboard was created as `Production Overview` and configured with
the observability panels.

------------------------------------------------------------------------

## Task 6: Compare Your Stack with the Reference and Document

The reference repository structure was inspected. The complete previous
Day 73-76 configuration files were not available for a reliable
line-by-line comparison, so differences were not assumed or invented.

  ---------------------------------------------------------------------------------------------
  Component                     Your Version      Reference Repo            Differences
  ----------------------------- ----------------- ------------------------- -------------------
  `prometheus.yml`              Day 73-74         Root directory            Reference
                                configuration                               configuration was
                                                                            used in the
                                                                            integrated stack;
                                                                            detailed comparison
                                                                            was not performed

  `loki-config.yml`             Day 75            `loki/` directory         Detailed comparison
                                configuration                               was not performed

  `promtail-config.yml`         Day 75            `promtail/` directory     Reference Docker
                                configuration                               service-discovery
                                                                            configuration was
                                                                            used; detailed
                                                                            comparison was not
                                                                            performed

  `otel-collector-config.yml`   Day 76            `otel-collector/`         Reference OTLP
                                configuration     directory                 configuration was
                                                                            used; detailed
                                                                            comparison was not
                                                                            performed

  `datasources.yml`             Day 74            `grafana/provisioning/`   Reference
                                configuration                               provisioned
                                                                            datasources were
                                                                            used

  `docker-compose.yml`          Days 73-76        Root directory            Reference Compose
                                                                            file integrated all
                                                                            8 services
  ---------------------------------------------------------------------------------------------

### Observability Concept Mapping

  Day   What You Built
  ----- -----------------------------------------------
  73    Prometheus, PromQL, metrics fundamentals
  74    Node Exporter, cAdvisor, Grafana dashboards
  75    Loki, Promtail, LogQL, log-metric correlation
  76    OTEL Collector, traces, alerting rules
  77    Full stack integration and unified dashboard

### What Would Be Added for Production Readiness

The following components and improvements would be considered before
using this architecture in production:

-   **Alertmanager** for routing alerts to Slack, PagerDuty, or other
    notification systems.
-   **Grafana Tempo** for persistent trace storage instead of relying on
    the OTEL debug exporter.
-   **HTTPS/TLS** for securing communication between observability
    components.
-   **Authentication and authorization** for Grafana, Prometheus, Loki,
    and other exposed endpoints.
-   **Log retention and storage limits** to control storage consumption.
-   **High availability** using multiple Prometheus and Loki replicas.
-   Persistent and appropriately sized storage for production telemetry
    data.
-   Backup and recovery procedures for important observability data.

### Comparison with Managed Solutions

This Docker Compose stack provides direct control over the observability
components and their configurations.

Managed platforms such as **Datadog**, **New Relic**, and **AWS
CloudWatch** provide hosted observability services that reduce the
infrastructure and maintenance required for running the monitoring
backend.

  --------------------------------------------------------------------------------
  Area             Self-Hosted Stack            Managed Observability
  ---------------- ---------------------------- ----------------------------------
  Infrastructure   Managed by the team          Provider managed

  Configuration    Full control                 Provider-defined capabilities plus
                                                configuration

  Scaling          Must be designed and         Scaling is largely handled by
                   operated                     provider

  Storage          Team manages storage and     Provider-managed storage options
                   retention                    

  Maintenance      Team maintains components    Provider maintains platform

  Customization    High control over components Depends on provider capabilities

  Cost model       Infrastructure and           Service/usage-based pricing
                   operational costs            
  --------------------------------------------------------------------------------

The choice depends on operational requirements, required control,
existing cloud infrastructure, telemetry volume, security requirements,
and team responsibilities.

------------------------------------------------------------------------

## Architecture

``` mermaid
flowchart LR

    APP[Notes App]

    subgraph Metrics
        NE[Node Exporter]
        CAD[cAdvisor]
        PROM[Prometheus]
    end

    subgraph Logs
        PT[Promtail]
        LOKI[Loki]
    end

    subgraph Traces
        OTEL[OpenTelemetry Collector]
    end

    GRAF[Grafana]

    APP -->|Application metrics| PROM
    NE -->|Host metrics| PROM
    CAD -->|Container metrics| PROM
    OTEL -->|OTLP metrics| PROM

    APP -->|Container logs| PT
    PT -->|Log streams| LOKI

    APP -->|OTLP traces| OTEL

    PROM -->|Metrics| GRAF
    LOKI -->|Logs| GRAF
    OTEL -->|Trace processing| GRAF
```

### Data Flow

**Metrics:**

``` text
Node Exporter ──────┐
                    │
cAdvisor ───────────┼──> Prometheus ──> Grafana
                    │
OTEL Collector ─────┘
```

**Logs:**

``` text
Notes App / Containers
        │
        ▼
    Promtail
        │
        ▼
      Loki
        │
        ▼
     Grafana
```

**Traces:**

``` text
Notes App
    │
    │ OTLP
    ▼
OTEL Collector
    │
    ▼
Trace processing/export
```

------------------------------------------------------------------------

## Cleanup

After completing the observability exploration, the stack was cleaned up
using:

``` bash
docker compose down -v
```

The cleanup successfully removed:

-   All 8 containers
-   Prometheus data volume
-   Grafana data volume
-   Loki data volume
-   Docker Compose monitoring network

The final verification was:

``` bash
docker compose ps
```

Result:

``` text
NAME      IMAGE     COMMAND   SERVICE   CREATED   STATUS    PORTS
```

No services were running after cleanup.

------------------------------------------------------------------------

## Key Takeaways from the 5-Day Observability Block

1.  **Prometheus and PromQL** provided the foundation for collecting and
    querying time-series metrics.
2.  **Node Exporter and cAdvisor** extended monitoring from Prometheus
    itself to host and container resources.
3.  **Grafana** provided a single interface for visualizing system and
    application telemetry.
4.  **Loki and Promtail** added centralized container log collection and
    LogQL-based log analysis.
5.  **OpenTelemetry Collector** provided a standardized telemetry
    pipeline for receiving and processing traces and metrics.
6.  **Correlation of metrics, logs, and traces** provided a broader view
    of application and infrastructure behavior.
7.  **Docker Compose** brought the complete observability architecture
    together as one reproducible stack.
8.  The final dashboard demonstrated how multiple telemetry sources can
    be presented through a single observability interface.
9.  Production deployment would require additional considerations such
    as alert routing, persistent trace storage, security, retention,
    backups, and high availability.

------------------------------------------------------------------------

## Reflection

Day 77 brought together the concepts learned throughout the previous
four observability days into one complete stack.

The five-day progression moved from collecting and querying metrics, to
monitoring hosts and containers, to centralized logging, and finally to
distributed tracing and alerting. The final integration demonstrated how
these individual components work together as a complete observability
platform.
