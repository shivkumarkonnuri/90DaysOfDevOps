# Day 82 — EKS Networking with Gateway API and Persistent Storage

## Overview

Today's session covered production-grade Kubernetes networking using the **Gateway API with Envoy Gateway** on EKS, automated TLS with cert-manager, and EBS persistent storage verification. The AI-BankApp (Spring Boot + MySQL + Ollama/TinyLlama) served as the target workload.

**Cluster:** 3x t3.medium nodes, EKS v1.35.5, us-west-2
**App repo:** `TrainWithShubham/AI-BankApp-DevOps` (branch: `feat/gitops`)

---

## Gateway API Architecture

```
[Browser]
    |
[nip.io DNS] → resolves 44.239.129.144.nip.io → 44.239.129.144
    |
[AWS NLB] — auto-provisioned by Envoy Gateway
    |
[Envoy Proxy pods] — deployed by Envoy Gateway in bankapp namespace
    |
[Gateway: bankapp-gateway]
  ├── Listener: HTTP  (port 80)  — also used for cert-manager ACME challenge
  └── Listener: HTTPS (port 443) — TLS terminated, cert from bankapp-tls Secret
    |
[HTTPRoute: bankapp-route]
  └── match: PathPrefix /  → backendRef: bankapp-service:8080
    |
[BackendTrafficPolicy: bankapp-session]
  └── ConsistentHash (Cookie: BANKAPP_AFFINITY, TTL: 3600s)
    |
[Service: bankapp-service] → [BankApp Pods x2-4]
```

---

## Gateway API vs Ingress

| Feature | Ingress | Gateway API |
|---|---|---|
| API maturity | Stable but limited | GA since Kubernetes 1.26 |
| Traffic splitting | Not supported | Built-in (weighted backends) |
| Header matching | Annotation-dependent | Native HTTPRoute rules |
| Role separation | Single resource | GatewayClass → Gateway → HTTPRoute |
| TLS management | Annotation-based | Native TLS config in Gateway listeners |
| Session affinity | Not standardized | BackendTrafficPolicy (Envoy-specific) |
| Multi-namespace | Limited | First-class via ReferenceGrant |
| Controller binding | Implicit (annotation) | Explicit via `controllerName` |

**Key insight:** Gateway API separates concerns across three roles — infrastructure team owns GatewayClass, platform team owns Gateway, app team owns HTTPRoute. This role-based model maps cleanly to real org structures.

---

## Gateway API Resources Explained

### 1. GatewayClass
Defines which controller manages Gateways. Think of it as the "driver registration" — tells Kubernetes that `gateway.envoyproxy.io/gatewayclass-controller` handles anything using this class.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

**Note:** GatewayClass is cluster-scoped (no namespace). One GatewayClass can serve many Gateways.

### 2. Gateway
The actual load balancer configuration. When applied on EKS, Envoy Gateway watches for Gateway resources and automatically provisions an AWS NLB. Each listener defines a port/protocol/TLS combination.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bankapp-gateway
  namespace: bankapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All          # needed for cert-manager ACME challenge
    - name: https
      protocol: HTTPS
      port: 443
      hostname: 44.239.129.144.nip.io
      tls:
        mode: Terminate
        certificateRefs:
          - name: bankapp-tls
      allowedRoutes:
        namespaces:
          from: Same
```

**What happened when this was applied:** Envoy Gateway created an AWS NLB (`aa448edd76c87413489424dba07c4d64-72794197.us-west-2.elb.amazonaws.com`) within ~12 seconds and the Gateway moved to `Programmed: True`.

### 3. HTTPRoute
Routes traffic from the Gateway listeners to backend Services. Supports path matching, header matching, and weighted backends — all natively, without annotations.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bankapp-route
  namespace: bankapp
spec:
  hostnames:
    - 44.239.129.144.nip.io
  parentRefs:
    - name: bankapp-gateway
      sectionName: https
    - name: bankapp-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: bankapp-service
          port: 8080
          weight: 1
```

### 4. BackendTrafficPolicy (Envoy-specific)
Envoy Gateway extension for session persistence. Unlike Kubernetes `Service.spec.sessionAffinity`, this operates at the Envoy proxy layer and works even when Envoy bypasses kube-proxy.

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: bankapp-session
  namespace: bankapp
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: bankapp-route
  loadBalancer:
    type: ConsistentHash
    consistentHash:
      type: Cookie
      cookie:
        name: BANKAPP_AFFINITY
        ttl: 3600s
```

---

## Why Cookie-Based Session Affinity for AI-BankApp

The BankApp uses **Spring Security with form-based login**. Spring stores session state (CSRF tokens, authenticated principal, login status) in server-side `HttpSession` objects, which are stored in-memory per JVM instance.

**Without session affinity:**
1. User logs in → request hits Pod A → session created in Pod A's JVM
2. Next request hits Pod B (different pod) → no session found → user gets 401/redirect to login
3. User is effectively logged out on every other request

**With `BANKAPP_AFFINITY` cookie:**
1. User logs in → Envoy sets `BANKAPP_AFFINITY` cookie with a hash of Pod A's identity
2. All subsequent requests from this browser carry the cookie
3. Envoy uses consistent hashing to always route to Pod A
4. Session persists for the 3600s TTL

**Proof from curl output:**
```
< set-cookie: JSESSIONID=1BD84E5434438D6B22AD27F3B54A73C7; Path=/; HttpOnly
< set-cookie: BANKAPP_AFFINITY="1e0f53146e51a512"; Max-Age=3600; HttpOnly
```

Both cookies were set on the very first request — Spring Security created the session and Envoy immediately assigned pod affinity.

---

## cert-manager and Automated TLS

### How it works with Gateway API

```
cert-manager watches Gateway annotations
    |
Sees: cert-manager.io/cluster-issuer: letsencrypt-prod
    |
Creates Certificate resource → requests cert from Let's Encrypt ACME
    |
Let's Encrypt sends HTTP-01 challenge:
"Serve this token at http://<hostname>/.well-known/acme-challenge/<token>"
    |
cert-manager creates temporary HTTPRoute on HTTP listener (port 80)
    |
Let's Encrypt verifies → issues certificate
    |
cert-manager stores cert in Secret: bankapp-tls
    |
Gateway HTTPS listener picks up the Secret → TLS enabled
```

### ClusterIssuer configuration

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: shivkumarkonnuri@gmail.com
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - group: gateway.networking.k8s.io
                kind: Gateway
                name: bankapp-gateway
                namespace: bankapp
```

**Important:** cert-manager must be installed with `--set config.enableGatewayAPI=true` to use `gatewayHTTPRoute` solver. The standard install only supports Ingress-based HTTP-01.

**ClusterIssuer status:** `READY: True` — ACME account registered successfully with Let's Encrypt.

**Why nip.io for learning:** `nip.io` is a wildcard DNS service — `44.239.129.144.nip.io` resolves to `44.239.129.144`. No domain purchase needed, instant DNS for TLS testing.

---

## EBS Persistent Storage on EKS

### Storage flow

```
StorageClass (gp3)
    ↓
PersistentVolumeClaim (mysql-pvc: 5Gi, ollama-pvc: 10Gi)
    ↓ [WaitForFirstConsumer — volume created only when pod is scheduled]
PersistentVolume (dynamically provisioned by EBS CSI driver)
    ↓
EBS Volume (created in same AZ as the pod's node)
    ↓
Pod mounts volume at /var/lib/mysql or /root/.ollama
```

### StorageClass configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### PVC status

```
NAME         STATUS   VOLUME                                     CAPACITY   STORAGECLASS
mysql-pvc    Bound    pvc-13c12941-81c3-4936-9cda-0387aaa3f241   5Gi        gp3
ollama-pvc   Bound    pvc-c31bf588-38f1-4a89-a478-628f2ac80a50   10Gi       gp3
```

### EBS volumes in AWS (us-west-2)

```
+------------+-------------------------+-------+---------+--------------+
|     AZ     |           ID            | Size  |  State  |    PVC       |
+------------+-------------------------+-------+---------+--------------+
| us-west-2a | vol-00dc0b4075e1ef63b   |  10Gi | in-use  | ollama-pvc   |
| us-west-2c | vol-0f886c3d549fd0f6f   |   5Gi | in-use  | mysql-pvc    |
+------------+-------------------------+-------+---------+--------------+
```

**WaitForFirstConsumer in action:** MySQL pod was scheduled to a node in us-west-2c, so the EBS volume was created in us-west-2c. Ollama landed on us-west-2a. The volumes are AZ-locked — if a pod moves AZs, it cannot follow.

### Key EBS concepts

**ReadWriteOnce (RWO):** EBS volumes attach to exactly one node at a time. This is why MySQL and Ollama use `strategy: Recreate` — the old pod must fully terminate and detach the volume before the new pod can attach it. `RollingUpdate` would cause both pods to try attaching simultaneously, which EBS rejects.

**gp3 vs gp2:** gp3 provides 3000 IOPS and 125 MB/s throughput baseline at a lower cost than gp2, and IOPS/throughput can be tuned independently of volume size.

**allowVolumeExpansion: true:** You can grow a gp3 volume without deleting and recreating it — just edit the PVC's `spec.resources.requests.storage`.

### EBS persistence test — data survives pod restart

```bash
# Before: bankappdb exists
kubectl exec -n bankapp deploy/mysql -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"
# Database: bankappdb ✓

# Delete the pod
kubectl delete pod -n bankapp -l app=mysql
# pod "mysql-778d8d585d-4xtlz" deleted

# New pod came up in 22 seconds
# mysql-778d8d585d-9d8bm   1/1   Running   0   22s

# After: bankappdb still exists
kubectl exec -n bankapp deploy/mysql -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"
# Database: bankappdb ✓
```

The EBS volume persisted independently of the pod lifecycle. The Deployment controller created a new pod, it attached the same EBS volume, and MySQL found its data intact.

---

## HPA and Resource Budget

### HPA status

```
NAME          REFERENCE            TARGETS       MINPODS   MAXPODS   REPLICAS
bankapp-hpa   Deployment/bankapp   cpu: 1%/70%   2         4         2
```

CPU at 1% — app is idle. HPA will scale up when CPU crosses 70% threshold, adding pods up to max 4.

### Resource budget on 3x t3.medium (2 vCPU, 4Gi RAM each)

| Component | CPU Request | Memory Request | Pods | Notes |
|---|---|---|---|---|
| BankApp | 250m | 256Mi | 2 (min) | Scales to 4 under load |
| MySQL | 250m | 256Mi | 1 | Recreate strategy, RWO volume |
| Ollama | 900m | 2Gi | 1 | Heaviest consumer, TinyLlama model |
| System/kube | ~500m | ~500Mi | per node | DNS, CSI, metrics-server etc |
| **Total available** | **6000m** | **~12Gi** | 3 nodes | |

**Actual observed usage (idle):**

| Pod | CPU | Memory |
|---|---|---|
| bankapp (x2) | 3-4m | 232-235Mi |
| mysql | 9m | 355Mi |
| ollama | 4m | 64Mi |

Nodes at 2-3% CPU, 29-50% memory — healthy headroom for HPA scale-out and traffic spikes.

---

## Issues Encountered and Resolved

| Issue | Cause | Fix |
|---|---|---|
| `gp3` StorageClass not found | EBS CSI driver installed but StorageClass not created | Created manually with `WaitForFirstConsumer` and `allowVolumeExpansion` |
| GatewayClass not auto-created | Envoy Gateway watches for GatewayClass but doesn't create it | Applied GatewayClass manifest manually |
| HTTPRoute returning 404 | Old NLB IP hardcoded in gateway.yml | `sed` replaced old IP with current NLB IP, reapplied |
| ClusterIssuer missing `enableGatewayAPI` | cert-manager installed without Gateway API feature gate | `helm upgrade` with `--set config.enableGatewayAPI=true` |

---

## Key Takeaways

- **Gateway API is the future** — it is officially GA and will eventually replace the Ingress resource. Envoy Gateway, Istio, and Cilium all implement it.
- **Envoy Gateway provisions AWS NLB automatically** — applying a Gateway resource triggers cloud infrastructure creation. No manual load balancer setup needed.
- **BackendTrafficPolicy is Envoy-specific** — other Gateway implementations (Istio, Cilium) have their own session affinity mechanisms. The concept is standard, the CRD is not.
- **EBS is AZ-locked** — `WaitForFirstConsumer` prevents cross-AZ volume attachment issues by deferring volume creation until pod scheduling.
- **Recreate strategy is intentional for RWO volumes** — EBS cannot attach to two nodes simultaneously. RollingUpdate + RWO = attachment conflict.
- **cert-manager + Gateway API needs `enableGatewayAPI=true`** — this feature gate enables the `gatewayHTTPRoute` solver type in ClusterIssuer.
- **nip.io enables quick TLS testing** without buying a domain — `<IP>.nip.io` resolves to `<IP>` via wildcard DNS.

---

## Commands Reference

```bash
# Install Envoy Gateway
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version v1.4.0 -n envoy-gateway-system --create-namespace --wait

# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

# Get NLB address
kubectl get gateway bankapp-gateway -n bankapp -o jsonpath='{.status.addresses[0].value}'

# Install cert-manager with Gateway API support
helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set crds.enabled=true \
  --set config.enableGatewayAPI=true --wait

# Find EBS volumes for PVCs
aws ec2 describe-volumes \
  --filters "Name=status,Values=in-use" \
  --query "Volumes[*].{ID:VolumeId,Size:Size,AZ:AvailabilityZone,Tags:Tags[?Key=='kubernetes.io/created-for/pvc/name'].Value|[0]}" \
  --output table --region us-west-2

# Test session affinity
curl -v http://<NLB-IP>.nip.io  # look for BANKAPP_AFFINITY cookie in response
```

---

`#90DaysOfDevOps` `#DevOpsKaJosh` `#TrainWithShubham`
