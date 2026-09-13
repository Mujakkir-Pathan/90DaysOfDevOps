# Day 75 -- Log Management with Loki and Promtail

## Task

Metrics tell me *what* is broken. Logs help me understand *why*.
Yesterday I built the metrics pipeline with Prometheus, Node Exporter,
cAdvisor, and Grafana. Today I added the second pillar of observability
-- logs.

I set up Grafana Loki to store logs and Promtail to collect Docker
container logs and send them to Loki. By the end, I could view metrics
and logs together in Grafana.

------------------------------------------------------------------------

## All project

[today's stack](observability-stack/)

---

## All screenshot's

[screenshots of task](screenshots/)

------------------------------------------------------------------------

## Challenge Tasks

### Task 1: Understand the Logging Pipeline

I followed this logging flow:

``` text
[Docker Containers]
       |
       | (write JSON logs to /var/lib/docker/containers/)
       v
  [Promtail]
       |
       | (reads logs, adds labels, sends them to Loki)
       v
    [Loki]
       |
       | (stores logs and indexes labels)
       v
   [Grafana]
       |
       | (queries Loki with LogQL)
       v
   [You]
```

The main thing I learned is that Loki does not index the full text of
every log line. Instead, it indexes labels such as `container_name`,
`job`, and `filename`.

This keeps Loki simpler and generally cheaper to operate than a
full-text logging system. The trade-off is that full-text searching is
not as powerful as it is with systems such as ELK. Loki fits especially
well with Prometheus and Grafana because it follows the same label-based
approach.

------------------------------------------------------------------------

### Task 2: Add Loki to the Stack

I created the Loki directory:

``` bash
mkdir -p loki
```

Then I created `loki/loki-config.yml`:

``` yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /loki

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  filesystem:
    directory: /loki/chunks
```

I understood the main settings as:

-   `auth_enabled: false` -- kept Loki in single-tenant mode without
    authentication.
-   `http_listen_port: 3100` -- exposed Loki's HTTP endpoint.
-   `store: tsdb` -- used Loki's time-series database for indexing.
-   `object_store: filesystem` -- stored log chunks on local disk.
-   `replication_factor: 1` -- used a single Loki instance, which was
    enough for this lab.
-   `/loki` -- was persisted using the Docker volume.

I added Loki to `docker-compose.yml`:

``` yaml
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml
      - loki_data:/loki
    command: -config.file=/etc/loki/loki-config.yml
    restart: unless-stopped
```

I also added its volume:

``` yaml
volumes:
  prometheus_data:
  grafana_data:
  loki_data:
```

I started Loki:

``` bash
docker compose up -d loki
```

Then I checked whether it was ready:

``` bash
curl http://localhost:3100/ready
```

The response was:

``` text
ready
```

So Loki was running correctly.

------------------------------------------------------------------------

### Task 3: Add Promtail to Collect Container Logs

I created the Promtail directory:

``` bash
mkdir -p promtail
```

I initially used a static Docker log path. While testing the labels, I
found that I needed the `container_name` label for the queries in the
task. I changed Promtail to use Docker service discovery and relabeling.

The final `promtail/promtail-config.yml` was:

``` yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

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
      # container name
      - source_labels: ["__meta_docker_container_name"]
        regex: "/(.*)"
        target_label: "container_name"

      # container id -> log path
      - source_labels: ["__meta_docker_container_id"]
        regex: "(.+)"
        target_label: "__path__"
        replacement: "/var/lib/docker/containers/$1/*-json.log"

      # job label
      - target_label: "job"
        replacement: "docker"

    pipeline_stages:
      - docker: {}
```

The important parts were:

-   `positions` kept track of which lines Promtail had already sent.
-   `clients` pointed Promtail to Loki.
-   `docker_sd_configs` let Promtail discover Docker containers through
    the Docker socket.
-   `container_name` converted Docker's container name metadata into a
    Loki label.
-   `__path__` used the container ID to find its JSON log file.
-   `docker: {}` parsed Docker's JSON log format.

I added Promtail to `docker-compose.yml`:

``` yaml
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/promtail-config.yml
    restart: unless-stopped
```

The volume mounts gave Promtail access to:

-   `/var/lib/docker/containers` -- Docker's log files, mounted
    read-only.
-   `/var/run/docker.sock` -- Docker metadata and container discovery.

I restarted the stack:

``` bash
docker compose up -d
```

Then I generated some logs from `notes-app`:

``` bash
for i in $(seq 1 20); do curl -s http://localhost:8000 > /dev/null; done
```

Promtail successfully discovered the Docker containers. Its logs showed
messages such as:

``` text
added Docker target containerID=...
```

There were no configuration errors.

I also checked the labels in Loki. After the Promtail configuration was
corrected, `container_name` was available and worked in Grafana.

------------------------------------------------------------------------

### Task 4: Add Loki as a Grafana Datasource

I updated `grafana/provisioning/datasources/datasources.yml`:

``` yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```

I restarted Grafana:

``` bash
docker compose restart grafana
```

The restart completed successfully:

``` text
[+] Restarting 1/1
 ✔ Container grafana Started
```

In Grafana, the Loki datasource showed:

``` text
Data source is working.
```

At this point, both Prometheus and Loki were available in Grafana.

------------------------------------------------------------------------

### Task 5: Query Logs with LogQL

I opened **Grafana \> Explore** and selected Loki.

#### 1. Stream selector

``` logql
{job="docker"}
```

This returned logs from the Docker containers.

#### 2. Filter by container name

``` logql
{container_name="prometheus"}
```

This returned Prometheus container logs.

#### 3. Keyword search

``` logql
{job="docker"} |= "error"
```

This searched for log lines containing `error`.

#### 4. Negative filter

``` logql
{job="docker"} != "health"
```

This filtered out lines containing `health`.

#### 5. Regex filter

``` logql
{job="docker"} |~ "status=[45]\\d{2}"
```

This searched for 4xx and 5xx status codes.

#### 6. Count log lines

``` logql
count_over_time({job="docker"}[5m])
```

This counted matching log lines over five-minute windows.

#### 7. Rate of logs

``` logql
rate({job="docker"}[5m])
```

This calculated the rate of matching log lines.

#### 8. Top containers by log volume

``` logql
topk(5, sum by (container_name) (rate({job="docker"}[5m])))
```

This showed the containers producing logs at the highest rate.

#### Exercise: Find error logs from notes-app

I used:

``` logql
{container_name="notes-app"} |= "error"
```

The query returned no matching data. I checked the actual `notes-app`
logs and found 404 entries such as:

``` text
Not Found: /does-not-exist
[13/Sep/2026 11:42:34] "GET /does-not-exist HTTP/1.1" 404 2513
```

There was no literal `error` text in those logs, so the result was
expected.

I also ran:

``` logql
count_over_time({container_name="notes-app"} |= "error"[1m])
```

It returned zero/no matching data for the same reason.

------------------------------------------------------------------------

### Task 6: Correlate Metrics and Logs in Grafana

#### 1. Add a logs panel to the dashboard

I opened the Day 74 dashboard and added a new panel with:

-   **Datasource:** Loki
-   **Query:** `{job="docker"}`
-   **Visualization:** Logs
-   **Title:** `Container Logs`

This allowed the dashboard to show container logs alongside the existing
metrics.

#### 2. Use the Explore split view

I opened Grafana Explore and enabled the split view.

The left side used **Prometheus**, while the right side used **Loki**.

The task originally suggested:

``` promql
rate(container_cpu_usage_seconds_total{name="notes-app"}[5m])
```

But this returned no data in my environment because cAdvisor did not
expose `name="notes-app"` for this metric.

I checked the raw metric:

``` promql
container_cpu_usage_seconds_total{job="cadvisor"}
```

The available labels included:

``` text
cpu
id
instance
job
```

I then found the actual Docker container ID:

``` bash
docker inspect -f '{{.Id}}' notes-app
```

The result was:

``` text
d791e208d2ab739a39ab6186b0b820834162eafdbce1d7ca774ac00494bd9896
```

cAdvisor represented that container using this cgroup ID:

``` text
/system.slice/docker-d791e208d2ab739a39ab6186b0b820834162eafdbce1d7ca774ac00494bd9896.scope
```

Using that actual ID, the Prometheus query worked:

``` promql
container_cpu_usage_seconds_total{
  job="cadvisor",
  id="/system.slice/docker-d791e208d2ab739a39ab6186b0b820834162eafdbce1d7ca774ac00494bd9896.scope"
}
```

The right side used Loki:

``` logql
{container_name="notes-app"}
```

This returned the `notes-app` logs.

So the split view became:

``` text
Prometheus                         Loki
CPU metrics                       Container logs
     |                                  |
     +---------- notes-app -------------+
```

This gave me both sides of the incident: the metric showing what was
happening and the logs showing what the application was doing.

#### 3. Time sync

I used the same time range for both sides of the split view so the CPU
metric and logs could be compared together.

This is useful during an incident because I can find a spike in a metric
and then check the application logs from the same period instead of
switching between separate tools.

------------------------------------------------------------------------

## Architecture Diagram

``` text
+-------------------+
| Docker Containers |
|                   |
| notes-app         |
| prometheus        |
| grafana           |
| etc.              |
+---------+---------+
          |
          | JSON container logs
          v
+-------------------+
|     Promtail      |
|                   |
| Reads logs        |
| Adds labels       |
| Sends to Loki     |
+---------+---------+
          |
          | HTTP push
          v
+-------------------+
|       Loki        |
|                   |
| Stores logs       |
| Indexes labels    |
+---------+---------+
          |
          | LogQL
          v
+-------------------+
|      Grafana      |
|                   |
| Metrics + Logs    |
+-------------------+
          |
          v
        You
```

------------------------------------------------------------------------

## Updated `docker-compose.yml`

The stack now contains all the services built so far:

``` yaml
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
      - '--storage.tsdb.retention.time=30d'
      - '--storage.tsdb.retention.size=1GB'
    restart: unless-stopped

  notes-app:
    image: mujakkirpathan/notes-app:latest
    container_name: notes-app
    ports:
      - "8000:8000"
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped

  grafana:
    image: grafana/grafana-enterprise:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml
      - loki_data:/loki
    command: -config.file=/etc/loki/loki-config.yml
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/promtail-config.yml
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
```

------------------------------------------------------------------------

## Loki vs ELK Stack

  --------------------------------------------------------------------------
  Feature                 Loki                       ELK
  ----------------------- -------------------------- -----------------------
  Main purpose            Log aggregation and        Log aggregation,
                          querying                   search, and analysis

  Indexing                Labels instead of full log Full-text and
                          text                       field-based indexing

  Resource usage          Generally lighter and      Generally more
                          simpler                    resource-intensive

  Query language          LogQL                      Elasticsearch Query DSL
                                                     / Kibana search

  Best fit                Grafana/Prometheus-based   Deep search and complex
                          observability              log analysis

  Setup                   Simpler                    More components to
                                                     manage
  --------------------------------------------------------------------------

### When would I use each?

I would choose **Loki** when I already have Grafana and Prometheus and
want a lightweight logging solution that follows the same label-based
approach.

I would choose **ELK** when detailed full-text searching, indexing, and
complex log analysis are more important than keeping the logging stack
simple.

For this project, Loki made more sense because it fit naturally into the
Prometheus and Grafana stack I had already built.

------------------------------------------------------------------------

## Final Result

By the end of Day 75, my observability pipeline looked like this:

``` text
Docker
  |
  +--> cAdvisor --> Prometheus --+
  |                              |
  +--> Promtail --> Loki --------+--> Grafana --> You
```

The biggest thing I learned today was the difference between **metrics
and logs**.

Prometheus helps me understand **what is happening**, while Loki helps
me investigate **why it is happening**. Bringing both into Grafana makes
troubleshooting much easier because I can compare the metric and the
related logs in the same place.
