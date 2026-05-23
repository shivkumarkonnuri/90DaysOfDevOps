# Day 83 — EKS Project: Production Deployment of AI-BankApp

## Overview

Day 83 is the capstone of the 3-day EKS block (Days 81–83). The AI-BankApp — a Spring Boot banking application with MySQL persistence and an Ollama/TinyLlama AI chatbot — was deployed as a production-grade application on Amazon EKS with full observability via Prometheus and Grafana.

**Cluster:** 3x t3.medium nodes, EKS v1.35.5, us-west-2
**App URL:** `https://44.239.129.144.nip.io` (Let's Encrypt TLS via cert-manager)
**Repo:** `TrainWithShubham/AI-BankApp-DevOps` (branch: `feat/gitops`)

---

## Full Architecture

```
[Internet]
    |
[Route53 / nip.io DNS] → 44.239.129.144
    |
[AWS NLB] — auto-provisioned by Envoy Gateway
    |
[Envoy Proxy] — Gateway API implementation
    |
[Gateway: bankapp-gateway]
  ├── HTTP  :80  → cert-manager ACME challenge + redirect
  └── HTTPS :443 → TLS terminated (Let's Encrypt cert)
    |
[HTTPRoute: bankapp-route] → PathPrefix /
[BackendTrafficPolicy] → Cookie: BANKAPP_AFFINITY (session pinning)
    |
[Service: bankapp-service:8080]
    |
    ├── [BankApp Pod 1] ──→ [MySQL Service:3306] ──→ [MySQL Pod] ──→ [EBS 5Gi]
    └── [BankApp Pod 2] ──→ [Ollama Service:11434] ──→ [Ollama Pod] ──→ [EBS 10Gi]

[HPA] → scales BankApp 2→4 pods at 70% CPU

[Prometheus] ←── ServiceMonitor ←── /actuator/prometheus (15s interval)
[Grafana] ←── Prometheus datasource → Kubernetes dashboards
```

---

## EKS Infrastructure (Terraform-provisioned)

| Component | Detail |
|---|---|
| EKS version | v1.35.5 |
| Node type | t3.medium (2 vCPU, 4Gi RAM) |
| Node count | 3 (one per AZ: us-west-2a/b/c) |
| VPC | 3-AZ with public/private subnets |
| EKS add-ons | CoreDNS, VPC CNI, kube-proxy, Pod Identity, EBS CSI, Metrics Server |
| StorageClass | gp3 (3000 IOPS, 125 MB/s, WaitForFirstConsumer) |

---

## Application Stack Deployed

```bash
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/pv.yml
kubectl apply -f k8s/pvc.yml
kubectl apply -f k8s/configmap.yml
kubectl apply -f k8s/secrets.yml
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/ollama-deployment.yml
kubectl apply -f k8s/bankapp-deployment.yml
kubectl apply -f k8s/hpa.yml
kubectl apply -f k8s/gateway.yml
```

### Complete stack status

```
NAME                           READY   STATUS    RESTARTS   AGE
pod/bankapp-6d69cdf947-88qm5   1/1     Running   0          63m
pod/bankapp-6d69cdf947-src94   1/1     Running   0          63m
pod/mysql-778d8d585d-9d8bm     1/1     Running   0          37m
pod/ollama-cbd479bf4-nqcrk     1/1     Running   0          63m

NAME                      TYPE        CLUSTER-IP       PORT(S)
service/bankapp-service   ClusterIP   172.20.137.248   8080/TCP
service/mysql-service     ClusterIP   172.20.117.142   3306/TCP
service/ollama-service    ClusterIP   172.20.110.38    11434/TCP

NAME                      READY   UP-TO-DATE   AVAILABLE
deployment.apps/bankapp   2/2     2            2
deployment.apps/mysql     1/1     1            1
deployment.apps/ollama    1/1     1            1

NAME                                              TARGETS       MINPODS   MAXPODS   REPLICAS
horizontalpodautoscaler.autoscaling/bankapp-hpa   cpu: 0%/70%   2         4         2
```

---

## Monitoring Stack

### Installation

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=3d \
  --set prometheus.prometheusSpec.resources.requests.memory=256Mi \
  --set prometheus.prometheusSpec.resources.requests.cpu=100m \
  --wait --timeout 600s
```

### Monitoring pods

```
NAME                                                     READY   STATUS
alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running
monitoring-grafana-79c78fb6ff-4hpps                      3/3     Running
monitoring-kube-prometheus-operator-6c855c6b6b-lwblv     1/1     Running
monitoring-kube-state-metrics-868694bc4b-76skb           1/1     Running
monitoring-prometheus-node-exporter-7r247                1/1     Running  ← node 1
monitoring-prometheus-node-exporter-k9m6b                1/1     Running  ← node 2
monitoring-prometheus-node-exporter-wxc6h                1/1     Running  ← node 3
prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running
```

### ServiceMonitor for BankApp

The BankApp service had no port name by default, causing the ServiceMonitor selector to not match. Fix was to name the port and reference it in the ServiceMonitor:

```bash
# Add port name to service
kubectl patch svc bankapp-service -n bankapp --type='json' \
  -p='[{"op":"replace","path":"/spec/ports/0/name","value":"http-metrics"}]'

# Add label for ServiceMonitor selector
kubectl label svc bankapp-service -n bankapp app=bankapp
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bankapp-monitor
  namespace: monitoring
  labels:
    release: monitoring
spec:
  namespaceSelector:
    matchNames:
      - bankapp
  selector:
    matchLabels:
      app: bankapp
  endpoints:
    - port: http-metrics
      path: /actuator/prometheus
      interval: 15s
```

### PromQL queries for BankApp metrics

```promql
# JVM heap memory used per pod
jvm_memory_used_bytes{namespace="bankapp", area="heap"}

# HTTP request rate (5m window)
rate(http_server_requests_seconds_count{namespace="bankapp"}[5m])

# HTTP 95th percentile latency
histogram_quantile(0.95,
  rate(http_server_requests_seconds_bucket{namespace="bankapp"}[5m])
)

# HikariCP connection pool size
hikaricp_connections{namespace="bankapp"}

# JVM memory used (confirmed working)
jvm_memory_used_bytes{namespace="bankapp"}
# Result: Eden Space ~7MB, Survivor Space ~148KB per pod
```

### Grafana dashboard observations (namespace: bankapp)

| Metric | Value |
|---|---|
| CPU utilisation | 1.34% (requests), 0.739% (limits) |
| Memory utilisation | 27.4% (requests), 19.8% (limits) |
| MySQL memory | 355 MiB (139% of 256Mi request) |
| BankApp memory | ~246 MiB per pod (96% of 256Mi request) |
| Ollama memory | 64.7 MiB (2.53% of 2.5Gi request) |
| BankApp transmit bandwidth | ~20 kb/s per pod |
| Packet drops | 0 — clean network |
| BankApp storage writes | ~0.2 io/s |

**Key observation:** MySQL is using 355MiB against a 256Mi request (139%) — in production this would warrant increasing the memory request to avoid OOMKill risk under load.

---

## End-to-End Validation Checklist

### Application layer ✅

| Check | Result |
|---|---|
| All pods Running 1/1 | ✅ 4/4 pods |
| Health endpoint `/actuator/health` | ✅ `{"status":"UP"}` |
| Prometheus metrics `/actuator/prometheus` | ✅ JVM, HikariCP, HTTP metrics |
| HPA active | ✅ `cpu: 1%/70%`, 2 replicas |
| HTTPS accessible | ✅ `https://44.239.129.144.nip.io` |

### Data layer ✅

| Check | Result |
|---|---|
| MySQL ping | ✅ `mysqld is alive` |
| mysql-pvc Bound | ✅ 5Gi gp3 EBS |
| ollama-pvc Bound | ✅ 10Gi gp3 EBS |
| Ollama model loaded | ✅ `tinyllama:latest` 637MB |
| EBS persistence test | ✅ Data survived pod deletion (22s recovery) |

### Infrastructure layer ✅

| Check | Result |
|---|---|
| Nodes healthy | ✅ 3/3 Ready, 2-4% CPU |
| Gateway Programmed | ✅ NLB active |
| Monitoring running | ✅ 8/8 pods Running |
| Prometheus scraping BankApp | ✅ 2 targets UP |

### Security layer ✅

| Check | Result |
|---|---|
| Non-root container | ✅ runs as `devsecops` user |
| Secrets not in env | ✅ `MYSQL_ROOT_PASSWORD` stored in K8s Secret |
| HTTPS TLS | ✅ Let's Encrypt cert, `READY: True` |

---

## Issues Encountered and Resolved

| Issue | Root Cause | Fix |
|---|---|---|
| gp3 StorageClass missing | EBS CSI installed but SC not created | Created manually |
| GatewayClass not auto-created | Envoy Gateway watches for it, doesn't create it | Applied manifest manually |
| HTTPRoute 404 | Old NLB IP hardcoded in gateway.yml | `sed` replaced IP, reapplied |
| cert-manager Gateway solver failing | Missing `--set config.enableGatewayAPI=true` | `helm upgrade` with flag |
| ServiceMonitor not scraping | Service port had no name + missing `app=bankapp` label | Patched port name, added label |
| Port 3000 already in use | Previous port-forward still running | `lsof -ti:3000 \| xargs kill -9` |

---

## 3-Day EKS Journey Summary

| Day | What Was Built | Key Learning |
|---|---|---|
| 81 | EKS cluster via Terraform, kubectl, raw manifest deploy | Terraform EKS module, node groups, IAM OIDC, EKS add-ons |
| 82 | Gateway API, Envoy, TLS, EBS storage, session persistence | GatewayClass→Gateway→HTTPRoute, BackendTrafficPolicy, cert-manager HTTP-01 |
| 83 | Full production stack, Prometheus+Grafana, validation, teardown | kube-prometheus-stack, ServiceMonitor, PromQL, end-to-end validation |

### What the AI-BankApp EKS setup includes

- Terraform-provisioned VPC with 3-AZ networking
- Managed node group with auto-scaling
- 6 EKS add-ons (CoreDNS, VPC CNI, kube-proxy, Pod Identity, EBS CSI, Metrics Server)
- Gateway API with Envoy for traffic management
- cert-manager for automated HTTPS with Let's Encrypt
- Cookie-based session persistence (`BANKAPP_AFFINITY`) for Spring Security
- EBS persistent storage for MySQL and Ollama (gp3, WaitForFirstConsumer)
- HPA scaling BankApp 2→4 pods at 70% CPU threshold
- Spring Boot Actuator + Micrometer + Prometheus metrics
- Init containers for MySQL/Ollama dependency ordering
- Non-root container security (`devsecops` user)

### What production would additionally need

- Route 53 + ExternalDNS for proper domain management
- Network Policies for pod-to-pod isolation
- Pod Disruption Budgets for safe node draining
- External Secrets Operator for AWS Secrets Manager
- Automated MySQL backups to S3
- Log aggregation with Loki
- Multi-environment clusters (dev + staging + prod)
- Increased MySQL memory request (currently OOM risk at 139% usage)

---

## Teardown Procedure

```bash
# 1. Delete monitoring
helm uninstall monitoring -n monitoring

# 2. Delete Gateway (releases NLB — do this before terraform destroy)
kubectl delete -f k8s/gateway.yml

# 3. Delete BankApp stack
kubectl delete -f k8s/hpa.yml
kubectl delete -f k8s/bankapp-deployment.yml
kubectl delete -f k8s/ollama-deployment.yml
kubectl delete -f k8s/mysql-deployment.yml
kubectl delete -f k8s/service.yml
kubectl delete -f k8s/secrets.yml
kubectl delete -f k8s/configmap.yml
kubectl delete -f k8s/pvc.yml   # releases EBS volumes
kubectl delete -f k8s/pv.yml
kubectl delete -f k8s/namespace.yml

# 4. Delete Helm releases
helm uninstall envoy-gateway -n envoy-gateway-system
helm uninstall cert-manager -n cert-manager

# 5. Delete namespaces
kubectl delete namespace monitoring envoy-gateway-system cert-manager

# 6. Verify no lingering LoadBalancers or PVCs
kubectl get svc -A | grep LoadBalancer
kubectl get pvc -A

# 7. Destroy infrastructure
cd terraform
terraform destroy
```

**Critical:** Delete Gateway resources before `terraform destroy`. If the NLB is not released first, Terraform cannot delete the VPC (NLB holds ENIs in the subnets) and the destroy will fail.

**Post-destroy verification in AWS Console:**
- EKS: no clusters
- EC2: no instances, no load balancers, no EBS volumes
- VPC: `bankapp-eks` VPC deleted
- CloudFormation: no lingering stacks

---

## Cost Report (3-day lab)

| Resource | Approximate cost |
|---|---|
| 3x t3.medium EC2 (nodes) | ~$0.125/hr = ~$9/day |
| EKS control plane | $0.10/hr = ~$2.40/day |
| NAT Gateway | ~$0.045/hr + data = ~$1.50/day |
| NLB (when active) | ~$0.008/hr = ~$0.20/day |
| EBS volumes (15Gi total) | ~$0.05/day |
| **Estimated 3-day total** | **~$15–25** |

Cost optimization: tear down cluster between sessions. EKS control plane charges even when idle.

---

## Key Takeaways

- **Gateway API is production-ready** — GA since Kubernetes 1.26, Envoy/Istio/Cilium all implement it. Ingress is legacy.
- **ServiceMonitor port names matter** — Prometheus operator matches on named ports. Unnamed ports cause silent scrape failures.
- **MySQL memory requests need headroom** — 355MiB actual vs 256Mi request = OOMKill risk under load. Always set requests based on observed usage + buffer.
- **WaitForFirstConsumer prevents cross-AZ issues** — EBS volumes are AZ-locked. This binding mode ensures the volume lands in the same AZ as the pod.
- **Delete Gateway before terraform destroy** — NLB holds VPC ENIs. Skipping this step causes terraform destroy to fail on VPC deletion.
- **kube-prometheus-stack is batteries-included** — Prometheus, Grafana, Alertmanager, Node Exporter, kube-state-metrics, all pre-wired with Kubernetes dashboards.
- **Spring Boot Actuator is production-grade** — `/actuator/health`, `/actuator/prometheus`, `/actuator/info` out of the box. Zero custom instrumentation needed for JVM, HTTP, DB pool metrics.

---

`#90DaysOfDevOps` `#DevOpsKaJosh` `#TrainWithShubham`
