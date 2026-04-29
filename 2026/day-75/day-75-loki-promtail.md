# 🚀 Day 75 – Log Management with Loki and Promtail

## 📌 Task Summary
Added the second pillar of observability → **Logs**

Stack used:
- Loki (log storage)
- Promtail (log collector)
- Grafana (visualization)

---

## 🏗️ Architecture

~~~
Docker Containers
        ↓
Promtail
        ↓
Loki
        ↓
Grafana
        ↓
User
~~~

---

## 🔍 Loki Concept

Loki only indexes **labels**, not full log text.

### Why?
- Faster queries
- Lower storage
- Simple design

### Trade-off
- Limited full-text search

---

## 🧰 Loki Setup

### Create directory
~~~
mkdir -p loki
~~~

### loki-config.yml
~~~yaml
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
~~~

---

## 🐳 Promtail Setup

### Create directory
~~~
mkdir -p promtail
~~~

### promtail-config.yml
~~~yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*-json.log

    pipeline_stages:
      - docker: {}
~~~

---

## 🐳 Docker Compose

~~~yaml
loki:
  image: grafana/loki:latest
  container_name: loki
  ports:
    - "3100:3100"
  volumes:
    - ./loki/loki-config.yml:/etc/loki/loki-config.yml
    - loki_data:/loki
  command: -config.file=/etc/loki/loki-config.yml

promtail:
  image: grafana/promtail:latest
  container_name: promtail
  volumes:
    - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
    - /var/lib/docker/containers:/var/lib/docker/containers:ro
    - /var/run/docker.sock:/var/run/docker.sock
  command: -config.file=/etc/promtail/promtail-config.yml
~~~

---

## 🔎 Verification

~~~bash
curl http://localhost:3100/ready
~~~

Expected:
~~~
ready
~~~

---

## 📊 Grafana Setup

Add Loki datasource:
~~~
http://loki:3100
~~~

---

## 🔍 LogQL Queries

### All logs
~~~
{job="docker"}
~~~

### Container logs
~~~
{container_name="notes-app"}
~~~

### Error logs
~~~
{job="docker"} |= "error"
~~~

### Exclude logs
~~~
{job="docker"} != "health"
~~~

### Regex
~~~
{job="docker"} |~ "status=[45]\\d{2}"
~~~

---

## 📈 Log Metrics

### Count logs
~~~
count_over_time({job="docker"}[5m])
~~~

### Rate
~~~
rate({job="docker"}[5m])
~~~

### Top containers
~~~
topk(5, sum by (container_name) (rate({job="docker"}[5m])))
~~~

---

## 🔥 Custom Queries

### Notes-app errors
~~~
{container_name="notes-app"} |= "error"
~~~

### Error count per minute
~~~
count_over_time({container_name="notes-app"} |= "error"[1m])
~~~

---

## 📊 Grafana Dashboard

### Logs Panel
- Datasource → Loki
- Query:
~~~
{job="docker"}
~~~

---

## 🔗 Metrics + Logs Correlation

### Metrics
~~~
rate(container_cpu_usage_seconds_total{name="notes-app"}[5m])
~~~

### Logs
~~~
{container_name="notes-app"}
~~~

👉 Helps debug spikes instantly

---

## ⚠️ Troubleshooting

- No logs → check Promtail
- Wrong path → check:
~~~
/var/lib/docker/containers
~~~
- Check targets:
~~~
http://localhost:9080/targets
~~~

---

## 🎯 Outcome

- Loki setup complete
- Promtail shipping logs
- Logs visible in Grafana
- LogQL queries working
- Metrics + logs correlated

---

## 🚀 Key Learning

Observability =  
- Metrics → what happened  
- Logs → why it happened  

Both together = powerful debugging 🔥
