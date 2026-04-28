# Day 73 -- Introduction to Observability and Prometheus

---

## Task 1: Observability Concepts

### Monitoring vs Observability

| | Traditional Monitoring | Observability |
|---|---|---|
| **Tells you** | *When* something is wrong | *Why, what, and where* it is wrong |
| **How** | Fixed thresholds and alerts | Query, explore, correlate |
| **Example** | "CPU > 90% -- alert fired" | "CPU spiked because this API endpoint got 10x traffic at 2:47 AM" |
| **Analogy** | Car warning light 🔴 | Full dashboard + diagnostics port 📊 |

**In simple words:** Monitoring tells you *when* something is wrong. Observability tells you *why* it is wrong, *what* exactly broke, and *where* it broke -- using metrics, logs, and traces together.

---

### The Three Pillars of Observability

~~~
METRICS  → "What is broken?"     → Numbers over time (CPU %, error rate, req/sec)
LOGS     → "Why did it break?"   → Text records of events (stack traces, error messages)
TRACES   → "Where did it break?" → Journey of a single request across services
~~~

**Real example -- a slow checkout page:**
- **Metrics** show: error rate on `/checkout` jumped from 0.1% to 8% at 2:47 AM
- **Logs** show: `TimeoutException: payment-service did not respond within 5s`
- **Traces** show: the call to payment-service took 12 seconds, everything else was fine

Without all three, you are guessing.

---

### Tools for Each Pillar

| Pillar | Tools |
|---|---|
| Metrics | Prometheus, Datadog, CloudWatch |
| Logs | Loki, ELK Stack, Fluentd, Promtail |
| Traces | OpenTelemetry, Jaeger, Zipkin |

---

### Why DevOps Engineers Need All Three

Metrics tell you *what* is broken. Logs tell you *why* it broke. Traces tell you *where* it broke across services. Using only one or two pillars leaves blind spots -- you might know there is a problem but not be able to fix it quickly.

---

### Observability Stack Architecture (Days 73-77)

~~~
[Your App]  --> metrics --> [Prometheus]       --> [Grafana Dashboards]
[Your App]  --> logs    --> [Promtail]         --> [Loki] --> [Grafana]
[Your App]  --> traces  --> [OTEL Collector]   --> [Grafana/Debug]
[Host]      --> metrics --> [Node Exporter]    --> [Prometheus]
[Docker]    --> metrics --> [cAdvisor]         --> [Prometheus]
~~~

---

## Task 2: Prometheus Setup

### How Prometheus Works -- Pull Model

Most monitoring tools use a **push model** -- the app sends data to the monitoring tool.

Prometheus uses a **pull model** (also called scraping):

~~~
Prometheus → "Hey App, give me your metrics" → App responds with data
~~~

**Why pull model matters:**
- Prometheus is in control of when and how often it collects data
- If an app dies, Prometheus knows immediately -- it tried to scrape and got nothing
- Apps just expose an endpoint, they do not need to know where Prometheus is

---

### Project Directory Structure

~~~bash
mkdir observability-stack && cd observability-stack
~~~

---

### prometheus.yml

~~~yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "notes-app"
    static_configs:
      - targets: ["notes-app:8080"]
~~~

**Key points:**
- `scrape_interval: 15s` -- Prometheus scrapes every target every 15 seconds
- `evaluation_interval: 15s` -- how often alerting rules are evaluated
- `job_name` -- a logical name for a group of targets
- `targets: ["localhost:9090"]` -- Prometheus scraping itself (its own `/metrics` endpoint)
- `targets: ["notes-app:8080"]` -- uses container name, not `localhost`, because Docker resolves container names to IPs automatically

---

### docker-compose.yml

~~~yaml
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

  notes-app:
    image: trainwithshubham/notes-app:latest
    container_name: notes-app
    ports:
      - "8080:8080"
    restart: unless-stopped

volumes:
  prometheus_data:
~~~

**Key points:**
- `ports: "9090:9090"` -- maps EC2 port 9090 to container port 9090
- `./prometheus.yml:/etc/prometheus/prometheus.yml` -- mounts your config file into the container
- `prometheus_data:/prometheus` -- named volume so data survives container restarts
- Top-level `volumes:` uses key-value format (no dash) -- common YAML mistake to avoid

---

### Start the Stack

~~~bash
docker compose up -d
docker ps
docker logs prometheus
~~~

Look for: `msg="Server is ready to receive web requests."`

---

## Task 3: Prometheus Concepts

### Metric Types

| Type | Behaviour | Real Example |
|---|---|---|
| **Counter** | Only goes UP, never down | `total_requests`, `total_errors`, `bytes_sent` |
| **Gauge** | Goes up AND down | `cpu_usage`, `memory_used`, `active_connections` |
| **Histogram** | Counts values in buckets | How many requests took <100ms, <500ms, <1s |
| **Summary** | Percentiles calculated on client side | p50, p95, p99 latency |

### Counter vs Gauge -- Key Difference

> A **Counter** is like your car's **odometer** -- it only goes up, and you calculate speed (rate) from it.
> A **Gauge** is like your car's **speedometer** -- the current value IS the answer.

**Examples:**
- Number of errors since app started → **Counter** (only goes up)
- Number of users logged in right now → **Gauge** (goes up and down)
- Memory used by Prometheus → **Gauge** (`process_resident_memory_bytes`)

---

### Labels and Time Series

Labels are key-value pairs attached to every metric:

~~~
prometheus_http_requests_total{code="200", handler="/api/v1/query", instance="localhost:9090", job="prometheus"}
~~~

A **time series** is a unique combination of metric name + labels. This is why `prometheus_http_requests_total` returned 60 result series -- one for each unique combination of `code`, `handler`, `instance`, and `job`.

---

### Scrape Targets

Go to **Status > Target health** in the Prometheus UI to see all configured targets.

**Screenshot: Targets page showing both targets**

| Target | State | Note |
|---|---|---|
| prometheus | UP ✅ | Scraping its own metrics successfully |
| notes-app | DOWN ❌ | App does not expose `/metrics` endpoint natively |

**Why notes-app is DOWN:** The notes-app is a Django application not instrumented with Prometheus. When Prometheus tried to scrape `http://notes-app:8080/metrics`, the app returned `404 Not Found` because that endpoint does not exist. This is expected -- not all apps expose Prometheus metrics natively. In later days, exporters like Node Exporter and cAdvisor will solve this for host and container metrics.

---

## Task 4: PromQL Queries

### Query 1 -- Health check
~~~promql
up
~~~
Returns `1` (UP) or `0` (DOWN) for each scrape target. Automatically created by Prometheus -- you never define it yourself.

**Result:** `up{instance="localhost:9090", job="prometheus"} = 1`

---

### Query 2 -- Memory usage in bytes
~~~promql
process_resident_memory_bytes
~~~
**Result:** `96718848` (bytes) -- this is a **Gauge** because memory goes up and down.

---

### Query 3 -- Memory usage in MB
~~~promql
process_resident_memory_bytes / 1024 / 1024
~~~
**Result:** `91.33 MB` -- PromQL supports arithmetic directly on metric values.

---

### Query 4 -- Raw HTTP request counter
~~~promql
prometheus_http_requests_total
~~~
**Result:** 60 time series, one per unique `code` + `handler` combination. Raw counter values -- useful as a starting point but not actionable alone.

---

### Query 5 -- Rate of requests per second
~~~promql
rate(prometheus_http_requests_total[5m])
~~~
**What `[5m]` means:** The lookback window -- Prometheus looks at the last 5 minutes of data points and calculates the average per-second rate of increase across that window.

**Why `rate()` is needed:** A raw counter only tells you the total since the app started. `rate()` converts it to a per-second speed -- like converting odometer readings into speedometer readings.

> ⚠️ `rate()` only works on Counters. Applying it to a Gauge gives meaningless results.

---

### Query 6 -- Total request rate (single number)
~~~promql
sum(rate(prometheus_http_requests_total[5m]))
~~~
**Result:** `0.0807` requests/sec -- all 60 time series collapsed into one number.

> ⚠️ Always write `sum(rate(...))` -- never `rate(sum(...))`. The `rate()` function needs raw counter values over time. Summing first destroys the individual time series and breaks the calculation.

---

### Query 7 -- Non-200 HTTP requests
~~~promql
rate(prometheus_http_requests_total{code!="200"}[5m])
~~~
**Result:** Only `code="302"` series returned -- a redirect on `/`. Value is `0` meaning no active error traffic.

---

### Label Filtering Operators

| Operator | Meaning | Example |
|---|---|---|
| `=` | Exact match | `{code="200"}` |
| `!=` | Not equal | `{code!="200"}` |
| `=~` | Regex match | `{handler=~"/api.*"}` |
| `!~` | Regex not match | `{handler!~"/api.*"}` |

---

## Task 5: Not All Apps Expose Metrics Natively

This is why **exporters** exist:

| What to monitor | Exporter needed |
|---|---|
| Linux host CPU/memory/disk | Node Exporter |
| Docker container stats | cAdvisor |
| Django/Flask app | prometheus-client library |
| MySQL/PostgreSQL | mysqld_exporter / postgres_exporter |

These will be covered in Days 74-77.

---

## Task 6: Storage and Retention

### Disk Usage Check

~~~bash
docker exec prometheus du -sh /prometheus
# Result: 1.1M  (after ~30 minutes of running)

docker exec prometheus ls /prometheus
# Result: data
~~~

### What Happens When Retention is Exceeded

Prometheus automatically **deletes the oldest data** to stay within the configured limit. Default retention is **15 days**. You can override this:

~~~yaml
command:
  - '--config.file=/etc/prometheus/prometheus.yml'
  - '--storage.tsdb.retention.time=30d'
  - '--storage.tsdb.retention.size=1GB'
~~~

### Why the Volume Mount is Critical

Without `prometheus_data:/prometheus`, all collected metrics live only inside the container's writable layer. Running `docker compose down` destroys the container and **all historical data is permanently lost**.

The named volume stores data on the EC2 host disk -- containers come and go, but the data survives restarts and redeployments.

---

## Key Takeaways from Day 73

1. **Observability = Metrics + Logs + Traces** -- you need all three to fully understand system behaviour
2. **Prometheus uses a pull model** -- it scrapes targets, targets do not push to it
3. **Counters only go up** -- use `rate()` to make them useful
4. **Gauges are current values** -- use them directly
5. **Labels create dimensions** -- filter and aggregate metrics by any label combination
6. **Always `sum(rate(...))` not `rate(sum(...))`**
7. **Not all apps expose `/metrics` natively** -- exporters bridge that gap
8. **Volume mounts are essential** -- without them, data is lost on container restart
9. **`localhost` inside Docker = the container itself** -- use container names for cross-container communication
10. **Always check `docker logs <container>`** when a container is in Restarting state

---

## Resources
- Prometheus Docs: https://prometheus.io/docs/
- Reference repo: https://github.com/LondheShubham153/observability-for-devops
- PromQL cheatsheet: https://promlabs.com/promql-cheat-sheet/

