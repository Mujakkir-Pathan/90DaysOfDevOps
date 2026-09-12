# Day 74 -- Node Exporter, cAdvisor, and Grafana Dashboards

## Task

Prometheus was already running and querying metrics, but it was initially monitoring only itself. During Day 74, I extended the observability stack to monitor the host machine and Docker containers, and then visualized the metrics through Grafana dashboards.

---

## All project

[today's stack](observability-stack/)

---

## All screenshot's

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Add Node Exporter for Host Metrics

Node Exporter was added to `docker-compose.yml` to expose Linux host metrics such as CPU, memory, disk, filesystem, and network statistics.

The Node Exporter container used read-only mounts for `/proc`, `/sys`, and `/`, allowing it to read host information without modifying the host.

Prometheus was configured with Node Exporter as a scrape target:

```yaml
- job_name: "node-exporter"
  static_configs:
    - targets: ["node-exporter:9100"]
```

The stack was restarted with:

```bash
docker compose up -d
```

Node Exporter was verified through its metrics endpoint and the Prometheus Targets page. The `node-exporter:9100` target was shown as **UP**.

#### Host Metrics PromQL

**CPU idle time per core:**

```promql
node_cpu_seconds_total{mode="idle"}
```

The query returned CPU metrics for two cores:

```text
cpu="0", instance="node-exporter:9100", job="node-exporter", mode="idle" -> 8150.78
cpu="1", instance="node-exporter:9100", job="node-exporter", mode="idle" -> 8145.14
```

**Available memory:**

```promql
node_memory_MemAvailable_bytes
```

The available memory returned was:

```text
1390886912 bytes
```

**Memory usage percentage:**

```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

The result was approximately:

```text
30.33%
```

**Disk usage percentage:**

```promql
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100
```

For the root filesystem `/`, the result was approximately:

```text
53.69%
```

**Network received bytes per second:**

```promql
rate(node_network_receive_bytes_total[5m])
```

The `eth0` interface returned approximately:

```text
46.47 bytes/second
```

The `lo` interface returned:

```text
0 bytes/second
```

These queries confirmed that Node Exporter was exposing host-level metrics to Prometheus.

---

### Task 2: Add cAdvisor for Container Metrics

cAdvisor was added to `docker-compose.yml` to monitor resource usage and performance of Docker containers.

The service used the Docker socket, `/sys`, and `/var/lib/docker/` as read-only mounts so cAdvisor could discover containers and collect container-level statistics.

Prometheus was configured with cAdvisor as a scrape target:

```yaml
- job_name: "cadvisor"
  static_configs:
    - targets: ["cadvisor:8080"]
```

The stack was restarted with:

```bash
docker compose up -d
```

The cAdvisor web UI was accessible at port `8080`, and the Prometheus Targets page showed:

```text
cadvisor:8080/metrics -> UP
```

#### Container Metrics PromQL

The task queries used `{name!=""}` to filter for named containers. However, in the cAdvisor version used during this task, the returned metrics did not contain a `name` label. They exposed an `id` label instead.

Because of this environment-specific behavior, the exact CPU and memory queries with `{name!=""}` returned no data. Removing the `name` filter confirmed that the metrics themselves were available.

**CPU usage per container:**

```promql
rate(container_cpu_usage_seconds_total[5m])
```

The query returned container and system-level data, including examples such as:

```text
id="/" -> 0.0373
id="/system.slice" -> 0.0414
```

**Memory usage:**

```promql
container_memory_usage_bytes
```

The query returned memory data, including examples such as:

```text
id="/" -> 1585254400
id="/system.slice" -> 1457094656
id="/system.slice/containerd.service" -> 255123456
```

**Network received bytes per second:**

```promql
rate(container_network_receive_bytes_total{name!=""}[5m])
```

This query returned network data in the environment, with the `eth0` interface showing approximately:

```text
268.06 bytes/second
```

**Top memory users:**

```promql
topk(3, container_memory_usage_bytes{name!=""})
```

The result showed the highest memory entries, including:

```text
id="/" -> 1564663808 bytes
id="/system.slice" -> 1436295168 bytes
id="/system.slice/containerd.service" -> 254955520 bytes
```

The important difference discovered during verification was that the current cAdvisor metrics did not provide the `name` label expected by the task. Therefore, the CPU and memory metrics were verified by removing the filter rather than falsely claiming that the original filtered queries returned data.

#### Node Exporter vs cAdvisor

| Tool | What it monitors | When to use it |
|---|---|---|
| Node Exporter | Host/server CPU, memory, disk, filesystem, and network | When I need to understand the health and resource usage of the Linux machine |
| cAdvisor | Docker/container CPU, memory, network, and filesystem usage | When I need to understand resource usage of individual containers |

In simple terms, **Node Exporter tells me how the machine is doing, while cAdvisor tells me how the containers running on that machine are doing.**

---

### Task 3: Set Up Grafana

Grafana was added to the Docker Compose stack using:

```yaml
grafana:
  image: grafana/grafana-enterprise:latest
  container_name: grafana
  ports:
    - "3000:3000"
  volumes:
    - grafana_data:/var/lib/grafana
  environment:
    - GF_SECURITY_ADMIN_USER=admin
    - GF_SECURITY_ADMIN_PASSWORD=admin123
  restart: unless-stopped
```

The persistent Grafana volume was also added:

```yaml
volumes:
  prometheus_data:
  grafana_data:
```

Grafana was started with:

```bash
docker compose up -d
```

I opened Grafana on port `3000` and logged in with the configured admin credentials.

Prometheus was configured as the datasource using:

```text
http://prometheus:9090
```

Grafana displayed:

```text
Successfully queried the Prometheus API.
```

This confirmed that Grafana could communicate with Prometheus over the Docker network.

---

### Task 4: Build Your First Dashboard

A custom Grafana dashboard named **DevOps Observability Overview** was created.

The dashboard contained the required five panels.

#### Panel 1 -- CPU Usage

PromQL:

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Visualization: **Gauge**

Title:

```text
CPU Usage %
```

The gauge thresholds were configured as requested.

#### Panel 2 -- Memory Usage

PromQL:

```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

Visualization: **Gauge**

Title:

```text
Memory Usage %
```

#### Panel 3 -- Container CPU Usage

PromQL:

```promql
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
```

Visualization: **Time series**

Title:

```text
Container CPU Usage
```

The panel used the requested container CPU query. The cAdvisor `name`-label limitation described in Task 2 also applied to this query, so the displayed container information used the labels available from the running cAdvisor version.

#### Panel 4 -- Container Memory Usage

PromQL:

```promql
container_memory_usage_bytes{name!=""} / 1024 / 1024
```

Visualization: **Bar chart**

Title:

```text
Container Memory (MB)
```

The panel displayed container memory data using the labels available from cAdvisor.

#### Panel 5 -- Disk Usage

PromQL:

```promql
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```

Visualization: **Stat**

Title:

```text
Disk Usage %
```

The completed dashboard showed all five panels with live data. At verification time, the dashboard displayed approximately:

- Disk Usage: **54.3%**
- Memory Usage: **33.7%**
- CPU Usage: **2.22%**

Container CPU and container memory panels also displayed data.

The dashboard was saved as:

```text
DevOps Observability Overview
```

---

### Task 5: Auto-Provision Datasources with YAML

Grafana datasource provisioning was configured so that the Prometheus datasource could be recreated automatically instead of being configured manually through the Grafana UI.

The provisioning directories were created:

```bash
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards
```

The datasource configuration was created at:

```text
grafana/provisioning/datasources/datasources.yml
```

Its contents were:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

The Grafana service was updated to mount the provisioning directory:

```yaml
volumes:
  - grafana_data:/var/lib/grafana
  - ./grafana/provisioning:/etc/grafana/provisioning
```

Grafana was restarted with:

```bash
docker compose up -d grafana
```

The container remained running:

```text
✔ Container grafana Running
```

I then checked **Connections → Data sources** and confirmed that the **Prometheus** datasource was already present without manually adding it through the UI.

#### Why YAML Provisioning Is Better

Provisioning datasources with YAML is better because it is:

- **Repeatable** — the same datasource can be created automatically every time.
- **Consistent** — different environments can use the same configuration.
- **Automated** — no manual Grafana setup is required after deployment.
- **Version-controlled** — the configuration can be stored and tracked in Git.
- **Easy to recreate** — redeploying Grafana can automatically restore the datasource.
- **Production-friendly** — it supports Infrastructure as Code and reduces manual configuration errors.

In this setup, Grafana loaded the Prometheus datasource from `datasources.yml`, making the configuration repeatable.

---

### Task 6: Import a Community Dashboard

I imported two Grafana community dashboards.

#### Node Exporter Dashboard

Dashboard ID:

```text
1860
```

This was imported as **Node Exporter Full** and connected to the Prometheus datasource.

It provided pre-built panels for host metrics such as CPU, memory, disk, and network.

#### Docker / cAdvisor Dashboard

Dashboard ID:

```text
193
```

This dashboard was imported using Prometheus as the datasource and explored for container-level statistics.

#### Final Docker Compose Verification

The final `docker-compose.yml` contained all required services:

- `prometheus`
- `node-exporter`
- `cadvisor`
- `grafana`
- `notes-app`

The following command was used to verify them:

```bash
docker compose ps
```

Actual verification output:

```text
NAME            IMAGE                               COMMAND                  SERVICE         CREATED          STATUS                 PORTS
cadvisor        gcr.io/cadvisor/cadvisor:latest     "/usr/bin/cadvisor -…"   cadvisor        6 hours ago      Up 3 hours (healthy)   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
grafana         grafana/grafana-enterprise:latest   "/run.sh"                grafana         22 minutes ago   Up 22 minutes          0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp
node-exporter   prom/node-exporter:latest           "/bin/node_exporter …"   node-exporter   10 hours ago     Up 3 hours             0.0.0.0:9100->9100/tcp, [::]:9100->9100/tcp
notes-app       mujakkirpathan/notes-app:latest     "/bin/sh -c 'python …"   notes-app       10 hours ago     Up 3 hours             0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp
prometheus      prom/prometheus:latest              "/bin/prometheus --c…"   prometheus      10 hours ago     Up 3 hours             0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp
```

All five required services were running successfully, and cAdvisor was reported as **healthy**.

---

## Key Learnings

### Node Exporter and cAdvisor

Node Exporter monitors the **host machine**, while cAdvisor monitors **containers**. Using both gives visibility from the infrastructure level down to individual workloads.

### PromQL

PromQL was used to calculate and inspect:

- CPU usage
- Memory usage
- Disk usage
- Network traffic
- Container CPU usage
- Container memory usage

### Grafana

Grafana turned raw Prometheus metrics into dashboards that made system health easier to understand at a glance.

### Datasource Provisioning

YAML provisioning removed the need for repeated manual datasource configuration and made the Grafana setup reproducible.

### Community Dashboards

Community dashboards showed how the same Prometheus metrics can be turned into complete monitoring views without manually creating every visualization.

---

## Final Result

By the end of Day 74, the observability stack monitored both the Linux host and Docker containers:

```text
                    Grafana
                       |
                       v
                  Prometheus
                  /        \
                 v          v
         Node Exporter    cAdvisor
              |              |
              v              v
          Host Metrics   Container Metrics

                  notes-app
```

The stack now provides a complete monitoring flow from **host → containers → Prometheus → Grafana dashboards**.
