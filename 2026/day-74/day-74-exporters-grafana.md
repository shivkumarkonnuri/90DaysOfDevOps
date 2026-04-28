# Day 74 — Node Exporter, cAdvisor, and Grafana Dashboards

## 📌 Objective
Extend the Prometheus setup to monitor:
- Host machine (CPU, memory, disk, network)
- Docker containers
- Visualize everything using Grafana dashboards
- Configure datasource provisioning using YAML

---

# 🧠 Architecture Overview

~~~text
Node Exporter  ---> 
                     \
                      --> Prometheus --> Grafana
                     /
cAdvisor      ----->
~~~

- Node Exporter → Host metrics
- cAdvisor → Container metrics
- Prometheus → Collects metrics
- Grafana → Visualizes metrics

---

# 🏗️ Task 1 — Node Exporter (Host Monitoring)

## 📌 Purpose
Expose Linux system metrics:
- CPU
- Memory
- Disk
- Network

---

## 🔧 docker-compose.yml

~~~yaml
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
~~~

---

## 📊 Prometheus Config

~~~yaml
- job_name: "node-exporter"
  static_configs:
    - targets: ["node-exporter:9100"]
~~~

---

## ✅ Verification

~~~bash
curl http://localhost:9100/metrics | head -20
~~~

---

## 📊 Sample Queries

~~~promql
node_cpu_seconds_total{mode="idle"}
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100
rate(node_network_receive_bytes_total[5m])
~~~

---

# 🐳 Task 2 — cAdvisor (Container Monitoring)

## 📌 Purpose
Monitor Docker containers:
- CPU usage
- Memory usage
- Network

---

## 🔧 docker-compose.yml

~~~yaml
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
~~~

---

## 📊 Prometheus Config

~~~yaml
- job_name: "cadvisor"
  static_configs:
    - targets: ["cadvisor:8080"]
~~~

---

## 📊 Sample Queries

~~~promql
rate(container_cpu_usage_seconds_total{name!=""}[5m])
container_memory_usage_bytes{name!=""}
rate(container_network_receive_bytes_total{name!=""}[5m])
topk(3, container_memory_usage_bytes{name!=""})
~~~

---

# 📊 Task 3 — Grafana Setup

## 📌 Purpose
Visualization layer for monitoring

---

## 🔧 docker-compose.yml

~~~yaml
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
~~~

---

## 🌐 Access Grafana

~~~text
http://<your-ec2-ip>:3000
~~~

Login:
~~~text
admin / admin123
~~~

---

## 🔗 Add Prometheus Datasource (Manual)

~~~text
URL: http://prometheus:9090
~~~

---

# 📈 Task 4 — Custom Dashboard

## 📌 Panels Created

---

### 🔹 CPU Usage

~~~promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
~~~

- Type: Gauge
- Thresholds:
  - <60 → Green
  - <80 → Yellow
  - ≥80 → Red

---

### 🔹 Memory Usage

~~~promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
~~~

- Type: Gauge

---

### 🔹 Container Memory

~~~promql
topk(5, container_memory_usage_bytes{id=~".*docker-[a-f0-9]+.*"}) / 1024 / 1024
~~~

- Type: Bar Gauge
- Used regex + transformations for clean labels

---

### 🔹 Disk Usage

~~~promql
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
~~~

- Type: Stat
- Threshold Mode: Absolute
- Thresholds:
  - 70 → Yellow
  - 85 → Red

---

# ⚙️ Task 5 — Auto-Provision Datasource (YAML)

## 📌 Purpose
Avoid manual UI setup → automate datasource creation

---

## 📁 Directory Structure

~~~bash
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards
~~~

---

## 📄 datasources.yml

~~~yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
~~~

---

## 🔧 docker-compose.yml Update

~~~yaml
- ./grafana/provisioning:/etc/grafana/provisioning
~~~

---

## 🔄 Restart

~~~bash
docker compose up -d grafana
~~~

---

## ✅ Verification

- Open Grafana
- Datasource already present
- No manual setup required

---

## 🧠 Why YAML Provisioning?

- Repeatable setup
- No manual work
- Version-controlled configuration

---

# 📊 Task 6 — Import Dashboard

## 📌 Import Community Dashboard

Go to:
~~~text
Dashboards → Import
~~~

Use:

~~~text
ID: 11074 (or 1860 if compatible)
~~~

---

## 🔧 Fix Variables

Set:
- JOB → node-exporter
- Instance → node-exporter:9100
- NIC → eth0 / ens5

---

## ✅ Result

- CPU per core
- Memory stats
- Disk metrics
- Network traffic

---

# 🧠 Key Learnings

- Difference between Node Exporter and cAdvisor
- Docker DNS usage (~~~prometheus:9090~~~ vs localhost)
- PromQL debugging and filtering
- Grafana transformations (regex, labels)
- Threshold configuration (Absolute vs Percentage)
- YAML provisioning for automation

---

# 📊 Final Stack

~~~text
Prometheus ✔
Node Exporter ✔
cAdvisor ✔
Grafana ✔
Custom Dashboard ✔
Provisioning ✔
Imported Dashboard ✔
~~~

---

# 🎯 Conclusion

Day 74 setup provides complete observability:

- Host monitoring via Node Exporter
- Container monitoring via cAdvisor
- Visualization via Grafana
- Automation via YAML provisioning

This forms a strong foundation for production monitoring systems.
