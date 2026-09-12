# Day 73 -- Introduction to Observability and Prometheus

## Task
I learned why observability is important after building infrastructure with Terraform, configuring servers with Ansible, and containerizing applications with Docker. I learned how metrics, logs, and traces help understand whether systems are healthy and why failures happen.

---

## All project

[whole project](observability-stack/)

---

## All screenshot's

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Understand Observability

#### 1. What is observability? How is it different from traditional monitoring?

**Monitoring** tells me **when** something is wrong by using alerts and thresholds.

**Observability** helps me understand **why** something is wrong by allowing me to explore, query, and correlate system information.

#### 2. The three pillars of observability

In my own words:

- **Metrics** are numbers that show what is happening in a system over time. For example, CPU usage, request count, memory usage, and error rate.
- **Logs** are timestamped records that explain events happening inside an application or system. They are useful for finding errors and understanding what happened.
- **Traces** follow a single request as it moves through different services. They help identify where a request became slow or failed.

My simple understanding is:

```text
Metrics → WHAT is happening?
Logs    → WHY did it happen?
Traces  → WHERE did it happen?
```

#### 3. Why do DevOps engineers need all three?

I need all three because each one provides different information:

- Metrics can show a high error rate on `/api/users`.
- Logs can show the database timeout or stack trace that caused the error.
- Traces can show which service in the request path was slow or failed.

Together, they provide a better picture of system health and help troubleshoot problems faster.

#### 4. Architecture

The observability architecture I will build over Days 73-77 is:

```text
[Your App] --> metrics --> [Prometheus] --> [Grafana Dashboards]
[Your App] --> logs    --> [Promtail]   --> [Loki] --> [Grafana]
[Your App] --> traces  --> [OTEL Collector] --> [Grafana/Debug]
[Host]     --> metrics --> [Node Exporter] --> [Prometheus]
[Docker]   --> metrics --> [cAdvisor] --> [Prometheus]
```

This shows how different types of observability data move from applications and infrastructure into the tools that collect and visualize them.

---

### Task 2: Set Up Prometheus with Docker

I created the project directory:

```bash
mkdir observability-stack && cd observability-stack
```

Project location:

```text
/home/ubuntu/observability-stack
```

I created `prometheus.yml` with a 15-second scrape interval:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

I created `docker-compose.yml` to run Prometheus:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped

volumes:
  prometheus_data:
```

I started Prometheus with:

```bash
docker compose up -d
```

Prometheus was running in Docker and the web UI was accessible on port `9090`.

**Verify:** Go to Status > Targets. You should see one target (`prometheus`) with state `UP`.

**Result:** The `prometheus` target was shown as `UP`.

---

### Task 3: Understand Prometheus Concepts

I explored the Prometheus UI and learned these concepts:

1. **Scrape targets** are endpoints that Prometheus pulls metrics from at regular intervals. Prometheus uses a pull-based model.
2. **Counter** only increases, such as total requests or total errors.
3. **Gauge** can increase and decrease, such as current CPU usage or memory usage.
4. **Histogram** stores observations in buckets to show the distribution of values such as request duration.
5. **Summary** calculates summary statistics and percentiles on the client side.
6. **Labels** are key-value pairs that add dimensions to metrics.
7. **Time series** are identified by a metric name together with a unique set of label values.

#### Queries and results

**How many metrics is Prometheus collecting about itself?**

```promql
count({__name__=~".+"})
```

Result:

```text
909
```

This query returned `909` matching metric series at that time.

**How much memory is Prometheus using?**

```promql
process_resident_memory_bytes
```

Result:

```text
95461376 bytes
```

I also converted this value to approximately:

```text
91.48 MB
```

**Total HTTP requests to the Prometheus server**

```promql
prometheus_http_requests_total
```

This returned the HTTP request counters broken down by labels such as `code`, `handler`, `instance`, and `job`.

**Break it down by handler**

```promql
prometheus_http_requests_total{handler="/api/v1/query"}
```

Result observed during the exercise:

```text
code="200", handler="/api/v1/query" = 10
```

The exact counter value changed as more queries were made because it is a counter.

#### Counter vs Gauge

A **counter** is a value that only increases during normal operation. It is useful for counting things that have happened.

Real-world example:

```text
HTTP requests served = 1000 → 1001 → 1002 → ...
```

A **gauge** can increase or decrease depending on the current state.

Real-world example:

```text
Memory usage = 500 MB → 700 MB → 450 MB → ...
```

---

### Task 4: Learn PromQL Basics

I practiced the following PromQL queries.

#### 1. Instant vector

```promql
up
```

Result:

```text
up{instance="localhost:9090", job="prometheus"} = 1
```

A value of `1` means the target is up.

#### 2. Range vector

```promql
prometheus_http_requests_total[5m]
```

This returned the values of the HTTP request counter over the previous five minutes.

#### 3. Rate

```promql
rate(prometheus_http_requests_total[5m])
```

This converted the request counter into a per-second rate over five minutes.

An observed result included approximately:

```text
/metrics = 0.0667 requests/second
```

#### 4. Aggregation

```promql
sum(rate(prometheus_http_requests_total[5m]))
```

Observed result:

```text
0.115649216374269
```

This represented the combined request rate across the matching label combinations.

#### 5. Filter by label

```promql
prometheus_http_requests_total{code="200"}
```

This returned `63` result series during the exercise, filtered to HTTP status code `200`.

```promql
prometheus_http_requests_total{code!="200"}
```

Observed non-200 counters included:

```text
302 / = 1
400 /api/v1/query_range = 6
400 /api/v1/query = 1
```

#### 6. Arithmetic

```promql
process_resident_memory_bytes / 1024 / 1024
```

Result:

```text
91.48046875 MB
```

#### 7. Top-K

```promql
topk(5, prometheus_http_requests_total)
```

The top results observed were:

```text
/metrics                    = 2398
/api/v1/notifications/live  = 149
/api/v1/query               = 42
/api/v1/metadata            = 16
/api/v1/label/:name/values  = 13
```

#### PromQL exercise

I wrote the query for the per-second rate of non-200 requests:

```promql
rate(prometheus_http_requests_total{code!="200"}[5m])
```

The result was `0` for the observed non-200 series during the measured five-minute window. The historical counters still contained non-200 requests, but there were no non-200 requests occurring at a measurable rate in that window.

---

### Task 5: Add a Sample Application as a Scrape Target

I added the Notes App to `docker-compose.yml`:

```yaml
notes-app:
  image: trainwithshubham/notes-app:latest
  container_name: notes-app
  ports:
    - "8000:8000"
  restart: unless-stopped
```

I added the Notes App as a Prometheus scrape target:

```yaml
- job_name: "notes-app"
  static_configs:
    - targets: ["notes-app:8000"]
```

Initially, Prometheus reported the target as `DOWN` because:

```text
http://notes-app:8000/metrics
```

returned:

```text
404 Not Found
```

I fixed the application so that the `/metrics` endpoint was available.

I verified the endpoint with:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/metrics
```

Result:

```text
200
```

After the fix, Prometheus showed both targets as `UP`.

I generated traffic to the application:

```bash
curl http://localhost:8000
curl http://localhost:8000
curl http://localhost:8000
```

The application was successfully reachable and Prometheus successfully scraped its metrics.

**Screenshot: Prometheus Targets page showing all targets UP**

![Prometheus Targets page](prometheus-targets.png)

The Targets page showed:

```text
notes-app   → UP
prometheus  → UP
```

**Note:** Not all applications expose Prometheus metrics natively. An application must expose a Prometheus-compatible metrics endpoint for Prometheus to scrape it directly. In later days, exporters such as Node Exporter and cAdvisor can provide metrics for systems that do not expose them directly.

---

### Task 6: Explore Data Retention and Storage

#### 1. Check Prometheus disk usage

I checked the Prometheus storage directory:

```bash
docker exec prometheus du -sh /prometheus
```

Result:

```text
2.7M    /prometheus
```

Prometheus was using approximately `2.7 MB` of storage at the time of the check.

#### 2. Prometheus TSDB retention

Prometheus stores metrics in its local time-series database (TSDB).

The default retention period is **15 days**.

Retention can be changed with:

```yaml
command:
  - '--config.file=/etc/prometheus/prometheus.yml'
  - '--storage.tsdb.retention.time=30d'
  - '--storage.tsdb.retention.size=1GB'
```

Here:

- `30d` means data can be retained for up to 30 days.
- `1GB` limits the storage size to 1 GB.

When a configured retention limit is exceeded, Prometheus removes older data so that newer data can continue to be stored.

#### 3. TSDB Status

I checked **Status > TSDB Status** in the Prometheus UI.

The TSDB Head Status showed:

```text
Number of Series       = 1001
Number of Chunks       = 1001
Number of Label Pairs  = 622
```

The current time range shown was:

```text
Current Min Time = 2026-09-12T03:31:38Z
Current Max Time = 2026-09-12T04:38:53Z
```

#### Volume mount

The Docker Compose configuration uses:

```yaml
volumes:
  - prometheus_data:/prometheus
```

The volume is important because Prometheus stores its TSDB data under `/prometheus`. Using a Docker volume keeps that data separate from the container lifecycle, so the stored metrics can persist when the Prometheus container is stopped, removed, or recreated.

# Additional Documentation

### The Three Pillars of Observability in My Own Words

1. **Metrics** -- Metrics are numbers collected over time that help me understand the current health and performance of a system. Examples include CPU usage, memory usage, request count, and error rate.

2. **Logs** -- Logs are records of events generated by applications and systems. They help me understand what happened and are useful for finding errors, warnings, and detailed failure information.

3. **Traces** -- Traces follow a single request as it travels through different services. They help me understand where a request became slow or where a failure occurred.

In simple words:

```text
Metrics → WHAT is happening?
Logs    → WHY did it happen?
Traces  → WHERE did it happen?
```

### Five PromQL Queries I Ran and What They Returned

#### 1. Check whether the Prometheus target is up

```promql
up
```

**Result:**

```text
up{instance="localhost:9090", job="prometheus"} = 1
```

`1` means the Prometheus target is UP.

#### 2. Check Prometheus memory usage

```promql
process_resident_memory_bytes
```

**Result:**

```text
95461376 bytes
```

This was approximately:

```text
91.48 MB
```

#### 3. Calculate the HTTP request rate

```promql
rate(prometheus_http_requests_total[5m])
```

**Result:**

The query returned the per-second request rate for different Prometheus HTTP endpoints.

One observed result was approximately:

```text
/metrics = 0.0667 requests/second
```

#### 4. Calculate the total HTTP request rate

```promql
sum(rate(prometheus_http_requests_total[5m]))
```

**Result:**

```text
0.115649216374269
```

This represented the combined HTTP request rate across the matching label combinations.

#### 5. Find the top five HTTP request counters

```promql
topk(5, prometheus_http_requests_total)
```

**Result:**

```text
/metrics                    = 2398
/api/v1/notifications/live  = 149
/api/v1/query               = 42
/api/v1/metadata            = 16
/api/v1/label/:name/values  = 13
```

This returned the five HTTP request series with the highest counter values at that time.

### Counter vs Gauge

A **Counter** is a metric that normally only increases. It is useful for counting events that have happened.

**Real-world example:**

```text
HTTP requests:
1000 → 1001 → 1002 → 1003
```

The total number of HTTP requests keeps increasing.

A **Gauge** represents a value that can increase or decrease depending on the current state.

**Real-world example:**

```text
Memory usage:
500 MB → 700 MB → 450 MB → 800 MB
```

Memory usage can go up and down, so it is represented using a Gauge.

| Metric Type | Behavior | Example |
|---|---|---|
| Counter | Only increases | Total HTTP requests |
| Gauge | Can increase or decrease | Current memory usage |

### Architecture Diagram for Days 73-77

The observability architecture I will build over Days 73-77 is:

```text
                         ┌──────────────────────┐
                         │      Your App        │
                         └──────────┬───────────┘
                                    │
                  ┌─────────────────┼─────────────────┐
                  │                 │                 │
               metrics            logs             traces
                  │                 │                 │
                  ▼                 ▼                 ▼
          ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐
          │  Prometheus  │  │   Promtail   │  │ OTEL Collector  │
          └──────┬───────┘  └──────┬───────┘  └────────┬────────┘
                 │                 │                   │
                 ▼                 ▼                   ▼
          ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐
          │    Grafana   │  │     Loki     │  │ Grafana / Debug │
          └──────────────┘  └──────┬───────┘  └─────────────────┘
                                   │
                                   ▼
                              ┌──────────┐
                              │ Grafana  │
                              └──────────┘


          ┌──────────────┐
          │     Host     │
          └──────┬───────┘
                 │ metrics
                 ▼
          ┌──────────────┐
          │Node Exporter │
          └──────┬───────┘
                 │
                 ▼
          ┌──────────────┐
          │  Prometheus  │
          └──────────────┘


          ┌──────────────┐
          │    Docker    │
          └──────┬───────┘
                 │ metrics
                 ▼
          ┌──────────────┐
          │   cAdvisor   │
          └──────┬───────┘
                 │
                 ▼
          ┌──────────────┐
          │  Prometheus  │
          └──────────────┘
```

The overall idea is:

```text
Metrics → Prometheus → Grafana
Logs    → Loki       → Grafana
Traces  → OTEL       → Grafana / Debug
```

The project roadmap for Days 73-77 builds this observability stack step by step.
