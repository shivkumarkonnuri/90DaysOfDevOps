# Day 77 — Observability Project: Full Stack with Docker Compose

## Overview

This document covers the capstone day of the 5-day observability block. The goal was to clone a reference repository, spin up a complete 8-service observability stack in a single command, validate every data pipeline end to end, build a unified Grafana dashboard, and document the entire setup.

**Environment:** AWS EC2 Ubuntu (t3.small), Docker Compose

---

## Architecture — All 8 Services and Data Flows

```
                        ┌─────────────────────────────────────────────┐
                        │           Docker Network: monitoring          │
                        │                                               │
  ┌──────────────┐      │  ┌─────────────┐    scrape     ┌──────────┐ │
  │  EC2 Host    │──────┼─▶│ Node Exporter│◀─────────────│          │ │
  │  (metrics)   │      │  └─────────────┘               │          │ │
  └──────────────┘      │                                 │Prometheus│ │
                        │  ┌─────────────┐    scrape     │          │ │
  ┌──────────────┐      │  │  cAdvisor   │◀─────────────│  :9090   │ │
  │Docker daemon │──────┼─▶│  :8080      │               │          │ │
  │  (container  │      │  └─────────────┘               └──────────┘ │
  │   metrics)   │      │                                      │       │
  └──────────────┘      │  ┌─────────────┐    scrape           │       │
                        │  │OTEL Collector│◀────────────────────┘       │
  ┌──────────────┐      │  │ :4317/:4318 │                              │
  │  notes-app  │─OTLP─┼─▶│             │                              │
  │  Django API  │      │  └─────────────┘                              │
  │  :8000       │      │                                               │
  └──────────────┘      │  ┌─────────────┐  push logs  ┌──────────┐   │
                        │  │  Promtail   │────────────▶│  Loki    │   │
  ┌──────────────┐      │  │  :9080      │             │  :3100   │   │
  │Docker logs   │──────┼─▶│             │             └──────────┘   │
  │/var/lib/     │      │  └─────────────┘                  │         │
  │docker/       │      │                                    │         │
  │containers/   │      │  ┌─────────────────────────────────▼───────┐ │
  └──────────────┘      │  │              Grafana :3000               │ │
                        │  │  Datasources: Prometheus + Loki          │ │
                        │  │  Dashboard: Production Overview          │ │
                        │  └─────────────────────────────────────────┘ │
                        └─────────────────────────────────────────────┘

Data flows:
  Metrics:  Node Exporter / cAdvisor / OTEL Collector → Prometheus → Grafana
  Logs:     Docker containers → Promtail → Loki → Grafana
  Traces:   notes-app (OTLP) → OTEL Collector → debug exporter (stdout)
```

---

## Task 1: Clone and Launch the Reference Stack

```bash
git clone https://github.com/LondheShubham153/observability-for-devops.git
cd observability-for-devops

# Removed leftover containers from previous project first
cd ~/observability-stack && docker compose down
cd ~/observability-for-devops

docker compose up -d
docker compose ps
```

**Issue encountered:** `node-exporter` container name conflict with a previous `observability-stack` project running on the same EC2 instance. Resolved by tearing down the old stack first.

**All 8 containers started successfully:**

| Service | Port | Status |
|---|---|---|
| prometheus | 9090 | ✅ Running |
| node-exporter | 9100 (internal) | ✅ Running |
| cadvisor | 8080 | ✅ Running |
| grafana | 3000 | ✅ Running |
| loki | 3100 | ✅ Running |
| promtail | 9080 (internal) | ✅ Running |
| otel-collector | 4317/4318 | ✅ Running |
| notes-app | 8000 | ✅ Running |

**Key observation:** `node-exporter` uses `expose: 9100` not `ports:`, so it is only reachable inside the Docker network. Verified internally:

```bash
docker exec prometheus wget -qO- http://node-exporter:9100/metrics | head -5
# Returns: go_gc_duration_seconds metrics — confirmed healthy
```

---

## Task 2: Validate the Metrics Pipeline

### Prometheus Targets — All 4 UP

Verified at `http://<ec2-ip>:9090/targets`:

| Job | Instance | State |
|---|---|---|
| prometheus | localhost:9090 | UP ✅ |
| node-exporter | node-exporter:9100 | UP ✅ |
| docker (cadvisor) | cadvisor:8080 | UP ✅ |
| otel-collector | otel-collector:8889 | UP ✅ |

### PromQL Validation Queries

```promql
# All targets healthy — returns 4 series all with value 1
up

# CPU usage — result: ~2.06% (idle EC2)
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage — result: ~14.7%
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Container memory (newer cAdvisor labels cgroups by path, not name)
topk(3, container_memory_usage_bytes)
# Returns: /system.slice (~6.99GB), / (~6.95GB), /system.slice/containerd.service (~5.3GB)
```

**Note on cAdvisor labels:** Newer cAdvisor versions label by cgroup path (`id` label) rather than container name (`name` label). The query `container_memory_usage_bytes{name!=""}` returns empty. Use `container_memory_usage_bytes` without the name filter instead.

---

## Task 3: Validate the Logs Pipeline

### Traffic Generation

```bash
for i in $(seq 1 50); do
  curl -s http://localhost:8000 > /dev/null
  curl -s http://localhost:8000/api/ > /dev/null
done
```

### Promtail Label Issue and Fix

**Problem:** The reference repo's `promtail-config.yml` uses `static_configs` with a glob path and `docker: {}` pipeline stage. In newer Promtail versions, this stage does not reliably enrich all container log streams with the `container` label. Only 3 containers appeared in Loki (`grafana`, `loki`, `tempo` — carried over from a previous stack).

**Diagnosis:**
```bash
curl -s "http://localhost:3100/loki/api/v1/label/container/values" | python3 -m json.tool
# Only showed: grafana, loki, tempo

docker exec promtail ls /var/run/docker.sock  # Socket accessible ✅
docker logs promtail 2>&1 | grep -i "error"   # Only one harmless warn ✅
```

Promtail was watching all container directories but couldn't resolve container IDs to names.

**Fix — switched to Docker service discovery:**

```yaml
# promtail/promtail-config.yml (updated)
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
      - source_labels: [__meta_docker_container_name]
        regex: '/(.*)'
        target_label: container
      - source_labels: [__meta_docker_container_log_stream]
        target_label: stream
      - source_labels: [__meta_docker_compose_service]
        target_label: compose_service
    pipeline_stages:
      - docker: {}
```

```bash
docker compose restart promtail
```

**Result:** `{container="notes-app"}` now returns real Django HTTP logs:
```
[03/May/2026 11:50:24] "GET /api/notes/ HTTP/1.1" 200 1169
[03/May/2026 11:50:24] "POST /api/notes/create/ HTTP/1.1" 200 148
```

### LogQL Queries Validated

```logql
{job="docker"}                          # All container logs
{container="notes-app"}                 # notes-app Django logs
{container="notes-app"} |= "GET"        # HTTP GET requests only
{job="docker"} |= "error"              # Errors across all containers
sum by (container) (rate({job="docker"}[5m]))  # Log rate per container
```

---

## Task 4: Validate the Traces Pipeline

Sent a two-span OTLP trace simulating an HTTP request calling a database query:

```bash
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"notes-app"}}]},"scopeSpans":[{"spans":[{"traceId":"aaaabbbbccccdddd1111222233334444","spanId":"1111222233334444","name":"GET /api/notes","kind":2,...},{"traceId":"aaaabbbbccccdddd1111222233334444","spanId":"5555666677778888","parentSpanId":"1111222233334444","name":"SELECT notes FROM database","kind":3,...}]}]}]}'
```

**OTEL Collector confirmed receipt:**

```
2026-05-03T12:04:17.430Z  info  Traces  ...  "resource spans": 1, "spans": 2
```

- 1 resource span (notes-app service) ✅
- 2 spans (parent HTTP + child DB query) ✅
- Parent-child relationship preserved via `parentSpanId` ✅

**Note:** The collector uses `debug` exporter (logs to stdout). For production, replace with Grafana Tempo for persistent trace storage and UI visualization.

---

## Task 5: Production Overview Dashboard

Built a unified Grafana dashboard with 4 rows and 11 panels.

### Row 1 — System Health

| Panel | Type | Query | Value |
|---|---|---|---|
| CPU Usage % | Gauge | `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` | ~2.43% 🟢 |
| Memory Usage % | Gauge | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` | ~15.3% 🟢 |
| Disk Usage % | Gauge | `(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100` | 56.1% 🟢 |
| Targets Up | Stat | `sum(up)` / `count(up)` | 4/4 🟢 |

### Row 2 — Container Metrics

| Panel | Type | Query |
|---|---|---|
| Container CPU | Time series | `rate(container_cpu_usage_seconds_total[5m]) * 100` |
| Container Memory | Bar chart | `topk(5, container_memory_usage_bytes) / 1024 / 1024` |

### Row 3 — Application Logs

| Panel | Type | Query (Loki) |
|---|---|---|
| Notes App Logs | Logs | `{container="notes-app"}` |
| Error Rate | Time series | `sum(rate({job="docker"} \|= "error" [5m]))` |
| Log Volume by Container | Time series | `sum by (container) (rate({job="docker"}[5m]))` |

### Row 4 — Service Overview

| Panel | Type | Query |
|---|---|---|
| Prometheus Scrape Duration | Time series | `prometheus_target_interval_length_seconds{quantile="0.99"}` |
| Prometheus Ingest Rate | Stat | `rate(prometheus_tsdb_head_samples_appended_total{type="float"}[5m])` |

**Dashboard settings:** Time range = Last 30 minutes, Auto-refresh = 10s

---

## Task 6: Config Comparison — My Stack vs Reference Repo

| Component | Reference Repo | My Day 73-76 Version | Key Differences |
|---|---|---|---|
| `prometheus.yml` | 4 scrape jobs: prometheus, node-exporter, docker(cadvisor), otel-collector | Similar structure, same targets | Reference uses 15s interval throughout |
| `loki-config.yml` | filesystem storage, tsdb schema | Same storage backend | Minor: schema date differences |
| `promtail-config.yml` | static_configs + `docker: {}` stage | **Fixed to docker_sd_configs** | docker_sd_configs is more reliable for container label enrichment |
| `otel-collector-config.yml` | OTLP receiver + debug exporter | Same pipeline | Reference lacks Tempo exporter for trace persistence |
| `datasources.yml` | Prometheus + Loki auto-provisioned | Same | Identical |
| `docker-compose.yml` | 8 services, monitoring network, named volumes | Similar but fewer services | Reference includes notes-app sample application |

---

## 5-Day Observability Learning Map

| Day | What Was Built | Key Concepts |
|---|---|---|
| 73 | Prometheus setup, PromQL fundamentals | Metrics types, scrape configs, time-series queries |
| 74 | Node Exporter, cAdvisor, Grafana dashboards | Infrastructure metrics, visualization, datasource provisioning |
| 75 | Loki, Promtail, LogQL | Log aggregation, label-based querying, log-metric correlation |
| 76 | OTEL Collector, traces, alerting rules | Distributed tracing, OTLP protocol, alert rule syntax |
| 77 | Full stack integration, unified dashboard | End-to-end pipeline validation, dashboard-as-code concepts |

---

## What Would Be Added for Production

1. **Alertmanager** — Route alerts to Slack/PagerDuty based on severity. Currently alert rules fire but have nowhere to send notifications.

2. **Grafana Tempo** — Replace the `debug` OTEL exporter with Tempo for persistent trace storage. Add Tempo as a Grafana datasource to visualize traces and link them to logs (TraceID correlation).

3. **HTTPS/TLS** — All endpoints (Grafana, Prometheus, Loki) are currently plain HTTP. Use a reverse proxy (Nginx/Caddy) with TLS certificates.

4. **Authentication** — Prometheus and Loki have no auth. Use basic auth or integrate with an identity provider via Grafana's OAuth support.

5. **Log retention policies** — Add `retention_period` to Loki config. Without it, logs grow indefinitely and will fill the disk.

6. **High availability** — Single Prometheus instance is a SPOF. Use Thanos or Prometheus federation for HA. Loki supports horizontal scaling in microservices mode.

7. **Resource limits** — Add `mem_limit` and `cpus` constraints to each Docker Compose service to prevent one container starving others.

8. **Structured logging** — The notes-app outputs plain text Django logs. Structured JSON logs would enable richer LogQL filtering and metric extraction via Promtail pipeline stages.

---

## Self-Managed vs Managed Observability

| Aspect | This Stack (Self-managed) | Datadog / New Relic / CloudWatch |
|---|---|---|
| Cost | Infrastructure cost only | Per-host or per-metric pricing, can be very expensive at scale |
| Setup time | Hours to days | Minutes (agent install) |
| Maintenance | You manage upgrades, storage, HA | Fully managed |
| Data ownership | All data stays in your infrastructure | Data sent to vendor |
| Customization | Full control over retention, pipelines, dashboards | Limited by vendor UI |
| Trace storage | Requires Tempo (additional component) | Built-in |
| Alerting | Alertmanager (powerful but complex to configure) | Built-in with integrations |
| Best for | Cost-sensitive teams, data-sovereignty requirements, learning | Teams wanting managed simplicity, enterprise integrations |

---

## Key Takeaways from the 5-Day Observability Block

1. **The three pillars work together** — Metrics tell you *what* is wrong, logs tell you *why*, traces tell you *where* in the call chain. All three are needed for production debugging.

2. **Label design matters** — The Promtail `container` label issue showed that how you label data at ingestion determines what you can query later. Fix labels at the source, not at query time.

3. **cAdvisor label evolution** — Newer versions use cgroup paths instead of container names. Always verify actual label values with `/loki/api/v1/label/<name>/values` or Prometheus label browser before building dashboards.

4. **Docker service discovery > static configs** — `docker_sd_configs` in Promtail is more reliable than static path globs for container log collection because it actively queries the Docker API to resolve metadata.

5. **The debug exporter is for learning** — OTEL's debug exporter logs traces to stdout. In production, you need Tempo (or Jaeger/Zipkin) as a backend to store and query traces.

6. **Disk usage monitoring is critical** — Prometheus TSDB, Loki log storage, and Docker image cache all consume disk. At 56% disk usage on a fresh EC2 running this stack for one day, retention policies are not optional in production.

7. **Dashboard-as-code** — Grafana dashboards can be exported as JSON and stored in Git. The `grafana/provisioning/dashboards/` directory in the reference repo shows how to auto-provision dashboards on startup — eliminating manual UI setup.

---

## All Config Files

### docker-compose.yml

```yaml
networks:
  monitoring:
    driver: bridge
volumes:
  prometheus_data: {}
  grafana_data: {}
  loki_data: {}
services:
  grafana:
    image: grafana/grafana-enterprise
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources
      - ./grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards
    networks:
      - monitoring
    restart: unless-stopped
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - prometheus_data:/prometheus
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    networks:
      - monitoring
    restart: unless-stopped
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    expose:
      - 9100
    networks:
      - monitoring
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring
    restart: unless-stopped
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/config.yaml:ro
      - loki_data:/loki
    command: -config.file=/etc/loki/config.yaml
    networks:
      - monitoring
    restart: unless-stopped
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/config.yaml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/config.yaml
    depends_on:
      - loki
    networks:
      - monitoring
    restart: unless-stopped
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector
    command: ["--config=/etc/otelcol-contrib/config.yaml"]
    volumes:
      - ./otel-collector/otel-collector-config.yml:/etc/otelcol-contrib/config.yaml:ro
    ports:
      - "4317:4317"
      - "4318:4318"
    expose:
      - 8889
    networks:
      - monitoring
    restart: unless-stopped
  notes-app:
    image: notes-app:latest
    build:
      context: ./notes-app
      dockerfile: Dockerfile
    container_name: notes-app
    ports:
      - "8000:8000"
    networks:
      - monitoring
    restart: unless-stopped
```

### promtail-config.yml (fixed version)

```yaml
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
      - source_labels: [__meta_docker_container_name]
        regex: '/(.*)'
        target_label: container
      - source_labels: [__meta_docker_container_log_stream]
        target_label: stream
      - source_labels: [__meta_docker_compose_service]
        target_label: compose_service
    pipeline_stages:
      - docker: {}
```

---

## Submission

```bash
mkdir -p ~/observability-for-devops/2026/day-77
cp day-77-observability-project.md ~/observability-for-devops/2026/day-77/
cd ~/observability-for-devops
git add 2026/day-77/day-77-observability-project.md
git commit -m "Day 77: Full observability stack capstone - Prometheus, Grafana, Loki, Promtail, OTEL Collector"
git push origin main
```

---

## Cleanup

```bash
# When done exploring — removes containers AND named volumes
docker compose down -v
```

---

*Day 77 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
