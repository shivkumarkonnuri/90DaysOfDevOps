# 🚀 Day 76 — OpenTelemetry & Alerting (Complete Notes)

---

## 📌 Objective
Today’s goal was to complete the **3rd pillar of observability (TRACES)** using OpenTelemetry and implement **end-to-end alerting**.

By the end:
- Metrics ✅ (Prometheus)
- Logs ✅ (Loki)
- Traces ✅ (OpenTelemetry)
- Alerting ✅ (Prometheus + Grafana)

---

## 🧠 Task 1: Understanding OpenTelemetry

### 🔹 What is OpenTelemetry (OTEL)?
- Open-source, vendor-neutral observability framework
- Collects:
  - Metrics
  - Logs
  - Traces
- NOT a storage system
- Sends data to backends like:
  - Prometheus
  - Loki
  - Jaeger
  - Tempo

---

### 🔹 What is OTEL Collector?
A service that:
- Receives telemetry
- Processes it
- Exports it

### 🔁 Pipeline:
```
Receivers → Processors → Exporters
```

- **Receivers** → Accept data (OTLP, Prometheus, etc.)
- **Processors** → Modify data (batch, filter)
- **Exporters** → Send data (Prometheus, debug, Jaeger)

---

### 🔹 What is OTLP?
- OpenTelemetry Protocol
- Standard format to send telemetry
- Supports:
  - gRPC → Port 4317
  - HTTP → Port 4318

---

### 🔹 What are Traces?
- Track request across services

Example:
```
User → API → Auth → DB
```

Each step = **Span**

Span contains:
- Trace ID
- Span ID
- Parent ID
- Duration
- Attributes

---

## ⚙️ Task 2: Setup OTEL Collector

### 📁 Create directory
```bash
mkdir -p otel-collector
```

---

### 📄 Config file
`otel-collector-config.yml`

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

---

### 🔍 What this does
- Receives OTLP data (4317, 4318)
- Batches data
- Sends:
  - Metrics → Prometheus (8889)
  - Traces → Debug logs
  - Logs → Debug logs

---

### 🐳 Docker Setup
```yaml
otel-collector:
  image: otel/opentelemetry-collector-contrib:latest
  container_name: otel-collector
  ports:
    - "4317:4317"
    - "4318:4318"
    - "8889:8889"
  volumes:
    - ./otel-collector/otel-collector-config.yml:/etc/otelcol-contrib/config.yaml
  restart: unless-stopped
```

---

### 🔗 Prometheus Integration
Add in `prometheus.yml`:

```yaml
- job_name: "otel-collector"
  static_configs:
    - targets: ["otel-collector:8889"]
```

---

### 🔁 Restart
```bash
docker compose up -d
```

---

### ✅ Verification
```bash
docker logs otel-collector | tail -5
```

👉 Prometheus Targets → otel-collector should be **UP**

---

## 📡 Task 3: Send Test Traces

### 🔹 Send Trace
```bash
curl -X POST http://localhost:4318/v1/traces ...
```

---

### 🔹 Verify Trace
```bash
docker logs otel-collector | grep test-span
```

👉 You should see span logs

---

### 🔹 Send Metrics
```bash
curl -X POST http://localhost:4318/v1/metrics ...
```

---

### 🔹 Verify in Prometheus
```promql
test_requests_total
```

---

### 🔄 Flow
```
curl → OTEL → Prometheus exporter → Prometheus
```

---

## 🚨 Task 4: Prometheus Alerting

### 📄 alert-rules.yml
- HighCPUUsage
- HighMemoryUsage
- ContainerDown
- TargetDown
- HighDiskUsage

---

### 🔑 Key Concepts
- `expr` → condition
- `for` → delay before firing
- `labels` → routing
- `annotations` → human message

---

### 🔗 Add in Prometheus config
```yaml
rule_files:
  - /etc/prometheus/alert-rules.yml
```

---

### 🐳 Mount in Docker
```yaml
- ./alert-rules.yml:/etc/prometheus/alert-rules.yml
```

---

### 🔁 Restart
```bash
docker compose up -d prometheus
```

---

### ✅ Verify
- Prometheus → Status → Rules
- Alerts page → shows:
  - inactive
  - pending
  - firing

---

### 🧪 Test
```bash
docker compose stop notes-app
```

👉 Trigger:
- ContainerDown
- TargetDown

---

## 🔔 Task 5: Grafana Alerting

### ✔️ Contact Point
- Name: DevOps Team
- Type: Email/Slack

---

### ✔️ Alert Rule
Example:
```promql
container_memory_usage_bytes{name="notes-app"} / 1024 / 1024
```

Condition:
- > 100 MB

---

### ✔️ Notification Policy
- Default → DevOps Team
- Nested:
  ```
  severity = critical
  ```

---

### ✔️ Alert State
- Normal
- Pending
- Firing

---

## ⚖️ Prometheus vs Grafana Alerts

| Feature | Prometheus | Grafana |
|--------|------------|---------|
| Evaluation | Prometheus server | Grafana |
| Storage | In Prometheus | In Grafana |
| Notifications | Needs Alertmanager | Built-in |
| Use case | Infra alerts | UI-based alerts |

---

## 🧩 Task 6: Full Architecture

```
METRICS:
Node Exporter → Prometheus → Grafana

LOGS:
Promtail → Loki → Grafana

TRACES:
App/curl → OTEL → Debug/Jaeger

ALERTING:
Prometheus → Rules
Grafana → Notifications
```

---

## 📊 Services

| Service | Port |
|--------|------|
| Prometheus | 9090 |
| Node Exporter | 9100 |
| cAdvisor | 8080 |
| Grafana | 3000 |
| Loki | 3100 |
| Promtail | 9080 |
| OTEL Collector | 4317/4318/8889 |
| Notes App | 8000 |

---

### 🔍 Verify All
```bash
docker compose ps
```

---

## 🎯 Final Outcome

✔ Metrics pipeline working  
✔ Logs pipeline working  
✔ Traces pipeline working  
✔ Prometheus alerting working  
✔ Grafana alerting working  

👉 **All 3 pillars achieved**

---

## 🧠 Key Learnings

- Observability = Metrics + Logs + Traces  
- OTEL bridges different telemetry systems  
- Alerting reduces manual monitoring  
- Labels help route alerts  
- `for` avoids alert flapping  

---

## 📌 Submission

- File: `day-76-otel-alerting.md`
- Location: `2026/day-76/`
- Push to GitHub

---

## 🚀 Conclusion

Today you built a **production-level observability system**:

👉 Not just monitoring  
👉 But **intelligent alerting + tracing**

---
